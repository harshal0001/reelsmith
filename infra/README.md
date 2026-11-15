# Deployment

Reelsmith runs five services. All of them bind to loopback, or stay internal to Compose, except the web ingress.

| Service | Port | Runtime |
| --- | --- | --- |
| API (FastAPI) | 8000 | Python 3.12 venv at the repo root |
| Remotion render worker | 8001 | Node 22 |
| Chatterbox Turbo (speech) | 8002 | Python 3.11 venv, CUDA |
| WhisperX (alignment) | 8003 | Python 3.11 venv, CUDA |
| Editor (Vite preview) | 5173 | Node 22 |

The speech services need an NVIDIA GPU. They were developed on an RTX PRO 6000 Blackwell with Torch 2.8.0+cu128. The API and renderer use only the CPU.

## Option A: run the processes directly on a GPU host

Put the private `.env`, the reference WAVs (`assets/voices/{sundar,jensen}/reference.wav`), and the character PNGs (`assets/characters/{sundar,jensen}.png`) on the host. Keep `.env` at mode 600.

### Speech services

Create separate Python 3.11 environments, `.venv-chatterbox` and `.venv-whisperx`, from each service's `requirements.txt` under `backend/modules/voice/gpu_services/`. The Chatterbox lock pins `setuptools==80.9.0`, which its Perth watermarker needs for `pkg_resources`.

```bash
ROOT=$PWD
nohup bash -lc "set -a; source $ROOT/.env; set +a; cd $ROOT/backend/modules/voice/gpu_services/chatterbox; exec $ROOT/.venv-chatterbox/bin/python -m uvicorn app:app --host 127.0.0.1 --port 8002 --workers 1" > logs/chatterbox-service.log 2>&1 &
nohup bash -lc "set -a; source $ROOT/.env; set +a; cd $ROOT/backend/modules/voice/gpu_services/whisperx; exec $ROOT/.venv-whisperx/bin/python -m uvicorn app:app --host 127.0.0.1 --port 8003 --workers 1" > logs/whisperx-service.log 2>&1 &
curl --fail http://127.0.0.1:8002/health
curl --fail http://127.0.0.1:8003/health
```

A service is ready only when its health response shows `status: ok` and `model_loaded: true`. The synthesis and alignment routes require their own bearer keys, `VOICE_API_KEY` and `ALIGNMENT_API_KEY`.

### API, worker and editor

```bash
nohup .venv/bin/python -m uvicorn backend.app:app --host 127.0.0.1 --port 8000 --workers 1 > logs/api-service.log 2>&1 &

nohup bash -lc 'set -a; source .env; set +a; exec env -i HOME=$HOME PATH="$PATH" LANG=C.UTF-8 ARTIFACT_ROOT="$ARTIFACT_ROOT" RENDER_WORKER_TOKEN="$RENDER_WORKER_TOKEN" RENDER_CONCURRENCY=1 RENDER_WORKER_HOST=127.0.0.1 RENDER_WORKER_PORT=8001 npm run worker --prefix renderer' > logs/renderer-service.log 2>&1 &

nohup npm run preview --prefix frontend -- --host 127.0.0.1 --port 5173 > logs/frontend-service.log 2>&1 &
```

The worker receives only its render settings, never the provider keys. If the host lacks Chromium's NSS libraries, install them in user space (for example with mamba). Then export `LD_LIBRARY_PATH` and `REMOTION_BROWSER_EXECUTABLE` before you start the worker.

To serve the editor under a path prefix, such as an authenticated reverse proxy at `/proxy/5173/`, build the frontend with `VITE_BASE_PATH=./`. Set `VITE_API_BASE_URL` to the API's same-origin path and `VITE_PREVIEW_HOST` to the public hostname. Set `FRONTEND_ORIGIN` for API CORS, and rebuild the frontend after changing any of these. Keep one API worker, because project state and operation locks are file-backed and in-process.

## Option B: Docker Compose

`infra/compose.yml` and `infra/compose.gpu.yml` define the five-service deployment for a Docker host with the NVIDIA Container Toolkit. Prepare the shared artifact directory for API UID/GID 10001, then run these from the repository root:

```bash
docker compose --env-file .env -f infra/compose.yml -f infra/compose.gpu.yml config --quiet
docker compose --env-file .env -f infra/compose.yml -f infra/compose.gpu.yml up -d --build
docker compose --env-file .env -f infra/compose.yml -f infra/compose.gpu.yml ps
```

`config --quiet` avoids printing interpolated secrets. The web service proxies same-origin `/api`, and only its ingress should be reachable. The API, renderer, and speech ports stay internal to Compose.
