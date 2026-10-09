# JobMilgya — AI Job Application Agent

An AI-powered job application platform that parses resumes, matches jobs, generates cover letters, and automates applications.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | Next.js 14, TailwindCSS, TypeScript |
| Backend | FastAPI, Python 3.11 |
| Database | PostgreSQL 15 |
| AI / LLM | OpenRouter (Claude/GPT) + Ollama fallback |
| Automation | Playwright (Chromium) |
| Proxy / TLS | Nginx |
| Containers | Docker, Docker Compose |

---

## Project Structure

```
job_agent/
├── backend/          # FastAPI app (agents, API routes, DB, scheduler)
├── frontend/         # Next.js app
├── docker/
│   ├── backend/Dockerfile
│   ├── frontend/Dockerfile
│   └── nginx/nginx.conf
├── docker-compose.yml
├── .env.example
└── alembic.ini       # DB migrations
```

---

## Prerequisites

- Python 3.11+
- Node.js 20+
- PostgreSQL 15 (or Docker)
- `git`

---

## Running Locally (without Docker)

### 1. Clone & configure environment

```bash
git clone <repo-url>
cd job_agent
cp .env.example .env
# Edit .env — set SECRET_KEY, DATABASE_URL, OPENROUTER_API_KEY at minimum
```

### 2. Backend

Run all of the following from the **project root** (`job_agent/`):

**Linux / macOS**
```bash
# Create and activate a virtual environment
python3.11 -m venv backend/venv
source backend/venv/bin/activate

# Install dependencies
pip install -r backend/requirements.txt

# Download NLP model used by the resume parser
python -m spacy download en_core_web_sm

# Install Playwright browser (used for job automation)
playwright install chromium
playwright install-deps chromium

# Run database migrations
alembic upgrade head

# Start the API server
source backend/venv/bin/activate && uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
```

**Windows (PowerShell)**
```powershell
# Create and activate a virtual environment
python -m venv backend\venv
backend\venv\Scripts\Activate.ps1

# Install dependencies
pip install -r backend\requirements.txt

# Download NLP model used by the resume parser
python -m spacy download en_core_web_sm

# Install Playwright browser (used for job automation)
playwright install chromium

# Run database migrations
alembic upgrade head

# Start the API server
backend\venv\Scripts\activate && uvicorn backend.main:app --reload --host 0.0.0.0 --port 8000
```

> **Note:** `playwright install-deps` is Linux-only. On Windows, Playwright manages its own browser dependencies automatically.

Backend is now available at:
- API: `http://localhost:8000/api/v1`
- Swagger docs: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`

### 3. Frontend

Open a new terminal tab and run from the **project root** (`job_agent/`):

**Linux / macOS**
```bash
# Install dependencies
npm install --prefix frontend

# Create frontend env file
echo "NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1" > frontend/.env.local

# Fix binary permissions (required on Linux), then start dev server
cd frontend && chmod +x node_modules/.bin/* && npm run dev
```

**Windows (PowerShell)**
```powershell
# Install dependencies
npm install --prefix frontend

# Create frontend env file
echo "NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1" > frontend\.env.local

# Start dev server
cd frontend && npm run dev
```

> **Note:** The `chmod +x node_modules/.bin/*` step is Linux/macOS only. Windows does not need it.

Frontend is now available at `http://localhost:3000`

---

## Running with Docker Compose (recommended)

```bash
# 1. Configure environment
cp .env.example .env
# Edit .env with your real values

# 2. Build and start all services (DB, backend, frontend, nginx)
docker-compose up --build

# 3. Run DB migrations (first time only)
docker-compose exec backend alembic upgrade head
```

| Service | URL |
|---------|-----|
| App (via Nginx) | http://localhost |
| Frontend (direct) | http://localhost:3000 |
| Backend API | http://localhost:8000 |
| Swagger Docs | http://localhost:8000/docs |
| PostgreSQL | localhost:5432 |

To stop:
```bash
docker-compose down          # keep volumes
docker-compose down -v       # also delete DB data
```

---

## Environment Variables

Copy `.env.example` to `.env` and fill in the required values:

| Variable | Required | Description |
|----------|----------|-------------|
| `SECRET_KEY` | Yes | Random 32+ char string for JWT signing |
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `OPENROUTER_API_KEY` | Yes | Get from https://openrouter.ai/keys |
| `LINKEDIN_EMAIL` / `LINKEDIN_PASSWORD` | Optional | For LinkedIn Easy Apply automation |
| `NAUKRI_EMAIL` / `NAUKRI_PASSWORD` | Optional | For Naukri automation |

---

## Deploying on AWS EC2

### 1. Launch an EC2 instance

- **AMI**: Ubuntu 22.04 LTS (64-bit x86)
- **Instance type**: `t3.medium` (2 vCPU, 4 GB RAM) minimum; `t3.large` recommended if using Ollama
- **Storage**: 20 GB gp3 minimum
- **Security Group — open these ports**:

  | Port | Protocol | Source | Purpose |
  |------|----------|--------|---------|
  | 22 | TCP | Your IP | SSH |
  | 80 | TCP | 0.0.0.0/0 | HTTP |
  | 443 | TCP | 0.0.0.0/0 | HTTPS (after adding TLS) |

  Do **not** expose 3000, 8000, or 5432 to the internet — Nginx handles all public traffic on port 80.

### 2. SSH into the instance

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
```

### 3. Install Docker & Docker Compose

```bash
sudo apt-get update && sudo apt-get upgrade -y

# Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker ubuntu
newgrp docker                 # apply group change without re-login

# Docker Compose plugin
sudo apt-get install -y docker-compose-plugin
docker compose version        # verify
```

### 4. Clone the repository

```bash
git clone <repo-url>
cd job_agent
```

### 5. Configure environment

```bash
cp .env.example .env
nano .env
```

Key values to set for production:

```env
SECRET_KEY=<random-64-char-string>     # openssl rand -hex 32
ENVIRONMENT=production
DEBUG=false
DATABASE_URL=postgresql+asyncpg://jobagent:<strong-password>@db:5432/jobagentdb
POSTGRES_USER=jobagent
POSTGRES_PASSWORD=<strong-password>
POSTGRES_DB=jobagentdb
OPENROUTER_API_KEY=<your-key>
```

### 6. Update the frontend API URL

Edit `docker-compose.yml` and replace `localhost` with your EC2 public IP (or domain):

```yaml
frontend:
  environment:
    NEXT_PUBLIC_API_URL: http://<EC2_PUBLIC_IP>/api/v1
```

### 7. Build and start services

```bash
docker compose up --build -d
```

### 8. Run database migrations

```bash
docker compose exec backend alembic upgrade head
```

### 9. Verify everything is running

```bash
docker compose ps                          # all services should show "Up"
curl http://localhost/health               # should return {"status":"ok"}
```

Open `http://<EC2_PUBLIC_IP>` in your browser — the app should load.

### 10. (Optional) Point a domain + enable HTTPS

```bash
# Install Certbot
sudo apt-get install -y certbot

# Stop Nginx container temporarily
docker compose stop nginx

# Issue certificate
sudo certbot certonly --standalone -d yourdomain.com

# Update docker/nginx/nginx.conf to add HTTPS listener and cert paths
# Then restart
docker compose start nginx
```

---

## Useful Commands

```bash
# View live logs
docker compose logs -f

# View logs for a single service
docker compose logs -f backend

# Restart a single service after a code change
docker compose build backend && docker compose up -d --no-deps backend

# Open a shell inside the backend container
docker compose exec backend bash

# Pull latest code and redeploy
git pull
docker compose up --build -d
```

---

## API Documentation

- Swagger UI: `http://<host>/docs`
- ReDoc: `http://<host>/redoc`
