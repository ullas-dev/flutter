# How to Run — Streaming & Productivity Assistant

This guide covers two things: running the project in mock mode (no API key, no cost), and switching it to live mode with a real OpenAI key.

---

## 1. One-Time Setup

Unzip the project. Inside `streaming_assistant/`, copy the environment template:

```
cp backend/.env.sample backend/.env
```

Open `backend/.env`. By default it looks like this:

```
AGENT_USE_MOCK=true
OPENAI_API_KEY=sk-your-key-here
OPENAI_MODEL=gpt-4o-mini
```

This is the only file you ever need to edit to change how the agent runs.

---

## 2. Running in Mock Mode (default)

Mock mode needs no API key. The agent returns a deterministic canned response so the entire WebSocket → SQLite → React → trigger flow works for free.

**With Docker:**

```
docker compose up --build
```

**Without Docker:**

```
cd backend
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn main:app --reload
```

```
cd frontend
npm install
npm run dev
```

Backend: `http://localhost:8000`
Dashboard: `http://localhost:3000` (Docker) or `http://localhost:5173` (npm run dev)

Send a trigger to test:

```
python triggers/send_trigger.py --type gmail
```

---

## 3. Switching to Live Mode — Yes, This Works

**Short answer: yes, you can run this with `AGENT_USE_MOCK=false` and a real OpenAI key.** The agent code already supports it — `agent.py` has a complete `_openai_respond` function that sends the full conversation history to the OpenAI chat completions API.

### Step-by-step

**Step 1 — Get an API key**

From `https://platform.openai.com/api-keys`. It looks like `sk-proj-...` or `sk-...`.

**Step 2 — Edit `backend/.env`**

```
AGENT_USE_MOCK=false
OPENAI_API_KEY=sk-your-real-key-here
OPENAI_MODEL=gpt-4o-mini
```

`gpt-4o-mini` is the cheapest model that handles this task well. You can change it to `gpt-4o` for higher quality at higher cost.

**Step 3 — Restart the server**

This is the part that catches people out, so read this carefully.

`agent.py` reads `AGENT_USE_MOCK` into a constant called `USE_MOCK` **once, when the file is first imported** — not on every request. If the server is already running when you edit `.env`, it is still using the old value. You must restart it.

**With Docker:**
```
docker compose down
docker compose up --build
```

**Without Docker:**
Stop uvicorn with Ctrl+C, then run `uvicorn main:app --reload` again. (The `--reload` flag watches your `.py` files for changes — it does not watch `.env`.)

**Step 4 — Verify mock mode is off**

```
curl http://localhost:8000/health
```

Expected:
```json
{"status": "ok", "mock_mode": "false"}
```

If this still says `"true"`, the server did not pick up the new `.env` — go back to Step 3.

---

## 4. What Actually Changes in Live Mode

Everything about the architecture stays identical — WebSockets, SQLite, broadcasting, the React dashboard, the trigger endpoint. The only thing that changes is one function call inside `agent.py`.

In mock mode, `_mock_respond` runs: it slices the first 120 characters of the input and returns a fixed-format string. No network call.

In live mode, `_openai_respond` runs: it loads the full conversation history for that session from the `conversations` table in SQLite, prepends a system prompt describing the assistant's role, and sends all of it to OpenAI. The model's reply is what gets saved back to SQLite and broadcast to the dashboard.

This means in live mode, the agent's replies will genuinely reference earlier messages in the same session — try sending two related messages in a row on the same channel with the same username and watch the second reply refer back to the first.

---

## 5. A Fix You Need Before Live Mode Works

Two small things in the project needed correcting for `.env` to actually be read. Both are already fixed in the zip you have, but it's worth knowing what they were, in case you're comparing against an older copy:

**`docker-compose.yml`** was pointing at `backend/.env.sample` instead of `backend/.env`. If you edit `.env` but Docker is still reading `.env.sample`, your real key never reaches the container. Fixed — it now reads `backend/.env`.

**`main.py`** never called `load_dotenv()`. The `python-dotenv` package was installed but unused, so in non-Docker mode `.env` was never loaded into the environment at all — `os.getenv("OPENAI_API_KEY")` would return `None`. Fixed — `main.py` now calls `load_dotenv()` as the very first thing, before `agent.py` is imported (the ordering matters, for the same reason as Step 3 above).

---

## 6. Cost and Rate Limits — What to Expect

Each agent reply is one API call with `max_tokens=300`. With `gpt-4o-mini`, a full classroom demo — a handful of WebSocket messages plus two or three triggers — costs a fraction of a cent. This is safe to run live in front of a class.

If multiple students each run their own copy with their own key, costs stay per-key and tiny. If you're sharing one key across the room, be aware that everyone's agent calls draw from the same rate limit — unlikely to be an issue at classroom scale, but worth knowing.

---

## 7. Switching Back to Mock Mode

Set `AGENT_USE_MOCK=true` in `.env` and restart the server the same way as Step 3. No code changes needed either direction — it's a single environment variable.

---

## 8. Troubleshooting Live Mode

**`mock_mode` still shows `"true"` after editing `.env`**
The server wasn't restarted, or (Docker) `docker-compose.yml` is reading the wrong file. Confirm it says `./backend/.env`, not `./backend/.env.sample`.

**Agent reply is an error message instead of a summary**
Usually an invalid or missing API key. Check `backend/.env` has no extra spaces, no quotes around the key, and that the key hasn't been revoked on the OpenAI dashboard.

**`ModuleNotFoundError: No module named 'openai'`**
Only happens in live mode — the `openai` import in `agent.py` is lazy and only runs when `_openai_respond` is called. Run `pip install -r requirements.txt` again inside your active virtual environment, or rebuild the Docker image with `docker compose up --build`.

**Responses are slow**
Live mode makes a real network call to OpenAI, typically 1–3 seconds. This is normal and is exactly the kind of delay yesterday's background-task pattern was designed to absorb — the WebSocket broadcast still happens after the agent responds, so the dashboard waits a moment longer but nothing times out.
