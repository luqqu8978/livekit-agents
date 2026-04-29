# ELearn Recruitment System

ML-powered recruitment platform built with **FastAPI**, **React + TypeScript + Tailwind**, and a **hybrid feature-learning matching engine** (Sentence-BERT semantic similarity + TF-IDF keyword overlap + skill-coverage Jaccard + experience proximity).

> **Deep-dive explanation with flow diagrams:** [`docs/SYSTEM_EXPLAINED.md`](docs/SYSTEM_EXPLAINED.md) — architecture, data model, matching-engine internals, sequence diagrams, security model, and more.

---

## What's in the box

- **Admin dashboard** — overview charts, jobs CRUD, candidates browser, applications pipeline (Applied → Hired), match explorer, PDF reports.
- **Candidate portal** — profile editor, resume upload/download, job search, "My Matches" (top-N ranked open jobs), application tracking.
- **Resume parsing** — PDF, DOCX, TXT → extracts skills, experience, education automatically and updates the candidate profile.
- **Hybrid matching engine** — combines four signals into a single 0–100% score:
  | Signal              | Weight | What it measures                                                       |
  | ------------------- | ------ | ---------------------------------------------------------------------- |
  | Skill coverage      | 0.45   | Required-skill Jaccard + bonus for nice-to-have                        |
  | Experience fit      | 0.15   | Smooth proximity to the job's experience window                        |
  | Semantic similarity | 0.25   | Sentence-BERT (`all-MiniLM-L6-v2`) cosine of resume vs job description |
  | Keyword overlap     | 0.15   | TF-IDF cosine (1–2 grams)                                              |

  Falls back gracefully to TF-IDF if the transformer can't be loaded (e.g., offline).

- **PDF reports** — per-candidate (top matching jobs) and per-job (top matching candidates).

---

## Project layout

```text
ELearn/
├── backend/                  FastAPI + SQLAlchemy + ML
│   ├── app/
│   │   ├── core/             security (JWT, bcrypt), deps
│   │   ├── models/           SQLAlchemy ORM
│   │   ├── schemas/          Pydantic I/O
│   │   ├── routers/          auth, jobs, candidates, resumes,
│   │   │                      applications, matching, reports, admin
│   │   ├── services/         resume_parser, matching_service,
│   │   │                      report_service, skill_taxonomy
│   │   ├── main.py           FastAPI entrypoint
│   │   ├── seed.py           Sample data: 1 admin + 20 jobs +
│   │   │                                  50 candidates + ~125 apps
│   │   ├── config.py         env-driven settings
│   │   └── database.py       SQLAlchemy engine/session
│   ├── uploads/resumes/      Stored resume files (server-managed names)
│   ├── requirements.txt
│   └── .env.example
└── frontend/                 React 18 + Vite + TypeScript + Tailwind
    ├── src/
    │   ├── api/              axios client + endpoint wrappers
    │   ├── store/            Zustand auth store
    │   ├── components/       Layout, ProtectedRoute, UI primitives
    │   ├── pages/
    │   │   ├── candidate/    Dashboard, Profile, Resumes, Jobs,
    │   │   │                 JobDetail, Matches, Applications
    │   │   └── admin/        Overview, Jobs, Candidates,
    │   │                     Applications, Matching, Reports
    │   ├── App.tsx           Routing
    │   └── main.tsx
    ├── package.json
    └── tailwind.config.js
```

---

## Running on a fresh machine

### 0. Prerequisites

| Tool    | Min version | Notes                                                       |
| ------- | ----------- | ----------------------------------------------------------- |
| Python  | 3.11+       | https://www.python.org/downloads/ (tick "Add to PATH")      |
| Node.js | 18+         | https://nodejs.org/ — Node 22 LTS tested                    |
| Git     | any         | https://git-scm.com/downloads                               |

> First-time setup downloads ~500 MB of Python packages (mostly PyTorch) and ~80 MB Sentence-BERT model. **Internet required for the very first run.** Subsequent runs work offline.

### 1. Get the code

Either clone from a Git repo, or copy the project folder. **Before zipping/copying, delete these to keep things clean:**

- `backend/.venv/`
- `backend/elearn_recruitment.db`
- `backend/uploads/resumes/*`
- `frontend/node_modules/`
- `frontend/dist/`

The `.gitignore` files in each subproject already exclude these from Git.

### 2. Backend (Terminal 1)

```powershell
cd <project>\backend

# Create and activate venv
python -m venv .venv
.\.venv\Scripts\Activate.ps1   # PowerShell
# .\.venv\Scripts\activate.bat # cmd.exe
# source .venv/bin/activate    # macOS / Linux

# Install dependencies (~5 min the first time)
pip install --upgrade pip
pip install -r requirements.txt

# Configure environment
Copy-Item .env.example .env
# IMPORTANT: open .env and set a strong SECRET_KEY:
# python -c "import secrets; print(secrets.token_urlsafe(64))"

# Seed sample data (admin + 20 jobs + 50 candidates + ~125 applications)
python -m app.seed

# Run the API
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Backend is now live at:

- API: http://localhost:8000
- Swagger docs: http://localhost:8000/docs

### 3. Frontend (Terminal 2)

```powershell
cd <project>\frontend
npm install         # first time only (~1-2 min)
npm run dev
```

Open http://localhost:5173.

### 4. Sign in

| Role      | Email                  | Password      |
| --------- | ---------------------- | ------------- |
| Admin     | `admin@elearn.io`      | `Admin@12345` |
| Candidate | any seeded candidate * | `Password@123`|

\* Candidate emails follow the pattern `<first>.<last><idx>@example.com` — the admin "Candidates" page lists them all.

---

## Running on a LAN (so other devices can use it)

1. Backend already binds `0.0.0.0` above; it's reachable at `http://<your-ip>:8000`.
2. Update the frontend dev script in `frontend/package.json`:
   ```json
   "dev": "vite --host 0.0.0.0"
   ```
3. Add the LAN origin to `backend/.env`:
   ```
   CORS_ORIGINS=http://localhost:5173,http://192.168.1.42:5173
   ```
4. Open Windows Firewall (or your OS firewall) for ports `8000` and `5173`.

---

## Production-ish build

```powershell
# Backend without auto-reload, multi-worker
cd backend
.\.venv\Scripts\Activate.ps1
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 2

# Frontend static build
cd ..\frontend
npm run build       # outputs frontend/dist/
npm run preview     # serves dist/ on :5173
```

For a real production deployment:

- Put **Nginx** or **Caddy** in front of both, terminate TLS.
- Serve `frontend/dist/` as static files.
- Reverse-proxy `/api` → `http://localhost:8000`.
- Use **PostgreSQL** instead of SQLite (set `DATABASE_URL=postgresql+psycopg://user:pass@host/db`, install `psycopg[binary]`).
- Run uvicorn behind `gunicorn -k uvicorn.workers.UvicornWorker` or use `uvicorn --workers N`.

---

## API surface (a few highlights)

| Method | Path                                       | Auth      | Purpose                                      |
| ------ | ------------------------------------------ | --------- | -------------------------------------------- |
| POST   | `/api/auth/register`                       | public    | Create candidate account                     |
| POST   | `/api/auth/login`                          | public    | OAuth2 password grant → JWT                  |
| GET    | `/api/auth/me`                             | any       | Current user                                 |
| GET    | `/api/candidates/me` / `PUT`               | candidate | Read / update own profile                    |
| GET    | `/api/candidates`                          | admin     | Search / list candidates                     |
| GET    | `/api/jobs`, `POST`, `PUT/{id}`, `DELETE`  | mixed     | Job CRUD (write = admin)                     |
| POST   | `/api/resumes/upload`                      | candidate | Upload PDF/DOCX/TXT — auto-parses skills     |
| GET    | `/api/resumes/{id}/download`               | owner+adm | Stream original file                         |
| POST   | `/api/applications`                        | candidate | Apply (computes & caches match score)        |
| GET    | `/api/applications`                        | admin     | List with filters: status, min_score, etc.   |
| PATCH  | `/api/applications/{id}/status`            | admin     | Move through the pipeline                    |
| GET    | `/api/matches/jobs-for-me?top=20`          | candidate | Best-matching jobs                           |
| GET    | `/api/matches/candidates-for-job/{job_id}` | admin     | Best-matching candidates for a job           |
| GET    | `/api/admin/overview`                      | admin     | Dashboard counts + chart series              |
| GET    | `/api/reports/candidate/{id}.pdf`          | admin     | Per-candidate PDF report                     |
| GET    | `/api/reports/job/{id}.pdf`                | admin     | Per-job PDF report                           |

Full interactive list: http://localhost:8000/docs

---

## Security notes

- All secrets (JWT key, DB URL) come from environment variables — none are hardcoded.
- Passwords are bcrypt-hashed; verification uses bcrypt's constant-time `checkpw`.
- Resume uploads: extension + size whitelisted, stored under server-generated UUID names, paths resolved with directory-traversal protection.
- File reads of user-uploaded resumes only happen on server-controlled paths — client-supplied filenames are never used as paths.
- No use of `eval`, `exec`, `pickle`, or dynamic `__import__()` on untrusted data.
- CORS is restricted to the origins listed in `CORS_ORIGINS`.

---

## Troubleshooting

| Symptom                                                            | Fix                                                                                                              |
| ------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------- |
| `pip install` hangs on `torch`                                     | First-time download is large (~600 MB). Be patient or run on faster network.                                     |
| `huggingface_hub` symlink warning on Windows                       | Harmless — caching just uses more disk. Enable Developer Mode in Windows Settings to silence it.                 |
| `Could not validate credentials` on every API call after restart   | Your token expired (24 h default). Log in again.                                                                 |
| Login returns 500 with `value is not a valid email address`        | You changed the seed admin email to a reserved TLD (e.g. `.local`). Use a real TLD.                              |
| Frontend can't reach backend                                       | Confirm uvicorn is running on :8000 and `CORS_ORIGINS` in `.env` includes your frontend origin.                  |
| Match scores look low for everybody                                | Candidates need either skills on profile or an uploaded resume (resume text feeds the semantic similarity term). |

---

## License

Proprietary / TBD.
