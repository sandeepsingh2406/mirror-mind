# Mirror Mind

Mirror Mind is an AI-powered digital twin chatbot. It answers questions about a person's background, skills, and experience in their voice, grounded in a curated set of facts, a resume/LinkedIn export, and notes on communication style — with conversation memory and a one-command path to deploying it as a fully serverless app on AWS.

## Features

- **Conversational digital twin** — a FastAPI backend that builds a system prompt from structured facts, a summary, a LinkedIn PDF export, and a style guide, then answers as that person via AWS Bedrock.
- **Persistent conversation memory** — each chat session is tracked by a `session_id` and its history is stored either on local disk or in S3, so follow-up questions have context.
- **Simple chat UI** — a Next.js + Tailwind frontend with a single chat widget component.
- **Serverless deployment** — Terraform provisions API Gateway, a Lambda function (via Mangum), S3 buckets for the static frontend and memory storage, and an optional CloudFront + custom domain setup.
- **One-command deploy/destroy** — shell/PowerShell scripts and GitHub Actions workflows wrap the Terraform + build steps.

## Architecture

```
frontend/ (Next.js, static export)
   │  fetch /chat, /conversation/{id}
   ▼
API Gateway (HTTP API)
   │
   ▼
Lambda (FastAPI via Mangum) ──► AWS Bedrock (chat completion)
   │
   ▼
S3 (conversation memory) / local filesystem in dev
```

The same FastAPI app (`backend/server.py`) runs locally with `uvicorn` for development and inside Lambda (`backend/lambda_handler.py`) in production — no code changes between the two.

## Tech stack

- **Backend:** Python 3.12, FastAPI, boto3 (AWS Bedrock), Mangum, uv
- **Frontend:** Next.js 16, React 19, TypeScript, Tailwind CSS
- **Infrastructure:** Terraform (API Gateway, Lambda, S3, CloudFront, Route 53, ACM)
- **CI/CD:** GitHub Actions

## Project structure

```
backend/
  server.py          FastAPI app and chat/memory endpoints
  context.py          Builds the system prompt from data files
  resources.py         Loads facts, summary, style, and LinkedIn PDF
  lambda_handler.py     Mangum entrypoint for AWS Lambda
  deploy.py            Builds the Lambda deployment zip
  data/                Personal facts, summary, style notes, LinkedIn export
frontend/
  app/                 Next.js app router pages
  components/twin.tsx   Chat widget
terraform/            Infrastructure as code for the AWS deployment
scripts/               Deploy/destroy helper scripts (bash + PowerShell)
.github/workflows/     CI/CD pipelines for deploy and destroy
```

## Getting started

### Prerequisites

- Python 3.12+ and [uv](https://docs.astral.sh/uv/)
- Node.js 18+ and npm
- An AWS account with access to Amazon Bedrock, and credentials configured locally (`aws configure`)

### 1. Configure your digital twin's data

Add your own information to `backend/data/`:

- `facts.json` — structured facts (name, role, etc.)
- `summary.txt` — a free-text summary
- `style.txt` — notes on tone/communication style
- `linkedin.pdf` — a PDF export of your LinkedIn profile

### 2. Backend setup

```bash
cd backend
cp ../.env.example .env   # fill in your AWS account ID / region
uv sync
uv run server.py
```

The API will be available at `http://localhost:8000`, with `/`, `/health`, `/chat`, and `/conversation/{session_id}` endpoints.

### 3. Frontend setup

```bash
cd frontend
npm install
npm run dev
```

Open `http://localhost:3000` to chat with your digital twin. By default the frontend talks to `http://localhost:8000`; set `NEXT_PUBLIC_API_URL` to point it elsewhere.

## Configuration

Backend environment variables (see `.env.example`):

| Variable | Description | Default |
|---|---|---|
| `AWS_ACCOUNT_ID` | AWS account used for deployment | — |
| `DEFAULT_AWS_REGION` | AWS region for Bedrock and resources | `us-east-1` |
| `PROJECT_NAME` | Prefix used for deployed resource names | `twin` |
| `CORS_ORIGINS` | Comma-separated list of allowed origins | `http://localhost:3000` |
| `BEDROCK_MODEL_ID` | Bedrock model used for chat completion | `global.amazon.nova-2-lite-v1:0` |
| `USE_S3` | Store conversation memory in S3 instead of local disk | `false` |
| `S3_BUCKET` | S3 bucket name for memory storage (when `USE_S3=true`) | — |
| `MEMORY_DIR` | Local directory for conversation memory (when `USE_S3=false`) | `../memory` |

## Deployment

Infrastructure is managed with Terraform and deployed via the provided scripts, which build the Lambda package, apply Terraform, and sync the built frontend to S3:

```bash
./scripts/deploy.sh <environment> <project_name>   # e.g. ./scripts/deploy.sh prod twin
```

To tear everything down:

```bash
./scripts/destroy.sh <environment> <project_name>
```

PowerShell equivalents (`deploy.ps1`, `destroy.ps1`) are also provided. GitHub Actions workflows under `.github/workflows/` run the same deploy/destroy steps in CI.

## License

This project is licensed under the [MIT License](LICENSE).
