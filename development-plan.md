# Predictive Analytics Workbench -- Development Plan

> Project: Candidate #030 · Plan created: 2026-05-25

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Data Model Selection](#data-model-selection)
4. [Phase Dependency Graph](#phase-dependency-graph)
5. [Phase 1: Foundation & Data Layer](#phase-1-foundation--data-layer)
6. [Phase 2: Data Ingestion & Profiling](#phase-2-data-ingestion--profiling)
7. [Phase 3: AutoML Training Engine](#phase-3-automl-training-engine)
8. [Phase 4: Model Leaderboard & Explainability](#phase-4-model-leaderboard--explainability)
9. [Phase 5: LLM Copilot Integration](#phase-5-llm-copilot-integration)
10. [Phase 6: Prediction & Batch Export](#phase-6-prediction--batch-export)
11. [Phase 7: Time-Series Forecasting](#phase-7-time-series-forecasting)
12. [Phase 8: Drift Detection & Retraining](#phase-8-drift-detection--retraining)
13. [Phase 9: Context-Aware Feature Engineering](#phase-9-context-aware-feature-engineering)
14. [Phase 10: Narrative Generation & Reporting](#phase-10-narrative-generation--reporting)
15. [Phase 11: REST API & Real-Time Serving](#phase-11-rest-api--real-time-serving)
16. [Phase 12: Production Hardening & SaaS](#phase-12-production-hardening--saas)
17. [Definition of Done](#definition-of-done)

---

## Technology Decisions

### Backend Framework: Python + FastAPI

**Rationale:** The ML ecosystem (AutoGluon, SHAP, pandas, scikit-learn) is Python-native. FastAPI provides async request handling, automatic OpenAPI docs, and Pydantic validation. The entire model training pipeline, feature engineering, and explainability stack runs in Python without language boundary overhead.

### ML Engine: AutoGluon (Tabular + TimeSeries)

**Rationale:** AutoGluon is Apache 2.0 licensed with no patent encumbrances. It won the NeurIPS 2025 TabArena benchmark (peer-reviewed best-in-class tabular accuracy). AutoGluon-TimeSeries integrates Chronos foundation models (also Apache 2.0) for zero-shot forecasting. Three lines of code produce competitive results. No IP risk from DataRobot or H2O patents since AutoGluon uses independent ensemble stacking (LightGBM, XGBoost, CatBoost, neural nets) rather than evolutionary feature engineering.

### Time-Series Foundation Models: Chronos (Amazon)

**Rationale:** Apache 2.0 model weights freely usable. Enables zero-shot forecasting for cold-start datasets. Integrates natively with AutoGluon-TimeSeries. No runtime dependency on AWS services.

### Explainability: SHAP

**Rationale:** MIT licensed, no patent encumbrances. Industry standard for feature importance. Produces both global (model-level) and local (per-prediction) explanations. Integrates with all AutoGluon-supported algorithms.

### LLM Integration: Anthropic Claude API

**Rationale:** The README explicitly names Claude for copilot and narrative generation. Used for: conversational workflow guidance, domain-aware feature engineering, plain-English prediction explanations, and drift narratives.

### Database: PostgreSQL 16+

**Rationale:** Mature JSONB support with GIN indexes. Supports table partitioning for time-series data. Optional TimescaleDB extension for hypertables. UUID primary keys for distributed deployment. The entire data model runs on a single database engine.

### Data Model: Hybrid Relational + JSONB (Model 3) with Feature Store extensions from Model 4

**Rationale:** See [Data Model Selection](#data-model-selection) below.

### Frontend: Next.js 15 + React 19

**Rationale:** Server-side rendering for dashboard performance. React ecosystem has the best charting libraries (Recharts, Nivo) for model metrics visualization. TypeScript provides type safety for the complex data structures (JSONB payloads). Tailwind CSS + shadcn/ui for rapid UI development.

### Task Queue: Celery + Redis

**Rationale:** Model training is long-running (minutes to hours). Celery provides distributed task execution, progress tracking, and retry logic. Redis serves as both the message broker and the cache layer for real-time prediction serving.

### Experiment Tracking: MLflow (compatibility layer)

**Rationale:** MLflow is Apache 2.0 and the de facto standard. The data model aligns with MLflow concepts (experiments, runs, params, metrics). Providing MLflow-compatible export enables data science teams to use familiar tooling alongside the no-code workbench.

### Containerization: Docker + Docker Compose

**Rationale:** Self-hosted deployment is a core requirement (free tier). Docker Compose orchestrates PostgreSQL, Redis, the API server, the Celery worker, and the frontend. Single-command startup for development and self-hosted production.

### Object Storage: S3-compatible (MinIO for self-hosted)

**Rationale:** Model artifacts, dataset files, and prediction outputs require object storage. S3 API is the standard. MinIO provides S3-compatible self-hosted storage. Production SaaS deployment uses AWS S3 or GCS.

---

## Project Structure

```
predictive-analytics-workbench/
├── docker-compose.yml
├── docker-compose.prod.yml
├── .env.example
├── Makefile
│
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── alembic/
│   │   └── versions/                    # Database migrations
│   │
│   ├── app/
│   │   ├── main.py                      # FastAPI application entry point
│   │   ├── config.py                    # Settings (Pydantic BaseSettings)
│   │   ├── database.py                  # SQLAlchemy engine + session
│   │   │
│   │   ├── models/                      # SQLAlchemy ORM models
│   │   │   ├── organisation.py
│   │   │   ├── user.py
│   │   │   ├── project.py
│   │   │   ├── dataset.py
│   │   │   ├── feature_set.py
│   │   │   ├── experiment.py
│   │   │   ├── run.py
│   │   │   ├── model_registry.py
│   │   │   ├── explanation.py
│   │   │   ├── prediction.py
│   │   │   ├── drift.py
│   │   │   ├── copilot.py
│   │   │   └── audit.py
│   │   │
│   │   ├── schemas/                     # Pydantic request/response schemas
│   │   │   ├── project.py
│   │   │   ├── dataset.py
│   │   │   ├── experiment.py
│   │   │   ├── model.py
│   │   │   ├── prediction.py
│   │   │   └── copilot.py
│   │   │
│   │   ├── api/                         # FastAPI routers
│   │   │   ├── v1/
│   │   │   │   ├── projects.py
│   │   │   │   ├── datasets.py
│   │   │   │   ├── experiments.py
│   │   │   │   ├── models.py
│   │   │   │   ├── predictions.py
│   │   │   │   ├── copilot.py
│   │   │   │   ├── drift.py
│   │   │   │   └── auth.py
│   │   │   └── router.py
│   │   │
│   │   ├── services/                    # Business logic layer
│   │   │   ├── dataset_service.py       # Upload, profile, quality check
│   │   │   ├── feature_service.py       # Feature engineering + LLM generation
│   │   │   ├── training_service.py      # AutoGluon training orchestration
│   │   │   ├── explanation_service.py   # SHAP computation + narrative generation
│   │   │   ├── prediction_service.py    # Batch prediction + warehouse writeback
│   │   │   ├── drift_service.py         # Drift detection + retraining trigger
│   │   │   ├── copilot_service.py       # LLM conversation management
│   │   │   ├── narrative_service.py     # Plain-English explanation generation
│   │   │   └── storage_service.py       # S3/MinIO file operations
│   │   │
│   │   ├── ml/                          # ML engine wrappers
│   │   │   ├── autogluon_tabular.py     # Classification + regression
│   │   │   ├── autogluon_timeseries.py  # Time-series forecasting
│   │   │   ├── feature_engineering.py   # Lag, rolling window, seasonal features
│   │   │   ├── data_profiler.py         # Column statistics, quality checks
│   │   │   ├── shap_explainer.py        # SHAP value computation
│   │   │   └── drift_detector.py        # PSI, KS test, performance comparison
│   │   │
│   │   ├── llm/                         # LLM integration
│   │   │   ├── claude_client.py         # Anthropic SDK wrapper
│   │   │   ├── prompts/                 # Prompt templates
│   │   │   │   ├── copilot_onboarding.py
│   │   │   │   ├── feature_generation.py
│   │   │   │   ├── model_explanation.py
│   │   │   │   ├── drift_narrative.py
│   │   │   │   └── forecast_narrative.py
│   │   │   └── tools.py                 # Tool definitions for Claude tool use
│   │   │
│   │   └── tasks/                       # Celery async tasks
│   │       ├── celery_app.py
│   │       ├── training_tasks.py
│   │       ├── prediction_tasks.py
│   │       ├── drift_tasks.py
│   │       └── feature_tasks.py
│   │
│   └── tests/
│       ├── conftest.py
│       ├── unit/
│       ├── integration/
│       └── fixtures/
│           ├── sample_tabular.csv
│           ├── sample_timeseries.csv
│           └── sample_churn.csv
│
├── frontend/
│   ├── package.json
│   ├── next.config.ts
│   ├── tailwind.config.ts
│   │
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx                 # Landing / dashboard
│   │   │   ├── (auth)/
│   │   │   │   ├── login/page.tsx
│   │   │   │   └── register/page.tsx
│   │   │   ├── projects/
│   │   │   │   ├── page.tsx             # Project list
│   │   │   │   └── [slug]/
│   │   │   │       ├── page.tsx         # Project overview
│   │   │   │       ├── data/page.tsx    # Dataset management
│   │   │   │       ├── features/page.tsx
│   │   │   │       ├── experiments/page.tsx
│   │   │   │       ├── models/page.tsx
│   │   │   │       ├── predictions/page.tsx
│   │   │   │       ├── drift/page.tsx
│   │   │   │       └── copilot/page.tsx
│   │   │   └── settings/page.tsx
│   │   │
│   │   ├── components/
│   │   │   ├── ui/                      # shadcn/ui primitives
│   │   │   ├── layout/
│   │   │   │   ├── sidebar.tsx
│   │   │   │   ├── header.tsx
│   │   │   │   └── project-nav.tsx
│   │   │   ├── datasets/
│   │   │   │   ├── upload-form.tsx
│   │   │   │   ├── quality-report.tsx
│   │   │   │   └── column-profile.tsx
│   │   │   ├── experiments/
│   │   │   │   ├── leaderboard.tsx
│   │   │   │   └── run-details.tsx
│   │   │   ├── models/
│   │   │   │   ├── shap-chart.tsx
│   │   │   │   └── model-card.tsx
│   │   │   ├── predictions/
│   │   │   │   ├── forecast-chart.tsx
│   │   │   │   └── prediction-table.tsx
│   │   │   ├── copilot/
│   │   │   │   ├── chat-panel.tsx
│   │   │   │   └── message-bubble.tsx
│   │   │   └── drift/
│   │   │       ├── drift-timeline.tsx
│   │   │       └── drift-alert.tsx
│   │   │
│   │   ├── lib/
│   │   │   ├── api-client.ts            # Typed fetch wrapper
│   │   │   └── utils.ts
│   │   │
│   │   └── hooks/
│   │       ├── use-project.ts
│   │       ├── use-experiment.ts
│   │       └── use-copilot.ts
│   │
│   └── tests/
│       └── e2e/
│
└── docs/
    ├── architecture.md
    ├── api-reference.md
    └── deployment.md
```

---

## Data Model Selection

**Selected: Hybrid Relational + JSONB (Data Model Suggestion 3), extended with Feature Store tables from Data Model Suggestion 4.**

### Rationale

| Criterion | Model 1 (Normalized) | Model 2 (Event-Sourced) | Model 3 (Hybrid) | Model 4 (TS-First) |
|-----------|----------------------|------------------------|-------------------|---------------------|
| Table count | ~30 | ~12 + projections | ~15 | ~21 |
| MVP velocity | Moderate | Slow (CQRS complexity) | Fast | Moderate |
| Schema flexibility | Low (ALTER TABLE) | High (JSONB events) | High (JSONB columns) | Moderate |
| Time-series support | Generic | Generic | Generic | Purpose-built |
| Audit trail | Separate table | Built-in (event store) | Separate table | Separate table |
| Team familiarity | High | Low (event sourcing) | High | Moderate |

**Decision:** Model 3 provides the best velocity-to-flexibility ratio for an MVP. Its ~15 tables are manageable, JSONB handles the inherent variability of ML hyperparameters and metrics, and GIN indexes keep query performance strong. However, Model 4's dedicated feature store tables (`feature_catalog`, `feature_views`, `feature_values_offline`, `feature_values_online`) and time-series-specific structures (`forecast_points` with prediction intervals, `forecast_accuracy` by horizon) are adopted for Phases 7 and 9 because time-series forecasting is the workbench's primary differentiator.

**What is deferred:** Model 2's event-sourcing architecture is deferred to a post-v1 compliance module. For MVP, the `audit_log` table from Model 3 provides sufficient audit capability. If regulated-industry demand materializes, event sourcing can be layered in as a separate write-path without restructuring the read models.

---

## Phase Dependency Graph

```
Phase 1 ─── Foundation & Data Layer
  │
  v
Phase 2 ─── Data Ingestion & Profiling
  │
  ├──────────────────────────────┐
  v                              v
Phase 3 ─── AutoML Training    Phase 5 ─── LLM Copilot (basic)
  │                              │
  v                              │
Phase 4 ─── Leaderboard &       │
            Explainability       │
  │                              │
  ├──────────────────────────────┘
  v
Phase 6 ─── Prediction & Batch Export
  │
  ├──────────────────────────────┐
  v                              v
Phase 7 ─── Time-Series         Phase 8 ─── Drift Detection &
            Forecasting                      Retraining
  │                              │
  v                              │
Phase 9 ─── Context-Aware       │
            Feature Engineering  │
  │                              │
  ├──────────────────────────────┘
  v
Phase 10 ── Narrative Generation & Reporting
  │
  v
Phase 11 ── REST API & Real-Time Serving
  │
  v
Phase 12 ── Production Hardening & SaaS
```

**Key constraints:**
- Phase 3 requires Phase 2 (training needs ingested data)
- Phase 4 requires Phase 3 (leaderboard needs trained models)
- Phase 5 can start in parallel with Phase 3 (copilot for data guidance is independent of training)
- Phase 6 requires Phases 4 and 5 (predictions need models + copilot integration)
- Phase 7 requires Phase 6 (time-series extends the prediction pipeline)
- Phase 8 can start in parallel with Phase 7 (drift detection is independent of time-series)
- Phase 9 requires Phase 7 (context-aware features are most valuable for time-series)
- Phase 10 requires Phases 8 and 9 (narratives cover both drift and forecasts)
- Phase 11 requires Phase 10 (API serving includes narrative responses)
- Phase 12 requires Phase 11 (production hardening wraps everything)

---

## Phase 1: Foundation & Data Layer

**Goal:** Establish the project scaffold, database schema, authentication, and core infrastructure so all subsequent phases build on a stable foundation.

**Duration:** 2-3 weeks

### Task 1.1: Project Scaffold & Docker Compose

**What:** Create the monorepo structure with backend (FastAPI), frontend (Next.js), and Docker Compose for local development. Configure PostgreSQL 16, Redis 7, and MinIO containers.

**Design:**

```python
# backend/app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.api.router import api_router

app = FastAPI(
    title="Predictive Analytics Workbench",
    version="0.1.0",
    docs_url="/api/docs",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(api_router, prefix="/api/v1")
```

```python
# backend/app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str = "postgresql+asyncpg://paw:paw@localhost:5432/paw"
    redis_url: str = "redis://localhost:6379/0"
    s3_endpoint: str = "http://localhost:9000"
    s3_bucket: str = "paw-artifacts"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    anthropic_api_key: str = ""
    cors_origins: list[str] = ["http://localhost:3000"]
    jwt_secret: str = "change-me-in-production"
    jwt_algorithm: str = "HS256"
    jwt_expire_minutes: int = 1440  # 24 hours

    class Config:
        env_file = ".env"

settings = Settings()
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: paw
      POSTGRES_USER: paw
      POSTGRES_PASSWORD: paw
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data

  backend:
    build: ./backend
    ports:
      - "8000:8000"
    depends_on:
      - postgres
      - redis
      - minio
    env_file: .env
    volumes:
      - ./backend:/app

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend
    volumes:
      - ./frontend:/app

volumes:
  pgdata:
  minio_data:
```

**Testing:**
- `docker compose up` starts all services without errors
- `curl http://localhost:8000/api/docs` returns OpenAPI spec
- `curl http://localhost:3000` returns the Next.js page shell
- PostgreSQL accepts connections on port 5432
- MinIO console accessible at port 9001

### Task 1.2: Database Schema & Migrations

**What:** Implement the Hybrid Relational + JSONB schema (Model 3) using Alembic migrations. Create all 15 core tables: organisations, users, organisation_members, projects, datasets, feature_sets, experiments, runs, models, model_explanations, prediction_jobs, drift_monitors, drift_events, copilot_sessions, audit_log.

**Design:**

```python
# backend/app/models/project.py
from sqlalchemy import Column, String, Text, ForeignKey, Index
from sqlalchemy.dialects.postgresql import UUID, JSONB, TIMESTAMP
from sqlalchemy.sql import func
import uuid

from app.database import Base

class Project(Base):
    __tablename__ = "projects"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    organisation_id = Column(UUID(as_uuid=True), ForeignKey("organisations.id", ondelete="CASCADE"), nullable=False)
    name = Column(String(255), nullable=False)
    slug = Column(String(100), nullable=False)
    description = Column(Text)
    problem_type = Column(String(50), nullable=False)  # classification, regression, time_series
    domain_context = Column(Text)  # NL description for LLM feature generation
    status = Column(String(50), nullable=False, default="draft")
    config = Column(JSONB, nullable=False, default={})
    created_by = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    created_at = Column(TIMESTAMP(timezone=True), server_default=func.now(), nullable=False)
    updated_at = Column(TIMESTAMP(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)

    __table_args__ = (
        Index("idx_projects_org", "organisation_id"),
        Index("idx_projects_status", "status"),
        {"schema": None},
    )
```

```python
# backend/app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
from app.config import settings

engine = create_async_engine(settings.database_url, echo=False)
async_session = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session
```

**Testing:**
- `alembic upgrade head` runs all migrations without errors on a clean database
- `alembic downgrade base` + `alembic upgrade head` round-trips cleanly
- All 15 tables exist with correct columns, indexes, and foreign keys: `SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'` returns exactly the expected set
- UUID default generation works: `INSERT INTO organisations (name, slug) VALUES ('Test', 'test') RETURNING id` returns a valid UUID
- JSONB columns accept valid JSON and reject invalid: insert `{"key": "value"}` succeeds; insert `'not json'` fails
- Foreign key cascades work: deleting an organisation cascades to its projects

### Task 1.3: Authentication & Multi-Tenancy

**What:** Implement JWT-based authentication with email/password registration, login, and organisation membership. Support local auth initially with SSO extension points.

**Design:**

```python
# backend/app/api/v1/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.schemas.auth import RegisterRequest, LoginRequest, TokenResponse
from app.services.auth_service import AuthService

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/register", response_model=TokenResponse, status_code=201)
async def register(body: RegisterRequest, db: AsyncSession = Depends(get_db)):
    service = AuthService(db)
    user = await service.register(body.email, body.display_name, body.password)
    token = service.create_token(user.id)
    return TokenResponse(access_token=token, token_type="bearer")

@router.post("/login", response_model=TokenResponse)
async def login(body: LoginRequest, db: AsyncSession = Depends(get_db)):
    service = AuthService(db)
    user = await service.authenticate(body.email, body.password)
    if not user:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="Invalid credentials")
    token = service.create_token(user.id)
    return TokenResponse(access_token=token, token_type="bearer")
```

```python
# backend/app/services/auth_service.py
from datetime import datetime, timedelta, timezone
from passlib.context import CryptContext
import jwt
from app.config import settings

class AuthService:
    pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

    def __init__(self, db):
        self.db = db

    async def register(self, email: str, display_name: str, password: str):
        hashed = self.pwd_context.hash(password)
        user = User(email=email, display_name=display_name, password_hash=hashed)
        self.db.add(user)
        await self.db.commit()
        await self.db.refresh(user)
        # Auto-create a personal organisation
        org = Organisation(name=f"{display_name}'s Workspace", slug=self._slugify(display_name))
        self.db.add(org)
        await self.db.commit()
        member = OrganisationMember(organisation_id=org.id, user_id=user.id, role="owner")
        self.db.add(member)
        await self.db.commit()
        return user

    def create_token(self, user_id) -> str:
        expire = datetime.now(timezone.utc) + timedelta(minutes=settings.jwt_expire_minutes)
        payload = {"sub": str(user_id), "exp": expire}
        return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)
```

**Testing:**
- POST `/api/v1/auth/register` with valid email/password returns 201 with JWT token
- POST `/api/v1/auth/register` with duplicate email returns 409
- POST `/api/v1/auth/login` with correct credentials returns 200 with token
- POST `/api/v1/auth/login` with wrong password returns 401
- JWT token decodes correctly and contains `sub` (user_id) and `exp` claims
- Registration auto-creates a personal organisation and owner membership
- Protected endpoints return 401 without Authorization header
- Protected endpoints return 401 with expired token
- Password is stored as bcrypt hash, not plaintext

### Task 1.4: Frontend Shell & Navigation

**What:** Set up the Next.js 15 application with Tailwind CSS, shadcn/ui component library, application layout (sidebar, header), and routing structure. Implement the project list page as the first functional view.

**Design:**

```tsx
// frontend/src/app/layout.tsx
import { Inter } from "next/font/google";
import "./globals.css";
import { Sidebar } from "@/components/layout/sidebar";
import { Header } from "@/components/layout/header";

const inter = Inter({ subsets: ["latin"] });

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <div className="flex h-screen">
          <Sidebar />
          <div className="flex-1 flex flex-col overflow-hidden">
            <Header />
            <main className="flex-1 overflow-y-auto p-6 bg-gray-50">
              {children}
            </main>
          </div>
        </div>
      </body>
    </html>
  );
}
```

```tsx
// frontend/src/components/layout/sidebar.tsx
"use client";
import Link from "next/link";
import { usePathname } from "next/navigation";
import { BarChart3, FolderOpen, Settings } from "lucide-react";

const navItems = [
  { label: "Projects", href: "/projects", icon: FolderOpen },
  { label: "Analytics", href: "/analytics", icon: BarChart3 },
  { label: "Settings", href: "/settings", icon: Settings },
];

export function Sidebar() {
  const pathname = usePathname();
  return (
    <aside className="w-64 bg-white border-r border-gray-200 flex flex-col">
      <div className="p-4 border-b border-gray-200">
        <h1 className="text-lg font-semibold">PAW</h1>
        <p className="text-xs text-gray-500">Predictive Analytics Workbench</p>
      </div>
      <nav className="flex-1 p-4 space-y-1">
        {navItems.map((item) => (
          <Link
            key={item.href}
            href={item.href}
            className={`flex items-center gap-3 px-3 py-2 rounded-md text-sm
              ${pathname.startsWith(item.href) ? "bg-blue-50 text-blue-700" : "text-gray-700 hover:bg-gray-50"}`}
          >
            <item.icon className="w-4 h-4" />
            {item.label}
          </Link>
        ))}
      </nav>
    </aside>
  );
}
```

**Testing:**
- `npm run build` completes without TypeScript errors
- The landing page renders with sidebar and header
- Navigation links highlight the active route
- The project list page loads and displays a "Create Project" button
- Responsive layout collapses sidebar on mobile viewports (< 768px)
- Dark mode toggle (if implemented) switches CSS variables correctly
- Login page renders and submits credentials to the API

---

## Phase 2: Data Ingestion & Profiling

**Goal:** Allow users to upload CSV files, connect to cloud warehouses, and automatically profile datasets with column statistics and data quality assessments.

**Duration:** 2-3 weeks

### Task 2.1: CSV Upload & Storage

**What:** Implement file upload endpoint that accepts CSV files (up to 500MB), stores them in MinIO/S3, creates a `datasets` record, and returns the dataset ID for downstream processing.

**Design:**

```python
# backend/app/api/v1/datasets.py
from fastapi import APIRouter, UploadFile, File, Depends, HTTPException
from app.services.dataset_service import DatasetService

router = APIRouter(prefix="/projects/{project_id}/datasets", tags=["datasets"])

@router.post("/upload", status_code=201)
async def upload_csv(
    project_id: str,
    file: UploadFile = File(...),
    db: AsyncSession = Depends(get_db),
    user=Depends(get_current_user),
):
    if not file.filename.endswith(".csv"):
        raise HTTPException(400, "Only CSV files are supported")
    if file.size and file.size > 500 * 1024 * 1024:
        raise HTTPException(413, "File exceeds 500MB limit")

    service = DatasetService(db)
    dataset = await service.ingest_csv(project_id, file, user.id)
    return {"id": str(dataset.id), "name": dataset.name, "status": "profiling"}
```

```python
# backend/app/services/dataset_service.py
import pandas as pd
from io import BytesIO
from app.services.storage_service import StorageService

class DatasetService:
    def __init__(self, db):
        self.db = db
        self.storage = StorageService()

    async def ingest_csv(self, project_id: str, file: UploadFile, user_id: str):
        # Read into memory for small files; stream to disk for large files
        content = await file.read()
        s3_path = f"datasets/{project_id}/{file.filename}"
        await self.storage.upload(s3_path, content)

        # Quick row/column count
        df = pd.read_csv(BytesIO(content), nrows=0)
        row_count = sum(1 for _ in BytesIO(content)) - 1  # Subtract header

        dataset = Dataset(
            project_id=project_id,
            name=file.filename,
            source_type="csv_upload",
            row_count=row_count,
            column_count=len(df.columns),
            file_path=s3_path,
            file_format="csv",
            created_by=user_id,
        )
        self.db.add(dataset)
        await self.db.commit()

        # Trigger async profiling
        from app.tasks.training_tasks import profile_dataset
        profile_dataset.delay(str(dataset.id))

        return dataset
```

**Testing:**
- Upload a 100-row CSV returns 201 with dataset ID
- Upload a non-CSV file returns 400
- Upload a file exceeding 500MB returns 413
- The CSV file is retrievable from MinIO at the expected path
- A `datasets` record exists with correct `row_count`, `column_count`, `file_path`
- The profiling Celery task is enqueued (visible in Redis queue)
- Upload with invalid project_id returns 404
- Upload without authentication returns 401

### Task 2.2: Automated Data Profiling

**What:** Build a Celery task that reads the uploaded dataset, computes per-column statistics (null rate, unique count, mean/std/min/max for numerics, cardinality for categoricals), and writes the profile to `datasets.schema_info` JSONB.

**Design:**

```python
# backend/app/ml/data_profiler.py
import pandas as pd
import numpy as np
from typing import Any

class DataProfiler:
    def profile(self, df: pd.DataFrame) -> list[dict[str, Any]]:
        columns = []
        for col in df.columns:
            profile = {
                "name": col,
                "dtype": str(df[col].dtype),
                "nullable": bool(df[col].isnull().any()),
                "null_rate": round(float(df[col].isnull().mean()), 4),
                "unique_count": int(df[col].nunique()),
            }
            if pd.api.types.is_numeric_dtype(df[col]):
                desc = df[col].describe()
                profile["stats"] = {
                    "mean": round(float(desc["mean"]), 4) if not np.isnan(desc["mean"]) else None,
                    "std": round(float(desc["std"]), 4) if not np.isnan(desc["std"]) else None,
                    "min": float(desc["min"]),
                    "max": float(desc["max"]),
                    "p25": float(desc["25%"]),
                    "p75": float(desc["75%"]),
                }
            elif pd.api.types.is_categorical_dtype(df[col]) or df[col].dtype == "object":
                top = df[col].value_counts().head(10)
                profile["stats"] = {
                    "cardinality": int(df[col].nunique()),
                    "top_values": [{"value": str(v), "count": int(c)} for v, c in top.items()],
                }
            elif pd.api.types.is_datetime64_any_dtype(df[col]):
                profile["stats"] = {
                    "min": str(df[col].min()),
                    "max": str(df[col].max()),
                }
            columns.append(profile)
        return columns
```

```python
# backend/app/ml/data_profiler.py (continued)
class QualityChecker:
    def check(self, df: pd.DataFrame, target_column: str | None = None) -> dict:
        issues = []
        # High null rate check
        for col in df.columns:
            null_rate = df[col].isnull().mean()
            if null_rate > 0.3:
                issues.append({
                    "type": "high_null_rate",
                    "severity": "warning",
                    "column": col,
                    "null_rate": round(float(null_rate), 4),
                })
        # Class imbalance check
        if target_column and target_column in df.columns:
            if df[target_column].dtype == "object" or df[target_column].nunique() < 10:
                dist = df[target_column].value_counts(normalize=True)
                if dist.min() < 0.1:
                    issues.append({
                        "type": "class_imbalance",
                        "severity": "warning",
                        "detail": f"Target '{target_column}' minority class is {dist.min():.1%}",
                    })
        # Target leakage check (columns highly correlated with target)
        if target_column and target_column in df.columns:
            for col in df.select_dtypes(include=[np.number]).columns:
                if col != target_column and df[col].corr(df[target_column].astype(float)) > 0.95:
                    issues.append({
                        "type": "potential_leakage",
                        "severity": "critical",
                        "column": col,
                        "detail": f"Column '{col}' has >0.95 correlation with target",
                    })
        overall_score = max(0, 1.0 - len(issues) * 0.1)
        return {"overall_score": round(overall_score, 2), "issues": issues}
```

**Testing:**
- Profile a dataset with 5 numeric, 3 categorical, and 1 datetime column: verify each column type produces the correct stats structure
- Null rate computation: a column with 20/100 nulls reports `null_rate: 0.2`
- Numeric stats: verify mean, std, min, max, p25, p75 match pandas `describe()` output
- Categorical stats: verify cardinality and top_values match `value_counts()` output
- Quality check with 40% null column flags `high_null_rate` warning
- Quality check with 95/5 class split flags `class_imbalance` warning
- Quality check with a leakage column (copied target with noise) flags `potential_leakage` critical
- Profile of a 1M-row dataset completes in under 30 seconds
- Profile result is valid JSONB stored in `datasets.schema_info`

### Task 2.3: Data Preview & Quality Report UI

**What:** Build frontend components to display the dataset column profile (data types, null rates, distributions) and the quality report (issues with severity badges, recommendations).

**Design:**

```tsx
// frontend/src/components/datasets/quality-report.tsx
"use client";

interface QualityIssue {
  type: string;
  severity: "info" | "warning" | "critical";
  column?: string;
  detail?: string;
  null_rate?: number;
}

interface QualityReportProps {
  score: number;
  issues: QualityIssue[];
}

const severityColors = {
  info: "bg-blue-100 text-blue-800",
  warning: "bg-yellow-100 text-yellow-800",
  critical: "bg-red-100 text-red-800",
};

export function QualityReport({ score, issues }: QualityReportProps) {
  return (
    <div className="space-y-4">
      <div className="flex items-center gap-4">
        <div className="text-3xl font-bold">{(score * 100).toFixed(0)}%</div>
        <div className="text-sm text-gray-500">Data Quality Score</div>
      </div>
      {issues.length === 0 ? (
        <p className="text-green-600 text-sm">No quality issues detected.</p>
      ) : (
        <ul className="space-y-2">
          {issues.map((issue, i) => (
            <li key={i} className="flex items-start gap-2 text-sm">
              <span className={`px-2 py-0.5 rounded text-xs font-medium ${severityColors[issue.severity]}`}>
                {issue.severity}
              </span>
              <span>{issue.detail || `${issue.type} on column "${issue.column}"`}</span>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

**Testing:**
- Quality report renders with correct score percentage
- Critical issues display red badges; warnings display yellow; info displays blue
- A dataset with zero issues shows "No quality issues detected" message
- Column profile table shows all columns with correct data types
- Numeric columns show histogram sparkline or min/max range
- Categorical columns show cardinality count
- The component handles empty datasets (0 rows) gracefully
- The component handles datasets with 100+ columns without layout overflow

### Task 2.4: Cloud Warehouse Connectors (Snowflake, BigQuery)

**What:** Implement data source connection management. Users register a warehouse connection (credentials stored as a reference to a secrets manager, never inline), run a preview query, and import query results as a dataset.

**Design:**

```python
# backend/app/services/dataset_service.py (warehouse extension)
from sqlalchemy import text as sql_text
import snowflake.connector
from google.cloud import bigquery

class WarehouseConnector:
    async def preview(self, source_type: str, config: dict, query: str, limit: int = 100) -> pd.DataFrame:
        if source_type == "snowflake":
            conn = snowflake.connector.connect(
                account=config["account"],
                user=config["user"],
                password=self._resolve_credential(config["credentials_ref"]),
                warehouse=config["warehouse"],
                database=config["database"],
                schema=config.get("schema", "PUBLIC"),
            )
            cursor = conn.cursor()
            cursor.execute(f"SELECT * FROM ({query}) LIMIT {limit}")
            df = cursor.fetch_pandas_all()
            conn.close()
            return df
        elif source_type == "bigquery":
            client = bigquery.Client()
            df = client.query(f"SELECT * FROM ({query}) LIMIT {limit}").to_dataframe()
            return df
        else:
            raise ValueError(f"Unsupported source type: {source_type}")

    def _resolve_credential(self, ref: str) -> str:
        # In production: resolve from AWS Secrets Manager, HashiCorp Vault, etc.
        # For development: read from environment variables
        import os
        return os.environ.get(ref, "")
```

**Testing:**
- Register a Snowflake connection with valid credentials: connection test returns success
- Register with invalid credentials: returns a clear error message (not a stack trace)
- Preview query returns the correct columns and row count (capped at limit)
- Import query results creates a dataset record and stores data in S3
- Credentials are never logged or stored in the database (only `credentials_ref`)
- BigQuery connection with service account JSON works
- Connection timeout after 30 seconds for unreachable warehouses
- SQL injection prevention: parameterized queries only

---

## Phase 3: AutoML Training Engine

**Goal:** Implement the core model training pipeline using AutoGluon. Users select a dataset and target column; the system trains multiple models, evaluates them, and persists results.

**Duration:** 3-4 weeks

### Task 3.1: AutoGluon Tabular Integration

**What:** Wrap AutoGluon-Tabular with a service layer that accepts a dataset ID and training configuration, runs AutoGluon `TabularPredictor.fit()`, and stores the resulting models as artifacts in S3.

**Design:**

```python
# backend/app/ml/autogluon_tabular.py
from autogluon.tabular import TabularPredictor
import pandas as pd
import tempfile
import shutil
from app.services.storage_service import StorageService

class AutoGluonTabularTrainer:
    def __init__(self):
        self.storage = StorageService()

    def train(
        self,
        df: pd.DataFrame,
        target_column: str,
        problem_type: str,  # "binary", "multiclass", "regression"
        time_limit: int = 3600,
        presets: str = "medium_quality",
    ) -> dict:
        with tempfile.TemporaryDirectory() as tmpdir:
            predictor = TabularPredictor(
                label=target_column,
                problem_type=problem_type,
                eval_metric=self._default_metric(problem_type),
                path=tmpdir,
            )
            predictor.fit(
                train_data=df,
                time_limit=time_limit,
                presets=presets,
            )
            leaderboard = predictor.leaderboard(extra_info=True)
            results = []
            for _, row in leaderboard.iterrows():
                results.append({
                    "algorithm": row["model"],
                    "score": float(row["score_val"]),
                    "fit_time": float(row["fit_time"]),
                    "pred_time": float(row["pred_time_val"]),
                })
            # Upload model artifact
            artifact_path = f"models/{target_column}/{predictor.path}"
            self.storage.upload_directory(tmpdir, artifact_path)

            return {
                "artifact_path": artifact_path,
                "leaderboard": results,
                "best_model": predictor.model_best,
                "best_score": float(predictor.info()["best_model_score_val"]),
            }

    def _default_metric(self, problem_type: str) -> str:
        return {
            "binary": "f1",
            "multiclass": "accuracy",
            "regression": "root_mean_squared_error",
        }.get(problem_type, "auto")
```

**Testing:**
- Train on the Iris dataset (classification): returns leaderboard with 3+ models
- Train on a regression dataset (Boston housing equivalent): returns RMSE metric
- Best model is correctly identified as the model with highest validation score
- Model artifact is uploaded to S3 and can be downloaded
- Training with `time_limit=60` completes within 90 seconds (allowing overhead)
- Training with an invalid target column raises a clear error
- Training with a dataset containing only null values in the target column raises an error
- The leaderboard DataFrame contains columns: model, score_val, fit_time, pred_time_val

### Task 3.2: Experiment & Run Management

**What:** Implement the experiment lifecycle: create an experiment (selecting dataset, target, problem type), trigger training as a Celery task, track run status in real-time, and store per-run metrics in JSONB.

**Design:**

```python
# backend/app/tasks/training_tasks.py
from celery import shared_task
from app.ml.autogluon_tabular import AutoGluonTabularTrainer
from app.database import get_sync_session

@shared_task(bind=True, max_retries=1, soft_time_limit=7200)
def train_experiment(self, experiment_id: str):
    db = get_sync_session()
    experiment = db.query(Experiment).get(experiment_id)
    experiment.status = "running"
    db.commit()

    try:
        dataset = db.query(Dataset).get(experiment.dataset_id)
        df = load_dataframe_from_s3(dataset.file_path)

        config = experiment.config
        trainer = AutoGluonTabularTrainer()
        results = trainer.train(
            df=df,
            target_column=config["target_column"],
            problem_type=config["problem_type"],
            time_limit=config.get("time_limit_seconds", 3600),
            presets=config.get("presets", "medium_quality"),
        )

        # Create run records for each model in the leaderboard
        for i, model_result in enumerate(results["leaderboard"]):
            run = Run(
                experiment_id=experiment_id,
                run_number=i + 1,
                algorithm=model_result["algorithm"],
                status="completed",
                hyperparameters={},  # Populated from AutoGluon info
                metrics={"score": model_result["score"], "fit_time": model_result["fit_time"]},
                artifact_path=results["artifact_path"],
                duration_seconds=model_result["fit_time"],
            )
            db.add(run)

        experiment.status = "completed"
        experiment.summary = {
            "total_runs": len(results["leaderboard"]),
            "best_algorithm": results["best_model"],
            "best_score": results["best_score"],
        }
        db.commit()

    except Exception as e:
        experiment.status = "failed"
        experiment.summary = {"error": str(e)}
        db.commit()
        raise
```

```python
# backend/app/api/v1/experiments.py
@router.post("/", status_code=201)
async def create_experiment(
    project_id: str,
    body: CreateExperimentRequest,
    db: AsyncSession = Depends(get_db),
    user=Depends(get_current_user),
):
    experiment = Experiment(
        project_id=project_id,
        name=body.name,
        dataset_id=body.dataset_id,
        config={
            "target_column": body.target_column,
            "problem_type": body.problem_type,
            "time_limit_seconds": body.time_limit_seconds or 3600,
            "presets": body.presets or "medium_quality",
        },
        status="created",
        created_by=user.id,
    )
    db.add(experiment)
    await db.commit()

    # Trigger async training
    train_experiment.delay(str(experiment.id))

    return {"id": str(experiment.id), "status": "created"}
```

**Testing:**
- Create an experiment via API: returns 201 with experiment ID
- Experiment status transitions: created -> running -> completed
- A failed training (bad target column) transitions to: created -> running -> failed with error message
- Each model in the AutoGluon leaderboard has a corresponding `runs` record
- Run metrics JSONB contains at minimum: score and fit_time
- Experiment summary JSONB contains total_runs, best_algorithm, best_score
- Concurrent experiment creation does not cause database deadlocks
- Celery task retries once on transient failure
- GET `/experiments/{id}` returns current status during training (polling)

### Task 3.3: Training Progress & Status UI

**What:** Build the experiment creation form (dataset selector, target column dropdown, problem type radio, time limit slider) and a real-time training progress view that polls for status updates.

**Design:**

```tsx
// frontend/src/components/experiments/experiment-form.tsx
"use client";
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Select } from "@/components/ui/select";

interface ExperimentFormProps {
  projectId: string;
  datasets: Array<{ id: string; name: string; columns: string[] }>;
  onCreated: (experimentId: string) => void;
}

export function ExperimentForm({ projectId, datasets, onCreated }: ExperimentFormProps) {
  const [datasetId, setDatasetId] = useState("");
  const [targetColumn, setTargetColumn] = useState("");
  const [problemType, setProblemType] = useState<"classification" | "regression">("classification");
  const [timeLimit, setTimeLimit] = useState(600);
  const [isSubmitting, setIsSubmitting] = useState(false);

  const selectedDataset = datasets.find((d) => d.id === datasetId);

  const handleSubmit = async () => {
    setIsSubmitting(true);
    const res = await fetch(`/api/v1/projects/${projectId}/experiments`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        name: `${targetColumn}_${problemType}_${Date.now()}`,
        dataset_id: datasetId,
        target_column: targetColumn,
        problem_type: problemType,
        time_limit_seconds: timeLimit,
      }),
    });
    const data = await res.json();
    onCreated(data.id);
    setIsSubmitting(false);
  };

  return (
    <div className="space-y-6 max-w-lg">
      <div>
        <label className="block text-sm font-medium mb-1">Dataset</label>
        <Select value={datasetId} onChange={setDatasetId}
          options={datasets.map((d) => ({ value: d.id, label: d.name }))} />
      </div>
      {selectedDataset && (
        <div>
          <label className="block text-sm font-medium mb-1">Target Column</label>
          <Select value={targetColumn} onChange={setTargetColumn}
            options={selectedDataset.columns.map((c) => ({ value: c, label: c }))} />
        </div>
      )}
      <div>
        <label className="block text-sm font-medium mb-1">Problem Type</label>
        <div className="flex gap-4">
          {["classification", "regression"].map((t) => (
            <label key={t} className="flex items-center gap-2">
              <input type="radio" value={t} checked={problemType === t}
                onChange={() => setProblemType(t as any)} />
              <span className="text-sm capitalize">{t}</span>
            </label>
          ))}
        </div>
      </div>
      <div>
        <label className="block text-sm font-medium mb-1">
          Time Limit: {timeLimit}s ({Math.round(timeLimit / 60)} min)
        </label>
        <input type="range" min={60} max={7200} step={60} value={timeLimit}
          onChange={(e) => setTimeLimit(Number(e.target.value))}
          className="w-full" />
      </div>
      <Button onClick={handleSubmit} disabled={!datasetId || !targetColumn || isSubmitting}>
        {isSubmitting ? "Starting..." : "Train Models"}
      </Button>
    </div>
  );
}
```

**Testing:**
- Dataset dropdown shows all datasets for the current project
- Selecting a dataset populates the target column dropdown with column names
- Problem type defaults to classification
- Time limit slider ranges from 1 minute to 2 hours
- Submit button is disabled until dataset and target column are selected
- Clicking "Train Models" shows a loading state and navigates to progress view
- Progress view shows real-time status (polling every 3 seconds)
- Progress view shows elapsed time and estimated completion
- Completed experiments show a link to the leaderboard

---

## Phase 4: Model Leaderboard & Explainability

**Goal:** Display trained models in a ranked leaderboard, compute SHAP explanations, and allow users to promote the best model to the model registry.

**Duration:** 2-3 weeks

### Task 4.1: Model Leaderboard

**What:** Build an API endpoint and UI component that displays all runs for an experiment ranked by the primary metric. Include algorithm name, score, training time, and model size. Highlight the best model.

**Design:**

```python
# backend/app/api/v1/experiments.py
@router.get("/{experiment_id}/leaderboard")
async def get_leaderboard(
    experiment_id: str,
    db: AsyncSession = Depends(get_db),
):
    experiment = await db.get(Experiment, experiment_id)
    if not experiment:
        raise HTTPException(404, "Experiment not found")

    runs = await db.execute(
        select(Run)
        .where(Run.experiment_id == experiment_id)
        .where(Run.status == "completed")
        .order_by(Run.metrics["score"].as_float().desc())
    )
    runs = runs.scalars().all()

    return {
        "experiment_id": experiment_id,
        "primary_metric": experiment.config.get("primary_metric", "score"),
        "runs": [
            {
                "id": str(r.id),
                "rank": i + 1,
                "algorithm": r.algorithm,
                "score": r.metrics.get("score"),
                "fit_time": r.duration_seconds,
                "is_best": i == 0,
            }
            for i, r in enumerate(runs)
        ],
    }
```

```tsx
// frontend/src/components/experiments/leaderboard.tsx
"use client";

interface LeaderboardEntry {
  id: string;
  rank: number;
  algorithm: string;
  score: number;
  fit_time: number;
  is_best: boolean;
}

export function Leaderboard({ runs, primaryMetric }: { runs: LeaderboardEntry[]; primaryMetric: string }) {
  return (
    <table className="w-full text-sm">
      <thead>
        <tr className="border-b text-left">
          <th className="py-2 pr-4">Rank</th>
          <th className="py-2 pr-4">Algorithm</th>
          <th className="py-2 pr-4">{primaryMetric}</th>
          <th className="py-2 pr-4">Training Time</th>
          <th className="py-2">Actions</th>
        </tr>
      </thead>
      <tbody>
        {runs.map((run) => (
          <tr key={run.id} className={`border-b ${run.is_best ? "bg-green-50" : ""}`}>
            <td className="py-2 pr-4">
              {run.is_best ? "🏆 " : ""}{run.rank}
            </td>
            <td className="py-2 pr-4 font-mono text-xs">{run.algorithm}</td>
            <td className="py-2 pr-4 font-medium">{run.score.toFixed(4)}</td>
            <td className="py-2 pr-4 text-gray-500">{run.fit_time.toFixed(1)}s</td>
            <td className="py-2">
              <button className="text-blue-600 hover:underline text-xs">
                Promote to Registry
              </button>
            </td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

**Testing:**
- Leaderboard API returns runs sorted by score descending
- The top-ranked run has `is_best: true`; all others have `is_best: false`
- An experiment with 0 completed runs returns an empty leaderboard
- Algorithm names match AutoGluon model names (e.g., `LightGBM`, `XGBoost`, `WeightedEnsemble_L2`)
- UI highlights the best model row with a green background
- Each row has a "Promote to Registry" action

### Task 4.2: SHAP Explanations

**What:** After training completes, compute global SHAP values (mean |SHAP| per feature) for the best model. Store results in the `model_explanations` table. Provide API endpoint and chart component.

**Design:**

```python
# backend/app/ml/shap_explainer.py
import shap
import pandas as pd
import numpy as np

class ShapExplainer:
    def explain_global(self, predictor, df: pd.DataFrame, max_samples: int = 500) -> dict:
        """Compute global SHAP values for the best model in a predictor."""
        sample = df.sample(n=min(max_samples, len(df)), random_state=42)
        # Use AutoGluon's built-in feature importance as fallback
        try:
            explainer = shap.TreeExplainer(predictor._trainer.load_model(predictor.model_best))
            shap_values = explainer.shap_values(sample)
            if isinstance(shap_values, list):
                shap_values = shap_values[1]  # Binary classification: take positive class
            mean_abs = np.abs(shap_values).mean(axis=0)
            features = []
            for i, col in enumerate(sample.columns):
                features.append({
                    "name": col,
                    "mean_abs_shap": round(float(mean_abs[i]), 6),
                    "rank": 0,  # Will be set after sorting
                })
            features.sort(key=lambda x: x["mean_abs_shap"], reverse=True)
            for i, f in enumerate(features):
                f["rank"] = i + 1
            base_value = float(explainer.expected_value)
            if isinstance(base_value, np.ndarray):
                base_value = float(base_value[1])
        except Exception:
            # Fallback: use AutoGluon's feature importance
            importance = predictor.feature_importance(df)
            features = [
                {"name": row["feature"], "mean_abs_shap": float(row["importance"]), "rank": i + 1}
                for i, (_, row) in enumerate(importance.iterrows())
            ]
            base_value = 0.0

        return {
            "base_value": base_value,
            "features": features[:20],  # Top 20 features
        }
```

**Testing:**
- Global SHAP for a 10-feature classification model returns 10 features ranked by importance
- Features are sorted by `mean_abs_shap` descending
- `rank` values are sequential: 1, 2, 3, ...
- `base_value` is a valid float
- SHAP computation on a 10K-row dataset completes in under 60 seconds
- Fallback to AutoGluon `feature_importance` when TreeExplainer fails (e.g., ensemble models)
- Results are stored in `model_explanations` with `explanation_type: "global_shap"`
- SHAP chart component renders a horizontal bar chart with feature names and values

### Task 4.3: Model Promotion to Registry

**What:** Allow users to promote a run's model to the model registry with a version number and lifecycle stage (staging, production). Implement the promotion API and registry list view.

**Design:**

```python
# backend/app/api/v1/models.py
@router.post("/promote", status_code=201)
async def promote_model(
    project_id: str,
    body: PromoteModelRequest,  # run_id, model_name, description
    db: AsyncSession = Depends(get_db),
    user=Depends(get_current_user),
):
    run = await db.get(Run, body.run_id)
    if not run or run.status != "completed":
        raise HTTPException(400, "Run must be completed to promote")

    # Determine next version number
    latest = await db.execute(
        select(Model)
        .where(Model.project_id == project_id, Model.name == body.model_name)
        .order_by(Model.version.desc())
        .limit(1)
    )
    latest_model = latest.scalar_one_or_none()
    next_version = (latest_model.version + 1) if latest_model else 1

    model = Model(
        project_id=project_id,
        name=body.model_name,
        version=next_version,
        run_id=body.run_id,
        stage="staging",
        description=body.description,
        artifact_path=run.artifact_path,
        framework="autogluon",
        performance=run.metrics,
        promoted_by=user.id,
        promoted_at=func.now(),
    )
    db.add(model)
    await db.commit()

    # Audit log
    audit = AuditLog(
        organisation_id=project.organisation_id,
        user_id=user.id,
        action="model.promoted",
        resource_type="model",
        resource_id=model.id,
        details={"version": next_version, "stage": "staging", "run_id": str(body.run_id)},
    )
    db.add(audit)
    await db.commit()

    return {"id": str(model.id), "name": body.model_name, "version": next_version, "stage": "staging"}
```

**Testing:**
- Promote a completed run: returns 201 with model name, version 1, stage "staging"
- Promote the same model name again: version auto-increments to 2
- Promote a failed run: returns 400
- Promote to production: previous production model moves to "archived"
- Audit log entry is created for each promotion
- Model registry list endpoint returns all models with version, stage, and performance metrics
- Model registry UI shows a card per model with version history

---

## Phase 5: LLM Copilot Integration

**Goal:** Integrate Claude API as a conversational copilot that guides users through the modeling workflow, explains data quality issues, and recommends model configurations.

**Duration:** 2-3 weeks

### Task 5.1: Claude API Client & Prompt Architecture

**What:** Build a reusable Claude API client with structured prompt templates for different workflow stages (onboarding, data quality review, model recommendation, result interpretation). Implement tool use for copilot actions.

**Design:**

```python
# backend/app/llm/claude_client.py
import anthropic
from app.config import settings

class ClaudeClient:
    def __init__(self):
        self.client = anthropic.Anthropic(api_key=settings.anthropic_api_key)
        self.model = "claude-sonnet-4-20250514"  # Cost-effective for copilot

    async def chat(
        self,
        messages: list[dict],
        system: str,
        tools: list[dict] | None = None,
        max_tokens: int = 2048,
    ) -> dict:
        response = self.client.messages.create(
            model=self.model,
            max_tokens=max_tokens,
            system=system,
            messages=messages,
            tools=tools or [],
        )
        return {
            "content": response.content[0].text if response.content else "",
            "tool_use": [
                {"name": block.name, "input": block.input}
                for block in response.content
                if block.type == "tool_use"
            ],
            "usage": {
                "input_tokens": response.usage.input_tokens,
                "output_tokens": response.usage.output_tokens,
            },
        }
```

```python
# backend/app/llm/prompts/copilot_onboarding.py
SYSTEM_PROMPT = """You are a predictive analytics copilot helping business users build
forecasting and prediction models without data science expertise.

Your role:
1. Understand the user's business problem and data
2. Recommend the right problem type (classification, regression, time-series)
3. Identify potential data quality issues before they affect model training
4. Explain ML concepts in plain business language (never use jargon without explanation)
5. Flag potential target leakage, class imbalance, or insufficient data

You have access to the following tools:
- analyze_dataset: Get column statistics and quality report for a dataset
- recommend_problem_type: Suggest the best problem type based on the target column
- suggest_features: Recommend feature engineering based on the domain context

Always be encouraging but honest about data limitations. If the data has serious quality
issues, explain them clearly and suggest remediation steps."""
```

```python
# backend/app/llm/tools.py
COPILOT_TOOLS = [
    {
        "name": "analyze_dataset",
        "description": "Analyze a dataset's columns, data types, null rates, and quality issues",
        "input_schema": {
            "type": "object",
            "properties": {
                "dataset_id": {"type": "string", "description": "The dataset to analyze"},
            },
            "required": ["dataset_id"],
        },
    },
    {
        "name": "recommend_problem_type",
        "description": "Recommend whether to use classification, regression, or time-series based on the target column",
        "input_schema": {
            "type": "object",
            "properties": {
                "target_column": {"type": "string"},
                "dataset_id": {"type": "string"},
            },
            "required": ["target_column", "dataset_id"],
        },
    },
    {
        "name": "start_training",
        "description": "Start training models on a dataset with specified configuration",
        "input_schema": {
            "type": "object",
            "properties": {
                "dataset_id": {"type": "string"},
                "target_column": {"type": "string"},
                "problem_type": {"type": "string", "enum": ["classification", "regression", "time_series"]},
                "time_limit_seconds": {"type": "integer", "default": 600},
            },
            "required": ["dataset_id", "target_column", "problem_type"],
        },
    },
]
```

**Testing:**
- Claude client sends a message and receives a response with content
- System prompt is included in every API call
- Tool use responses are correctly parsed into `name` and `input`
- Token usage is tracked and returned
- API key missing or invalid raises a clear configuration error
- Rate limiting (429) is handled with exponential backoff
- Messages array correctly alternates user/assistant roles

### Task 5.2: Copilot Conversation API & Persistence

**What:** Implement WebSocket or Server-Sent Events (SSE) endpoint for streaming copilot responses. Persist conversation history in the `copilot_sessions` table. Execute tool calls against the backend services.

**Design:**

```python
# backend/app/api/v1/copilot.py
from fastapi import APIRouter, Depends
from fastapi.responses import StreamingResponse

router = APIRouter(prefix="/projects/{project_id}/copilot", tags=["copilot"])

@router.post("/sessions", status_code=201)
async def create_session(
    project_id: str,
    body: CreateSessionRequest,
    db: AsyncSession = Depends(get_db),
    user=Depends(get_current_user),
):
    session = CopilotSession(
        project_id=project_id,
        user_id=user.id,
        session_type=body.session_type or "general",
        messages=[],
    )
    db.add(session)
    await db.commit()
    return {"id": str(session.id)}

@router.post("/sessions/{session_id}/messages")
async def send_message(
    project_id: str,
    session_id: str,
    body: SendMessageRequest,
    db: AsyncSession = Depends(get_db),
    user=Depends(get_current_user),
):
    session = await db.get(CopilotSession, session_id)
    # Append user message
    session.messages.append({"role": "user", "content": body.content, "timestamp": now_iso()})

    # Build Claude messages from conversation history
    claude_messages = [{"role": m["role"], "content": m["content"]} for m in session.messages]

    # Get project context for system prompt
    project = await db.get(Project, project_id)
    system = build_system_prompt(project)

    client = ClaudeClient()
    response = await client.chat(messages=claude_messages, system=system, tools=COPILOT_TOOLS)

    # Handle tool calls
    while response["tool_use"]:
        for tool_call in response["tool_use"]:
            result = await execute_tool(tool_call["name"], tool_call["input"], db, project_id)
            claude_messages.append({"role": "assistant", "content": response["content"]})
            claude_messages.append({"role": "user", "content": f"Tool result: {result}"})
        response = await client.chat(messages=claude_messages, system=system, tools=COPILOT_TOOLS)

    # Append assistant response
    session.messages.append({
        "role": "assistant",
        "content": response["content"],
        "timestamp": now_iso(),
        "tokens": response["usage"],
    })
    session.message_count = len(session.messages)
    await db.commit()

    return {"content": response["content"], "tokens": response["usage"]}
```

**Testing:**
- Create a session: returns 201 with session ID
- Send a message: returns assistant response with content
- Conversation history persists: subsequent messages include prior context
- Tool call execution: "analyze this dataset" triggers the `analyze_dataset` tool
- Tool results are incorporated into the LLM response
- Multiple tool calls in a single turn are all executed
- Session messages JSONB is updated correctly after each turn
- Token count accumulates across the session
- Empty message body returns 400

### Task 5.3: Copilot Chat UI

**What:** Build the chat panel component with streaming message display, typing indicator, and tool execution status. Integrate into the project page layout.

**Design:**

```tsx
// frontend/src/components/copilot/chat-panel.tsx
"use client";
import { useState, useRef, useEffect } from "react";
import { MessageBubble } from "./message-bubble";
import { Send } from "lucide-react";

interface Message {
  role: "user" | "assistant";
  content: string;
  timestamp: string;
}

export function ChatPanel({ projectId, sessionId }: { projectId: string; sessionId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");
  const [isLoading, setIsLoading] = useState(false);
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  const sendMessage = async () => {
    if (!input.trim() || isLoading) return;
    const userMsg: Message = { role: "user", content: input, timestamp: new Date().toISOString() };
    setMessages((prev) => [...prev, userMsg]);
    setInput("");
    setIsLoading(true);

    const res = await fetch(`/api/v1/projects/${projectId}/copilot/sessions/${sessionId}/messages`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ content: userMsg.content }),
    });
    const data = await res.json();
    setMessages((prev) => [...prev, { role: "assistant", content: data.content, timestamp: new Date().toISOString() }]);
    setIsLoading(false);
  };

  return (
    <div className="flex flex-col h-full">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg, i) => (
          <MessageBubble key={i} role={msg.role} content={msg.content} />
        ))}
        {isLoading && <div className="text-sm text-gray-400 animate-pulse">Thinking...</div>}
        <div ref={bottomRef} />
      </div>
      <div className="border-t p-4 flex gap-2">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && sendMessage()}
          placeholder="Ask about your data or models..."
          className="flex-1 px-3 py-2 border rounded-md text-sm"
        />
        <button onClick={sendMessage} disabled={isLoading}
          className="p-2 bg-blue-600 text-white rounded-md hover:bg-blue-700 disabled:opacity-50">
          <Send className="w-4 h-4" />
        </button>
      </div>
    </div>
  );
}
```

**Testing:**
- User can type a message and press Enter to send
- User message appears immediately on the right side
- Assistant response appears on the left side after API returns
- "Thinking..." indicator shows while waiting for response
- Chat auto-scrolls to the bottom when new messages arrive
- Send button is disabled while loading
- Empty messages cannot be sent
- Long messages wrap correctly without overflow
- Tool execution status appears as a system message (e.g., "Analyzing dataset...")

---

## Phase 6: Prediction & Batch Export

**Goal:** Allow users to generate predictions from promoted models, export results to CSV, and write back to connected cloud warehouses.

**Duration:** 2 weeks

### Task 6.1: Batch Prediction Pipeline

**What:** Implement a prediction job system where users select a model version and input data (new CSV or existing dataset), run batch predictions as a Celery task, and store results.

**Design:**

```python
# backend/app/services/prediction_service.py
from autogluon.tabular import TabularPredictor

class PredictionService:
    def __init__(self, db):
        self.db = db
        self.storage = StorageService()

    async def run_batch_prediction(self, job_id: str):
        job = await self.db.get(PredictionJob, job_id)
        model = await self.db.get(Model, job.model_id)

        # Load model from S3
        predictor = TabularPredictor.load(
            self.storage.download_directory(model.artifact_path)
        )

        # Load input data
        input_config = job.input_config
        if input_config.get("source") == "dataset":
            dataset = await self.db.get(Dataset, input_config["dataset_id"])
            df = pd.read_csv(self.storage.download(dataset.file_path))
        else:
            df = pd.read_csv(self.storage.download(input_config["file_path"]))

        # Generate predictions
        predictions = predictor.predict(df)
        probas = predictor.predict_proba(df) if hasattr(predictor, "predict_proba") else None

        # Build output DataFrame
        output_df = df.copy()
        output_df["_prediction"] = predictions
        if probas is not None:
            for col in probas.columns:
                output_df[f"_proba_{col}"] = probas[col]

        # Store output
        output_path = f"predictions/{job_id}/output.csv"
        self.storage.upload(output_path, output_df.to_csv(index=False).encode())

        # Update job
        job.status = "completed"
        job.output_config = {"format": "csv", "path": output_path}
        job.result_summary = {
            "row_count": len(output_df),
            "prediction_distribution": predictions.value_counts().to_dict()
                if predictions.dtype == "object" else {"mean": float(predictions.mean()), "std": float(predictions.std())},
        }
        await self.db.commit()
```

**Testing:**
- Create a batch prediction job: returns 201 with job ID
- Job status transitions: pending -> running -> completed
- Output CSV contains original columns plus `_prediction` column
- Classification outputs include `_proba_<class>` columns
- Regression outputs include a numeric prediction
- Output file is downloadable from S3
- Prediction on data with missing columns (compared to training) returns a clear error
- Prediction on 100K rows completes in under 5 minutes
- `result_summary` contains correct prediction distribution

### Task 6.2: Prediction Export & Warehouse Writeback

**What:** Implement CSV download endpoint and warehouse writeback (write prediction results back to the user's connected Snowflake/BigQuery table).

**Design:**

```python
# backend/app/api/v1/predictions.py
@router.get("/{job_id}/download")
async def download_predictions(
    job_id: str,
    db: AsyncSession = Depends(get_db),
):
    job = await db.get(PredictionJob, job_id)
    if job.status != "completed":
        raise HTTPException(400, "Prediction job is not completed")

    content = StorageService().download(job.output_config["path"])
    return StreamingResponse(
        BytesIO(content),
        media_type="text/csv",
        headers={"Content-Disposition": f"attachment; filename=predictions_{job_id}.csv"},
    )

@router.post("/{job_id}/writeback")
async def writeback_predictions(
    job_id: str,
    body: WritebackRequest,  # target_table, data_source_id
    db: AsyncSession = Depends(get_db),
):
    job = await db.get(PredictionJob, job_id)
    output_df = pd.read_csv(StorageService().download(job.output_config["path"]))

    connector = WarehouseConnector()
    await connector.writeback(
        source_type=body.source_type,
        config=body.connection_config,
        table_name=body.target_table,
        df=output_df,
    )
    return {"status": "written", "rows": len(output_df), "table": body.target_table}
```

**Testing:**
- Download endpoint returns a valid CSV file with correct Content-Disposition header
- Downloaded CSV matches the prediction output (same columns and row count)
- Writeback to Snowflake creates/replaces the target table with prediction data
- Writeback to BigQuery appends or replaces data correctly
- Writeback with invalid credentials returns a clear error
- Download on a non-completed job returns 400
- Large predictions (1M rows) stream without memory overflow

### Task 6.3: Prediction Results UI

**What:** Build the prediction results page showing prediction distribution, download button, writeback form, and a sample of predictions.

**Testing:**
- Prediction results page shows prediction distribution chart (histogram for regression, bar chart for classification)
- "Download CSV" button triggers browser download
- Sample table shows first 20 prediction rows with prediction and probability columns
- Writeback form shows connected data sources as target options
- Successful writeback shows confirmation toast

---

## Phase 7: Time-Series Forecasting

**Goal:** Extend the training engine with AutoGluon-TimeSeries and Chronos for time-series forecasting. Add purpose-built time-series dataset handling, forecast visualization with prediction intervals, and accuracy backtesting.

**Duration:** 3-4 weeks

### Task 7.1: Time-Series Dataset Handling

**What:** Extend the dataset model with time-series metadata (time column, granularity, group columns). Implement time-series data validation and automatic temporal ordering. Add the feature store tables from Data Model 4.

**Design:**

```python
# backend/app/ml/timeseries_validator.py
import pandas as pd

class TimeSeriesValidator:
    def validate(self, df: pd.DataFrame, time_column: str, group_columns: list[str] | None = None) -> dict:
        issues = []

        # Validate time column exists and is parseable
        if time_column not in df.columns:
            return {"valid": False, "issues": [{"type": "missing_time_column", "severity": "critical"}]}

        df[time_column] = pd.to_datetime(df[time_column], errors="coerce")
        null_times = df[time_column].isnull().sum()
        if null_times > 0:
            issues.append({
                "type": "unparseable_timestamps",
                "severity": "critical",
                "count": int(null_times),
            })

        # Detect granularity
        df_sorted = df.sort_values(time_column)
        diffs = df_sorted[time_column].diff().dropna()
        median_diff = diffs.median()
        granularity = self._infer_granularity(median_diff)

        # Check for gaps
        if group_columns:
            for name, group in df.groupby(group_columns):
                gaps = self._check_gaps(group[time_column], granularity)
                if gaps:
                    issues.append({"type": "time_gaps", "group": str(name), "gap_count": len(gaps)})
        else:
            gaps = self._check_gaps(df_sorted[time_column], granularity)
            if gaps:
                issues.append({"type": "time_gaps", "gap_count": len(gaps)})

        return {
            "valid": len([i for i in issues if i["severity"] == "critical"]) == 0,
            "granularity": granularity,
            "start_time": str(df[time_column].min()),
            "end_time": str(df[time_column].max()),
            "period_count": int(df[time_column].nunique()),
            "group_count": int(df[group_columns].drop_duplicates().shape[0]) if group_columns else 1,
            "issues": issues,
        }

    def _infer_granularity(self, median_diff) -> str:
        hours = median_diff.total_seconds() / 3600
        if hours < 2: return "hourly"
        if hours < 36: return "daily"
        if hours < 168 * 1.5: return "weekly"
        if hours < 744 * 1.5: return "monthly"
        return "quarterly"
```

**Testing:**
- Daily sales dataset: granularity detected as "daily"
- Monthly revenue dataset: granularity detected as "monthly"
- Dataset with 5 missing days: reports `time_gaps` issue
- Dataset with unparseable timestamps: reports critical issue, valid = false
- Group columns correctly identify unique time series (e.g., 3 products x 12 months = 3 groups)
- Hourly sensor data: granularity detected as "hourly"
- Mixed-frequency data (some daily, some weekly): reports inconsistency issue

### Task 7.2: AutoGluon-TimeSeries Training

**What:** Integrate AutoGluon-TimeSeries which ensembles statistical (ETS, ARIMA), tree-based (LightGBM), deep learning (DeepAR, TFT), and foundation model (Chronos) forecasters. Support grouped forecasting and prediction intervals.

**Design:**

```python
# backend/app/ml/autogluon_timeseries.py
from autogluon.timeseries import TimeSeriesPredictor, TimeSeriesDataFrame
import pandas as pd

class AutoGluonTimeSeriesTrainer:
    def train(
        self,
        df: pd.DataFrame,
        target_column: str,
        time_column: str,
        group_columns: list[str] | None,
        forecast_horizon: int,
        time_limit: int = 3600,
    ) -> dict:
        # Convert to AutoGluon TimeSeriesDataFrame
        if group_columns:
            item_id_col = "_item_id"
            df[item_id_col] = df[group_columns].astype(str).agg("|".join, axis=1)
        else:
            item_id_col = "_item_id"
            df[item_id_col] = "series_0"

        ts_df = TimeSeriesDataFrame.from_data_frame(
            df,
            id_column=item_id_col,
            timestamp_column=time_column,
        )

        predictor = TimeSeriesPredictor(
            target=target_column,
            prediction_length=forecast_horizon,
            eval_metric="MAPE",
        )
        predictor.fit(
            train_data=ts_df,
            time_limit=time_limit,
            presets="medium_quality",
        )

        leaderboard = predictor.leaderboard()
        predictions = predictor.predict(ts_df)

        return {
            "leaderboard": [
                {
                    "algorithm": row["model"],
                    "score": float(row["score_val"]),
                    "fit_time": float(row.get("fit_time", 0)),
                }
                for _, row in leaderboard.iterrows()
            ],
            "best_model": predictor.model_best,
            "best_score": float(leaderboard.iloc[0]["score_val"]),
            "predictions": predictions,
        }
```

**Testing:**
- Train on a single time series (100 daily points, forecast 30 days): returns 3+ model results
- Train on grouped time series (5 products x 365 days): produces per-group forecasts
- Chronos appears in the leaderboard when included in presets
- Predictions contain point forecast and prediction intervals (0.1, 0.5, 0.9 quantiles)
- MAPE metric is correctly computed against the validation holdout
- Training with fewer data points than forecast horizon raises a clear error
- Leaderboard ranks models by MAPE (lower is better)
- Model artifact is uploaded to S3 and can be reloaded for prediction

### Task 7.3: Forecast Visualization with Prediction Intervals

**What:** Build a time-series chart component that displays historical actuals, point forecasts, and shaded prediction intervals (80% and 95%). Support group-level drill-down.

**Design:**

```tsx
// frontend/src/components/predictions/forecast-chart.tsx
"use client";
import { AreaChart, Area, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer } from "recharts";

interface ForecastPoint {
  timestamp: string;
  actual?: number;
  forecast?: number;
  lower_80?: number;
  upper_80?: number;
  lower_95?: number;
  upper_95?: number;
}

export function ForecastChart({ data, targetColumn }: { data: ForecastPoint[]; targetColumn: string }) {
  return (
    <ResponsiveContainer width="100%" height={400}>
      <AreaChart data={data} margin={{ top: 10, right: 30, left: 0, bottom: 0 }}>
        <CartesianGrid strokeDasharray="3 3" />
        <XAxis dataKey="timestamp" tick={{ fontSize: 11 }} />
        <YAxis tick={{ fontSize: 11 }} />
        <Tooltip />
        {/* 95% prediction interval */}
        <Area type="monotone" dataKey="upper_95" stroke="none" fill="#dbeafe" fillOpacity={0.4}
          name="95% upper" />
        <Area type="monotone" dataKey="lower_95" stroke="none" fill="#ffffff" fillOpacity={1}
          name="95% lower" />
        {/* 80% prediction interval */}
        <Area type="monotone" dataKey="upper_80" stroke="none" fill="#93c5fd" fillOpacity={0.5}
          name="80% upper" />
        <Area type="monotone" dataKey="lower_80" stroke="none" fill="#ffffff" fillOpacity={1}
          name="80% lower" />
        {/* Actual values */}
        <Line type="monotone" dataKey="actual" stroke="#1e40af" strokeWidth={2}
          dot={false} name="Actual" />
        {/* Forecast line */}
        <Line type="monotone" dataKey="forecast" stroke="#dc2626" strokeWidth={2}
          strokeDasharray="5 5" dot={false} name="Forecast" />
      </AreaChart>
    </ResponsiveContainer>
  );
}
```

**Testing:**
- Chart renders historical data as a solid blue line
- Forecast data renders as a dashed red line continuing from the last actual
- 80% prediction interval renders as a darker shaded band
- 95% prediction interval renders as a lighter shaded band around the 80% band
- Group selector dropdown filters the chart to a single time series
- Tooltip shows actual, forecast, and interval bounds on hover
- Chart handles 1000+ data points without performance degradation
- Empty forecast data shows only the historical line
- Y-axis auto-scales to fit the data range including intervals

### Task 7.4: Forecast Accuracy Backtesting

**What:** Implement backtesting that compares forecasts against actuals as they arrive. Track MAPE, RMSE, and prediction interval coverage by forecast horizon (1-day, 7-day, 30-day).

**Design:**

```python
# backend/app/services/forecast_accuracy_service.py
class ForecastAccuracyService:
    async def backtest(self, model_id: str, forecast_job_id: str) -> dict:
        # Load forecast points with actuals
        points = await self.db.execute(
            select(ForecastPoint)
            .where(ForecastPoint.forecast_job_id == forecast_job_id)
            .where(ForecastPoint.actual_value.isnot(None))
        )
        points = points.scalars().all()

        if not points:
            return {"status": "no_actuals", "message": "No actual values received yet"}

        # Compute metrics by horizon
        results = {}
        for horizon in [1, 7, 30]:
            horizon_points = [p for p in points if self._days_ahead(p) <= horizon]
            if not horizon_points:
                continue
            actuals = [p.actual_value for p in horizon_points]
            forecasts = [p.point_forecast for p in horizon_points]
            results[f"{horizon}d"] = {
                "mape": self._mape(actuals, forecasts),
                "rmse": self._rmse(actuals, forecasts),
                "mae": self._mae(actuals, forecasts),
                "coverage_80": self._coverage(horizon_points, "80"),
                "coverage_95": self._coverage(horizon_points, "95"),
                "point_count": len(horizon_points),
            }

        # Persist accuracy record
        for horizon_key, metrics in results.items():
            accuracy = ForecastAccuracy(
                model_id=model_id,
                forecast_job_id=forecast_job_id,
                horizon_days=int(horizon_key.replace("d", "")),
                **metrics,
            )
            self.db.add(accuracy)
        await self.db.commit()

        return results
```

**Testing:**
- Backtest with 30 days of actuals: returns MAPE, RMSE, MAE for 1d, 7d, 30d horizons
- Coverage_80 is the percentage of actuals within the 80% prediction interval
- Coverage_95 should be >= coverage_80
- Backtest with no actuals: returns `no_actuals` status
- MAPE is computed as mean of |actual - forecast| / |actual|
- RMSE matches sqrt(mean((actual - forecast)^2))
- Accuracy records persist in `forecast_accuracy` table
- Accuracy dashboard shows degradation curve (MAPE increasing with horizon)

---

## Phase 8: Drift Detection & Retraining

**Goal:** Implement automated model drift detection using Population Stability Index (PSI) and performance degradation monitoring. Trigger automatic retraining when thresholds are exceeded.

**Duration:** 2-3 weeks

### Task 8.1: Drift Detection Engine

**What:** Build a drift detector that compares the current data distribution against the training baseline. Compute PSI per feature and track model performance metrics over time.

**Design:**

```python
# backend/app/ml/drift_detector.py
import numpy as np
from scipy import stats

class DriftDetector:
    def compute_psi(self, baseline: np.ndarray, current: np.ndarray, bins: int = 10) -> float:
        """Population Stability Index between baseline and current distributions."""
        min_val = min(baseline.min(), current.min())
        max_val = max(baseline.max(), current.max())
        bin_edges = np.linspace(min_val, max_val, bins + 1)

        baseline_hist, _ = np.histogram(baseline, bins=bin_edges)
        current_hist, _ = np.histogram(current, bins=bin_edges)

        # Add small epsilon to avoid log(0)
        baseline_pct = (baseline_hist + 1e-6) / len(baseline)
        current_pct = (current_hist + 1e-6) / len(current)

        psi = np.sum((current_pct - baseline_pct) * np.log(current_pct / baseline_pct))
        return float(psi)

    def check_drift(
        self,
        baseline_df: pd.DataFrame,
        current_df: pd.DataFrame,
        feature_columns: list[str],
        threshold: float = 0.1,
    ) -> dict:
        feature_drifts = []
        for col in feature_columns:
            if pd.api.types.is_numeric_dtype(baseline_df[col]):
                psi = self.compute_psi(
                    baseline_df[col].dropna().values,
                    current_df[col].dropna().values,
                )
                feature_drifts.append({
                    "feature": col,
                    "psi": round(psi, 4),
                    "drifted": psi > threshold,
                    "baseline_mean": round(float(baseline_df[col].mean()), 4),
                    "current_mean": round(float(current_df[col].mean()), 4),
                })

        overall_score = np.mean([f["psi"] for f in feature_drifts]) if feature_drifts else 0.0
        is_drifted = any(f["drifted"] for f in feature_drifts)

        return {
            "overall_drift_score": round(overall_score, 4),
            "is_drifted": is_drifted,
            "feature_drifts": feature_drifts,
            "drifted_feature_count": sum(1 for f in feature_drifts if f["drifted"]),
        }
```

**Testing:**
- Two identical distributions: PSI is near 0 (< 0.01)
- Baseline N(0,1) vs current N(2,1) (shifted mean): PSI > 0.1 (drift detected)
- Baseline N(0,1) vs current N(0,3) (increased variance): PSI > 0.1
- Per-feature drift report correctly identifies which features drifted
- Overall drift score is the mean of per-feature PSI values
- `is_drifted` is true when any feature exceeds the threshold
- Handles columns with null values without error
- Empty current DataFrame raises a clear error

### Task 8.2: Scheduled Drift Monitoring

**What:** Implement a Celery Beat scheduled task that runs drift checks at the configured interval for all active drift monitors. Store results and trigger notifications.

**Design:**

```python
# backend/app/tasks/drift_tasks.py
from celery import shared_task
from celery.schedules import crontab

@shared_task
def run_drift_checks():
    """Runs every hour. Checks all active monitors whose check interval has elapsed."""
    db = get_sync_session()
    monitors = db.query(DriftMonitor).filter(DriftMonitor.is_active == True).all()

    for monitor in monitors:
        last_check = db.query(DriftEvent)\
            .filter(DriftEvent.drift_monitor_id == monitor.id)\
            .order_by(DriftEvent.check_time.desc())\
            .first()

        if last_check and (now() - last_check.check_time).total_seconds() < monitor.config.get("check_interval_hours", 24) * 3600:
            continue  # Not time to check yet

        model = db.query(Model).get(monitor.model_id)
        # Load baseline (training data) and current (recent predictions)
        baseline_df = load_training_data(model)
        current_df = load_recent_prediction_inputs(model)

        if current_df is None or len(current_df) < 100:
            continue  # Insufficient data for drift check

        detector = DriftDetector()
        results = detector.check_drift(baseline_df, current_df, feature_columns=baseline_df.columns.tolist())

        # Create drift event
        event = DriftEvent(
            drift_monitor_id=monitor.id,
            is_drifted=results["is_drifted"],
            results=results,
        )
        db.add(event)

        if results["is_drifted"]:
            # Generate narrative
            narrative = await generate_drift_narrative(results, model)
            event.narrative = narrative

            # Send notification
            if monitor.config.get("notification"):
                send_drift_notification(monitor, results, narrative)

            # Trigger retraining if configured
            if monitor.config.get("retrain_on_drift"):
                retrain_model.delay(str(model.id), str(event.id))

        db.commit()

# Celery Beat schedule
app.conf.beat_schedule = {
    "drift-check-hourly": {
        "task": "app.tasks.drift_tasks.run_drift_checks",
        "schedule": crontab(minute=0),  # Every hour on the hour
    },
}
```

**Testing:**
- Celery Beat triggers `run_drift_checks` every hour
- Monitor with 24h interval: check runs once per day, not every hour
- Drift detected: drift event created with `is_drifted=true`
- No drift: drift event created with `is_drifted=false`
- Notification sent via email when drift detected and notification configured
- Retraining triggered when `retrain_on_drift=true` and drift detected
- Insufficient data (< 100 recent predictions): check is skipped
- Multiple active monitors are all checked in a single run

### Task 8.3: Automatic Retraining Pipeline

**What:** When drift triggers retraining, automatically retrain the model on fresh data, compare new model performance to the old, and optionally auto-promote if the new model is better.

**Design:**

```python
# backend/app/tasks/drift_tasks.py
@shared_task
def retrain_model(model_id: str, drift_event_id: str):
    db = get_sync_session()
    model = db.query(Model).get(model_id)
    run = db.query(Run).get(model.run_id)
    experiment = db.query(Experiment).get(run.experiment_id)

    # Load fresh data (same source, newer data)
    fresh_df = load_fresh_training_data(model)

    # Retrain with same configuration
    config = experiment.config
    trainer = AutoGluonTabularTrainer()
    results = trainer.train(
        df=fresh_df,
        target_column=config["target_column"],
        problem_type=config["problem_type"],
        time_limit=config.get("time_limit_seconds", 3600),
    )

    # Compare old vs new
    old_score = model.performance.get("score", 0)
    new_score = results["best_score"]
    improvement = ((new_score - old_score) / abs(old_score)) * 100 if old_score != 0 else 0

    # Create new model version
    new_model = Model(
        project_id=model.project_id,
        name=model.name,
        version=model.version + 1,
        run_id=new_run.id,
        stage="staging",
        artifact_path=results["artifact_path"],
        performance={"score": new_score},
    )
    db.add(new_model)

    # Auto-promote if improvement > 5%
    if improvement > 5:
        model.stage = "archived"
        new_model.stage = "production"
        new_model.promoted_at = func.now()

    db.commit()
```

**Testing:**
- Retraining produces a new model version (version N+1)
- New model is initially in "staging" stage
- If new model improves by >5%: auto-promoted to "production", old model archived
- If new model does not improve: stays in "staging" for manual review
- Retraining failure does not affect the current production model
- Comparison metrics are stored for audit (old score, new score, improvement %)
- Drift event is linked to the retraining job

### Task 8.4: Drift Dashboard UI

**What:** Build a drift monitoring dashboard showing drift timeline, per-feature PSI scores, and retraining history.

**Testing:**
- Drift timeline chart shows PSI score over time with a threshold line
- Drifted features are highlighted in red
- Non-drifted features are shown in green
- Retraining events are marked on the timeline
- Drift narrative text is displayed for each drift event
- Active/inactive monitor toggle works
- Threshold configuration is editable

---

## Phase 9: Context-Aware Feature Engineering

**Goal:** Leverage LLM understanding of business domain context to generate semantically meaningful features that blind AutoML would miss (lag features, rolling windows, cohort flags, seasonal components).

**Duration:** 2-3 weeks

### Task 9.1: LLM-Driven Feature Generation

**What:** Users describe their business problem in natural language. The LLM analyzes the domain context and dataset schema to propose features with transformation logic. Features are generated as SQL/pandas expressions and validated before training.

**Design:**

```python
# backend/app/llm/prompts/feature_generation.py
FEATURE_GENERATION_PROMPT = """You are a feature engineering expert. Given a business problem 
description and a dataset schema, propose features that would improve predictive accuracy.

Business problem: {domain_context}

Dataset columns:
{column_summary}

Target column: {target_column}
Problem type: {problem_type}

For each proposed feature, provide:
1. name: A descriptive snake_case name
2. type: One of: lag, rolling_window, seasonal, cohort, interaction, calendar, ratio
3. source_columns: Which existing columns are used
4. transform: A pandas expression to compute the feature
5. rationale: Why this feature would improve predictions for this specific business problem

Focus on domain-specific features. For time-series data, include lag features, 
rolling statistics, seasonal indicators, and trend components. For classification, 
include interaction terms and ratio features.

Return valid JSON array of feature objects."""
```

```python
# backend/app/services/feature_service.py
class FeatureService:
    def __init__(self, db):
        self.db = db
        self.llm = ClaudeClient()

    async def generate_features(self, project_id: str, dataset_id: str) -> list[dict]:
        project = await self.db.get(Project, project_id)
        dataset = await self.db.get(Dataset, dataset_id)

        prompt = FEATURE_GENERATION_PROMPT.format(
            domain_context=project.domain_context,
            column_summary=self._format_columns(dataset.schema_info),
            target_column=project.config.get("target_column", ""),
            problem_type=project.problem_type,
        )

        response = await self.llm.chat(
            messages=[{"role": "user", "content": prompt}],
            system="You are a feature engineering expert. Return only valid JSON.",
            max_tokens=4096,
        )

        features = json.loads(response["content"])

        # Validate each feature's transform expression
        validated = []
        for f in features:
            try:
                self._validate_transform(f["transform"], dataset.schema_info)
                validated.append(f)
            except Exception as e:
                f["validation_error"] = str(e)
                validated.append(f)

        # Store as feature set
        feature_set = FeatureSet(
            project_id=project_id,
            name=f"llm_generated_{datetime.now().strftime('%Y%m%d_%H%M%S')}",
            generation_mode="llm_context_driven",
            domain_prompt=project.domain_context,
            features=validated,
            feature_count=len(validated),
            created_by=project.created_by,
        )
        self.db.add(feature_set)
        await self.db.commit()

        return validated
```

**Testing:**
- For SaaS churn dataset: LLM proposes lag features (revenue_lag_7d, usage_lag_30d), rolling windows (avg_sessions_30d), cohort flags (signup_month)
- For demand forecasting: LLM proposes seasonal features (day_of_week, month, is_holiday), trend components, promotional flags
- Invalid transform expressions are flagged with `validation_error`
- Feature set is stored in the database with `generation_mode: "llm_context_driven"`
- Domain prompt is preserved for reproducibility
- Generated features improve model MAPE by 5-15% on test datasets compared to raw features only
- LLM returns valid JSON that parses without errors
- Features with invalid source_columns reference are caught during validation

### Task 9.2: Feature Computation Engine

**What:** Build the engine that takes a feature set definition and a DataFrame, computes all features, and returns an enriched DataFrame ready for training. Support lag, rolling window, seasonal, cohort, and interaction feature types.

**Design:**

```python
# backend/app/ml/feature_engineering.py
import pandas as pd
import numpy as np

class FeatureComputer:
    def compute(self, df: pd.DataFrame, features: list[dict], time_column: str | None = None) -> pd.DataFrame:
        result = df.copy()
        for feature in features:
            if feature.get("validation_error"):
                continue  # Skip invalid features
            try:
                result[feature["name"]] = self._compute_feature(result, feature, time_column)
            except Exception as e:
                # Log but don't fail the entire computation
                print(f"Failed to compute feature {feature['name']}: {e}")
        return result

    def _compute_feature(self, df: pd.DataFrame, feature: dict, time_column: str | None) -> pd.Series:
        ftype = feature["type"]
        params = feature.get("params", {})

        if ftype == "lag":
            col = feature["source_columns"][0]
            periods = params.get("lag_periods", 1)
            if time_column:
                df = df.sort_values(time_column)
            return df[col].shift(periods)

        elif ftype == "rolling_window":
            col = feature["source_columns"][0]
            window = params.get("window_size", 7)
            agg = params.get("aggregation", "mean")
            if time_column:
                df = df.sort_values(time_column)
            return df[col].rolling(window=window, min_periods=1).agg(agg)

        elif ftype == "seasonal":
            if time_column:
                dt = pd.to_datetime(df[time_column])
                period_type = params.get("period", "month")
                if period_type == "month": return dt.dt.month
                if period_type == "day_of_week": return dt.dt.dayofweek
                if period_type == "quarter": return dt.dt.quarter
                if period_type == "day_of_year": return dt.dt.dayofyear
            raise ValueError("Seasonal features require a time column")

        elif ftype == "ratio":
            numerator = feature["source_columns"][0]
            denominator = feature["source_columns"][1]
            return df[numerator] / df[denominator].replace(0, np.nan)

        elif ftype == "interaction":
            cols = feature["source_columns"]
            result = df[cols[0]]
            for col in cols[1:]:
                result = result * df[col]
            return result

        elif ftype == "cohort":
            col = feature["source_columns"][0]
            granularity = params.get("granularity", "month")
            dt = pd.to_datetime(df[col])
            if granularity == "month": return dt.dt.to_period("M").astype(str)
            if granularity == "quarter": return dt.dt.to_period("Q").astype(str)
            if granularity == "year": return dt.dt.year.astype(str)

        raise ValueError(f"Unknown feature type: {ftype}")
```

**Testing:**
- Lag feature: `revenue_lag_7d` shifts revenue by 7 rows, first 7 values are NaN
- Rolling window: `revenue_rolling_30d_mean` computes 30-period moving average
- Seasonal (month): extracts month number (1-12) from datetime column
- Seasonal (day_of_week): extracts day of week (0-6)
- Ratio: revenue/users computes per-user revenue; division by zero produces NaN, not error
- Interaction: price * quantity produces total_value
- Cohort: signup_date -> "2025-01" monthly cohort
- Invalid feature type: raises ValueError
- Feature with missing source column: raises KeyError
- Performance: computing 20 features on 1M rows completes in under 30 seconds

### Task 9.3: Feature Importance Feedback Loop

**What:** After training with LLM-generated features, compare model performance with and without the generated features. Report which LLM features added value and which did not. Feed this back to improve future feature generation.

**Testing:**
- Train baseline model (raw features only) and enhanced model (raw + LLM features)
- Report improvement: "LLM features improved F1 from 0.89 to 0.94 (+5.6%)"
- Per-feature importance shows which LLM features ranked in the top 10
- Features that ranked below median importance are flagged as "low impact"
- Feedback is stored in the feature set for future LLM prompt refinement

---

## Phase 10: Narrative Generation & Reporting

**Goal:** Auto-generate plain-English explanations for every model output, forecast, and drift event. Make model outputs actionable for non-technical stakeholders.

**Duration:** 2-3 weeks

### Task 10.1: Prediction Narrative Generator

**What:** For each prediction or forecast run, generate a plain-English narrative explaining the key drivers, expected outcomes, and confidence levels. Use Claude to translate SHAP values and metrics into business language.

**Design:**

```python
# backend/app/services/narrative_service.py
class NarrativeService:
    def __init__(self, db):
        self.db = db
        self.llm = ClaudeClient()

    async def generate_forecast_narrative(
        self,
        model_id: str,
        forecast_summary: dict,
        shap_values: dict,
        domain_context: str,
    ) -> str:
        prompt = f"""Generate a plain-English forecast narrative for business stakeholders.

Domain context: {domain_context}

Forecast summary:
- Forecast horizon: {forecast_summary['horizon_days']} days
- Point forecast trend: {forecast_summary['trend_direction']}
- Overall change: {forecast_summary['overall_change_pct']:.1f}%
- Prediction interval width (80%): {forecast_summary['interval_width_80']:.1f}%

Top contributing factors (SHAP analysis):
{json.dumps(shap_values['features'][:5], indent=2)}

Write 2-3 paragraphs suitable for a board presentation. Include:
1. What the forecast predicts and the expected trajectory
2. The key drivers behind the forecast (referencing specific features)
3. Confidence level and what could cause the forecast to be wrong

Use specific numbers. Do not use technical terms like SHAP, MAPE, or RMSE.
Write as if explaining to a VP of Operations."""

        response = await self.llm.chat(
            messages=[{"role": "user", "content": prompt}],
            system="You are a business analytics writer. Write clear, actionable forecast narratives.",
            max_tokens=1024,
        )

        # Store narrative
        explanation = ModelExplanation(
            model_id=model_id,
            explanation_type="forecast_narrative",
            content={"narrative": response["content"]},
            narrative=response["content"],
        )
        self.db.add(explanation)
        await self.db.commit()

        return response["content"]
```

**Testing:**
- Generated narrative is 2-3 paragraphs, 150-300 words
- Narrative references specific forecast numbers (e.g., "Revenue is expected to grow 12%")
- Narrative mentions top contributing features in business language (not column names)
- Narrative includes a confidence statement ("with 80% confidence between X and Y")
- Narrative does not contain technical jargon (SHAP, RMSE, MAPE, p-value)
- Narrative is stored in `model_explanations` table
- Narrative generation completes in under 10 seconds

### Task 10.2: Drift Explanation Narratives

**What:** When drift is detected, generate a plain-English explanation of what changed, why the model may be degrading, and what action is recommended.

**Design:**

```python
# backend/app/llm/prompts/drift_narrative.py
DRIFT_NARRATIVE_PROMPT = """A predictive model is showing signs of data drift. 
Explain what happened in plain English for a business user.

Model: {model_name} (predicts {target_column})
Domain: {domain_context}

Drift details:
- Overall drift score: {drift_score} (threshold: {threshold})
- Drifted features:
{drifted_features_detail}

Performance impact:
- Current accuracy: {current_metric}
- Baseline accuracy: {baseline_metric}
- Degradation: {degradation_pct:.1f}%

Write a 1-2 paragraph explanation including:
1. What changed in the data (in business terms, not statistical)
2. How this affects predictions
3. What action is recommended

Do not use terms like PSI, KS test, or distribution shift. 
Explain as if talking to a marketing manager."""
```

**Testing:**
- Drift narrative explains the change in business terms (e.g., "Customer spending patterns shifted lower in March")
- Narrative quantifies the impact (e.g., "Predictions may be off by 15% more than usual")
- Narrative recommends action (e.g., "We recommend retraining the model with data from the last 90 days")
- Narrative does not contain statistical jargon
- Narrative is 100-200 words

### Task 10.3: Compliance Report Generator

**What:** Generate an audit-ready PDF/HTML report documenting the entire model lifecycle: data provenance, feature selection rationale, model performance, deployment history, and drift events.

**Testing:**
- Report includes: dataset source, row count, column list, quality issues
- Report includes: experiment configuration, algorithm selection, validation metrics
- Report includes: feature engineering steps and rationale
- Report includes: model version history with promotion dates
- Report includes: drift check history and retraining events
- Report exports to HTML and PDF format
- Report timestamp and generation metadata are included
- Report covers the full lineage from data upload to current production model

---

## Phase 11: REST API & Real-Time Serving

**Goal:** Expose production models via a REST API for real-time single-prediction serving. Implement API key management, rate limiting, and response caching.

**Duration:** 2 weeks

### Task 11.1: Real-Time Prediction API

**What:** Create a public-facing REST endpoint that accepts a JSON payload of feature values and returns a prediction from the currently deployed production model. Include confidence scores and optional narrative explanation.

**Design:**

```python
# backend/app/api/v1/predictions.py
@router.post("/projects/{project_id}/predict")
async def predict_realtime(
    project_id: str,
    body: PredictRequest,  # features: dict[str, Any]
    include_explanation: bool = False,
    db: AsyncSession = Depends(get_db),
    api_key: str = Depends(verify_api_key),
):
    # Find production model
    model = await db.execute(
        select(Model)
        .where(Model.project_id == project_id, Model.stage == "production")
        .order_by(Model.version.desc())
        .limit(1)
    )
    model = model.scalar_one_or_none()
    if not model:
        raise HTTPException(404, "No production model deployed for this project")

    # Load model (cached in Redis)
    predictor = await load_cached_predictor(model.artifact_path)

    # Predict
    input_df = pd.DataFrame([body.features])
    prediction = predictor.predict(input_df)
    response = {
        "prediction": prediction.iloc[0],
        "model_name": model.name,
        "model_version": model.version,
    }

    # Optional: probability scores
    if hasattr(predictor, "predict_proba"):
        probas = predictor.predict_proba(input_df)
        response["probabilities"] = probas.iloc[0].to_dict()

    # Optional: explanation
    if include_explanation:
        explanation = generate_local_explanation(predictor, input_df)
        response["explanation"] = explanation

    return response
```

**Testing:**
- POST with valid features returns prediction with model name and version
- Classification returns probabilities for each class
- Regression returns a numeric prediction value
- Request with missing required features returns 400 with list of missing fields
- Request with invalid API key returns 401
- Request for project with no production model returns 404
- Response time is under 200ms for a cached model (P95)
- Include_explanation flag adds SHAP values and narrative to response
- Concurrent requests (100 RPS) are handled without errors

### Task 11.2: API Key Management

**What:** Implement API key generation, rotation, and rate limiting for the real-time prediction API.

**Design:**

```python
# backend/app/services/api_key_service.py
import secrets
import hashlib

class APIKeyService:
    def generate_key(self, project_id: str, name: str) -> tuple[str, str]:
        """Returns (raw_key, key_prefix) where raw_key is shown once."""
        raw_key = f"paw_{secrets.token_urlsafe(32)}"
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        prefix = raw_key[:12]

        api_key = APIKey(
            project_id=project_id,
            name=name,
            key_hash=key_hash,
            key_prefix=prefix,
            rate_limit=1000,  # requests per hour
        )
        self.db.add(api_key)
        return raw_key, prefix

    async def verify(self, raw_key: str) -> APIKey | None:
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        result = await self.db.execute(
            select(APIKey).where(APIKey.key_hash == key_hash, APIKey.is_active == True)
        )
        return result.scalar_one_or_none()
```

**Testing:**
- Generate key: returns a key starting with "paw_", 44+ characters long
- Key is shown once at creation; only the prefix is stored
- Verify valid key: returns the APIKey record
- Verify invalid key: returns None
- Revoked key: verify returns None
- Rate limit exceeded: returns 429 with retry-after header
- Key rotation: new key works, old key still works until explicitly revoked

### Task 11.3: Model Caching & Performance

**What:** Implement Redis-based model caching to avoid loading model artifacts from S3 on every prediction request. Cache the loaded predictor object serialized in memory.

**Testing:**
- First prediction request loads model from S3 (cold start < 5 seconds)
- Subsequent requests serve from Redis cache (< 100ms)
- Cache invalidation on model promotion (new production version clears old cache)
- Cache TTL of 1 hour prevents stale models
- Memory usage is bounded (max 10 cached models per project)
- Concurrent cache writes do not corrupt the cached model

---

## Phase 12: Production Hardening & SaaS

**Goal:** Prepare for production deployment with multi-tenant isolation, security hardening, monitoring, CI/CD, and SaaS billing integration.

**Duration:** 3-4 weeks

### Task 12.1: Multi-Tenant Data Isolation

**What:** Implement Row-Level Security (RLS) in PostgreSQL to ensure organisation-level data isolation. All queries are automatically scoped to the authenticated user's organisation.

**Design:**

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation_projects ON projects
    USING (organisation_id = current_setting('app.current_org_id')::UUID);

-- Set the org context per request (in middleware)
-- SET LOCAL app.current_org_id = 'org-uuid-here';
```

```python
# backend/app/middleware/tenant.py
from starlette.middleware.base import BaseHTTPMiddleware

class TenantMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        user = request.state.user  # Set by auth middleware
        if user and user.organisation_id:
            # Set PostgreSQL session variable for RLS
            async with request.state.db.begin():
                await request.state.db.execute(
                    text(f"SET LOCAL app.current_org_id = '{user.organisation_id}'")
                )
        return await call_next(request)
```

**Testing:**
- User A in Org-1 cannot see User B's data in Org-2
- API request without auth cannot access any tenant data
- RLS policies are active on: projects, datasets, experiments, runs, models, predictions
- Superadmin role can query across all tenants (for admin dashboard)
- Performance: RLS adds < 5ms overhead per query

### Task 12.2: Security Hardening

**What:** Implement HTTPS enforcement, CORS restriction, input validation, SQL injection prevention, secret rotation, and security headers.

**Testing:**
- All API endpoints enforce HTTPS in production
- CORS allows only configured origins
- SQL injection attempts in query parameters are rejected
- XSS payloads in user input are sanitized
- Security headers present: Strict-Transport-Security, X-Content-Type-Options, X-Frame-Options
- Secrets (API keys, database passwords) are not logged
- Rate limiting active on auth endpoints (10 attempts per minute per IP)

### Task 12.3: Monitoring & Observability

**What:** Implement structured logging, health check endpoints, Prometheus metrics, and error tracking (Sentry integration).

**Design:**

```python
# backend/app/api/health.py
@router.get("/health")
async def health_check(db: AsyncSession = Depends(get_db)):
    checks = {}
    # Database
    try:
        await db.execute(text("SELECT 1"))
        checks["database"] = "ok"
    except Exception:
        checks["database"] = "error"
    # Redis
    try:
        redis = get_redis()
        await redis.ping()
        checks["redis"] = "ok"
    except Exception:
        checks["redis"] = "error"
    # S3
    try:
        StorageService().check()
        checks["storage"] = "ok"
    except Exception:
        checks["storage"] = "error"

    status = "healthy" if all(v == "ok" for v in checks.values()) else "degraded"
    return {"status": status, "checks": checks}
```

**Testing:**
- `/health` returns 200 with all checks passing
- `/health` returns 503 with `degraded` status when database is down
- Prometheus `/metrics` endpoint exposes: request count, latency histogram, error rate
- Structured logs include: request_id, user_id, org_id, endpoint, duration
- Sentry captures unhandled exceptions with full context
- Training task failures are logged with experiment_id and error details

### Task 12.4: CI/CD Pipeline

**What:** Set up GitHub Actions for continuous integration (lint, type check, unit tests, integration tests) and continuous deployment (Docker build, push, deploy to staging/production).

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  backend:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: paw_test
          POSTGRES_USER: paw
          POSTGRES_PASSWORD: paw
        ports:
          - 5432:5432
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[test]"
        working-directory: backend
      - run: ruff check .
        working-directory: backend
      - run: mypy app/
        working-directory: backend
      - run: pytest tests/ -v --cov=app --cov-report=xml
        working-directory: backend
        env:
          DATABASE_URL: postgresql+asyncpg://paw:paw@localhost:5432/paw_test

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
      - run: npm ci
        working-directory: frontend
      - run: npm run lint
        working-directory: frontend
      - run: npm run type-check
        working-directory: frontend
      - run: npm run build
        working-directory: frontend
```

**Testing:**
- CI passes on a clean branch with no code changes
- Lint failures block the PR merge
- Unit test failures block the PR merge
- Integration tests run against a real PostgreSQL instance
- Docker build produces a working image
- Staging deployment auto-triggers on merge to main
- Production deployment requires manual approval

### Task 12.5: SaaS Billing & Plan Management

**What:** Integrate Stripe for subscription billing with free, pro, and enterprise plans. Enforce plan limits (concurrent training jobs, prediction API rate limits, storage quota).

**Testing:**
- Free plan: 3 projects, 5 training runs/month, 1K predictions/month
- Pro plan: unlimited projects, 100 training runs/month, 100K predictions/month
- Enterprise plan: unlimited everything, SSO, priority support
- Stripe checkout redirects correctly
- Webhook processes subscription.created, subscription.updated, subscription.deleted
- Plan limits are enforced at the API level (403 when exceeded)
- Usage metering tracks training runs and predictions per billing cycle

---

## Definition of Done

Each phase is considered complete when ALL of the following criteria are met:

### Code Quality
- [ ] All new code has type annotations (Python: mypy strict, TypeScript: strict mode)
- [ ] Linting passes with zero warnings (ruff for Python, eslint for TypeScript)
- [ ] No TODO comments remain in shipped code (tracked in issue tracker instead)
- [ ] Code follows the project's naming conventions (snake_case Python, camelCase TypeScript)

### Testing
- [ ] Unit test coverage >= 80% for new code (measured by pytest-cov / istanbul)
- [ ] Integration tests cover all API endpoints added in the phase
- [ ] All tests pass in CI (GitHub Actions green)
- [ ] Edge cases documented in the phase are covered by specific test cases
- [ ] No flaky tests (all tests deterministic)

### Documentation
- [ ] API endpoints have OpenAPI descriptions (auto-generated from FastAPI docstrings)
- [ ] Complex business logic has inline comments explaining "why" (not "what")
- [ ] New configuration options are documented in `.env.example`
- [ ] Database migrations have descriptive names and are reversible

### Deployment
- [ ] Docker build succeeds without errors
- [ ] `docker compose up` starts all services and passes health checks
- [ ] Database migrations run without data loss on existing data
- [ ] No secrets committed to the repository

### User Acceptance
- [ ] Feature works end-to-end in the UI (not just API)
- [ ] Error states show user-friendly messages (not stack traces)
- [ ] Loading states are shown during async operations
- [ ] The feature is accessible from the project navigation

### Performance
- [ ] API endpoints respond in under 500ms (P95, excluding training/prediction)
- [ ] Training tasks report progress and are cancellable
- [ ] Frontend pages load in under 3 seconds (LCP)
- [ ] No N+1 query patterns in ORM queries
