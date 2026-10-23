<div align="center">

# ATS Resume Scorer

**Score your resume for Applicant Tracking Systems, match it against a job description, and get an actionable checklist to improve it.**

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-backend-009688?logo=fastapi&logoColor=white)
![Streamlit](https://img.shields.io/badge/Streamlit-frontend-FF4B4B?logo=streamlit&logoColor=white)
![Supabase](https://img.shields.io/badge/Supabase-auth%20%2B%20db-3ECF8E?logo=supabase&logoColor=white)
![Groq](https://img.shields.io/badge/Groq-LLM-F55036)

</div>

---

## Overview

ATS Resume Scorer analyses a resume the way an Applicant Tracking System would. It returns an overall score out of 100, a breakdown by category, the keywords and skills a job description asks for that the resume is missing, and a prioritised list of fixes. Results are saved per user and can be exported as a PDF report.

## Features

- **Overall ATS score (0-100)** with a five-part breakdown
- **Job description comparison**: keyword match %, semantic similarity %, matched and missing keywords, skills gap
- **Skill validation**: checks that skills listed in the Skills section are backed by evidence in Projects or Experience
- **Feedback and action items**, sorted by priority
- **PDF report** (4 pages) and a plain-text summary download
- **User accounts** with saved **analysis history**
- Supports **PDF, DOC and DOCX** resumes up to 10 MB

### Score breakdown

| Component | Max points |
|---|---|
| Formatting | 20 |
| Keywords & Skills | 25 |
| Content Quality | 25 |
| Skill Validation | 15 |
| ATS Compatibility | 15 |
| **Total** | **100** |

## Architecture

```
 Streamlit frontend  ──HTTP──▶  FastAPI backend  ──▶  Groq LLM (skill / keyword extraction)
        │                            │           ──▶  spaCy + Sentence Transformers (NLP, matching)
        │                            │           ──▶  WeasyPrint (PDF report)
        └──── Supabase Auth          └──── Supabase Postgres (analysis history)
```

1. The user signs in through Supabase Auth in the Streamlit app.
2. The resume, and an optional job description, are sent to the FastAPI backend with the user's access token.
3. The backend verifies the token, extracts the text, parses it with the Groq API, scores it with rule-based checks plus spaCy and Sentence Transformers, and stores the result in Supabase.
4. The frontend renders the results and can request a PDF report.

> **Privacy:** resume text is sent to the Groq API for structured extraction. Do not upload resumes you are not comfortable sharing with that provider.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | Streamlit |
| Backend | FastAPI, Uvicorn |
| NLP | spaCy (`en_core_web_md`, `en_core_web_sm`), Sentence Transformers (`all-MiniLM-L6-v2`), RapidFuzz |
| LLM | Groq API (`openai/gpt-oss-120b`) |
| Auth and database | Supabase (Auth, Postgres via REST) |
| Document parsing | pdfplumber, PyPDF2, python-magic |
| PDF export | WeasyPrint |
| Token verification | PyJWT (ES256 / RS256 via JWKS, HS256 fallback) |

## Project structure

```
ai-resume-ats/
├── backend/
│   ├── api/              # routes.py, auth.py (JWT verification)
│   ├── core/             # config.py (environment variables, model names)
│   ├── database/         # supabase_db.py (history persistence)
│   ├── models/           # schemas.py (Pydantic models)
│   ├── services/         # scoring, JD matching, parsing, feedback, PDF export
│   ├── templates/        # HTML templates for the PDF report
│   ├── utils/            # file_utils.py, matching.py
│   └── main.py           # FastAPI app and start-up (model loading)
├── frontend/
│   ├── .streamlit/       # config.toml
│   ├── assets/           # styles.css
│   ├── components/       # UI blocks (score display, feedback, JD comparison, ...)
│   ├── services/         # api_client.py, supabase_client.py
│   ├── views/            # landing, scorer, history, resources pages
│   └── streamlit_app.py  # Streamlit entry point
├── jupyter notebooks/    # EDA and BERT experiments (not needed to run the app)
├── requirements.txt
└── README.md
```

## API reference

Base path: `/api/v1`. Interactive docs: `http://localhost:8000/docs`.

| Method | Endpoint | Description |
|---|---|---|
| POST | `/analyze-resume` | Analyse a resume, optionally against a job description |
| GET | `/health` | Health check |
| GET | `/history` | List the signed-in user's past analyses |
| DELETE | `/history/{analysis_id}` | Delete a history entry |
| POST | `/generate-pdf` | Generate a PDF report for a result |
| GET | `/history/{analysis_id}/pdf` | PDF report for a saved analysis |

All endpoints except `/health` require an `Authorization: Bearer <Supabase access token>` header.

---

## Getting started

### Prerequisites

- Python 3.11
- A free [Groq](https://console.groq.com) API key
- A free [Supabase](https://supabase.com) project
- About 2 GB of free disk space for models and libraries

### 1. Clone and install

```bash
git clone https://github.com/Himanshu1425/ai-resume-ats.git
cd ai-resume-ats
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

### 2. Download the spaCy models

```bash
python -m spacy download en_core_web_md
python -m spacy download en_core_web_sm
```

The Sentence Transformers model (~90 MB) downloads automatically on the first backend start.

### 3. Install system libraries

`python-magic` and WeasyPrint need native libraries.

**macOS (Homebrew)**

```bash
brew install libmagic pango
echo 'export DYLD_FALLBACK_LIBRARY_PATH="/opt/homebrew/lib:$DYLD_FALLBACK_LIBRARY_PATH"' >> ~/.zprofile
```

**Debian / Ubuntu**

```bash
sudo apt-get update
sudo apt-get install -y libmagic1 libpango-1.0-0 libpangoft2-1.0-0 libharfbuzz0b libffi-dev
```

### 4. Set up Supabase

Open **SQL Editor** in your project and run:

```sql
create table if not exists public.analyses (
  id uuid primary key default gen_random_uuid(),
  user_id text not null,
  filename text,
  ats_score numeric default 0,
  keyword_match numeric default 0,
  missing_keywords jsonb default '[]'::jsonb,
  created_at timestamptz default now(),
  analysis_result jsonb
);

create index if not exists analyses_user_id_idx
  on public.analyses (user_id, created_at desc);

alter table public.analyses enable row level security;
```

The backend uses the `service_role` key, which bypasses row level security, so no extra policy is needed.

Then configure authentication:

1. **Authentication → URL Configuration**: set **Site URL** and **Redirect URLs** to `http://localhost:8501`.
2. **Authentication → Sign In / Providers → Email**: for local testing you may turn **Confirm email** off. Keep it **on** for any public deployment.

### 5. Configure environment variables

Create a `.env` file in the project root. It is git-ignored; never commit it.

```env
GROQ_API_KEY=
SUPABASE_URL=https://<your-project-ref>.supabase.co
SUPABASE_ANON_KEY=
SUPABASE_KEY=            # service_role key
SUPABASE_JWT_SECRET=     # legacy JWT secret
AUTH_REDIRECT_URL=http://localhost:8501
```

| Variable | Where to find it |
|---|---|
| `GROQ_API_KEY` | Groq Console → API Keys |
| `SUPABASE_URL` | Supabase → Project home (Project URL) |
| `SUPABASE_ANON_KEY` | Project Settings → API Keys → Legacy → `anon` |
| `SUPABASE_KEY` | Project Settings → API Keys → Legacy → `service_role` |
| `SUPABASE_JWT_SECRET` | Project Settings → JWT Keys → Legacy JWT Secret |

Optional: `SENTENCE_TRANSFORMER_MODEL` (defaults to `all-MiniLM-L6-v2`).

### 6. Run

Backend, in one terminal:

```bash
source venv/bin/activate
uvicorn backend.main:app --reload --port 8000
```

Frontend, in another:

```bash
source venv/bin/activate
streamlit run frontend/streamlit_app.py
```

Open **http://localhost:8501**, sign up, upload a resume and run an analysis.

---

## Configuration

**LLM model.** The Groq model is set by `GROQ_MODEL` in `backend/services/groq_parser.py`. Groq retires models from time to time. To see which models your key can use:

```bash
python - <<'EOF'
import os, requests
from dotenv import load_dotenv
load_dotenv(".env")
r = requests.get("https://api.groq.com/openai/v1/models",
                 headers={"Authorization": f"Bearer {os.getenv('GROQ_API_KEY')}"})
print([m["id"] for m in r.json().get("data", [])])
EOF
```

**Backend URL for the frontend.** The frontend reads `st.secrets["backend"]["url"]` and falls back to `http://localhost:8000`. For a deployed frontend, add this to the platform's secrets:

```toml
[backend]
url = "https://your-backend-url"
```

## Deployment

The app runs as two services plus Supabase.

1. **Backend (FastAPI):** run in a container that includes the system libraries from step 3 and the environment variables above. Start command: `uvicorn backend.main:app --host 0.0.0.0 --port $PORT`. Choose a plan with enough memory for spaCy and Sentence Transformers.
2. **Frontend (Streamlit):** deploy `frontend/streamlit_app.py` and add the Supabase values and `[backend] url` to the platform secrets.
3. **Supabase:** add the deployed frontend URL to **Site URL** and **Redirect URLs**, and keep **Confirm email** on.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `failed to find libmagic` | Install the native library: `brew install libmagic` (macOS) or `apt-get install libmagic1` (Linux). |
| PDF export returns an error mentioning `gobject` or `pango` | Install Pango (`brew install pango`) and set `DYLD_FALLBACK_LIBRARY_PATH` as in step 3. |
| `CERTIFICATE_VERIFY_FAILED` on login or analysis (macOS) | Point Python at the certifi bundle: `export SSL_CERT_FILE="$(python -c 'import certifi; print(certifi.where())')"` and `export REQUESTS_CA_BUNDLE="$SSL_CERT_FILE"` before starting the backend. |
| `model_not_found` from Groq | The configured model is no longer available. List models with the snippet above and update `GROQ_MODEL`. |
| `Address already in use` | A previous server is still running: `kill -9 $(lsof -ti :8000)` (or `:8501` for Streamlit). |
| "Backend took too long" on the first PDF export | WeasyPrint builds a font cache on first use. Wait a minute and try again. |
| Login works but history is empty or errors | Check that the `analyses` table exists and that the Supabase project is not paused. |

## Known limitations

- Keyword matching is automated and can report a skill as missing even when it appears in the resume, or list job description phrases that are not skills. Treat results as guidance, not ground truth.
- Skill validation is strict: a skill counts as validated only when it appears by name in a project or experience bullet.
- Free-tier hosting (Groq, Supabase, the hosting platform) has rate limits and cold starts.

## My contributions

- Set up the Supabase project: the `analyses` table, email authentication and redirect URLs
- Configured environment variables, secrets handling and `.gitignore` rules
- Replaced the unavailable Groq model with `openai/gpt-oss-120b`
- Got the full stack running on macOS: spaCy models, `libmagic`, Pango for WeasyPrint and SSL certificates
- Tested the flow end to end: sign up, resume analysis, history and PDF export

## Notebooks

`jupyter notebooks/` contains experiments (exploratory data analysis, BERT embeddings and BERT fine-tuning). They are not required to run the app.