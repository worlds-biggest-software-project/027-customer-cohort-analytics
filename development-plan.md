# Customer Cohort Analytics — Phased Development Plan
> Project: 027-customer-cohort-analytics · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

### Language & Framework
| Layer | Choice | Rationale |
|-------|--------|-----------|
| Backend API | **Python 3.12 + FastAPI** | Best ML ecosystem (scikit-learn, XGBoost, SHAP); async support for high-throughput event ingestion; Pydantic for validation |
| Frontend | **Next.js 15 (TypeScript)** | React ecosystem for dashboard charting (Recharts/Nivo); SSR for initial load; strong typing |
| ML Pipeline | **XGBoost + SHAP + scikit-learn** | Industry-standard churn prediction; SHAP is the gold standard for ML explainability; no vendor lock-in |
| NL Layer | **Claude API (Anthropic)** | Best-in-class for SQL generation and natural language explanation; prompt caching for cost efficiency |

### Data Store
| Store | Choice | Rationale |
|-------|--------|-----------|
| Primary DB | **PostgreSQL 16** | MVP uses **Data Model Suggestion 3 (Hybrid Relational + JSONB)** — fastest path to MVP, fewest tables (26), flexible event properties via JSONB, single-database deployment. Upgrade path to Model 4 (PostgreSQL + ClickHouse dual-store) when event volume exceeds ~50M/month |
| Cache | **Redis 7** | Session storage, rate limiting, background job queues (via BullMQ or Celery) |
| Object Storage | **S3-compatible (MinIO for self-hosted)** | ML model artifact storage |
| Message Queue | **Redis Streams (MVP) -> Kafka (scale)** | Event buffering before DB write; upgrade path clear |

### Event Schema
| Decision | Choice | Rationale |
|----------|--------|-----------|
| Event format | **Segment Spec (`.track()`, `.identify()`, `.page()`, `.group()`)** | De facto industry standard; compatible with Segment, RudderStack, mParticle CDPs |
| Timestamps | **ISO 8601 / `TIMESTAMPTZ`** | Unambiguous timezone handling |
| Geography | **ISO 3166-1/2** | Country and subdivision codes |
| Explainability | **SHAP (SHapley Additive Explanations)** | Published research; not patent-blocked; industry-standard for ML explainability |

### Deployment
| Decision | Choice | Rationale |
|----------|--------|-----------|
| Self-hosted | **Docker Compose** | Single-command deployment for evaluation and small teams |
| Production | **Kubernetes (Helm chart)** | Horizontal scaling for ingestion API and ML workers |
| CI/CD | **GitHub Actions** | Standard for OSS projects |
| Licence | **MIT** | Maximally permissive; matches PostHog's approach; removes adoption friction |

### Project Structure
```
customer-cohort-analytics/
  backend/
    app/
      api/                  # FastAPI routers
        v1/
          events.py         # Event ingestion endpoints
          users.py          # User identity endpoints
          cohorts.py        # Cohort CRUD + membership
          retention.py      # Retention analysis endpoints
          funnels.py        # Funnel analysis endpoints
          predictions.py    # Churn prediction endpoints
          dashboards.py     # Dashboard CRUD
          alerts.py         # Alert rule management
          nl.py             # Natural language query endpoint
          auth.py           # Authentication endpoints
      core/
        config.py           # Settings (Pydantic BaseSettings)
        database.py         # SQLAlchemy engine + session
        security.py         # JWT, API key auth
        dependencies.py     # FastAPI dependency injection
      models/               # SQLAlchemy ORM models
        tenant.py
        user.py
        event.py
        cohort.py
        retention.py
        funnel.py
        prediction.py
        dashboard.py
        alert.py
        gdpr.py
      schemas/              # Pydantic request/response schemas
      services/             # Business logic layer
        event_service.py
        cohort_service.py
        retention_service.py
        funnel_service.py
        prediction_service.py
        alert_service.py
        nl_service.py
      ml/                   # ML pipeline
        feature_engineering.py
        churn_model.py
        shap_explainer.py
        cohort_discovery.py
      workers/              # Background job processors
        retention_computer.py
        cohort_evaluator.py
        feature_builder.py
        prediction_runner.py
        alert_checker.py
    migrations/             # Alembic migrations
    tests/
      unit/
      integration/
      fixtures/
    pyproject.toml
  frontend/
    src/
      app/                  # Next.js app router
      components/
        cohort-builder/
        retention-chart/
        funnel-chart/
        churn-dashboard/
        nl-query/
      lib/
        api-client.ts
        types.ts
      hooks/
    package.json
  sdk/
    javascript/             # Browser/Node SDK
      src/
        index.ts
        track.ts
        identify.ts
        page.ts
    python/                 # Server-side SDK
  docker-compose.yml
  Dockerfile
  helm/                     # Kubernetes Helm chart
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Event Ingestion & SDK -----> Phase 3: Cohort Engine
                                          |
                                          v
                                    Phase 4: Retention & Funnels
                                          |
                                          v
                                    Phase 5: Dashboards & Reporting
                                          |
                                          v
                              Phase 6: ML Churn Prediction
                                          |
                                          v
                              Phase 7: XAI Explainability
                                          |
                                          v
                              Phase 8: NL Query Interface
                                          |
                                          v
                              Phase 9: Automated Cohort Discovery
                                          |
                                          v
                              Phase 10: Alerts & Proactive Notifications
                                          |
                                          v
                              Phase 11: B2B Account Aggregation
                                          |
                                          v
                              Phase 12: GDPR, Security & Production Hardening
```

---

## Phase 1: Foundation — Project Scaffold, Database, Auth

### Definition of Done
- FastAPI application starts, connects to PostgreSQL, runs migrations, and returns `200 OK` on `/health`
- Multi-tenant auth works: JWT login, API key creation, tenant-scoped RBAC
- CI pipeline runs linting + tests on every push
- Docker Compose starts the full stack (API + DB + Redis) in one command

### Task 1.1: Project scaffold and configuration

**What:** Create the Python backend project with FastAPI, SQLAlchemy, Alembic, and Pydantic settings.

**Design:**
```python
# backend/app/core/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    PROJECT_NAME: str = "Customer Cohort Analytics"
    VERSION: str = "0.1.0"
    API_V1_PREFIX: str = "/api/v1"

    # Database
    DATABASE_URL: str = "postgresql+asyncpg://cca:cca@localhost:5432/cca"
    DATABASE_POOL_SIZE: int = 20
    DATABASE_MAX_OVERFLOW: int = 10

    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"

    # Auth
    JWT_SECRET_KEY: str  # required, no default
    JWT_ALGORITHM: str = "HS256"
    JWT_ACCESS_TOKEN_EXPIRE_MINUTES: int = 30

    # S3 / Object Storage
    S3_ENDPOINT: str = "http://localhost:9000"
    S3_BUCKET: str = "cca-models"
    S3_ACCESS_KEY: str = ""
    S3_SECRET_KEY: str = ""

    class Config:
        env_file = ".env"

settings = Settings()
```

```python
# backend/app/core/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase

class Base(DeclarativeBase):
    pass

engine = create_async_engine(
    settings.DATABASE_URL,
    pool_size=settings.DATABASE_POOL_SIZE,
    max_overflow=settings.DATABASE_MAX_OVERFLOW,
)

async_session_factory = async_sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        yield session
```

```python
# backend/app/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: verify DB connection
    async with engine.begin() as conn:
        await conn.execute(text("SELECT 1"))
    yield
    # Shutdown: dispose engine
    await engine.dispose()

app = FastAPI(
    title=settings.PROJECT_NAME,
    version=settings.VERSION,
    lifespan=lifespan,
)

@app.get("/health")
async def health():
    return {"status": "ok", "version": settings.VERSION}
```

**Testing:**
- `test_health_returns_200`: `GET /health` returns `{"status": "ok"}`
- `test_settings_loads_from_env`: verify `Settings` reads `DATABASE_URL` from environment
- `test_db_session_connects`: `get_db()` yields a valid `AsyncSession` that can execute `SELECT 1`
- `test_invalid_database_url_raises`: startup fails with a clear error when `DATABASE_URL` is unreachable

---

### Task 1.2: Multi-tenant data model and migrations

**What:** Create SQLAlchemy models for tenants, tenant members, and platform authentication, then generate the initial Alembic migration.

**Design:**
```python
# backend/app/models/tenant.py
import uuid
from datetime import datetime
from sqlalchemy import String, DateTime, func, UniqueConstraint
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship
from app.core.database import Base

class Tenant(Base):
    __tablename__ = "tenants"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255), nullable=False)
    slug: Mapped[str] = mapped_column(String(100), nullable=False, unique=True, index=True)
    plan: Mapped[str] = mapped_column(String(50), nullable=False, default="free")
    settings: Mapped[dict] = mapped_column(JSONB, nullable=False, default=dict)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    members: Mapped[list["TenantMember"]] = relationship(back_populates="tenant", cascade="all, delete-orphan")


class TenantMember(Base):
    __tablename__ = "tenant_members"
    __table_args__ = (UniqueConstraint("tenant_id", "email"),)

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id", ondelete="CASCADE"), nullable=False)
    email: Mapped[str] = mapped_column(String(255), nullable=False)
    display_name: Mapped[str | None] = mapped_column(String(255))
    role: Mapped[str] = mapped_column(String(50), nullable=False, default="member")
    password_hash: Mapped[str] = mapped_column(String(255), nullable=False)
    last_login_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())

    tenant: Mapped["Tenant"] = relationship(back_populates="members")
```

```sql
-- Generated Alembic migration (DDL representation)
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tenant_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    password_hash   VARCHAR(255) NOT NULL,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

**Testing:**
- `test_migration_creates_tenants_table`: run `alembic upgrade head`, verify `tenants` table exists with correct columns
- `test_tenant_slug_unique_constraint`: inserting two tenants with the same slug raises `IntegrityError`
- `test_tenant_member_cascade_delete`: deleting a tenant cascades to delete its members
- `test_tenant_member_email_unique_per_tenant`: same email in different tenants succeeds; same email in same tenant fails

---

### Task 1.3: Authentication and RBAC

**What:** Implement JWT-based authentication for platform users and API key authentication for SDK event ingestion, with role-based access control.

**Design:**
```python
# backend/app/core/security.py
from datetime import datetime, timedelta
from jose import jwt, JWTError
from passlib.context import CryptContext
from pydantic import BaseModel

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

class TokenPayload(BaseModel):
    sub: str          # tenant_member.id
    tenant_id: str    # tenant.id
    role: str         # owner, admin, member, viewer
    exp: datetime

def create_access_token(member_id: str, tenant_id: str, role: str) -> str:
    expires = datetime.utcnow() + timedelta(minutes=settings.JWT_ACCESS_TOKEN_EXPIRE_MINUTES)
    payload = TokenPayload(sub=member_id, tenant_id=tenant_id, role=role, exp=expires)
    return jwt.encode(payload.model_dump(), settings.JWT_SECRET_KEY, algorithm=settings.JWT_ALGORITHM)

def verify_token(token: str) -> TokenPayload:
    payload = jwt.decode(token, settings.JWT_SECRET_KEY, algorithms=[settings.JWT_ALGORITHM])
    return TokenPayload(**payload)
```

```python
# backend/app/schemas/auth.py
from pydantic import BaseModel, EmailStr

class RegisterRequest(BaseModel):
    tenant_name: str
    tenant_slug: str
    email: EmailStr
    password: str
    display_name: str | None = None

class LoginRequest(BaseModel):
    email: EmailStr
    password: str
    tenant_slug: str

class TokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"
    tenant_id: str
    role: str

class ApiKeyCreate(BaseModel):
    name: str
    scopes: list[str] = ["events:write"]  # events:write, events:read, admin

class ApiKeyResponse(BaseModel):
    id: str
    name: str
    key_prefix: str   # first 8 chars shown
    scopes: list[str]
    created_at: datetime
```

```python
# backend/app/api/v1/auth.py
from fastapi import APIRouter, Depends, HTTPException, status

router = APIRouter(prefix="/auth", tags=["auth"])

@router.post("/register", response_model=TokenResponse, status_code=201)
async def register(body: RegisterRequest, db: AsyncSession = Depends(get_db)):
    """Create a new tenant and owner account."""
    ...

@router.post("/login", response_model=TokenResponse)
async def login(body: LoginRequest, db: AsyncSession = Depends(get_db)):
    """Authenticate and return JWT."""
    ...

@router.post("/api-keys", response_model=ApiKeyResponse, status_code=201)
async def create_api_key(body: ApiKeyCreate, current_user=Depends(require_role("admin"))):
    """Create an API key for SDK event ingestion."""
    ...
```

```sql
CREATE TABLE api_keys (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    key_hash        VARCHAR(255) NOT NULL,
    key_prefix      VARCHAR(8) NOT NULL,
    scopes          TEXT[] NOT NULL DEFAULT '{"events:write"}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_used_at    TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES tenant_members(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_keys_hash ON api_keys (key_hash) WHERE is_active = true;
```

**Testing:**
- `test_register_creates_tenant_and_owner`: POST `/auth/register` creates tenant + member with role `owner`, returns valid JWT
- `test_login_returns_jwt`: POST `/auth/login` with correct credentials returns JWT; incorrect password returns 401
- `test_jwt_contains_tenant_and_role`: decoded JWT contains `tenant_id` and `role` claims
- `test_expired_jwt_rejected`: token with past `exp` returns 401
- `test_api_key_authenticates_event_ingestion`: API key in `Authorization: Bearer cca_...` header authenticates successfully
- `test_viewer_cannot_create_api_key`: member with `viewer` role gets 403 on POST `/auth/api-keys`
- `test_admin_can_create_api_key`: member with `admin` role successfully creates API key

---

### Task 1.4: Docker Compose and CI

**What:** Create Docker Compose configuration for local development and GitHub Actions CI pipeline.

**Design:**
```yaml
# docker-compose.yml
version: "3.9"
services:
  api:
    build: .
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://cca:cca@db:5432/cca
      REDIS_URL: redis://redis:6379/0
      JWT_SECRET_KEY: dev-secret-change-me
    depends_on:
      db: { condition: service_healthy }
      redis: { condition: service_started }

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: cca
      POSTGRES_USER: cca
      POSTGRES_PASSWORD: cca
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U cca"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_DB: cca_test, POSTGRES_USER: cca, POSTGRES_PASSWORD: cca }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install -e ".[dev]"
      - run: ruff check backend/
      - run: pytest backend/tests/ -v --tb=short
```

**Testing:**
- `test_docker_compose_starts`: `docker compose up -d` brings up api, db, redis; `curl localhost:8000/health` returns 200
- `test_alembic_migration_runs_in_container`: API container runs migrations on startup
- CI pipeline: push to any branch triggers linting + test suite

---

## Phase 2: Event Ingestion & JavaScript SDK

### Definition of Done
- SDK sends `.track()`, `.identify()`, `.page()` events to ingestion API
- Events are validated, written to PostgreSQL partitioned `events` table, and queryable
- Ingestion handles 1,000 events/second on a single node
- Event schema auto-discovery populates `event_schemas` table

### Task 2.1: Event data model and partitioned table

**What:** Create the tracked users, events, and event schema registry tables following the Hybrid JSONB model (Data Model 3).

**Design:**
```python
# backend/app/models/user.py
class TrackedUser(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id: Mapped[uuid.UUID] = mapped_column(UUID(as_uuid=True), ForeignKey("tenants.id", ondelete="CASCADE"), nullable=False)
    external_id: Mapped[str | None] = mapped_column(String(255))
    anonymous_ids: Mapped[list[str]] = mapped_column(ARRAY(String), nullable=False, default=list)
    email: Mapped[str | None] = mapped_column(String(255))
    display_name: Mapped[str | None] = mapped_column(String(255))
    plan_tier: Mapped[str | None] = mapped_column(String(50))
    signup_source: Mapped[str | None] = mapped_column(String(100))
    country_code: Mapped[str | None] = mapped_column(String(2))
    traits: Mapped[dict] = mapped_column(JSONB, nullable=False, default=dict)
    first_seen_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    last_seen_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    total_events: Mapped[int] = mapped_column(default=0)
    anonymized_at: Mapped[datetime | None] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
```

```sql
-- Events table with JSONB properties (from Data Model 3)
CREATE TABLE events (
    id              UUID NOT NULL DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(20) NOT NULL,       -- track, identify, page, screen, group
    event_name      VARCHAR(255),
    user_id         UUID,
    anonymous_id    VARCHAR(255),
    timestamp       TIMESTAMPTZ NOT NULL,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    session_id      VARCHAR(255),
    page_url        TEXT,
    utm_source      VARCHAR(255),
    utm_medium      VARCHAR(255),
    utm_campaign    VARCHAR(255),
    device_type     VARCHAR(50),
    country_code    CHAR(2),
    properties      JSONB NOT NULL DEFAULT '{}',
    context         JSONB NOT NULL DEFAULT '{}',
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

CREATE INDEX idx_events_tenant_user_ts ON events (tenant_id, user_id, timestamp);
CREATE INDEX idx_events_tenant_name_ts ON events (tenant_id, event_name, timestamp) WHERE event_name IS NOT NULL;
CREATE INDEX idx_events_properties ON events USING GIN (properties);

-- Auto-partition management function
CREATE OR REPLACE FUNCTION create_event_partition_if_not_exists(partition_date DATE)
RETURNS VOID AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    start_date := date_trunc('month', partition_date)::date;
    end_date := (start_date + interval '1 month')::date;
    partition_name := 'events_' || to_char(start_date, 'YYYY_MM');
    IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = partition_name) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF events FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );
    END IF;
END;
$$ LANGUAGE plpgsql;
```

**Testing:**
- `test_event_insert_creates_partition`: inserting an event auto-creates the monthly partition
- `test_event_properties_jsonb_queryable`: events with `properties @> '{"category": "shoes"}'` are retrievable via GIN index
- `test_users_traits_gin_indexed`: `users` rows with specific `traits` JSONB values are queryable via containment operator
- `test_event_partition_pruning`: `EXPLAIN` on a time-bounded query shows only the relevant partition is scanned

---

### Task 2.2: Event ingestion API

**What:** Build the REST endpoint that receives Segment-spec events, validates them, resolves user identity, and writes to PostgreSQL.

**Design:**
```python
# backend/app/schemas/event.py
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Literal

class SegmentContext(BaseModel):
    ip: str | None = None
    user_agent: str | None = Field(None, alias="userAgent")
    locale: str | None = None
    page: dict | None = None
    campaign: dict | None = None
    library: dict | None = None

class TrackEvent(BaseModel):
    type: Literal["track"] = "track"
    event: str                          # event name
    user_id: str | None = Field(None, alias="userId")
    anonymous_id: str | None = Field(None, alias="anonymousId")
    properties: dict = Field(default_factory=dict)
    context: SegmentContext = Field(default_factory=SegmentContext)
    timestamp: datetime | None = None

class IdentifyEvent(BaseModel):
    type: Literal["identify"] = "identify"
    user_id: str = Field(alias="userId")
    anonymous_id: str | None = Field(None, alias="anonymousId")
    traits: dict = Field(default_factory=dict)
    context: SegmentContext = Field(default_factory=SegmentContext)
    timestamp: datetime | None = None

class PageEvent(BaseModel):
    type: Literal["page"] = "page"
    user_id: str | None = Field(None, alias="userId")
    anonymous_id: str | None = Field(None, alias="anonymousId")
    name: str | None = None
    properties: dict = Field(default_factory=dict)
    context: SegmentContext = Field(default_factory=SegmentContext)
    timestamp: datetime | None = None

class BatchRequest(BaseModel):
    batch: list[TrackEvent | IdentifyEvent | PageEvent]

# backend/app/api/v1/events.py
router = APIRouter(prefix="/events", tags=["events"])

@router.post("/track", status_code=202)
async def track(event: TrackEvent, api_key: ApiKey = Depends(verify_api_key), db=Depends(get_db)):
    """Ingest a single track event."""
    ...

@router.post("/identify", status_code=202)
async def identify(event: IdentifyEvent, api_key: ApiKey = Depends(verify_api_key), db=Depends(get_db)):
    """Identify a user and merge anonymous activity."""
    ...

@router.post("/batch", status_code=202)
async def batch(request: BatchRequest, api_key: ApiKey = Depends(verify_api_key), db=Depends(get_db)):
    """Ingest a batch of events (up to 500)."""
    ...
```

```python
# backend/app/services/event_service.py
class EventService:
    async def ingest_track(self, tenant_id: UUID, event: TrackEvent, db: AsyncSession) -> None:
        """Resolve user, write event, update user last_seen_at, update event schema registry."""
        user = await self._resolve_user(tenant_id, event.user_id, event.anonymous_id, db)
        await self._ensure_partition(event.timestamp or datetime.utcnow(), db)
        db_event = Event(
            tenant_id=tenant_id,
            event_type="track",
            event_name=event.event,
            user_id=user.id if user else None,
            anonymous_id=event.anonymous_id,
            timestamp=event.timestamp or datetime.utcnow(),
            properties=event.properties,
            context=event.context.model_dump(),
            # extract promoted fields from context
            utm_source=event.context.campaign.get("source") if event.context.campaign else None,
            device_type=self._parse_device_type(event.context.user_agent),
        )
        db.add(db_event)
        await self._update_event_schema(tenant_id, event.event, event.properties, db)
        await db.commit()

    async def ingest_identify(self, tenant_id: UUID, event: IdentifyEvent, db: AsyncSession) -> None:
        """Create or update user, merge anonymous_id if provided."""
        ...

    async def _resolve_user(self, tenant_id: UUID, user_id: str | None, anonymous_id: str | None, db: AsyncSession) -> TrackedUser | None:
        """Find existing user by external_id or anonymous_id, or create new."""
        ...
```

**Testing:**
- `test_track_event_returns_202`: POST `/api/v1/events/track` with valid API key and event body returns 202
- `test_track_event_persists_to_db`: after POSTing a track event, querying `events` table returns the event with correct `event_name` and `properties`
- `test_identify_creates_user`: POST `/api/v1/events/identify` with `userId` and `traits` creates a row in `users`
- `test_identify_merges_anonymous`: POST identify with both `userId` and `anonymousId` adds the anonymous_id to `users.anonymous_ids`
- `test_batch_ingests_multiple_events`: POST `/api/v1/events/batch` with 3 events creates 3 event rows
- `test_batch_rejects_over_500`: batch with 501 events returns 422
- `test_invalid_api_key_returns_401`: request with wrong API key returns 401
- `test_event_schema_auto_discovered`: after first track event with `event="Purchase"`, `event_schemas` table has a row for that event name

---

### Task 2.3: JavaScript SDK

**What:** Build a lightweight browser SDK implementing Segment-compatible `.track()`, `.identify()`, `.page()` methods with automatic batching.

**Design:**
```typescript
// sdk/javascript/src/index.ts
export interface CCAConfig {
  writeKey: string;            // API key
  host?: string;               // default: https://api.cohortanalytics.dev
  flushAt?: number;            // batch size trigger (default: 20)
  flushInterval?: number;      // ms between flushes (default: 10000)
}

export interface TrackParams {
  event: string;
  properties?: Record<string, unknown>;
  userId?: string;
  anonymousId?: string;
  timestamp?: string;          // ISO 8601
}

export interface IdentifyParams {
  userId: string;
  traits?: Record<string, unknown>;
  anonymousId?: string;
}

export class CohortAnalytics {
  private queue: Array<Record<string, unknown>> = [];
  private anonymousId: string;
  private userId: string | null = null;
  private config: Required<CCAConfig>;

  constructor(config: CCAConfig) {
    this.config = {
      host: "https://api.cohortanalytics.dev",
      flushAt: 20,
      flushInterval: 10000,
      ...config,
    };
    this.anonymousId = this.getOrCreateAnonymousId();
    this.startFlushTimer();
  }

  track(params: TrackParams): void {
    this.enqueue({
      type: "track",
      event: params.event,
      properties: params.properties ?? {},
      userId: params.userId ?? this.userId,
      anonymousId: params.anonymousId ?? this.anonymousId,
      timestamp: params.timestamp ?? new Date().toISOString(),
      context: this.buildContext(),
    });
  }

  identify(params: IdentifyParams): void {
    this.userId = params.userId;
    this.enqueue({
      type: "identify",
      userId: params.userId,
      anonymousId: params.anonymousId ?? this.anonymousId,
      traits: params.traits ?? {},
      timestamp: new Date().toISOString(),
      context: this.buildContext(),
    });
  }

  page(name?: string, properties?: Record<string, unknown>): void {
    this.enqueue({
      type: "page",
      name: name ?? document.title,
      properties: { url: location.href, title: document.title, referrer: document.referrer, ...properties },
      userId: this.userId,
      anonymousId: this.anonymousId,
      timestamp: new Date().toISOString(),
      context: this.buildContext(),
    });
  }

  private async flush(): Promise<void> {
    if (this.queue.length === 0) return;
    const batch = this.queue.splice(0, this.config.flushAt);
    await fetch(`${this.config.host}/api/v1/events/batch`, {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${this.config.writeKey}` },
      body: JSON.stringify({ batch }),
    });
  }

  private buildContext(): Record<string, unknown> {
    return {
      userAgent: navigator.userAgent,
      locale: navigator.language,
      page: { url: location.href, title: document.title, referrer: document.referrer },
      library: { name: "cca-js", version: "0.1.0" },
    };
  }

  private getOrCreateAnonymousId(): string {
    const key = "cca_anonymous_id";
    let id = localStorage.getItem(key);
    if (!id) { id = crypto.randomUUID(); localStorage.setItem(key, id); }
    return id;
  }

  private startFlushTimer(): void {
    setInterval(() => this.flush(), this.config.flushInterval);
  }
}
```

**Testing:**
- `test_track_enqueues_event`: calling `.track()` adds an event to the internal queue
- `test_identify_sets_user_id`: after `.identify({userId: "u1"})`, subsequent `.track()` events include `userId: "u1"`
- `test_anonymous_id_persisted`: anonymous ID stored in localStorage and reused across instantiations
- `test_flush_sends_batch`: when queue reaches `flushAt`, a POST to `/api/v1/events/batch` is fired
- `test_page_captures_url_and_title`: `.page()` includes current URL, title, and referrer in properties
- `test_context_includes_library_version`: every event's `context.library` has `name: "cca-js"`

---

## Phase 3: Cohort Engine

### Definition of Done
- Users can create behavioral cohorts via API with AND/OR conditions over events and user traits
- Dynamic cohorts auto-recompute membership on a schedule
- Cohort membership is queryable and filterable across all analytics endpoints
- Static (manual) cohorts support explicit user addition/removal

### Task 3.1: Cohort data model

**What:** Create the cohorts, cohort_memberships, and discovered_cohorts tables.

**Design:**
```sql
CREATE TABLE cohorts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(50) NOT NULL,
    is_dynamic      BOOLEAN NOT NULL DEFAULT true,
    definition      JSONB NOT NULL DEFAULT '{}',
    member_count    INTEGER DEFAULT 0,
    last_computed   TIMESTAMPTZ,
    created_by      UUID,
    is_archived     BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE cohort_memberships (
    tenant_id       UUID NOT NULL,
    cohort_id       UUID NOT NULL REFERENCES cohorts(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    PRIMARY KEY (tenant_id, cohort_id, user_id)
);
CREATE INDEX idx_cohort_members_active ON cohort_memberships (tenant_id, cohort_id) WHERE exited_at IS NULL;
```

```python
# backend/app/schemas/cohort.py
from pydantic import BaseModel
from typing import Literal

class CohortCondition(BaseModel):
    type: Literal["did_event", "not_did_event", "user_trait", "cohort_membership"]
    event: str | None = None
    time_window: str | None = None           # "30d", "7d", "90d"
    count_operator: str | None = None        # ">=", "<=", "=", ">"
    count: int | None = None
    property_filters: list[dict] | None = None
    key: str | None = None                   # for user_trait
    op: str | None = None                    # "=", "!=", ">", "<", "contains"
    value: str | int | float | bool | None = None

class CohortDefinition(BaseModel):
    operator: Literal["AND", "OR"] = "AND"
    conditions: list[CohortCondition]

class CohortCreate(BaseModel):
    name: str
    description: str | None = None
    type: Literal["behavioral", "temporal", "demographic"] = "behavioral"
    is_dynamic: bool = True
    definition: CohortDefinition
```

**Testing:**
- `test_create_cohort_stores_definition`: POST cohort with conditions saves JSONB definition correctly
- `test_cohort_definition_validates`: invalid condition type rejected with 422
- `test_cohort_membership_unique_constraint`: duplicate `(tenant_id, cohort_id, user_id)` raises error
- `test_cascade_delete_removes_memberships`: deleting cohort removes all its memberships

---

### Task 3.2: Cohort evaluation engine

**What:** Build the query engine that translates cohort JSONB definitions into SQL queries and populates cohort_memberships.

**Design:**
```python
# backend/app/services/cohort_service.py
class CohortEvaluator:
    """Translates a CohortDefinition into a SQL query that returns matching user IDs."""

    def build_query(self, tenant_id: UUID, definition: CohortDefinition) -> TextClause:
        """Convert cohort definition to SQL.

        Example output for definition:
          AND(did_event("Purchase", 30d, >= 1), user_trait("plan_tier", "=", "pro"))

        SELECT DISTINCT u.id FROM users u
        WHERE u.tenant_id = :tenant_id
          AND EXISTS (
            SELECT 1 FROM events e
            WHERE e.tenant_id = :tenant_id AND e.user_id = u.id
              AND e.event_name = 'Purchase'
              AND e.timestamp >= now() - interval '30 days'
            HAVING count(*) >= 1
          )
          AND u.traits @> '{"plan_tier": "pro"}'::jsonb
        """
        ...

    async def evaluate_cohort(self, cohort_id: UUID, db: AsyncSession) -> int:
        """Evaluate cohort, update memberships (add new, exit removed), return member count."""
        cohort = await db.get(Cohort, cohort_id)
        definition = CohortDefinition.model_validate(cohort.definition)
        query = self.build_query(cohort.tenant_id, definition)

        # Get current matching users
        result = await db.execute(query, {"tenant_id": cohort.tenant_id})
        matching_user_ids = set(row[0] for row in result.fetchall())

        # Get current memberships
        current_members = await self._get_current_members(cohort.tenant_id, cohort_id, db)

        # Add new members
        new_members = matching_user_ids - current_members
        for user_id in new_members:
            db.add(CohortMembership(tenant_id=cohort.tenant_id, cohort_id=cohort_id, user_id=user_id))

        # Exit removed members
        exited_members = current_members - matching_user_ids
        await self._exit_members(cohort.tenant_id, cohort_id, exited_members, db)

        # Update cohort metadata
        cohort.member_count = len(matching_user_ids)
        cohort.last_computed = datetime.utcnow()
        await db.commit()
        return len(matching_user_ids)
```

**Testing:**
- `test_did_event_condition_matches`: cohort with `did_event("Purchase", "30d", >= 1)` matches users who have Purchase events in last 30 days
- `test_not_did_event_condition_excludes`: `not_did_event("Login", "7d")` excludes users who logged in within 7 days
- `test_user_trait_condition_matches`: `user_trait("plan_tier", "=", "pro")` matches users with that trait
- `test_and_operator_requires_all`: AND of two conditions only matches users satisfying both
- `test_or_operator_matches_either`: OR of two conditions matches users satisfying at least one
- `test_evaluate_adds_new_members`: after evaluation, new matching users appear in `cohort_memberships`
- `test_evaluate_exits_removed_members`: users no longer matching get `exited_at` set
- `test_evaluate_updates_member_count`: `cohort.member_count` reflects actual membership

---

### Task 3.3: Cohort CRUD API

**What:** Build REST endpoints for creating, listing, updating, and deleting cohorts, plus querying membership.

**Design:**
```python
# backend/app/api/v1/cohorts.py
router = APIRouter(prefix="/cohorts", tags=["cohorts"])

@router.post("/", response_model=CohortResponse, status_code=201)
async def create_cohort(body: CohortCreate, ...): ...

@router.get("/", response_model=list[CohortSummary])
async def list_cohorts(type: str | None = None, ...): ...

@router.get("/{cohort_id}", response_model=CohortDetailResponse)
async def get_cohort(cohort_id: UUID, ...): ...

@router.put("/{cohort_id}", response_model=CohortResponse)
async def update_cohort(cohort_id: UUID, body: CohortUpdate, ...): ...

@router.delete("/{cohort_id}", status_code=204)
async def delete_cohort(cohort_id: UUID, ...): ...

@router.post("/{cohort_id}/evaluate", response_model=CohortEvaluationResult)
async def evaluate_cohort(cohort_id: UUID, ...):
    """Trigger immediate cohort evaluation and return member count."""
    ...

@router.get("/{cohort_id}/members", response_model=PaginatedResponse[CohortMemberResponse])
async def list_cohort_members(cohort_id: UUID, page: int = 1, size: int = 50, ...): ...
```

**Testing:**
- `test_create_cohort_returns_201`: POST with valid definition returns created cohort with ID
- `test_list_cohorts_filters_by_type`: `GET /cohorts?type=behavioral` only returns behavioral cohorts
- `test_evaluate_returns_member_count`: POST `/cohorts/{id}/evaluate` returns `{"member_count": N}`
- `test_list_members_paginated`: GET `/cohorts/{id}/members?page=1&size=10` returns first 10 members
- `test_delete_cohort_returns_204`: DELETE removes cohort and its memberships
- `test_cross_tenant_isolation`: cohorts from tenant A are not visible to tenant B

---

## Phase 4: Retention Analysis & Funnel Engine

### Definition of Done
- Day-N retention curves computed and cached for configurable time periods
- Funnel analysis calculates step-by-step conversion rates within a time window
- Both retention and funnel results are filterable by any cohort
- Background worker keeps retention data fresh

### Task 4.1: Daily activity rollup worker

**What:** Build a background worker that aggregates raw events into daily_user_activity summary rows for fast retention queries.

**Design:**
```python
# backend/app/workers/retention_computer.py
class DailyActivityRollupWorker:
    """Aggregates events into daily_user_activity. Runs hourly via Celery/APScheduler."""

    async def rollup_day(self, tenant_id: UUID, target_date: date, db: AsyncSession) -> int:
        """Aggregate events for a specific tenant + date into daily_user_activity."""
        query = text("""
            INSERT INTO daily_user_activity (tenant_id, user_id, activity_date, event_count,
                                             session_count, distinct_events, event_names,
                                             first_event_at, last_event_at)
            SELECT
                tenant_id, user_id, :target_date,
                count(*),
                count(DISTINCT session_id),
                count(DISTINCT event_name),
                array_agg(DISTINCT event_name),
                min(timestamp),
                max(timestamp)
            FROM events
            WHERE tenant_id = :tenant_id
              AND timestamp >= :day_start AND timestamp < :day_end
              AND user_id IS NOT NULL
            GROUP BY tenant_id, user_id
            ON CONFLICT (tenant_id, user_id, activity_date)
            DO UPDATE SET
                event_count = EXCLUDED.event_count,
                session_count = EXCLUDED.session_count,
                distinct_events = EXCLUDED.distinct_events,
                event_names = EXCLUDED.event_names,
                first_event_at = EXCLUDED.first_event_at,
                last_event_at = EXCLUDED.last_event_at
        """)
        result = await db.execute(query, {
            "tenant_id": tenant_id,
            "target_date": target_date,
            "day_start": datetime.combine(target_date, time.min, tzinfo=timezone.utc),
            "day_end": datetime.combine(target_date + timedelta(days=1), time.min, tzinfo=timezone.utc),
        })
        await db.commit()
        return result.rowcount
```

**Testing:**
- `test_rollup_aggregates_events`: given 10 events for user A on 2026-05-01, rollup creates one `daily_user_activity` row with `event_count=10`
- `test_rollup_counts_distinct_sessions`: 3 events across 2 sessions produce `session_count=2`
- `test_rollup_idempotent`: running rollup twice for the same date produces the same result (upsert)
- `test_rollup_ignores_anonymous_events`: events with `user_id IS NULL` are excluded

---

### Task 4.2: Retention analysis engine

**What:** Compute Day-N retention curves using daily_user_activity, with configurable start/return events and cohort filtering.

**Design:**
```python
# backend/app/services/retention_service.py
from dataclasses import dataclass

@dataclass
class RetentionRow:
    cohort_period: date
    cohort_size: int
    period_offsets: dict[int, float]  # {0: 1.0, 1: 0.45, 7: 0.32, 30: 0.18}

class RetentionService:
    async def compute_retention(
        self,
        tenant_id: UUID,
        start_event: str,        # e.g., "Signed Up"
        return_event: str,       # e.g., "Logged In" (or "any" for any activity)
        granularity: str,        # "day", "week", "month"
        lookback_periods: int,   # number of periods to compute
        cohort_id: UUID | None,  # optional cohort filter
        db: AsyncSession,
    ) -> list[RetentionRow]:
        """Compute retention matrix. Returns one RetentionRow per cohort period."""
        ...

    async def save_retention_data(
        self, tenant_id: UUID, report_id: UUID, rows: list[RetentionRow], db: AsyncSession
    ) -> None:
        """Persist computed retention data to retention_data table for caching."""
        ...

# API endpoint
@router.get("/retention/{report_id}/data", response_model=RetentionMatrixResponse)
async def get_retention_data(report_id: UUID, ...):
    """Return cached retention matrix for a report."""
    ...

@router.post("/retention/{report_id}/compute", response_model=RetentionMatrixResponse)
async def compute_retention(report_id: UUID, ...):
    """Force recomputation of retention data."""
    ...
```

**Testing:**
- `test_day1_retention_correct`: 100 signups, 45 return on day 1 -> `retention_rate = 0.45`
- `test_week_granularity_groups_by_week`: weekly granularity groups signup cohorts by ISO week
- `test_cohort_filter_applied`: retention computed only for users in specified cohort
- `test_retention_data_cached`: second call to GET returns cached data without recomputation
- `test_empty_cohort_returns_zeros`: cohort period with no signups returns `cohort_size=0`

---

### Task 4.3: Funnel analysis engine

**What:** Compute step-by-step conversion rates for ordered event sequences within a time window, optionally filtered by cohort.

**Design:**
```python
# backend/app/services/funnel_service.py
@dataclass
class FunnelStep:
    order: int
    event_name: str
    entered_count: int
    converted_count: int
    conversion_rate: float         # from previous step
    overall_conversion_rate: float # from step 1
    median_time_to_convert: timedelta | None

class FunnelService:
    async def analyze_funnel(
        self,
        tenant_id: UUID,
        steps: list[dict],          # [{"event": "View", "filters": {}}, {"event": "Purchase"}]
        time_window: timedelta,     # max time to complete funnel
        date_range: tuple[date, date],
        cohort_id: UUID | None,
        db: AsyncSession,
    ) -> list[FunnelStep]:
        """Compute funnel conversion. Uses window function to find ordered event sequences per user."""
        ...

# API
@router.post("/funnels/{funnel_id}/analyze", response_model=FunnelAnalysisResponse)
async def analyze_funnel(funnel_id: UUID, date_from: date, date_to: date, cohort_id: UUID | None = None, ...): ...
```

**Testing:**
- `test_3_step_funnel_conversion`: 100 -> 60 -> 30 produces step rates [1.0, 0.6, 0.5] and overall [1.0, 0.6, 0.3]
- `test_time_window_enforced`: user who completes step 2 after the time window is not counted as converted
- `test_funnel_with_cohort_filter`: only users in specified cohort are included
- `test_median_time_to_convert`: median time between step 1 and step 2 is calculated correctly
- `test_funnel_with_property_filter_on_step`: step filter `{"properties.price": {">": 50}}` narrows step match

---

## Phase 5: Dashboards & Scheduled Reporting

### Definition of Done
- Users can create dashboards with configurable widget layouts
- Widget types: retention heatmap, funnel chart, cohort table, churn risk distribution
- Scheduled reports deliver dashboard snapshots via Slack webhook and email
- Dashboard API supports sharing via public link (read-only token)

### Task 5.1: Dashboard CRUD and widget system

**What:** Build the dashboard and widget data model and REST API, supporting grid-based layout management.

**Design:**
```python
# backend/app/schemas/dashboard.py
class WidgetCreate(BaseModel):
    widget_type: Literal["retention_chart", "funnel_chart", "cohort_table",
                          "churn_heatmap", "rfm_grid", "metric_card", "trend_line"]
    title: str
    config: dict  # widget-type-specific configuration

class DashboardCreate(BaseModel):
    name: str
    description: str | None = None
    widgets: list[WidgetCreate] = []

class DashboardLayout(BaseModel):
    widget_id: str
    x: int
    y: int
    w: int   # width in grid units (1-12)
    h: int   # height in grid units

# API endpoints
@router.post("/dashboards", response_model=DashboardResponse, status_code=201)
@router.get("/dashboards", response_model=list[DashboardSummary])
@router.get("/dashboards/{id}", response_model=DashboardDetailResponse)
@router.put("/dashboards/{id}/layout", response_model=DashboardDetailResponse)
@router.post("/dashboards/{id}/widgets", response_model=WidgetResponse, status_code=201)
@router.delete("/dashboards/{id}/widgets/{widget_id}", status_code=204)
```

**Testing:**
- `test_create_dashboard_with_widgets`: POST creates dashboard and nested widgets
- `test_update_layout`: PUT layout updates widget positions
- `test_add_widget_to_existing_dashboard`: POST widget to dashboard returns widget with ID
- `test_delete_widget_cascade`: deleting dashboard deletes its widgets

---

### Task 5.2: Widget data resolution

**What:** Build the service that resolves each widget type into rendered data by querying the appropriate analytics engine.

**Design:**
```python
# backend/app/services/dashboard_service.py
class WidgetDataResolver:
    async def resolve(self, widget: DashboardWidget, db: AsyncSession) -> dict:
        match widget.widget_type:
            case "retention_chart":
                return await self._resolve_retention(widget.config, db)
            case "funnel_chart":
                return await self._resolve_funnel(widget.config, db)
            case "cohort_table":
                return await self._resolve_cohort_table(widget.config, db)
            case "churn_heatmap":
                return await self._resolve_churn_heatmap(widget.config, db)
            case "metric_card":
                return await self._resolve_metric(widget.config, db)

@router.get("/dashboards/{id}/data", response_model=DashboardDataResponse)
async def get_dashboard_data(id: UUID, ...):
    """Resolve all widgets and return their data."""
    ...
```

**Testing:**
- `test_retention_widget_returns_matrix`: retention_chart widget returns `{periods: [...], data: [[...]]}`
- `test_funnel_widget_returns_steps`: funnel_chart widget returns step conversion data
- `test_unknown_widget_type_errors`: unrecognized widget type returns 400

---

### Task 5.3: Scheduled report delivery

**What:** Implement cron-based scheduled delivery of dashboard snapshots to Slack and email.

**Design:**
```python
# backend/app/workers/report_sender.py
class ReportSender:
    async def send_scheduled_report(self, report_id: UUID, db: AsyncSession) -> None:
        report = await db.get(ScheduledReport, report_id)
        dashboard_data = await widget_resolver.resolve_all(report.dashboard_id, db)
        rendered = self._render_summary(dashboard_data)

        match report.delivery.get("channel"):
            case "slack":
                await self._send_slack(report.delivery["slack_webhook_url"], rendered)
            case "email":
                await self._send_email(report.delivery["recipients"], rendered)

    async def _send_slack(self, webhook_url: str, content: dict) -> None:
        """POST Slack Block Kit message to webhook URL."""
        ...

    async def _send_email(self, recipients: list[str], content: dict) -> None:
        """Send HTML email summary via SMTP or SendGrid."""
        ...
```

**Testing:**
- `test_slack_delivery_posts_to_webhook`: mocked Slack webhook receives Block Kit message
- `test_email_delivery_sends_to_recipients`: mocked SMTP receives email with dashboard summary
- `test_inactive_report_not_sent`: report with `is_active=false` is skipped
- `test_cron_schedule_triggers_correctly`: report with `schedule_cron="0 9 * * 1"` triggers on Monday 9 AM

---

## Phase 6: ML Churn Prediction Pipeline

### Definition of Done
- Feature engineering worker computes rolling behavioral features for all users nightly
- XGBoost churn model trains on tenant's own data with configurable prediction horizons (30d, 60d, 90d)
- Trained models are versioned and stored in S3; active model flag controls which model serves predictions
- Batch prediction runs nightly, generating per-user churn probability scores with risk tiers

### Task 6.1: Feature engineering pipeline

**What:** Build the worker that computes ML features from daily_user_activity and events, writing to ml_feature_snapshots.

**Design:**
```python
# backend/app/ml/feature_engineering.py
from dataclasses import dataclass

@dataclass
class UserFeatures:
    user_id: uuid.UUID
    events_7d: int
    events_30d: int
    events_90d: int
    sessions_7d: int
    sessions_30d: int
    days_active_30d: int
    days_since_last_activity: int
    avg_session_duration_30d: float | None
    distinct_features_used_30d: int
    event_trend_7d_vs_30d: float | None       # events_7d / (events_30d / 4.28)
    session_trend_7d_vs_30d: float | None
    days_since_signup: int
    account_id: uuid.UUID | None
    account_user_count: int | None
    account_active_user_pct: float | None

class FeatureEngineer:
    FEATURE_SQL = """
    WITH user_activity AS (
        SELECT
            u.id AS user_id,
            u.first_seen_at,
            COALESCE(SUM(CASE WHEN da.activity_date >= :ref_date - 7 THEN da.event_count END), 0) AS events_7d,
            COALESCE(SUM(CASE WHEN da.activity_date >= :ref_date - 30 THEN da.event_count END), 0) AS events_30d,
            COALESCE(SUM(da.event_count), 0) AS events_90d,
            COALESCE(SUM(CASE WHEN da.activity_date >= :ref_date - 7 THEN da.session_count END), 0) AS sessions_7d,
            COALESCE(SUM(CASE WHEN da.activity_date >= :ref_date - 30 THEN da.session_count END), 0) AS sessions_30d,
            COUNT(DISTINCT CASE WHEN da.activity_date >= :ref_date - 30 THEN da.activity_date END) AS days_active_30d,
            :ref_date - MAX(da.activity_date)::date AS days_since_last_activity,
            :ref_date - u.first_seen_at::date AS days_since_signup
        FROM users u
        LEFT JOIN daily_user_activity da ON u.id = da.user_id
            AND da.tenant_id = :tenant_id
            AND da.activity_date >= :ref_date - 90
        WHERE u.tenant_id = :tenant_id
        GROUP BY u.id, u.first_seen_at
    )
    SELECT * FROM user_activity
    """

    async def compute_features(self, tenant_id: UUID, ref_date: date, db: AsyncSession) -> int:
        """Compute features for all users in a tenant and write to ml_feature_snapshots."""
        ...
```

**Testing:**
- `test_events_7d_counts_correct_window`: user with 5 events in last 7 days gets `events_7d=5`
- `test_event_trend_calculation`: `events_7d=10`, `events_30d=40` -> trend = `10 / (40/4.28) = 1.07`
- `test_days_since_last_activity`: user last active 5 days ago gets `days_since_last_activity=5`
- `test_no_activity_user_gets_zeros`: user with no events gets all zero counts
- `test_features_written_to_table`: after computation, `ml_feature_snapshots` has one row per user

---

### Task 6.2: Churn model training

**What:** Train an XGBoost binary classifier for churn prediction using historical feature snapshots and activity labels.

**Design:**
```python
# backend/app/ml/churn_model.py
import xgboost as xgb
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score, precision_score, recall_score, f1_score
import joblib

@dataclass
class TrainingResult:
    model_version: int
    algorithm: str
    hyperparameters: dict
    metrics: dict              # {"auc_roc": 0.93, "precision": 0.84, "recall": 0.78, "f1": 0.81}
    feature_names: list[str]
    training_rows: int
    artifact_path: str         # S3 path

class ChurnModelTrainer:
    DEFAULT_PARAMS = {
        "n_estimators": 500,
        "max_depth": 6,
        "learning_rate": 0.05,
        "subsample": 0.8,
        "colsample_bytree": 0.8,
        "objective": "binary:logistic",
        "eval_metric": "auc",
        "use_label_encoder": False,
    }

    FEATURE_COLUMNS = [
        "events_7d", "events_30d", "events_90d",
        "sessions_7d", "sessions_30d", "days_active_30d",
        "days_since_last_activity", "days_since_signup",
        "event_trend_7d_vs_30d", "session_trend_7d_vs_30d",
        "distinct_features_used_30d",
    ]

    async def train(self, tenant_id: UUID, prediction_horizon_days: int, db: AsyncSession) -> TrainingResult:
        """
        1. Load historical feature snapshots with churn labels
        2. Split 80/20 train/test
        3. Train XGBoost model
        4. Evaluate on test set
        5. Save model artifact to S3
        6. Register in ml_models table
        """
        ...

    def _label_churned_users(self, features_df: pd.DataFrame, horizon_days: int) -> pd.Series:
        """Label: user churned if they had zero activity in the `horizon_days` after feature snapshot date."""
        ...
```

**Testing:**
- `test_training_returns_metrics`: training produces `auc_roc > 0.5` (better than random)
- `test_model_saved_to_s3`: after training, model artifact exists at `s3://cca-models/{tenant_id}/{model_version}.joblib`
- `test_model_registered_in_db`: `ml_models` table has new row with correct `algorithm`, `hyperparameters`, `training_metrics`
- `test_only_one_active_model`: setting a new model as active deactivates the previous one
- `test_insufficient_data_raises`: training with fewer than 100 labeled users raises `InsufficientTrainingDataError`

---

### Task 6.3: Batch prediction runner

**What:** Run the active churn model against current feature snapshots for all users, writing churn_predictions with risk tiers.

**Design:**
```python
# backend/app/ml/prediction_runner.py
class PredictionRunner:
    RISK_TIERS = {
        (0.8, 1.0): "critical",
        (0.6, 0.8): "high",
        (0.3, 0.6): "medium",
        (0.0, 0.3): "low",
    }

    async def run_predictions(self, tenant_id: UUID, db: AsyncSession) -> int:
        """
        1. Load active model from S3
        2. Load latest feature snapshots for all users
        3. Run batch inference
        4. Classify risk tiers
        5. Write to churn_predictions table
        Returns: number of predictions generated
        """
        model_record = await self._get_active_model(tenant_id, db)
        model = joblib.load(self._download_model(model_record.artifact_path))
        features_df = await self._load_features(tenant_id, db)
        probabilities = model.predict_proba(features_df[self.FEATURE_COLUMNS])[:, 1]
        ...
```

**Testing:**
- `test_predictions_written_for_all_users`: after running, every user with features has a prediction row
- `test_risk_tier_classification`: probability 0.85 -> "critical", 0.65 -> "high", 0.4 -> "medium", 0.1 -> "low"
- `test_prediction_date_set_correctly`: predictions have today's date
- `test_no_active_model_skips`: if no active model exists, runner logs warning and returns 0
- `test_prediction_upsert_idempotent`: running twice for the same date updates existing rows

---

## Phase 7: XAI Explainability (SHAP)

### Definition of Done
- Every churn prediction includes top 5 SHAP feature attributions
- Each attribution has a plain-English explanation generated from a template
- API returns predictions with embedded explanations in a single response
- Explanations indicate direction (increases_risk / decreases_risk) and rank

### Task 7.1: SHAP explainer integration

**What:** Integrate SHAP TreeExplainer with the XGBoost churn model to generate per-user feature attributions.

**Design:**
```python
# backend/app/ml/shap_explainer.py
import shap
import numpy as np

@dataclass
class FeatureExplanation:
    rank: int
    feature: str
    shap_value: float
    feature_value: float | int
    direction: str                 # "increases_risk" or "decreases_risk"
    plain_english: str             # "No login for 45 days (cohort avg: 3 days)"

class ShapExplainer:
    TEMPLATES = {
        "days_since_last_activity": "No activity for {value} days (cohort avg: {avg:.0f} days)",
        "events_7d": "Only {value} events in last 7 days (cohort avg: {avg:.0f})",
        "events_30d": "{value} events in last 30 days (cohort avg: {avg:.0f})",
        "sessions_7d": "{value} sessions in last 7 days (cohort avg: {avg:.0f})",
        "event_trend_7d_vs_30d": "Activity trend is {pct:.0%} of 30-day average",
        "session_trend_7d_vs_30d": "Session frequency {change} {pct:.0%}",
        "days_since_signup": "Account is {value} days old",
        "days_active_30d": "Active {value} of last 30 days (cohort avg: {avg:.0f})",
    }

    def __init__(self, model, feature_names: list[str]):
        self.explainer = shap.TreeExplainer(model)
        self.feature_names = feature_names

    def explain(self, features: np.ndarray, cohort_averages: dict[str, float], top_k: int = 5) -> list[FeatureExplanation]:
        """Generate top-K SHAP explanations for a single user's feature vector."""
        shap_values = self.explainer.shap_values(features.reshape(1, -1))[0]
        indices = np.argsort(np.abs(shap_values))[::-1][:top_k]

        explanations = []
        for rank, idx in enumerate(indices, 1):
            feature = self.feature_names[idx]
            value = features[idx]
            sv = shap_values[idx]
            direction = "increases_risk" if sv > 0 else "decreases_risk"
            text = self._generate_text(feature, value, sv, cohort_averages)
            explanations.append(FeatureExplanation(
                rank=rank, feature=feature, shap_value=round(float(sv), 6),
                feature_value=value, direction=direction, plain_english=text
            ))
        return explanations

    def _generate_text(self, feature: str, value: float, shap_value: float, averages: dict) -> str:
        template = self.TEMPLATES.get(feature, "{feature} = {value}")
        avg = averages.get(feature, 0)
        ...
```

**Testing:**
- `test_shap_returns_top_5`: `explain()` returns exactly 5 explanations ranked by |SHAP value|
- `test_direction_correct`: positive SHAP value -> "increases_risk", negative -> "decreases_risk"
- `test_plain_english_generated`: explanation for `days_since_last_activity=45` produces "No activity for 45 days..."
- `test_rank_ordering`: explanations are sorted by descending |SHAP value|
- `test_explanations_stored_as_jsonb`: after prediction run, `churn_predictions.explanations` JSONB contains the array

---

### Task 7.2: Prediction API with explanations

**What:** Build API endpoints that return churn predictions with embedded SHAP explanations.

**Design:**
```python
# backend/app/api/v1/predictions.py
router = APIRouter(prefix="/predictions", tags=["predictions"])

@router.get("/churn/users", response_model=PaginatedResponse[ChurnPredictionResponse])
async def list_churn_predictions(
    risk_tier: str | None = None,     # filter: critical, high, medium, low
    cohort_id: UUID | None = None,    # filter by cohort membership
    sort_by: str = "churn_probability_30d",
    order: str = "desc",
    page: int = 1,
    size: int = 50,
    ...
): ...

@router.get("/churn/users/{user_id}", response_model=ChurnPredictionDetailResponse)
async def get_user_churn_prediction(user_id: UUID, ...):
    """Return churn prediction for a specific user with full SHAP explanations."""
    ...

@router.get("/churn/summary", response_model=ChurnSummaryResponse)
async def get_churn_summary(...):
    """Return aggregate churn risk distribution: {critical: N, high: N, medium: N, low: N}."""
    ...

# Response schemas
class ChurnPredictionResponse(BaseModel):
    user_id: str
    external_id: str | None
    email: str | None
    churn_probability_30d: float
    churn_probability_60d: float | None
    churn_probability_90d: float | None
    risk_tier: str
    top_reasons: list[str]           # plain_english from top 3 explanations
    prediction_date: date

class ChurnPredictionDetailResponse(ChurnPredictionResponse):
    explanations: list[FeatureExplanationResponse]  # full SHAP details
    model_version: int
    model_algorithm: str
    model_auc_roc: float
```

**Testing:**
- `test_list_predictions_filters_by_risk`: `?risk_tier=critical` only returns critical predictions
- `test_list_predictions_filters_by_cohort`: `?cohort_id=X` only returns predictions for cohort members
- `test_user_prediction_includes_explanations`: GET for specific user includes full explanations array
- `test_summary_returns_tier_counts`: summary endpoint returns `{"critical": 12, "high": 45, "medium": 120, "low": 823}`
- `test_top_reasons_are_plain_english`: `top_reasons` field contains human-readable strings

---

## Phase 8: Natural Language Query Interface

### Definition of Done
- Users can define cohorts, ask retention questions, and explore data using natural language
- NL queries are translated to SQL via Claude API, executed safely, and results returned
- Query history is stored with user feedback for quality improvement
- Guardrails prevent SQL injection and limit query scope to the user's tenant

### Task 8.1: NL-to-SQL translation engine

**What:** Build the service that translates natural language cohort/analytics queries into tenant-scoped SQL using Claude API.

**Design:**
```python
# backend/app/services/nl_service.py
from anthropic import Anthropic

class NLQueryService:
    SYSTEM_PROMPT = """You are a SQL analyst for a customer cohort analytics platform.
    Translate the user's natural language query into a PostgreSQL query.
    
    Available tables and schemas:
    - users (id UUID, tenant_id UUID, external_id, email, display_name, traits JSONB, plan_tier, signup_source, country_code, first_seen_at, last_seen_at)
    - events (id UUID, tenant_id UUID, event_type, event_name, user_id UUID, timestamp TIMESTAMPTZ, properties JSONB, session_id, utm_source, utm_campaign, device_type, country_code)
    - daily_user_activity (tenant_id, user_id, activity_date DATE, event_count, session_count, distinct_events)
    - cohort_memberships (tenant_id, cohort_id, user_id, entered_at, exited_at)
    - churn_predictions (tenant_id, user_id, prediction_date, churn_probability_30d, risk_tier, explanations JSONB)
    
    Rules:
    1. ALWAYS include WHERE tenant_id = :tenant_id
    2. NEVER use DELETE, UPDATE, INSERT, DROP, ALTER, or TRUNCATE
    3. ALWAYS use parameterized queries with :param syntax
    4. Limit results to 1000 rows maximum
    5. For JSONB property access, use ->> operator for text, cast for numbers
    6. Return the SQL query and a brief description of what it does
    """

    def __init__(self):
        self.client = Anthropic()

    async def translate(self, query_text: str, tenant_id: UUID) -> NLQueryResult:
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2000,
            system=self.SYSTEM_PROMPT,
            messages=[{"role": "user", "content": query_text}],
        )
        sql = self._extract_sql(response.content[0].text)
        self._validate_sql(sql)  # reject mutations, enforce tenant_id
        return NLQueryResult(sql=sql, description=self._extract_description(response.content[0].text))

    def _validate_sql(self, sql: str) -> None:
        """Reject any SQL containing mutation keywords. Raise NLQueryValidationError."""
        forbidden = ["DELETE", "UPDATE", "INSERT", "DROP", "ALTER", "TRUNCATE", "GRANT", "REVOKE"]
        upper_sql = sql.upper()
        for keyword in forbidden:
            if keyword in upper_sql:
                raise NLQueryValidationError(f"Query contains forbidden keyword: {keyword}")
        if ":tenant_id" not in sql:
            raise NLQueryValidationError("Query must be scoped to tenant")

    async def execute(self, sql: str, tenant_id: UUID, db: AsyncSession) -> list[dict]:
        """Execute validated SQL with tenant_id parameter binding."""
        result = await db.execute(text(sql), {"tenant_id": tenant_id})
        return [dict(row._mapping) for row in result.fetchmany(1000)]
```

```python
# backend/app/api/v1/nl.py
@router.post("/nl/query", response_model=NLQueryResponse)
async def nl_query(body: NLQueryRequest, ...):
    """Translate NL query to SQL, execute, and return results."""
    ...

@router.post("/nl/query/{query_id}/feedback")
async def submit_feedback(query_id: UUID, body: FeedbackRequest, ...):
    """Record user feedback on query quality."""
    ...
```

**Testing:**
- `test_nl_query_returns_results`: "show me users who signed up this month" returns user rows
- `test_mutation_rejected`: NL input that produces DELETE SQL raises validation error
- `test_tenant_scoping_enforced`: generated SQL without `:tenant_id` is rejected
- `test_query_history_saved`: after query execution, `nl_queries` table has a record with the SQL
- `test_feedback_recorded`: submitting feedback updates `nl_queries.feedback` field
- `test_complex_query_translated`: "users who made a purchase over $100 in the last 30 days but haven't logged in for 2 weeks" produces correct SQL with JSONB property access

---

## Phase 9: Automated Cohort Discovery

### Definition of Done
- Background job analyzes user behavioral data to surface statistically significant cohorts
- Discovery identifies behavioral patterns that correlate with retention or churn
- Discovered cohorts include p-value, effect size, and key distinguishing features
- Users can promote a discovered cohort to a standard cohort for ongoing tracking

### Task 9.1: Statistical cohort discovery engine

**What:** Build the ML pipeline that discovers statistically significant user segments correlated with retention/churn outcomes.

**Design:**
```python
# backend/app/ml/cohort_discovery.py
from scipy import stats
from sklearn.tree import DecisionTreeClassifier
import numpy as np

@dataclass
class DiscoveredSegment:
    name: str                              # AI-generated descriptive name
    description: str                       # plain-English description
    rules: list[dict]                      # [{feature: "events_7d", op: ">=", value: 15}, ...]
    member_count: int
    total_population: int
    retention_rate: float                  # segment's retention rate
    baseline_retention_rate: float         # overall retention rate
    effect_size: float                     # Cohen's h or similar
    p_value: float                         # chi-squared test
    key_features: list[str]               # top distinguishing features

class CohortDiscoveryEngine:
    MIN_SEGMENT_SIZE = 50                  # minimum users to consider
    MAX_P_VALUE = 0.01                     # significance threshold

    async def discover(self, tenant_id: UUID, target_metric: str, db: AsyncSession) -> list[DiscoveredSegment]:
        """
        1. Load feature snapshots + outcome labels (retained vs churned)
        2. Train shallow decision tree to find natural splits
        3. Extract leaf nodes as candidate segments
        4. For each candidate, compute chi-squared significance test
        5. Filter by p-value and minimum size
        6. Generate descriptive names and explanations
        """
        features_df = await self._load_features_with_labels(tenant_id, db)
        tree = DecisionTreeClassifier(max_depth=4, min_samples_leaf=self.MIN_SEGMENT_SIZE)
        tree.fit(features_df[self.FEATURE_COLUMNS], features_df["retained"])

        segments = self._extract_segments(tree, features_df)
        significant = [s for s in segments if s.p_value < self.MAX_P_VALUE]
        return sorted(significant, key=lambda s: abs(s.effect_size), reverse=True)

    def _extract_segments(self, tree, df) -> list[DiscoveredSegment]:
        """Extract decision tree leaf nodes as segments with their defining rules."""
        ...

    async def _generate_names(self, segments: list[DiscoveredSegment]) -> list[DiscoveredSegment]:
        """Use Claude API to generate descriptive names for discovered segments."""
        ...
```

**Testing:**
- `test_discovery_finds_significant_segments`: with synthetic data containing a known high-retention group, discovery finds it
- `test_p_value_filter_applied`: segments with p > 0.01 are excluded from results
- `test_minimum_size_enforced`: segments smaller than 50 users are excluded
- `test_effect_size_ordering`: results are sorted by descending |effect_size|
- `test_segment_rules_extractable`: each segment has interpretable rules like `events_7d >= 15 AND sessions_30d >= 10`

---

### Task 9.2: Discovery API and promotion workflow

**What:** Build API endpoints to trigger discovery, view results, and promote discovered cohorts.

**Design:**
```python
@router.post("/cohorts/discover", response_model=DiscoveryRunResponse, status_code=202)
async def trigger_discovery(target_metric: str = "30d_retention", ...):
    """Launch cohort discovery job. Returns run ID for polling."""
    ...

@router.get("/cohorts/discovered", response_model=list[DiscoveredCohortResponse])
async def list_discovered(status: str | None = None, ...): ...

@router.post("/cohorts/discovered/{id}/promote", response_model=CohortResponse)
async def promote_discovered(id: UUID, name: str | None = None, ...):
    """Promote a discovered cohort to a tracked cohort with ongoing membership evaluation."""
    ...

@router.post("/cohorts/discovered/{id}/dismiss", status_code=204)
async def dismiss_discovered(id: UUID, ...): ...
```

**Testing:**
- `test_discovery_returns_202`: triggering discovery returns a run ID
- `test_list_discovered_returns_results`: after discovery completes, GET returns discovered cohorts
- `test_promote_creates_tracked_cohort`: promoting creates a real cohort with the discovered rules as definition
- `test_dismiss_sets_status`: dismissing sets `status=dismissed`
- `test_promoted_cohort_evaluable`: promoted cohort can be evaluated via the standard cohort evaluation endpoint

---

## Phase 10: Alerts & Proactive Notifications

### Definition of Done
- Users can configure alert rules on cohort health metrics (retention drop, churn spike, active user change)
- Alert checker runs on schedule, evaluates rules, and delivers notifications
- Alert history records triggered alerts with LLM-generated narrative context
- Delivery channels: Slack webhook, email, in-app notification

### Task 10.1: Alert rule engine and checker

**What:** Build the alert rule evaluation engine that checks configured metrics against thresholds.

**Design:**
```python
# backend/app/services/alert_service.py
class AlertChecker:
    async def check_all_rules(self, tenant_id: UUID, db: AsyncSession) -> list[TriggeredAlert]:
        rules = await self._get_active_rules(tenant_id, db)
        triggered = []
        for rule in rules:
            current_value = await self._evaluate_metric(rule, db)
            if self._threshold_breached(rule, current_value):
                narrative = await self._generate_narrative(rule, current_value)
                alert = TriggeredAlert(rule=rule, value=current_value, narrative=narrative)
                triggered.append(alert)
                await self._deliver(rule, alert)
                await self._record_history(rule, alert, db)
        return triggered

    async def _evaluate_metric(self, rule: AlertRule, db: AsyncSession) -> float:
        """Compute current metric value based on rule configuration."""
        config = rule.config
        match config["metric"]:
            case "retention_rate":
                return await self._get_cohort_retention(config["cohort_id"], config.get("period", 7), db)
            case "churn_probability_avg":
                return await self._get_avg_churn_probability(config.get("cohort_id"), db)
            case "active_users":
                return await self._get_active_user_count(config.get("cohort_id"), config.get("lookback", "7d"), db)

    async def _generate_narrative(self, rule: AlertRule, value: float) -> str:
        """Use Claude API to generate a contextual alert narrative."""
        ...
```

**Testing:**
- `test_retention_drop_triggers_alert`: retention dropping below threshold fires alert
- `test_churn_spike_triggers_alert`: average churn probability exceeding threshold fires alert
- `test_threshold_not_breached_no_alert`: metric within bounds does not trigger
- `test_alert_recorded_in_history`: triggered alert creates `alert_history` row
- `test_narrative_generated`: alert includes LLM-generated narrative text

---

### Task 10.2: Alert delivery and history API

**What:** Build notification delivery (Slack, email) and alert history management API.

**Design:**
```python
@router.post("/alerts", response_model=AlertRuleResponse, status_code=201)
async def create_alert_rule(body: AlertRuleCreate, ...): ...

@router.get("/alerts", response_model=list[AlertRuleResponse])
async def list_alert_rules(...): ...

@router.get("/alerts/history", response_model=PaginatedResponse[AlertHistoryResponse])
async def list_alert_history(acknowledged: bool | None = None, ...): ...

@router.post("/alerts/history/{id}/acknowledge", status_code=204)
async def acknowledge_alert(id: UUID, ...): ...
```

**Testing:**
- `test_slack_delivery_sends_message`: mocked Slack webhook receives alert payload
- `test_email_delivery_sends_alert`: mocked SMTP sends alert email
- `test_alert_history_filterable`: `?acknowledged=false` returns only unacknowledged alerts
- `test_acknowledge_sets_flag`: acknowledging sets `acknowledged=true` with timestamp and user

---

## Phase 11: B2B Account-Level Aggregation

### Definition of Done
- Accounts (companies) created via `.group()` Segment calls
- User-to-account memberships tracked
- Account-level health scores aggregated from individual user behavioral features
- Churn predictions available at account level (aggregated risk)
- Account-level retention and engagement dashboards

### Task 11.1: Account data model and group call processing

**What:** Process Segment `.group()` calls to create/update accounts and manage user-account memberships.

**Design:**
```python
# backend/app/services/event_service.py (extension)
class EventService:
    async def ingest_group(self, tenant_id: UUID, event: GroupEvent, db: AsyncSession) -> None:
        """Create/update account, add user-account membership."""
        account = await self._upsert_account(tenant_id, event.group_id, event.traits, db)
        user = await self._resolve_user(tenant_id, event.user_id, event.anonymous_id, db)
        if user:
            await self._add_account_membership(tenant_id, account.id, user.id, db)
        await db.commit()

# API for account-level analytics
@router.get("/accounts", response_model=PaginatedResponse[AccountResponse])
@router.get("/accounts/{account_id}", response_model=AccountDetailResponse)
@router.get("/accounts/{account_id}/health", response_model=AccountHealthResponse)
@router.get("/accounts/{account_id}/members", response_model=list[AccountMemberResponse])
```

```python
# backend/app/services/account_health_service.py
class AccountHealthService:
    async def compute_account_health(self, tenant_id: UUID, account_id: UUID, db: AsyncSession) -> AccountHealth:
        """Aggregate user-level features and predictions to account level."""
        members = await self._get_account_members(tenant_id, account_id, db)
        user_predictions = await self._get_member_predictions(tenant_id, [m.user_id for m in members], db)

        return AccountHealth(
            account_id=account_id,
            member_count=len(members),
            active_member_count=sum(1 for m in members if m.last_seen_at and m.last_seen_at > cutoff),
            active_member_pct=...,
            avg_churn_probability=np.mean([p.churn_probability_30d for p in user_predictions]),
            max_churn_probability=max(p.churn_probability_30d for p in user_predictions),
            critical_risk_count=sum(1 for p in user_predictions if p.risk_tier == "critical"),
            health_score=self._compute_composite_score(...),
        )
```

**Testing:**
- `test_group_creates_account`: `.group()` call with new group_id creates account row
- `test_group_adds_membership`: `.group()` call links user to account
- `test_account_health_aggregates_users`: account with 3 users reflects average churn probability
- `test_active_member_percentage`: 2 of 5 users active -> `active_member_pct=0.4`
- `test_account_detail_includes_members`: GET account detail includes member list

---

## Phase 12: GDPR, Security & Production Hardening

### Definition of Done
- Consent tracking and right-to-erasure workflow fully operational
- Data deletion request processes all tables (events, users, predictions, memberships)
- Row-level security enforces tenant isolation at the database level
- Rate limiting on all API endpoints
- API key rotation without downtime
- Helm chart for Kubernetes deployment
- Comprehensive security headers and CORS configuration

### Task 12.1: GDPR consent and data deletion

**What:** Implement consent record tracking and right-to-erasure request processing.

**Design:**
```python
# backend/app/services/gdpr_service.py
class GDPRService:
    TABLES_TO_PURGE = [
        ("events", "user_id"),
        ("daily_user_activity", "user_id"),
        ("cohort_memberships", "user_id"),
        ("churn_predictions", "user_id"),
        ("ml_feature_snapshots", "user_id"),
        ("rfm_scores", "user_id"),
    ]

    async def process_deletion_request(self, request_id: UUID, db: AsyncSession) -> None:
        """
        1. Delete user data from all analytics tables
        2. Anonymize user record (clear PII, set anonymized_at)
        3. Update deletion request status
        """
        request = await db.get(DataDeletionRequest, request_id)
        user_id = request.user_id
        tenant_id = request.tenant_id
        tables_processed = []

        for table_name, user_column in self.TABLES_TO_PURGE:
            count = await db.execute(
                text(f"DELETE FROM {table_name} WHERE tenant_id = :tid AND {user_column} = :uid"),
                {"tid": tenant_id, "uid": user_id}
            )
            tables_processed.append(table_name)

        # Anonymize user record
        await db.execute(
            text("""UPDATE users SET email = NULL, display_name = NULL, external_id = NULL,
                     traits = '{}', anonymous_ids = '{}', anonymized_at = now()
                     WHERE id = :uid AND tenant_id = :tid"""),
            {"uid": user_id, "tid": tenant_id}
        )

        request.status = "completed"
        request.completed_at = datetime.utcnow()
        request.details = {"tables_processed": tables_processed}
        await db.commit()

# API
@router.post("/gdpr/deletion-requests", response_model=DeletionRequestResponse, status_code=201)
async def create_deletion_request(body: DeletionRequestCreate, ...): ...

@router.get("/gdpr/deletion-requests/{id}", response_model=DeletionRequestResponse)
async def get_deletion_status(id: UUID, ...): ...

@router.post("/gdpr/consent", response_model=ConsentResponse, status_code=201)
async def record_consent(body: ConsentCreate, ...): ...
```

**Testing:**
- `test_deletion_removes_events`: after deletion, user's events are gone from events table
- `test_deletion_anonymizes_user`: user's email, name, external_id are NULLed; `anonymized_at` set
- `test_deletion_removes_predictions`: user's churn predictions are deleted
- `test_deletion_status_tracking`: request status transitions: `pending -> processing -> completed`
- `test_consent_recorded`: POST consent creates record with type, granted status, and timestamp

---

### Task 12.2: Row-level security and rate limiting

**What:** Enable PostgreSQL row-level security for tenant isolation and implement API rate limiting.

**Design:**
```sql
-- Row-Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_users ON users
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

ALTER TABLE events ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation_events ON events
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Apply to all tenant-scoped tables...
```

```python
# backend/app/core/rate_limiter.py
from fastapi import Request
from redis.asyncio import Redis

class RateLimiter:
    def __init__(self, redis: Redis, requests_per_minute: int = 60):
        self.redis = redis
        self.rpm = requests_per_minute

    async def check(self, key: str) -> bool:
        """Returns True if request is allowed, False if rate limited."""
        pipe = self.redis.pipeline()
        pipe.incr(f"rate:{key}")
        pipe.expire(f"rate:{key}", 60)
        results = await pipe.execute()
        return results[0] <= self.rpm

# Rate limits by endpoint type
RATE_LIMITS = {
    "events:write": 10000,    # events/minute per API key
    "api:read": 300,          # reads/minute per user
    "nl:query": 20,           # NL queries/minute per user
    "auth": 10,               # auth attempts/minute per IP
}
```

**Testing:**
- `test_rls_prevents_cross_tenant_access`: query with tenant A's session cannot see tenant B's data
- `test_rate_limit_allows_under_threshold`: 59 requests in a minute all succeed
- `test_rate_limit_blocks_over_threshold`: 61st request in a minute returns 429
- `test_rate_limit_resets_after_window`: after 60 seconds, requests are allowed again

---

### Task 12.3: Production deployment (Helm chart)

**What:** Create Kubernetes Helm chart for production deployment with horizontal scaling.

**Design:**
```yaml
# helm/values.yaml
replicaCount:
  api: 3
  worker: 2
  scheduler: 1

image:
  repository: ghcr.io/customer-cohort-analytics/cca
  tag: latest

resources:
  api:
    requests: { cpu: 500m, memory: 512Mi }
    limits: { cpu: 2000m, memory: 2Gi }
  worker:
    requests: { cpu: 1000m, memory: 1Gi }
    limits: { cpu: 4000m, memory: 4Gi }

postgresql:
  enabled: true
  auth:
    database: cca
    username: cca

redis:
  enabled: true
  architecture: standalone

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.cohortanalytics.dev
      paths: [{ path: /, pathType: Prefix }]
  tls:
    - secretName: cca-tls
      hosts: [api.cohortanalytics.dev]

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

**Testing:**
- `test_helm_template_renders`: `helm template` produces valid Kubernetes manifests
- `test_health_probe_configured`: API deployment has liveness and readiness probes on `/health`
- `test_hpa_configured`: HorizontalPodAutoscaler targets 70% CPU
- `test_secrets_mounted`: JWT_SECRET_KEY and DATABASE_URL come from Kubernetes secrets, not env vars

---

## Summary

| Phase | Name | Tasks | Depends On |
|-------|------|-------|------------|
| 1 | Foundation | 4 tasks: scaffold, DB, auth, Docker/CI | -- |
| 2 | Event Ingestion & SDK | 3 tasks: data model, API, JS SDK | Phase 1 |
| 3 | Cohort Engine | 3 tasks: data model, evaluator, CRUD API | Phase 2 |
| 4 | Retention & Funnels | 3 tasks: daily rollup, retention, funnels | Phase 3 |
| 5 | Dashboards & Reporting | 3 tasks: CRUD, widget data, scheduled delivery | Phase 4 |
| 6 | ML Churn Prediction | 3 tasks: features, training, batch prediction | Phase 4 |
| 7 | XAI Explainability | 2 tasks: SHAP integration, prediction API | Phase 6 |
| 8 | NL Query Interface | 1 task: NL-to-SQL engine | Phase 4 |
| 9 | Automated Cohort Discovery | 2 tasks: discovery engine, promotion API | Phase 6 |
| 10 | Alerts & Notifications | 2 tasks: alert engine, delivery/history API | Phase 5, Phase 7 |
| 11 | B2B Account Aggregation | 1 task: account model, health scoring | Phase 7 |
| 12 | GDPR & Production | 3 tasks: GDPR, RLS/rate limiting, Helm chart | Phase 11 |
| **Total** | | **30 tasks** | |
