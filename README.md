# KelanaAI

AI-powered travel planner that turns a destination, trip length, budget, and travel style into a structured day-by-day itinerary, and offers a travel assistant chat grounded in a knowledge base.

**Live demo:** https://kelana-ai-rose.vercel.app/

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Backend Setup](#backend-setup)
  - [Frontend Setup](#frontend-setup)
- [Environment Variables](#environment-variables)
- [Database Migrations](#database-migrations)
- [API Reference](#api-reference)
- [Deployment](#deployment)

## Features

- **Account system** — register and log in with email and password (bcrypt-hashed, JWT bearer tokens valid for 60 minutes).
- **Trip planning** — create trips with destination, days, budget (USD), and travel style (Family, Solo, Couple, Group). Each trip is automatically categorised as *Backpacker* (< $1,000), *Standard* ($1,000–$3,000), or *Luxury* (> $3,000), with a computed daily budget.
- **AI itinerary generation** — generates a structured plan via Amazon Bedrock: daily morning/afternoon/evening activities, estimated costs, travel tips, local food recommendations, and a budget breakdown. Results are cached on the trip so each itinerary is generated only once.
- **Travel assistant chat** — multi-turn conversations backed by retrieval-augmented generation (RAG) over an Amazon Bedrock Knowledge Base, with the source document shown when context was found.
- **Per-user data** — trips and conversations are scoped to the authenticated user.

## Tech Stack

| Layer     | Technology                                                                 |
| --------- | -------------------------------------------------------------------------- |
| Frontend  | Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4              |
| Backend   | FastAPI, SQLAlchemy, Alembic, Pydantic                                     |
| Database  | PostgreSQL (Neon)                                                          |
| AI        | Amazon Bedrock (Converse API) and Bedrock Knowledge Bases (via `boto3`)    |
| Auth      | JWT (`python-jose`), bcrypt (`passlib`)                                    |
| Hosting   | Vercel (frontend)                                                          |

## Architecture

```
┌──────────────────┐   HTTPS + Bearer JWT   ┌──────────────────┐
│  Next.js client  │ ─────────────────────▶ │   FastAPI API    │
│   (frontend/)    │ ◀───────────────────── │    (backend/)    │
└──────────────────┘          JSON          └────────┬─────────┘
                                                     │
                         ┌───────────────────────────┼───────────────────────────┐
                         ▼                           ▼                           ▼
                ┌──────────────────┐      ┌──────────────────────┐    ┌──────────────────────┐
                │ PostgreSQL (Neon)│      │ Bedrock Runtime      │    │ Bedrock Knowledge    │
                │ users, trips,    │      │ itinerary + chat     │    │ Base (RAG retrieval) │
                │ conversations,   │      │ generation           │    │                      │
                │ messages         │      └──────────────────────┘    └──────────────────────┘
                └──────────────────┘
```

## Project Structure

```
.
├── backend/
│   ├── main.py               # FastAPI app, routes, auth dependency, CORS
│   ├── database.py           # SQLAlchemy engine, session, Base
│   ├── models/               # ORM models: users, trip, conversations, messages
│   ├── services/
│   │   ├── auth_service.py   # Registration, login, JWT issuing
│   │   ├── bedrock_service.py# Itinerary and chat generation via Bedrock
│   │   ├── kb_service.py     # Knowledge Base retrieval for RAG
│   │   └── trip_services.py  # Budget category and daily budget logic
│   ├── alembic/              # Database migrations
│   └── requirements.txt
└── frontend/
    ├── app/                  # Next.js routes: /, login, register, trips, trips/[id],
    │                         #   generate, assistant, profile
    ├── components/           # UI components and AuthContext
    ├── services/             # API clients (auth, trips, assistant)
    └── types/                # Shared TypeScript types
```

## Getting Started

### Prerequisites

- Python 3.10+
- Node.js 20+ and npm
- A PostgreSQL database (e.g. a [Neon](https://neon.tech) project)
- An AWS account with Amazon Bedrock model access and a Bedrock Knowledge Base

### Backend Setup

```bash
cd backend
python -m venv .venv
# Windows: .venv\Scripts\activate
source .venv/bin/activate
pip install -r requirements.txt
```

Create `backend/.env` (see [Environment Variables](#environment-variables)), then run:

```bash
alembic upgrade head
uvicorn main:app --reload
```

The API runs at http://localhost:8000, with interactive docs at http://localhost:8000/docs.

### Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env.local`:

```env
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
```

Then start the dev server:

```bash
npm run dev
```

Open http://localhost:3000.

## Environment Variables

### Backend (`backend/.env`)

| Variable                   | Description                                                         |
| -------------------------- | ------------------------------------------------------------------- |
| `DATABASE_URL`             | PostgreSQL connection string                                        |
| `JWT_SECRET_KEY`           | Secret used to sign JWTs; use a long random value                   |
| `FRONTEND_URL`             | Allowed CORS origin, e.g. `http://localhost:3000`                   |
| `AWS_REGION`               | AWS region for Bedrock, e.g. `us-east-1`                            |
| `AWS_BEARER_TOKEN_BEDROCK` | Amazon Bedrock API key                                              |
| `MODEL_ID`                 | Bedrock model or inference profile ID used for generation and chat  |
| `KNOWLEDGE_BASE_ID`        | Bedrock Knowledge Base ID used for chat retrieval                   |

### Frontend (`frontend/.env.local`)

| Variable              | Description                                              |
| --------------------- | -------------------------------------------------------- |
| `NEXT_PUBLIC_API_URL` | Backend base URL **including** `/api/v1`                 |

> Never commit `.env` files. They are already listed in `.gitignore`.

## Database Migrations

Migrations are managed with Alembic and read `DATABASE_URL` from the environment.

```bash
cd backend
alembic upgrade head                              # apply all migrations
alembic revision --autogenerate -m "describe change"  # create a new migration
```

## API Reference

All endpoints are prefixed with `/api/v1`. Endpoints marked 🔒 require an `Authorization: Bearer <token>` header.

| Method | Endpoint                                   | Description                                   |
| ------ | ------------------------------------------ | --------------------------------------------- |
| POST   | `/auth/register`                           | Create an account                             |
| POST   | `/auth/login`                              | Log in and receive an access token            |
| GET    | `/auth/me` 🔒                              | Current user profile and trip count           |
| GET    | `/trips` 🔒                                | List the user's trips                         |
| POST   | `/trips` 🔒                                | Create a trip                                 |
| GET    | `/trips/{trip_id}` 🔒                      | Get a trip                                    |
| PUT    | `/trips/{trip_id}?new_budget=` 🔒          | Update a trip's budget                        |
| DELETE | `/trips/{trip_id}` 🔒                      | Delete a trip                                 |
| POST   | `/trips/{trip_id}/generate` 🔒             | Generate (or return cached) AI itinerary      |
| GET    | `/conversations` 🔒                        | List conversations                            |
| POST   | `/conversations` 🔒                        | Start a conversation                          |
| PATCH  | `/conversations/{id}` 🔒                   | Rename a conversation                         |
| GET    | `/conversations/{id}/messages` 🔒          | List messages                                 |
| POST   | `/conversations/{id}/messages` 🔒          | Send a message and get the assistant's reply  |

`GET /health` returns `{"status": "OK"}` for health checks. Full schemas are available in the generated OpenAPI docs at `/docs`.

## Deployment

- **Frontend** is deployed on Vercel: https://kelana-ai-rose.vercel.app/. Set the Vercel project root to `frontend/` and configure `NEXT_PUBLIC_API_URL` to point at the deployed backend.
- **Backend** can run on any host that supports Python ASGI apps: `uvicorn main:app --host 0.0.0.0 --port $PORT`. Set `FRONTEND_URL` to the deployed frontend origin so CORS allows it, and run `alembic upgrade head` against the production database before starting.
