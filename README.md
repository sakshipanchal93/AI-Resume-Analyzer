# AlignAI — AI Resume Analyzer

Analyze your resume. Match it with the right job. Discover exactly what you're missing.

A privacy-first, full-stack resume analysis platform: upload a **PDF/DOCX/TXT** resume and (optionally) a job description, and get back structured, explainable NLP insights — skill gaps, keyword coverage, TF-IDF similarity, an 8-section completeness audit and prioritized recommendations — rendered on an interactive Recharts dashboard.

> **Honest by design:** scores estimate *textual alignment and document completeness* using transparent, deterministic heuristics. They do **not** guarantee ATS success, interviews, or hiring outcomes.

---

## Architecture

```
┌──────────────────────────┐         multipart/form-data          ┌────────────────────────────┐
│  React 19 + Vite + TS    │  ── POST /api/analyze ─────────────▶ │  FastAPI (Python) backend  │
│  Tailwind v4, Recharts,  │      resume file + job_description   │  ├─ parsers  PyMuPDF,      │
│  Framer Motion, Lucide   │ ◀──────── structured JSON ────────── │  │           python-docx   │
│                          │                                      │  ├─ nlp      TF normalization│
│  services/api.ts reads   │                                      │  │           (regex/NLTK-   │
│  VITE_API_URL, falls     │                                      │  │            style)        │
│  back to same-origin     │                                      │  ├─ ML       scikit-learn   │
│  /api serverless route   │                                      │  │           TfidfVectorizer│
│  running the identical   │                                      │  │           + cosine       │
│  pipeline 1:1 in Node    │                                      │  └─ services weighted score,│
└──────────────────────────┘                                      │              recs, summary │
                                                                  └────────────────────────────┘
```

Two interchangeable implementations of the **same** NLP pipeline:

- **`backend/` — Python FastAPI service** (`app/main.py`, `routers/`, `services/`, `nlp/`, `parsers/`, `models/`) using **scikit-learn** `TfidfVectorizer` + `cosine_similarity`, **PyMuPDF** for PDFs and **python-docx** for DOCX, with **Pydantic** response models. This is the canonical reference implementation.
- **`api/` — Node serverless mirror** (used by the hosted demo because it deploys as a Vercel function): the exact same tokenization, canonicalization, dictionary, TF-IDF (`sublinear_tf`, `smooth_idf`, L2 norm) and scoring math, with `pdf-parse`, `mammoth` and `busboy`.

Both process uploads **in memory only — resumes are never stored**.

## Tech Stack

| Layer    | Technology |
|----------|------------|
| Frontend | React 19, Vite, TypeScript, Tailwind CSS v4, Recharts, Framer Motion, Lucide |
| Backend  | Python, FastAPI, Pydantic, Uvicorn (Node serverless mirror for hosting) |
| NLP / ML | scikit-learn (TF-IDF, cosine similarity), regex normalization, curated 60+ skill dictionary |
| Parsing  | PyMuPDF (PDF), python-docx (DOCX), TXT; JS: pdf-parse, mammoth |

## NLP Methodology

1. **Extraction** — bytes → raw text (PDF/DOCX/TXT). Scanned/empty/corrupt files are rejected with actionable errors.
2. **Normalization** — unicode quotes/dashes, whitespace, then *canonicalization*: `C++→cplusplus`, `C#→csharp`, `Node.js→nodejs`, `Scikit-Learn→scikitlearn`, `CI/CD→cicd`, `PostgreSQL/Postgres→postgresql`, …
3. **Section detection** — heading-regex map for **Contact, Summary, Education, Experience, Projects, Skills, Certifications, Achievements**; each is graded `Present / Needs Attention / Missing` (contact is pattern-detected via email/phone links).
4. **Skill extraction** — a single declarative dictionary (`nlp/skills.py` / `api/lib/skills.js`) of 60+ skills × alias lists; extend by appending one entry.
5. **Keyword analysis** — stopword & JD-noise filtering, frequency × length scoring with skill-token boosting, plural stemming, presence check against the resume.
6. **Similarity** — `TfidfVectorizer` equivalent: sublinear TF (`1+ln tf`), smoothed IDF (`ln((1+N)/(1+df))+1`), L2-normalized cosine over the resume↔JD corpus.

## Matching Algorithm (transparent)

```
overall = 0.30 · skills_match        (JD skills found in resume / JD skills)
        + 0.25 · keyword_match       (rank-weighted coverage of top-12 JD keywords)
        + 0.20 · tfidf_similarity    (cosine of TF-IDF vectors, scaled ×100)
        + 0.25 · resume_quality      (50% section completeness + 50% content depth)
```

- **JD with no dictionary skills** → `0.40 keywords + 0.25 similarity + 0.35 quality`
- **No JD (resume-only mode)** → `0.60 quality + 0.25 skill breadth + 0.15 keyword richness`
- Content depth = contact info, summary length, bullet usage, quantified achievements (`45%`, `120k users`, `$2M`…), action verbs, word-count window (300–1100).
- All weights are returned in the response (`weights[]`) and shown in UI tooltips.

## API

### `POST /api/analyze`
`multipart/form-data`

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `resume` | file | ✅ | `.pdf` / `.docx` / `.txt`, ≤ 5 MB, ≥ 25 words |
| `job_description` | string | — | ≥10 words activates job-match mode |
| `demo` | `'true'\|'false'` | — | labels the response as demo data |

**Example**
```bash
curl -X POST http://localhost:8000/api/analyze \
  -F "resume=@./resume.pdf" \
  -F "job_description=Senior Software Engineer… (pasted JD text)"
```

**Response (trimmed)**
```json
{
  "overall_score": 72,
  "resume_quality_score": 84,
  "skills_match_score": 69,
  "keyword_match_score": 55,
  "similarity_score": 41,
  "skills": ["Python", "React", "…"],
  "matched_skills": ["React", "TypeScript"],
  "missing_skills": ["Kubernetes", "GraphQL"],
  "skill_match_ratio": { "matched": 9, "required": 13 },
  "keywords": ["postgresql", "docker"],
  "missing_keywords": ["payments", "observability"],
  "sections": [{ "key": "summary", "label": "Summary", "status": "present", "words": 52, "detail": "…" }],
  "completeness_score": 100,
  "recommendations": [{ "priority": "high", "title": "…", "detail": "…" }],
  "summary": "Your resume shows a strong alignment…",
  "charts": { "radar": […], "skill_distribution": […] },
  "weights": [{ "label": "Skill match", "value": 30 }],
  "meta": { "mode": "job_match", "processing_ms": 38, "disclaimer": "…" }
}
```

Errors: `400` missing/empty file · `413` > 5 MB · `415` unsupported type · `422` unreadable / too little text · `405` wrong method.

### `GET /api/health` → `{ "status": "ok", … }`

## Local Setup

### Python backend (FastAPI)
```bash
cd backend
cp .env.example .env          # optional; defaults work
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8000
# Swagger UI: http://localhost:8000/docs
```

### Frontend (React)
```bash
cp .env.example .env          # sets VITE_API_URL=http://localhost:8000
npm install
npm run dev                   # http://localhost:5173
```
`VITE_API_URL` points the React app at any backend. When unset, the app calls the same-origin `/api` serverless route (used in the hosted deployment), which runs the identical pipeline.

```bash
npm run build                 # production build
```

## Privacy & Limits

- Uploads are read into memory, analyzed, discarded — **no database, no disk writes**.
- 5 MB file ceiling, 60k-char JD ceiling, 60s client timeout, CORS allow-list via env.
- Recommendations explicitly tell users to add only skills/keywords that genuinely reflect their experience.

## Future Improvements

- spaCy NER for company/degree/certification entity extraction
- Sentence-embedding (SBERT) similarity alongside TF-IDF
- Multi-JD comparison, saved analyses, PDF report export
- Occupation-specific skill dictionaries and locale-aware parsing
