# url-summarizer-orchestrator

An event-driven job orchestration system (MVP) for URL summarization with Discord notifications.

## Run API

From repo root:

```bash
cd services/api
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

then open 
http://localhost:8000/health
http://localhost:8000/docs（Swagger UI）
```