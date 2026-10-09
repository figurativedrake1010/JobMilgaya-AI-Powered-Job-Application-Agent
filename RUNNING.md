# JobMilgya: Setup and Run Guide

This guide explains how to run the JobMilgya AI Job Application Agent from a ZIP file or a cloned repository.

## 1. Prerequisites

Install the following software before starting:

- Docker Desktop with Docker Compose support
- Git, only if cloning the repository instead of using a ZIP file
- At least 8 GB of free disk space for Docker images and dependencies
- An OpenRouter API key for AI features such as resume analysis, job matching, and cover-letter generation

Docker Desktop must be running before the project is started.

## 2. Get the Project

### ZIP file

Extract the ZIP file, then open a terminal in the extracted project directory. The directory should contain files such as `docker-compose.yml`, `backend`, `frontend`, and `docker`.

Windows PowerShell example:

```powershell
Expand-Archive .\ai-job-agent.zip -DestinationPath .
Set-Location .\ai-job-agent
```

### Git repository

```bash
git clone <repository-url>
cd ai-job-agent
```

All commands below must be run from the project root, where `docker-compose.yml` is located.

## 3. Configure Environment Variables

Create a local `.env` file from the supplied template.

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

macOS/Linux:

```bash
cp .env.example .env
```

Open `.env` and update at least these values:

```env
SECRET_KEY=replace-with-a-long-random-secret
OPENROUTER_API_KEY=replace-with-your-openrouter-api-key
```

The default Docker database values in `.env.example` are suitable for local development:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password
POSTGRES_DB=jobagentdb
```

Keep `.env` private. Do not commit or share it if it contains real API keys or passwords. Share `.env.example` instead.

The following settings are optional:

- `LINKEDIN_EMAIL` and `LINKEDIN_PASSWORD` for LinkedIn automation
- `NAUKRI_EMAIL` and `NAUKRI_PASSWORD` for Naukri automation
- `ADZUNA_APP_ID` and `ADZUNA_APP_KEY` for Adzuna job searches
- `OLLAMA_*` settings if local Ollama support is enabled

The application can start without optional job-platform credentials, but related automation or search features may not work.

## 4. Start the Project

From the project root, run:

```bash
docker compose up --build
```

The first run builds the backend and frontend images. It also installs Python and Node dependencies, downloads the Playwright Chromium browser, and downloads the spaCy language model. This can take several minutes.

To start the services in the background instead:

```bash
docker compose up --build -d
```

The default local Compose configuration starts:

- PostgreSQL database
- FastAPI backend
- Next.js frontend

Nginx and Ollama are disabled by default for local development.

Database migrations run automatically when the backend container starts.

## 5. Open the Application

Open the frontend in a browser:

```text
http://localhost:3000
```

The frontend redirects new users to the registration/login screen.

Useful service URLs:

| Service | URL |
|---|---|
| Frontend | http://localhost:3000 |
| Backend health check | http://localhost:8000/health |
| Swagger API documentation | http://localhost:8000/docs |
| ReDoc API documentation | http://localhost:8000/redoc |
| PostgreSQL | localhost:5432 |

A successful health check returns:

```json
{"status":"ok","version":"1.0.0"}
```

## 6. View Logs

Show logs for all services:

```bash
docker compose logs -f
```

Show only backend logs:

```bash
docker compose logs -f backend
```

Show only frontend logs:

```bash
docker compose logs -f frontend
```

List service status:

```bash
docker compose ps
```

The database should show `healthy`, and the backend and frontend should show `Up`.

## 7. Stop and Restart

Stop the containers while keeping database data:

```bash
docker compose down
```

Start them again without rebuilding:

```bash
docker compose up -d
```

Rebuild after changing dependencies or Docker configuration:

```bash
docker compose up --build -d
```

## 8. Reset the Local Database

To stop the project and permanently delete the local PostgreSQL data:

```bash
docker compose down -v
```

Then start from a clean database:

```bash
docker compose up --build
```

Use `down -v` only when a complete local database reset is intended.

## 9. Common Problems

### Docker is not running

Start Docker Desktop and wait until it reports that Docker is running. Then retry:

```bash
docker compose up --build
```

### Port already in use

The default ports are:

- `3000` for the frontend
- `8000` for the backend
- `5432` for PostgreSQL

Stop the process using the conflicting port, or update the port mappings in `docker-compose.yml`. If changing the frontend or backend host port, update any related URLs as well.

### Backend cannot connect to the database

Check that the database container is running and healthy:

```bash
docker compose ps
docker compose logs db
```

The `POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` values must match the database configuration in `.env`. If the database was initialized with old credentials, reset the volume with `docker compose down -v` and start again.

### AI requests do not work

Confirm that `OPENROUTER_API_KEY` in `.env` is a valid key, then recreate the backend container:

```bash
docker compose up --build -d backend
```

### The frontend cannot reach the backend

Check both URLs in a browser:

- http://localhost:3000
- http://localhost:8000/health

Then inspect the service logs:

```bash
docker compose logs backend frontend
```

## 10. Optional: Run Without Docker

The Docker method is recommended because it supplies the PostgreSQL database and keeps backend/frontend dependencies consistent.

A non-Docker run requires:

- Python 3.11+
- Node.js 20+
- PostgreSQL 15
- A PostgreSQL database matching `DATABASE_URL` in `.env`

Backend on Windows PowerShell:

```powershell
python -m venv backend\venv
backend\venv\Scripts\Activate.ps1
pip install -r backend\requirements.txt
python -m spacy download en_core_web_sm
playwright install chromium
alembic upgrade head
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
```

Backend on macOS/Linux:

```bash
python3 -m venv backend/venv
source backend/venv/bin/activate
pip install -r backend/requirements.txt
python -m spacy download en_core_web_sm
playwright install chromium
alembic upgrade head
uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
```

In a second terminal, install and start the frontend:

```bash
npm install --prefix frontend
cd frontend
npm run dev
```

The frontend runs at http://localhost:3000. The backend runs at http://localhost:8000.

## 11. Files That Should Be Shared

Include these in the ZIP:

- `backend/`
- `frontend/`
- `docker/`
- `alembic/`
- `docker-compose.yml`
- `docker-compose.override.yml`
- `alembic.ini`
- `.env.example`
- `README.md`
- `RUNNING.md`

Do not include:

- `.env` with real secrets
- `backend/venv/`
- `frontend/node_modules/`
- Docker database volumes
- temporary caches
- private resumes, logs, or screenshots
