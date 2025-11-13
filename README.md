# Hirelink
This project is an intelligent job-matching platform where users can upload their resumes and instantly receive a curated list of companies offering relevant job and internship opportunities. The system analyzes the user’s resume, identifies key skills, experience, and interests, and matches them with active job openings from various companies.  
# PathFinder Job Portal – Monorepo Guide

This repository groups the three services that power the PathFinder job and internship portal:

| Repo | Description | Runtime |
|------|-------------|---------|
| `job-portal-frontend` | Vite + React SPA for seekers (auth, profile, job search, applications) | Node 20 |
| `job-portal-api` | Express REST API (authentication, profiles, job search proxy, applications) | Node 20 |
| `job-portal-ingestor` | Django + Celery service ingesting external job feeds and exposing recommendations | Python 3.10 |

Each service lives in its own folder. Deployments are independent so you can host them on platforms that fit best (e.g. Vercel for the frontend, Render for the APIs).

---

## 1. Local Development

### Prerequisites
- Node.js 20+ and npm
- Python 3.10+ and `pip`
- PostgreSQL (local or Docker)
- Redis/SQS equivalent if you want to run Celery locally

### Frontend (React)
```bash
cd job-portal-frontend
npm install
npm run dev
```
Create `.env.local`:
```
VITE_API_URL=http://localhost:4000/api
VITE_JOB_SERVICE_URL=http://localhost:8000/api
```

### Node API (Express)
```bash
cd job-portal-api
npm install
npm run dev
```
Create `.env`:
```
PORT=4000
NODE_ENV=development
DATABASE_URL=postgres://postgres:postgres@localhost:5432/pathfinder
JWT_ACCESS_SECRET=local-access-secret
JWT_REFRESH_SECRET=local-refresh-secret
CLIENT_ORIGIN=http://localhost:5173
AWS_REGION=us-east-1
AWS_S3_BUCKET=pathfinder-local-resumes
JOB_SERVICE_BASE_URL=http://localhost:8000/api
```
The API auto-creates basic tables on startup. Swap in a managed Postgres instance for shared environments.

### Django Ingestor
```bash
cd job-portal-ingestor
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver 0.0.0.0:8000
```
`.env` example:
```
DJANGO_SECRET_KEY=dev-secret
DJANGO_DEBUG=true
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgres://postgres:postgres@localhost:5432/pathfinder
CORS_ALLOWED_ORIGINS=http://localhost:5173,http://localhost:4000
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0
ADZUNA_APP_ID=
ADZUNA_APP_KEY=
JOB_COUNTRY=in
```
Run a Celery worker for background ingestion:
```bash
celery -A job_ingestor worker -l info
```

---

## 2. Deployment Checklist

### Frontend on Vercel
1. Push `job-portal-frontend` to GitHub.
2. Import into Vercel, select the Vite preset.
3. Configure environment variables:
   - `VITE_API_URL=https://<node-api-domain>/api`
   - `VITE_JOB_SERVICE_URL=https://<django-domain>/api`
4. Deploy and (optionally) attach a custom domain.

### Node API on Render (Web Service)
1. Provision a Postgres database (Render → PostgreSQL).
2. Deploy `job-portal-api`:
   - Build: `npm install && npm run build`
   - Start: `npm run start`
3. Set environment variables:
   - `PORT=4000`
   - `NODE_ENV=production`
   - `DATABASE_URL=<render-postgres-url>`
   - `JWT_ACCESS_SECRET`, `JWT_REFRESH_SECRET`
   - `CLIENT_ORIGIN=https://<vercel-domain>`
   - `AWS_REGION`, `AWS_S3_BUCKET`, `CUSTOM_CDN_DOMAIN` (if using CloudFront)
   - `JOB_SERVICE_BASE_URL=https://<django-domain>/api`
4. Resulting URL becomes your backend base (`https://...onrender.com/api`).

### Django Ingestor on Render
1. Provision Redis (or another broker) for Celery.
2. Deploy `job-portal-ingestor`:
   - Build: `pip install -r requirements.txt`
   - Start: `gunicorn job_ingestor.wsgi:application --log-file -`
3. Env vars:
   - `DJANGO_SECRET_KEY`, `DJANGO_DEBUG=false`
   - `DJANGO_ALLOWED_HOSTS=<render-django-host>`
   - `DATABASE_URL=<render-postgres-url>`
   - `CORS_ALLOWED_ORIGINS=https://<vercel-domain>,https://<node-domain>`
   - `CELERY_BROKER_URL`, `CELERY_RESULT_BACKEND`
   - `ADZUNA_APP_ID`, `ADZUNA_APP_KEY`, `JOB_COUNTRY`
4. Create a Background Worker for Celery: command `celery -A job_ingestor worker -l info`.

### Connect Everything
- Update the frontend env vars with real API URLs and redeploy.
- Ensure CORS headers allow your Vercel domain.
- Resume uploads require S3 bucket + IAM credentials.

---

## 3. Useful Commands

| Service | Build | Lint/Test | Start |
|---------|-------|-----------|-------|
| Frontend | `npm run build` | `npm run lint` (if added) | `npm run dev` |
| Node API | `npm run build` | `npm run lint` | `npm run dev` |
| Ingestor | `python manage.py check` | `python manage.py test` (add tests) | `python manage.py runserver` |

Postgres tables can be inspected with:
```bash
psql "$DATABASE_URL"
\dt
SELECT * FROM users LIMIT 5;
```

---

## 4. Next Steps
- Add automated tests (unit + integration + e2e).
- Lock down security (HTTPS enforcement, rate limiting, logging).
- Introduce Terraform/Render YAML for reproducible infra.
- Expand ingestion beyond Adzuna, add employer/admin portal, schedule resume parsing, etc.

Feel free to adapt this README per repository when you split them into standalone repos. Each service can have its own README that focuses on its tech stack and deployment specifics.
