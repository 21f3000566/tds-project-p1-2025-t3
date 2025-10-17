# tds-project-p1-2025-t3

## 🚀 Automated Task Receiver & Deployment Service

This repository contains a FastAPI application that accepts evaluation tasks, asks an LLM (Google Gemini) to generate or update a small web project, commits the results to GitHub, and (optionally) publishes via GitHub Pages. It's packaged to run inside a Docker container and is suitable for deployment on Hugging Face Spaces (or any container host that exposes a web port).

This README replaces and documents the repository for:
- Deployment on Hugging Face Spaces
- Example usage
- Workflow (how `main.py` and `Dockerfile` operate)
- Status & Logs
- Documentation for the POST /ready API

---

## Quick facts

- Framework: FastAPI (entrypoint: `main:app`)
- Default HTTP port: 7860 (matching the `Dockerfile` and HF Spaces convention)
- Main entrypoint file: `main.py`
- Dockerized with a simple multi-stage `Dockerfile` (see `Dockerfile`)
- Logs written to `logs/app.log` by default (configurable via `LOG_FILE_PATH`)

---

## 1) Deploying to Hugging Face Spaces

This project can run on Hugging Face Spaces as a "Docker" Space (recommended for FastAPI). Follow these steps:

1. Create a new Space on Hugging Face and choose "Docker" as the runtime.
2. Push this repository (or a trimmed copy) to the Space's Git repository.
3. Configure Secrets in the Space settings (Repository > Settings > Secrets):
	 - GEMINI_API_KEY
	 - GITHUB_TOKEN
	 - GITHUB_USERNAME
	 - STUDENT_SECRET
	 - Optionally: GITHUB_PAGES_BASE, LOG_FILE_PATH, MAX_CONCURRENT_TASKS
4. Ensure the Space exposes port 7860 (the `Dockerfile` sets ENV PORT and `CMD` runs uvicorn on port 7860).

Notes and tips:
- The included `Dockerfile` installs dependencies from `requirements.txt`, creates a non-root user, copies the source, and starts the app with `uvicorn main:app --host 0.0.0.0 --port 7860`.
- If you need GPU access for model loading, replace the base image and add the required drivers (Spaces currently has limited GPU support — check HF docs).

---

## 2) Example usage

Run locally (recommended during development):

```bash
# create and populate a .env file (example below)
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 7860
```

Example `.env` (copy to repository root):

```env
GEMINI_API_KEY=your_gemini_api_key
GITHUB_TOKEN=your_github_personal_access_token
GITHUB_USERNAME=your_github_username
STUDENT_SECRET=super_secret_value
LOG_FILE_PATH=logs/app.log
MAX_CONCURRENT_TASKS=2
KEEP_ALIVE_INTERVAL_SECONDS=30
GITHUB_PAGES_BASE=https://<your-username>.github.io
```

HTTP example (call the POST /ready endpoint):

```bash
curl -X POST "http://localhost:7860/ready" \
	-H "Content-Type: application/json" \
	-d '{
		"task": "build a small landing page",
		"email": "student@example.com",
		"round": 1,
		"brief": "Create a single-page HTML landing page for a fictional product",
		"evaluation_url": "https://example.com/eval",
		"nonce": "random-nonce",
		"secret": "super_secret_value",
		"attachments": []
	}'
```

Response: the app returns JSON detailing queued status or synchronous failure. The service will then run generation, commit to GitHub, and notify the `evaluation_url` when done.

---

## 3) Workflow (what `main.py` and `Dockerfile` do)

High-level flow (in `main.py`):

- The app defines Pydantic models `TaskRequest` and `Attachment` for input validation.
- Entry endpoint (documented below) verifies a `secret` against `STUDENT_SECRET`.
- On receiving a task, the app will:
	1. Validate the request and attachments.
	2. Convert attachments into inlined image parts (data URIs -> base64) suitable for LLM prompts or save them locally.
	3. Call the Gemini LLM endpoint using `call_gemini_api()` to generate or update project files (HTML, README, LICENSE, etc.).
	4. Save generated files under `generated_tasks/<task_id>/`.
	5. Initialize or clone a GitHub repo (using `GitPython`) and push a commit.
	6. Attempt to configure GitHub Pages and assemble a `pages_url`.
	7. Notify the provided `evaluation_url` with the `repo_url`, `commit_sha`, and `pages_url`.

Concurrency and safety:

- A semaphore (`MAX_CONCURRENT_TASKS`) limits concurrent LLM/deploy tasks to avoid overloading resources.
- The app contains retry/backoff logic for Gemini and GitHub Pages configuration.
- A "round 2" surgical update routine exists to perform minimal edits on an existing `index.html` when `round` > 1.

Dockerfile summary:

- Based on `python:3.10-slim`.
- Installs system build tools (gcc, g++, cmake, git) needed by some Python packages.
- Creates a non-root `user` and copies files into `/app`.
- Installs Python dependencies from `requirements.txt`.
- Exposes port 7860 and runs `uvicorn main:app --host 0.0.0.0 --port 7860`.

---

## 4) Status & Logs

- Default log file: `logs/app.log` (configurable via `LOG_FILE_PATH` env variable).
- The app logs to both stdout and the log file. On Hugging Face Spaces you can view stdout via the Space logs; also configure `LOG_FILE_PATH` to a writable location.
- The process flushes logs after key operations. Use these logs to debug LLM errors, Git failures, and network timeouts.

Health endpoints (examples in `main.py`):
- GET / (FastAPI root) — default FastAPI metadata
- The codebase includes status variables (such as `background_tasks_list`, `last_received_task`) that can be used to implement a `/status` or `/healthz` endpoint quickly. You can add one like:

```python
@app.get('/status')
async def status():
		return {
				'queued_background_tasks': len(background_tasks_list),
				'last_received_task': last_received_task or {}
		}
```

Log extraction:

```bash
tail -n 200 logs/app.log
```

If running in Docker locally, view stdout/stderr with:

```bash
docker logs <container-id>
```

---

## 5) POST /ready API (contract & examples)

This is the primary webhook used by the evaluation server to submit tasks.

Endpoint: POST /ready

Content-Type: application/json

Request body JSON schema (Pydantic model `TaskRequest` in `main.py`):

- task: string — an identifier for the task
- email: string — contact or evaluator email
- round: integer — 1 for initial generation, >1 for surgical updates
- brief: string — the natural language brief given to the LLM
- evaluation_url: string — where to send the deployment result notification
- nonce: string — an opaque random value to pair request/response
- secret: string — must match `STUDENT_SECRET` for authorization
- attachments: array of Attachment objects — each has:
	- name: filename (string)
	- url: data URI (data:image/...) or an http(s) URL to fetch

Example request (full JSON):

```json
{
	"task": "task123",
	"email": "student@example.com",
	"round": 1,
	"brief": "Create a responsive single-page site with header, hero, and footer",
	"evaluation_url": "https://evaluation.example.com/notify",
	"nonce": "abc123",
	"secret": "super_secret_value",
	"attachments": [
		{"name": "logo.png", "url": "data:image/png;base64,iVBORw0KGgoAAAANS..."}
	]
}
```

Authorization:

- The endpoint validates the `secret` field against the `STUDENT_SECRET` environment variable. If it doesn't match, the request is rejected.

Behavior and response:

- On success, the server enqueues the generation + deploy job and returns a JSON acknowledgement. The service will run the task asynchronously (concurrent limit applied by `MAX_CONCURRENT_TASKS`).
- When the background job completes (or fails), the service POSTs to the provided `evaluation_url` with a payload containing `repo_url`, `commit_sha`, `pages_url`, `email`, `task`, `round`, and `nonce`.

Error handling:

- If LLM generation fails, the server returns an error and logs details. The background job contains retry/backoff logic for LLM and GitHub Pages operations.

---

## Other notes & troubleshooting

- If you hit GitHub API rate limits, ensure `GITHUB_TOKEN` has the right scopes (repo). For Pages configuration, the token must allow repo and pages management.
- The Gemini API calls require `GEMINI_API_KEY`; if missing the code will error early. For local testing, you can mock `call_gemini_api` or inject a stub.
- The `Dockerfile` sets a non-root `user`. Ensure mounted volumes and log paths are writable.

---

## Quick completion summary

- Replaced README with deployment instructions for Hugging Face Spaces, example usage, workflow summary (derived from `main.py` & `Dockerfile`), status/logs guidance, and a full contract for POST /ready API.

If you'd like, I can:
- Add a `/status` endpoint to `main.py` and commit it.
- Add a small `make run-local` helper and a Docker Compose file for easier local dev.
- Generate a minimal sample `.env.example` and a small test script that exercises `/ready` with a fake LLM response.
