# SpendSense: AI-Powered Personal Finance & Expense Tracker

Log expenses by typing a sentence or uploading a receipt photo. An LLM turns it into structured JSON
(amount, category, merchant, date), you set monthly budgets, and you can ask questions about your
spending in plain English.

**Stack:** FastAPI, SQLite, JWT login, LLM APIs (Anthropic or OpenAI-compatible), plain HTML/CSS/JS.

## Project structure

```
spendsense/
├── app/
│   ├── main.py          # creates the FastAPI app, connects routes, serves the web page
│   ├── config.py        # reads environment variables (.env)
│   ├── database.py      # SQLite connection + table creation
│   ├── auth.py          # password hashing + JWT tokens
│   ├── schemas.py       # input validation (Pydantic)
│   ├── llm.py           # all LLM calls + JSON cleaning + offline fallback
│   ├── utils.py         # month helpers
│   └── routes/
│       ├── auth_routes.py   # register, login, public config
│       ├── expenses.py      # CRUD + text and receipt input
│       ├── budgets.py       # set budgets, spent vs limit
│       └── insights.py      # monthly summary + natural-language answers
├── static/              # index.html, style.css, app.js (the web page)
├── tests/test_api.py    # automated tests
├── requirements.txt  .env.example  Procfile  render.yaml  .gitignore
```

## Run it on your computer

You need Python 3.10 or newer.

```bash
# 1. open a terminal inside the spendsense folder
cd spendsense

# 2. create and activate a virtual environment
python -m venv venv
venv\Scripts\activate          # Windows
source venv/bin/activate       # Mac / Linux

# 3. install packages
pip install -r requirements.txt

# 4. create your settings file
copy .env.example .env         # Windows
cp .env.example .env           # Mac / Linux
```

Open `.env` and set at least `SECRET_KEY` (any long random text). Generate one with:
`python -c "import secrets; print(secrets.token_hex(32))"`

```bash
# 5. start the server
uvicorn app.main:app --reload
```

Open **http://127.0.0.1:8000** in your browser. Register an account and start adding expenses.
Interactive API docs (Swagger) are at **http://127.0.0.1:8000/docs**.

## Turning on the AI

The app works without any key, using simple built-in rules for text and a plain-text summary for insights.
Receipt photos need an AI key. In `.env` set ONE of:

- **Anthropic:** `ANTHROPIC_API_KEY=your-key` (model is `ANTHROPIC_MODEL`, default `claude-haiku-5-5`)
- **OpenAI:** `OPENAI_API_KEY=your-key` (model `OPENAI_MODEL`, default `gpt-4o-mini`)
- **Any OpenAI-compatible service** (for example Groq or Gemini): set `OPENAI_API_KEY`, `OPENAI_BASE_URL`
  and `OPENAI_MODEL` for that service. The model must support images if you want receipt scanning.

Restart the server after changing `.env`. The model name is a setting, so if a provider retires a model,
change the name in `.env`; no code change is needed.

## Run the tests

```bash
pytest -q
```

## API summary

| Method | URL | What it does |
|---|---|---|
| POST | /api/auth/register, /api/auth/login | create account / log in, returns a token |
| POST | /api/expenses | add an expense manually |
| POST | /api/expenses/from-text | LLM turns a sentence into a saved expense |
| POST | /api/expenses/from-receipt | LLM reads a receipt image (multipart, field `file`) |
| GET | /api/expenses?month=YYYY-MM | list expenses |
| PUT / DELETE | /api/expenses/{id} | update / delete |
| PUT | /api/budgets | set a budget (`category` = `Overall` or a category) |
| GET | /api/budgets/status?month=YYYY-MM | spent vs limit |
| GET | /api/insights/summary?month=YYYY-MM | totals by category |
| POST | /api/insights/ask | ask a question about your spending |

Protected routes need the header `Authorization: Bearer <token>`.

## How the main ideas are used

- **SQLite & CRUD:** `database.py` creates three tables; `routes/expenses.py` has Create, Read, Update, Delete.
- **Structured outputs (JSON):** `llm.py` asks the model for a JSON object, extracts it, then `clean_expense()`
  validates the amount, category and date before anything reaches the database.
- **User authentication:** PBKDF2-hashed passwords, JWT tokens, and every query filters by `user_id`
  so users only ever see their own data.
- **Environment variables:** keys and secrets live in `.env`, which is ignored by Git.

## Deploy on Render

1. Push the project to GitHub (the `.gitignore` already keeps `.env` and `*.db` out).
2. On render.com choose **New → Web Service**, connect the repo (it can use `render.yaml`).
3. Build command: `pip install -r requirements.txt`
   Start command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`
4. Under **Environment**, add `SECRET_KEY`, your LLM key, and (optional) `CURRENCY_SYMBOL`.
5. **Important:** Render's free disk is wiped on every redeploy, so SQLite data would be lost.
   For real use, add a persistent disk (paid plan), mount it at `/data`, and set `DATABASE_PATH=/data/finance.db`.

## Deploy on Railway

1. Push to GitHub, then **New Project → Deploy from GitHub repo**. Railway reads the `Procfile`.
2. In **Variables**, add `SECRET_KEY` and your LLM key.
3. To keep your data, add a **Volume** mounted at `/data` and set `DATABASE_PATH=/data/finance.db`.
4. Under **Settings → Networking**, click **Generate Domain**.

## Common problems

| Problem | Fix |
|---|---|
| `ModuleNotFoundError` | activate the venv, then run `pip install -r requirements.txt` |
| `uvicorn` not found | run `python -m uvicorn app.main:app --reload` |
| Port 8000 busy | add `--port 8001` and open that port instead |
| "Receipt scanning needs an AI key" | add a key to `.env` and restart the server |
| Logged out after restart | normal if you changed `SECRET_KEY`; just log in again |
