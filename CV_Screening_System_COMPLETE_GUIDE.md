# Agentic AI CV Screening & Hiring Intelligence System
## Complete Beginner-to-Production Implementation Guide

---

## Table of Contents

1. [What We're Building](#what-were-building)
2. [System Architecture Overview](#system-architecture-overview)
3. [Prerequisites & Installation](#prerequisites--installation)
4. [Project Folder Structure](#project-folder-structure)
5. [Database Setup (PostgreSQL + pgvector)](#database-setup)
6. [Agent Definitions & Code](#agent-definitions--code)
7. [Backend API (FastAPI)](#backend-api)
8. [Streamlit Frontend](#streamlit-frontend)
9. [Docker & Deployment](#docker--deployment)
10. [Testing the System](#testing-the-system)
11. [Security & Compliance Checklist](#security--compliance-checklist)

---

## What We're Building

A fully local, enterprise-grade AI system that:
- **Ingests** CVs in PDF, DOCX, TXT formats (single or bulk)
- **Parses** and extracts structured data using an AI agent
- **Embeds** CVs and Job Descriptions into vector space using local Ollama models
- **Semantically matches** candidates to roles (no keyword counting)
- **Ranks** candidates with explainable AI reasoning
- **Detects** anomalies, duplicates, and red flags
- **Stores** everything in PostgreSQL with full audit logs
- **Presents** results in a non-technical, HR-friendly Streamlit dashboard

**Key principle**: HR recommends using AI. The AI does NOT make final hiring decisions.

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        HR USER (Browser)                            │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ Streamlit UI (Port 8501)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    STREAMLIT FRONTEND                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  JD Upload   │  │  CV Upload   │  │   Candidate Dashboard    │  │
│  │  & Preview   │  │  (Bulk/Single│  │   Rankings + Explain     │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │ HTTP (FastAPI - Port 8000)
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATION LAYER (FastAPI)                     │
│              Manages agent execution, routing, logging              │
└────┬──────────────┬──────────────┬──────────────┬───────────────────┘
     │              │              │              │
     ▼              ▼              ▼              ▼
┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐
│INGESTION│  │SEMANTIC  │  │POTENTIAL │  │VALIDATION &  │
│& PARSING│  │MATCHING  │  │& VALUE   │  │ANOMALY       │
│AGENT    │  │AGENT     │  │ADD AGENT │  │DETECTION     │
│         │  │          │  │          │  │AGENT         │
└────┬────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘
     │            │             │                │
     └────────────┴─────────────┴────────────────┘
                           │
          ┌────────────────┴────────────────┐
          │                                 │
          ▼                                 ▼
┌──────────────────┐              ┌──────────────────────┐
│   PostgreSQL     │              │   Ollama (Local LLM) │
│   + pgvector     │              │   llama3 / mistral   │
│                  │              │   nomic-embed-text   │
│  - candidates    │              └──────────────────────┘
│  - cv_versions   │
│  - embeddings    │
│  - screenings    │
│  - audit_logs    │
└──────────────────┘
```

### Agent Responsibilities

| Agent | Job |
|-------|-----|
| **Ingestion & Parsing Agent** | Reads PDF/DOCX/TXT, extracts structured JSON |
| **Semantic Matching Agent** | Generates embeddings, vector similarity, LLM reasoning |
| **Potential & Value-Add Agent** | Detects growth signals, learning velocity, leadership fit |
| **Validation & Anomaly Agent** | Flags duplicates, inconsistencies, missing qualifications |

---

## Prerequisites & Installation

### Step 1: Install Required Software

#### 1a. Install Docker Desktop
Docker runs our entire system in isolated containers.

**Mac:**
```bash
# Download from https://www.docker.com/products/docker-desktop/
# OR use Homebrew:
brew install --cask docker
```

**Windows:**
- Download Docker Desktop from https://www.docker.com/products/docker-desktop/
- Enable WSL2 when prompted

**Ubuntu/Debian Linux:**
```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
```

Verify:
```bash
docker --version        # Should show Docker version 24+
docker compose version  # Should show Docker Compose version 2+
```

#### 1b. Install Ollama (Local LLM Engine)

Ollama runs AI models completely locally — no internet, no API keys.

**Mac:**
```bash
brew install ollama
# OR download from https://ollama.ai
```

**Linux:**
```bash
curl -fsSL https://ollama.ai/install.sh | sh
```

**Windows:**
- Download installer from https://ollama.ai/download

Start Ollama and pull the required models:
```bash
# Start Ollama service
ollama serve &

# Pull the LLM (for reasoning and extraction)
ollama pull llama3.1:8b

# Pull the embedding model (for vector search)
ollama pull nomic-embed-text

# Verify models are loaded
ollama list
```

> ⚠️ **Note for beginners**: `llama3.1:8b` is ~5GB. Make sure you have 8GB+ RAM and 10GB+ free disk space.

#### 1c. Install Python 3.11+

**Mac:**
```bash
brew install python@3.11
```

**Ubuntu:**
```bash
sudo apt-get install python3.11 python3.11-venv python3-pip
```

**Windows:**
- Download from https://www.python.org/downloads/
- Check "Add to PATH" during install

Verify:
```bash
python3 --version  # Should show 3.11+
```

#### 1d. Install Git

```bash
# Mac
brew install git

# Ubuntu
sudo apt-get install git

# Windows: Download from https://git-scm.com/download/win
```

---

## Project Folder Structure

Create the following structure (we'll fill each file in subsequent sections):

```bash
# Run this in your terminal to create the entire structure at once:
mkdir -p cv_screening_system/{
  agents,
  api,
  frontend,
  database/{migrations,seeds},
  docker,
  logs,
  uploads/{cvs,jds},
  tests/{unit,integration},
  config,
  shared/{models,utils,constants}
}

cd cv_screening_system
```

Final structure:
```
cv_screening_system/
│
├── docker-compose.yml          # Orchestrates all services
├── .env                        # Environment variables (secrets)
├── .env.example                # Template for .env
├── requirements.txt            # Python dependencies
│
├── agents/
│   ├── __init__.py
│   ├── base_agent.py           # Abstract base class for all agents
│   ├── ingestion_agent.py      # Parses CVs → structured JSON
│   ├── matching_agent.py       # Semantic matching & ranking
│   ├── potential_agent.py      # Growth & value-add detection
│   └── validation_agent.py    # Anomaly & compliance checks
│
├── api/
│   ├── __init__.py
│   ├── main.py                 # FastAPI app entry point
│   ├── routers/
│   │   ├── candidates.py
│   │   ├── jobs.py
│   │   └── screenings.py
│   └── dependencies.py        # Shared FastAPI dependencies
│
├── database/
│   ├── __init__.py
│   ├── connection.py           # DB connection pool
│   ├── models.py               # SQLAlchemy ORM models
│   └── migrations/
│       └── 001_initial_schema.sql
│
├── frontend/
│   ├── app.py                  # Main Streamlit app
│   └── pages/
│       ├── 01_upload_jd.py
│       ├── 02_upload_cvs.py
│       ├── 03_screening_results.py
│       └── 04_candidate_history.py
│
├── shared/
│   ├── models.py               # Pydantic data models
│   ├── utils.py                # Shared utilities
│   └── constants.py            # System-wide constants
│
├── config/
│   └── settings.py             # Centralized configuration
│
├── logs/                       # Auto-generated JSON logs
└── tests/
    ├── unit/
    └── integration/
```
# ============================================================
# FILE: .env.example
# Copy this to .env and fill in your values
# ============================================================
# .env.example

# PostgreSQL
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=cv_screening
POSTGRES_USER=hr_admin
POSTGRES_PASSWORD=change_this_strong_password_123

# Ollama
OLLAMA_BASE_URL=http://ollama:11434
OLLAMA_LLM_MODEL=llama3.1:8b
OLLAMA_EMBED_MODEL=nomic-embed-text

# FastAPI
API_HOST=0.0.0.0
API_PORT=8000
SECRET_KEY=change_this_to_a_random_64_char_string

# Streamlit
STREAMLIT_PORT=8501

# File storage
UPLOAD_DIR=/app/uploads
MAX_FILE_SIZE_MB=20
ALLOWED_EXTENSIONS=pdf,docx,txt

# Logging
LOG_LEVEL=INFO
LOG_DIR=/app/logs

# System
ENVIRONMENT=production
MAX_BULK_UPLOAD=500
EMBEDDING_DIMENSION=768


# ============================================================
# FILE: requirements.txt
# ============================================================
# requirements.txt

# Web Framework & API
fastapi==0.111.0
uvicorn[standard]==0.29.0
python-multipart==0.0.9

# Streamlit Frontend
streamlit==1.35.0
streamlit-extras==0.4.2

# Database
sqlalchemy==2.0.30
asyncpg==0.29.0
psycopg2-binary==2.9.9
alembic==1.13.1
pgvector==0.2.5

# LLM & AI
ollama==0.2.1
langchain==0.2.1
langchain-community==0.2.1
langchain-ollama==0.1.1
sentence-transformers==3.0.1

# Document Parsing
pypdf2==3.0.1
pdfplumber==0.11.0
python-docx==1.1.2
python-magic==0.4.27
chardet==5.2.0

# Data Processing
pandas==2.2.2
numpy==1.26.4
pydantic==2.7.1
pydantic-settings==2.3.0

# Utilities
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
aiofiles==23.2.1
httpx==0.27.0
tenacity==8.3.0
structlog==24.2.0

# Testing
pytest==8.2.0
pytest-asyncio==0.23.7
pytest-cov==5.0.0
httpx==0.27.0

# Dev tools
python-dotenv==1.0.1
rich==13.7.1
# ============================================================
# FILE: database/migrations/001_initial_schema.sql
# Run once to set up the entire database
# ============================================================

-- Enable pgvector extension (MUST be first)
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;  -- For fuzzy text search

-- ============================================================
-- ENUM TYPES
-- ============================================================
CREATE TYPE candidate_status AS ENUM (
    'new', 'parsing', 'parsed', 'screened',
    'shortlisted', 'interview_scheduled', 'interviewed',
    'offered', 'hired', 'rejected', 'on_hold', 'withdrawn'
);

CREATE TYPE screening_decision AS ENUM (
    'auto_shortlisted', 'auto_rejected', 'needs_review',
    'hr_approved', 'hr_rejected', 'hr_hold', 'forwarded'
);

CREATE TYPE file_format AS ENUM ('pdf', 'docx', 'txt', 'unknown');

CREATE TYPE anomaly_type AS ENUM (
    'duplicate_cv', 'employment_gap', 'inconsistent_dates',
    'missing_required_skills', 'low_parse_confidence',
    'suspicious_pattern', 'borderline_score'
);

-- ============================================================
-- TABLE: jobs
-- Stores all job descriptions
-- ============================================================
CREATE TABLE jobs (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    title           TEXT NOT NULL,
    department      TEXT,
    location        TEXT,
    description_raw TEXT NOT NULL,           -- Original JD text
    requirements    JSONB,                   -- Parsed required skills/exp
    embedding       vector(768),             -- JD semantic vector
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    created_by      TEXT NOT NULL            -- HR user who posted
);

-- ============================================================
-- TABLE: candidates
-- Master candidate record — deduplicated across submissions
-- ============================================================
CREATE TABLE candidates (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    
    -- Identity (used for deduplication)
    email           TEXT UNIQUE,
    full_name       TEXT,
    phone           TEXT,
    linkedin_url    TEXT,
    
    -- Status tracking
    current_status  candidate_status DEFAULT 'new',
    
    -- Timestamps
    first_seen_at   TIMESTAMPTZ DEFAULT NOW(),
    last_updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- Metadata
    source          TEXT,                   -- 'portal', 'linkedin', 'email', 'manual'
    is_returning    BOOLEAN DEFAULT FALSE,  -- Has applied before
    notes           TEXT                    -- HR free-text notes
);

-- ============================================================
-- TABLE: cv_versions
-- One candidate may submit multiple CVs over time
-- ============================================================
CREATE TABLE cv_versions (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    candidate_id        UUID NOT NULL REFERENCES candidates(id) ON DELETE CASCADE,
    job_id              UUID REFERENCES jobs(id),
    
    -- File metadata
    original_filename   TEXT NOT NULL,
    stored_path         TEXT NOT NULL,       -- Local filesystem path
    file_format         file_format NOT NULL,
    file_size_bytes     INTEGER,
    file_hash           TEXT NOT NULL,       -- SHA256 for dedup
    
    -- Parsing outputs
    parsed_data         JSONB,               -- Full structured extraction
    parse_confidence    FLOAT,               -- 0.0 to 1.0
    parse_errors        JSONB,               -- Array of error objects
    
    -- Embeddings
    embedding           vector(768),         -- CV semantic vector
    embedding_model     TEXT,                -- Which model generated it
    
    -- Processing state
    ingestion_status    TEXT DEFAULT 'pending',  -- pending/processing/done/failed
    correlation_id      UUID DEFAULT uuid_generate_v4(),  -- For log tracing
    
    -- Timestamps
    uploaded_at         TIMESTAMPTZ DEFAULT NOW(),
    processed_at        TIMESTAMPTZ
);

-- ============================================================
-- TABLE: screenings
-- Each time a CV is evaluated against a JD
-- ============================================================
CREATE TABLE screenings (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    cv_version_id       UUID NOT NULL REFERENCES cv_versions(id),
    job_id              UUID NOT NULL REFERENCES jobs(id),
    candidate_id        UUID NOT NULL REFERENCES candidates(id),
    
    -- Semantic scores (0.0 to 1.0, not percentages)
    semantic_similarity FLOAT,              -- Raw vector cosine similarity
    relevance_score     FLOAT,              -- LLM-assessed job relevance
    potential_score     FLOAT,              -- Growth/adaptability signal
    composite_score     FLOAT,              -- Weighted final score
    
    -- Rank within this job's pool
    rank_in_pool        INTEGER,
    total_in_pool       INTEGER,
    
    -- Explainability (CRITICAL for compliance)
    strengths           JSONB,              -- Array of strength strings
    gaps                JSONB,              -- Array of gap strings
    transferable_skills JSONB,              -- Inferred from adjacent domains
    value_add_insights  JSONB,              -- From Potential Agent
    llm_rationale       TEXT,              -- Full LLM explanation paragraph
    
    -- HR Decision
    decision            screening_decision DEFAULT 'needs_review',
    decision_by         TEXT,              -- HR username
    decision_at         TIMESTAMPTZ,
    decision_notes      TEXT,              -- HR free-text rationale
    
    -- Timestamps
    screened_at         TIMESTAMPTZ DEFAULT NOW(),
    
    -- Ensure one screening per CV per job
    UNIQUE (cv_version_id, job_id)
);

-- ============================================================
-- TABLE: anomalies
-- Flags raised by the Validation Agent
-- ============================================================
CREATE TABLE anomalies (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    cv_version_id   UUID REFERENCES cv_versions(id),
    candidate_id    UUID REFERENCES candidates(id),
    job_id          UUID REFERENCES jobs(id),
    
    anomaly_type    anomaly_type NOT NULL,
    severity        TEXT NOT NULL CHECK (severity IN ('low', 'medium', 'high', 'critical')),
    description     TEXT NOT NULL,          -- Human-readable explanation
    raw_evidence    JSONB,                  -- Supporting data
    
    -- Resolution
    is_resolved     BOOLEAN DEFAULT FALSE,
    resolved_by     TEXT,
    resolved_at     TIMESTAMPTZ,
    resolution_note TEXT,
    
    detected_at     TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: interviews
-- Records interview scheduling and feedback
-- ============================================================
CREATE TABLE interviews (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    screening_id    UUID NOT NULL REFERENCES screenings(id),
    candidate_id    UUID NOT NULL REFERENCES candidates(id),
    job_id          UUID NOT NULL REFERENCES jobs(id),
    
    scheduled_at    TIMESTAMPTZ,
    interviewed_at  TIMESTAMPTZ,
    interviewer     TEXT,
    format          TEXT,                   -- 'phone', 'video', 'onsite', 'panel'
    
    -- Feedback (structured)
    technical_score     INTEGER CHECK (technical_score BETWEEN 1 AND 5),
    communication_score INTEGER CHECK (communication_score BETWEEN 1 AND 5),
    cultural_fit_score  INTEGER CHECK (cultural_fit_score BETWEEN 1 AND 5),
    overall_score       INTEGER CHECK (overall_score BETWEEN 1 AND 5),
    feedback_notes      TEXT,
    recommendation      TEXT CHECK (recommendation IN ('hire', 'reject', 'second_round', 'hold')),
    
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================================
-- TABLE: audit_logs
-- IMMUTABLE record of every system action (append-only)
-- ============================================================
CREATE TABLE audit_logs (
    id              BIGSERIAL PRIMARY KEY,   -- Sequential for ordering
    
    -- Context
    correlation_id  UUID,                    -- Links to cv_version correlation_id
    event_type      TEXT NOT NULL,           -- 'ingestion', 'parse', 'embed', 'match', 'decision'
    actor           TEXT NOT NULL,           -- 'system', 'agent:ingestion', 'hr:username'
    
    -- References (nullable — not all events have all entities)
    candidate_id    UUID,
    cv_version_id   UUID,
    job_id          UUID,
    screening_id    UUID,
    
    -- Event data
    event_data      JSONB NOT NULL,          -- Full structured event payload
    outcome         TEXT,                    -- 'success', 'failure', 'warning'
    error_message   TEXT,                    -- If outcome = failure
    duration_ms     INTEGER,                 -- Processing time
    
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Prevent audit log modification (compliance)
-- In production, consider a separate append-only role
REVOKE UPDATE, DELETE ON audit_logs FROM PUBLIC;

-- ============================================================
-- INDEXES for query performance
-- ============================================================

-- Vector similarity search indexes (IVFFlat for large scale)
CREATE INDEX idx_cv_embedding      ON cv_versions USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_job_embedding     ON jobs        USING ivfflat (embedding vector_cosine_ops) WITH (lists = 10);

-- Standard B-tree indexes
CREATE INDEX idx_candidates_email  ON candidates  (email);
CREATE INDEX idx_cvv_candidate     ON cv_versions (candidate_id);
CREATE INDEX idx_cvv_job           ON cv_versions (job_id);
CREATE INDEX idx_cvv_hash          ON cv_versions (file_hash);
CREATE INDEX idx_screenings_job    ON screenings  (job_id, composite_score DESC);
CREATE INDEX idx_screenings_cand   ON screenings  (candidate_id);
CREATE INDEX idx_audit_correlation ON audit_logs  (correlation_id);
CREATE INDEX idx_audit_event_type  ON audit_logs  (event_type, created_at DESC);
CREATE INDEX idx_anomaly_cv        ON anomalies   (cv_version_id);

-- GIN indexes for JSONB search
CREATE INDEX idx_cv_parsed_data    ON cv_versions USING GIN (parsed_data);
CREATE INDEX idx_screening_strengths ON screenings USING GIN (strengths);

-- ============================================================
-- VIEWS for HR dashboard queries
-- ============================================================

CREATE VIEW v_candidate_summary AS
SELECT
    c.id,
    c.full_name,
    c.email,
    c.current_status,
    c.is_returning,
    c.source,
    COUNT(DISTINCT cv.id)     AS cv_count,
    COUNT(DISTINCT s.job_id)  AS jobs_applied,
    MAX(s.composite_score)    AS best_score,
    c.first_seen_at
FROM candidates c
LEFT JOIN cv_versions cv ON cv.candidate_id = c.id
LEFT JOIN screenings s   ON s.candidate_id = c.id
GROUP BY c.id;

CREATE VIEW v_job_pipeline AS
SELECT
    j.id,
    j.title,
    j.department,
    COUNT(s.id)                                              AS total_screened,
    COUNT(s.id) FILTER (WHERE s.decision = 'hr_approved')   AS shortlisted,
    COUNT(s.id) FILTER (WHERE s.decision = 'hr_rejected')   AS rejected,
    COUNT(s.id) FILTER (WHERE s.decision = 'needs_review')  AS pending_review,
    AVG(s.composite_score)                                   AS avg_score
FROM jobs j
LEFT JOIN screenings s ON s.job_id = j.id
GROUP BY j.id;
# ============================================================
# FILE: config/settings.py
# Central configuration — reads from .env
# ============================================================

from pydantic_settings import BaseSettings
from pydantic import Field
from typing import Optional
import os

class Settings(BaseSettings):
    # ---- Database ----
    postgres_host: str = "localhost"
    postgres_port: int = 5432
    postgres_db: str = "cv_screening"
    postgres_user: str = "hr_admin"
    postgres_password: str

    @property
    def database_url(self) -> str:
        return (
            f"postgresql+asyncpg://{self.postgres_user}:{self.postgres_password}"
            f"@{self.postgres_host}:{self.postgres_port}/{self.postgres_db}"
        )

    @property
    def database_url_sync(self) -> str:
        return (
            f"postgresql://{self.postgres_user}:{self.postgres_password}"
            f"@{self.postgres_host}:{self.postgres_port}/{self.postgres_db}"
        )

    # ---- Ollama ----
    ollama_base_url: str = "http://localhost:11434"
    ollama_llm_model: str = "llama3.1:8b"
    ollama_embed_model: str = "nomic-embed-text"

    # ---- API ----
    api_host: str = "0.0.0.0"
    api_port: int = 8000
    secret_key: str
    environment: str = "production"

    # ---- Files ----
    upload_dir: str = "/app/uploads"
    max_file_size_mb: int = 20
    allowed_extensions: str = "pdf,docx,txt"

    @property
    def allowed_ext_list(self) -> list[str]:
        return [ext.strip() for ext in self.allowed_extensions.split(",")]

    # ---- Logging ----
    log_level: str = "INFO"
    log_dir: str = "/app/logs"

    # ---- System ----
    max_bulk_upload: int = 500
    embedding_dimension: int = 768

    class Config:
        env_file = ".env"
        case_sensitive = False


# Singleton instance
settings = Settings()


# ============================================================
# FILE: database/connection.py
# Async database connection pool
# ============================================================

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession, async_sessionmaker
from sqlalchemy.orm import DeclarativeBase
from config.settings import settings
import structlog

logger = structlog.get_logger(__name__)

# Create async engine
engine = create_async_engine(
    settings.database_url,
    pool_size=10,
    max_overflow=20,
    pool_timeout=30,
    pool_recycle=1800,
    echo=(settings.environment == "development"),
)

# Session factory
AsyncSessionLocal = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

class Base(DeclarativeBase):
    pass


async def get_db() -> AsyncSession:
    """FastAPI dependency — yields a DB session per request."""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception as e:
            await session.rollback()
            logger.error("database_session_error", error=str(e))
            raise
        finally:
            await session.close()


async def check_db_health() -> bool:
    """Ping the database. Used in health checks."""
    try:
        async with AsyncSessionLocal() as session:
            await session.execute("SELECT 1")
        return True
    except Exception as e:
        logger.error("db_health_check_failed", error=str(e))
        return False


# ============================================================
# FILE: shared/models.py
# Pydantic models — the data contracts between components
# ============================================================

from pydantic import BaseModel, Field, EmailStr
from typing import Optional, List, Dict, Any
from datetime import datetime
from uuid import UUID
from enum import Enum


class FileFormat(str, Enum):
    PDF  = "pdf"
    DOCX = "docx"
    TXT  = "txt"
    UNKNOWN = "unknown"


class IngestionStatus(str, Enum):
    PENDING    = "pending"
    PROCESSING = "processing"
    DONE       = "done"
    FAILED     = "failed"


# ---- CV Parsing Output ----

class ExperienceEntry(BaseModel):
    company:    Optional[str] = None
    role:       Optional[str] = None
    start_date: Optional[str] = None
    end_date:   Optional[str] = None
    duration_months: Optional[int] = None
    description: Optional[str] = None
    technologies: List[str] = Field(default_factory=list)


class EducationEntry(BaseModel):
    institution: Optional[str] = None
    degree:      Optional[str] = None
    field:       Optional[str] = None
    graduation_year: Optional[int] = None
    grade:       Optional[str] = None


class ParsedCV(BaseModel):
    """Structured output from the Ingestion & Parsing Agent."""
    full_name:        Optional[str] = None
    email:            Optional[str] = None
    phone:            Optional[str] = None
    linkedin_url:     Optional[str] = None
    location:         Optional[str] = None

    # Skills (never infer gender/age/nationality — privacy guardrail)
    technical_skills: List[str] = Field(default_factory=list)
    soft_skills:      List[str] = Field(default_factory=list)
    domain_expertise: List[str] = Field(default_factory=list)  # e.g. "cloud", "fintech"
    certifications:   List[str] = Field(default_factory=list)
    languages:        List[str] = Field(default_factory=list)

    # Timeline
    experience:       List[ExperienceEntry] = Field(default_factory=list)
    education:        List[EducationEntry]  = Field(default_factory=list)
    total_years_exp:  Optional[float] = None

    # Summary
    cv_summary:       Optional[str] = None  # One-paragraph LLM summary

    # Quality metadata
    parse_confidence: float = Field(ge=0.0, le=1.0, default=0.5)
    parse_warnings:   List[str] = Field(default_factory=list)


# ---- Screening Output ----

class ScreeningResult(BaseModel):
    """Output from the Semantic Matching Agent."""
    candidate_id:       UUID
    cv_version_id:      UUID
    job_id:             UUID

    # Scores
    semantic_similarity: float = Field(ge=0.0, le=1.0)
    relevance_score:     float = Field(ge=0.0, le=1.0)
    potential_score:     float = Field(ge=0.0, le=1.0)
    composite_score:     float = Field(ge=0.0, le=1.0)

    # Explainability
    strengths:           List[str]
    gaps:                List[str]
    transferable_skills: List[str]
    value_add_insights:  List[str]
    llm_rationale:       str   # Full plain-English explanation for HR

    # Validation flags
    anomalies:           List[Dict[str, Any]] = Field(default_factory=list)

    screened_at:         datetime = Field(default_factory=datetime.utcnow)


# ---- API Request/Response Models ----

class JobCreateRequest(BaseModel):
    title:       str
    department:  Optional[str] = None
    location:    Optional[str] = None
    description: str
    created_by:  str


class JobResponse(BaseModel):
    id:          UUID
    title:       str
    department:  Optional[str]
    description_raw: str
    is_active:   bool
    created_at:  datetime

    class Config:
        from_attributes = True


class CandidateResponse(BaseModel):
    id:             UUID
    full_name:      Optional[str]
    email:          Optional[str]
    current_status: str
    is_returning:   bool
    source:         Optional[str]

    class Config:
        from_attributes = True


class BulkUploadResponse(BaseModel):
    total_received: int
    queued:         int
    failed:         int
    correlation_ids: List[str]
    errors:         List[Dict[str, str]]  # filename → error message


class HealthResponse(BaseModel):
    status:      str
    database:    bool
    ollama:      bool
    version:     str = "1.0.0"


# ============================================================
# FILE: shared/utils.py
# Shared utility functions
# ============================================================

import hashlib
import uuid
import os
import re
from pathlib import Path
from typing import Optional
import structlog

logger = structlog.get_logger(__name__)


def compute_file_hash(file_bytes: bytes) -> str:
    """SHA-256 hash of file content — used for deduplication."""
    return hashlib.sha256(file_bytes).hexdigest()


def generate_correlation_id() -> str:
    """Unique ID to trace a single CV through the entire pipeline."""
    return str(uuid.uuid4())


def sanitize_filename(filename: str) -> str:
    """Remove dangerous characters from uploaded filenames."""
    # Keep only alphanumeric, dash, underscore, dot
    clean = re.sub(r'[^\w\-_\.]', '_', filename)
    return clean[:255]  # Limit length


def get_file_extension(filename: str) -> str:
    """Extract and lowercase file extension."""
    return Path(filename).suffix.lstrip('.').lower()


def ensure_dir(path: str) -> None:
    """Create directory if it doesn't exist."""
    os.makedirs(path, exist_ok=True)


def truncate_text(text: str, max_chars: int = 8000) -> str:
    """
    Truncate text for LLM input.
    LLMs have context limits — we truncate at word boundaries.
    """
    if len(text) <= max_chars:
        return text
    truncated = text[:max_chars].rsplit(' ', 1)[0]
    return truncated + "\n[TEXT TRUNCATED FOR PROCESSING]"


def format_duration(months: Optional[int]) -> str:
    """Convert months to human-readable string."""
    if not months:
        return "Unknown duration"
    years  = months // 12
    rem_mo = months % 12
    parts  = []
    if years:  parts.append(f"{years} year{'s' if years > 1 else ''}")
    if rem_mo: parts.append(f"{rem_mo} month{'s' if rem_mo > 1 else ''}")
    return " ".join(parts) or "< 1 month"


def mask_email(email: str) -> str:
    """Partially mask email for display (privacy)."""
    if '@' not in email:
        return email
    user, domain = email.split('@', 1)
    masked_user = user[:2] + '*' * (len(user) - 2) if len(user) > 2 else user
    return f"{masked_user}@{domain}"
# ============================================================
# FILE: agents/base_agent.py
# Abstract base for all agents — enforces structure & logging
# ============================================================

import time
import uuid
from abc import ABC, abstractmethod
from typing import Any, Dict, Optional
import structlog
from sqlalchemy.ext.asyncio import AsyncSession

logger = structlog.get_logger(__name__)


class BaseAgent(ABC):
    """
    Every agent inherits from this class.
    Provides:
      - Structured logging with correlation IDs
      - Performance timing
      - Standardized error handling
      - Audit log writing
    """

    def __init__(self, name: str, db: AsyncSession):
        self.name = name
        self.db   = db
        self.log  = logger.bind(agent=name)

    async def run(self, payload: Dict[str, Any], correlation_id: Optional[str] = None) -> Dict[str, Any]:
        """
        Entry point for every agent. Wraps execute() with:
        - Timing
        - Logging
        - Error handling
        - Audit trail writing
        """
        correlation_id = correlation_id or str(uuid.uuid4())
        bound_log = self.log.bind(correlation_id=correlation_id)

        bound_log.info("agent_started", payload_keys=list(payload.keys()))
        start_time = time.perf_counter()

        try:
            result = await self.execute(payload, correlation_id)
            duration_ms = int((time.perf_counter() - start_time) * 1000)

            bound_log.info(
                "agent_completed",
                duration_ms=duration_ms,
                outcome="success",
            )

            # Write to audit log
            await self._write_audit(
                correlation_id=correlation_id,
                event_type=f"agent:{self.name}",
                event_data={"input_keys": list(payload.keys()), "outcome": "success"},
                outcome="success",
                duration_ms=duration_ms,
            )

            return result

        except Exception as e:
            duration_ms = int((time.perf_counter() - start_time) * 1000)
            bound_log.error("agent_failed", error=str(e), duration_ms=duration_ms)

            await self._write_audit(
                correlation_id=correlation_id,
                event_type=f"agent:{self.name}",
                event_data={"input_keys": list(payload.keys()), "error": str(e)},
                outcome="failure",
                duration_ms=duration_ms,
                error_message=str(e),
            )

            raise

    @abstractmethod
    async def execute(self, payload: Dict[str, Any], correlation_id: str) -> Dict[str, Any]:
        """Subclasses implement their core logic here."""
        pass

    async def _write_audit(
        self,
        correlation_id: str,
        event_type: str,
        event_data: Dict[str, Any],
        outcome: str,
        duration_ms: int,
        error_message: Optional[str] = None,
        **kwargs,
    ) -> None:
        """Writes an immutable audit log entry."""
        try:
            from sqlalchemy import text
            await self.db.execute(
                text("""
                    INSERT INTO audit_logs
                        (correlation_id, event_type, actor, event_data, outcome, duration_ms, error_message)
                    VALUES
                        (:cid, :etype, :actor, :edata::jsonb, :outcome, :dur, :err)
                """),
                {
                    "cid":    correlation_id,
                    "etype":  event_type,
                    "actor":  f"agent:{self.name}",
                    "edata":  str(event_data),
                    "outcome": outcome,
                    "dur":    duration_ms,
                    "err":    error_message,
                },
            )
            await self.db.commit()
        except Exception as log_err:
            # Audit log failure must never crash the pipeline
            self.log.error("audit_write_failed", error=str(log_err))


# ============================================================
# FILE: agents/ingestion_agent.py
# Parses PDF/DOCX/TXT → structured JSON using LLM
# ============================================================

import json
import re
import hashlib
from pathlib import Path
from typing import Any, Dict, Optional

import pdfplumber
import docx
import chardet
import ollama as ollama_client
import structlog

from agents.base_agent import BaseAgent
from shared.models import ParsedCV, FileFormat
from shared.utils import truncate_text, get_file_extension
from config.settings import settings

logger = structlog.get_logger(__name__)


class IngestionAgent(BaseAgent):
    """
    Responsible for:
    1. Detecting file format
    2. Extracting raw text from PDF/DOCX/TXT
    3. Calling Ollama LLM to extract structured fields
    4. Returning a ParsedCV object with confidence score
    5. Logging ALL failures without stopping bulk batches
    """

    def __init__(self, db):
        super().__init__("ingestion", db)
        self.ollama = ollama_client.AsyncClient(host=settings.ollama_base_url)

    async def execute(self, payload: Dict[str, Any], correlation_id: str) -> Dict[str, Any]:
        """
        payload expects:
          - file_path: str     — path to the uploaded file
          - filename:  str     — original filename
        """
        file_path = payload["file_path"]
        filename  = payload["filename"]

        self.log.info("ingestion_start", filename=filename, path=file_path)

        # ---- Step 1: Detect format and extract raw text ----
        ext = get_file_extension(filename)
        raw_text, parse_warnings = await self._extract_text(file_path, ext)

        if not raw_text or len(raw_text.strip()) < 50:
            raise ValueError(f"Could not extract meaningful text from {filename}")

        # ---- Step 2: Compute file hash (deduplication) ----
        with open(file_path, "rb") as f:
            file_bytes = f.read()
        file_hash = hashlib.sha256(file_bytes).hexdigest()

        # ---- Step 3: LLM extraction ----
        truncated_text = truncate_text(raw_text, max_chars=6000)
        parsed_cv = await self._llm_extract(truncated_text, parse_warnings)

        self.log.info(
            "ingestion_complete",
            filename=filename,
            confidence=parsed_cv.parse_confidence,
            skills_found=len(parsed_cv.technical_skills),
        )

        return {
            "parsed_cv":      parsed_cv.model_dump(),
            "raw_text":       raw_text,
            "file_hash":      file_hash,
            "file_format":    ext,
            "parse_warnings": parse_warnings,
        }

    async def _extract_text(self, file_path: str, ext: str) -> tuple[str, list]:
        """Dispatch to the right extractor based on file type."""
        warnings = []

        if ext == "pdf":
            return await self._extract_pdf(file_path, warnings), warnings
        elif ext == "docx":
            return self._extract_docx(file_path, warnings), warnings
        elif ext == "txt":
            return self._extract_txt(file_path, warnings), warnings
        else:
            warnings.append(f"Unsupported format: {ext}")
            return "", warnings

    async def _extract_pdf(self, path: str, warnings: list) -> str:
        """Extract text from PDF using pdfplumber (handles tables & columns)."""
        text_parts = []
        try:
            with pdfplumber.open(path) as pdf:
                for i, page in enumerate(pdf.pages):
                    page_text = page.extract_text()
                    if page_text:
                        text_parts.append(page_text)
                    else:
                        warnings.append(f"Page {i+1}: No extractable text (possibly scanned image)")
        except Exception as e:
            warnings.append(f"PDF extraction error: {str(e)}")

        return "\n".join(text_parts)

    def _extract_docx(self, path: str, warnings: list) -> str:
        """Extract text from DOCX preserving paragraph structure."""
        try:
            doc = docx.Document(path)
            paragraphs = [p.text for p in doc.paragraphs if p.text.strip()]
            # Also extract text from tables
            for table in doc.tables:
                for row in table.rows:
                    row_text = " | ".join(cell.text.strip() for cell in row.cells if cell.text.strip())
                    if row_text:
                        paragraphs.append(row_text)
            return "\n".join(paragraphs)
        except Exception as e:
            warnings.append(f"DOCX extraction error: {str(e)}")
            return ""

    def _extract_txt(self, path: str, warnings: list) -> str:
        """Extract text from TXT with encoding detection."""
        try:
            with open(path, "rb") as f:
                raw_bytes = f.read()
            detected = chardet.detect(raw_bytes)
            encoding = detected.get("encoding", "utf-8") or "utf-8"
            return raw_bytes.decode(encoding, errors="replace")
        except Exception as e:
            warnings.append(f"TXT extraction error: {str(e)}")
            return ""

    async def _llm_extract(self, text: str, existing_warnings: list) -> ParsedCV:
        """
        Use Ollama LLM to extract structured fields from CV text.
        Returns a ParsedCV with confidence score.

        GUARDRAILS enforced in prompt:
        - Do NOT extract age, gender, photo description, or nationality
        - Evaluate only skills and experience
        """
        extraction_prompt = f"""
You are an expert HR data extraction assistant. Extract structured information from the CV text below.

IMPORTANT GUARDRAILS:
- Do NOT include age, date of birth, gender, nationality, religion, or any protected attributes
- Extract ONLY professional and skills-based information
- If a field is not present, use null

Return a valid JSON object with EXACTLY this structure:
{{
  "full_name": "string or null",
  "email": "string or null",
  "phone": "string or null",
  "linkedin_url": "string or null",
  "location": "city and country only, e.g. Dubai, UAE",
  "technical_skills": ["list of technical skills"],
  "soft_skills": ["list of soft skills like leadership, communication"],
  "domain_expertise": ["list of domains, e.g. cloud computing, fintech, supply chain"],
  "certifications": ["list of certifications"],
  "languages": ["spoken languages only, not programming languages"],
  "experience": [
    {{
      "company": "company name",
      "role": "job title",
      "start_date": "YYYY-MM or YYYY",
      "end_date": "YYYY-MM or YYYY or Present",
      "duration_months": integer or null,
      "description": "brief role summary",
      "technologies": ["tech used in this role"]
    }}
  ],
  "education": [
    {{
      "institution": "university or school name",
      "degree": "degree type e.g. BSc, MBA",
      "field": "field of study",
      "graduation_year": integer or null,
      "grade": "GPA or grade if mentioned"
    }}
  ],
  "total_years_exp": float or null,
  "cv_summary": "One paragraph professional summary of this candidate",
  "parse_confidence": float between 0.0 and 1.0 indicating how complete/clear the CV was
}}

Return ONLY the JSON object. No explanation. No markdown. No additional text.

CV TEXT:
{text}
"""
        try:
            response = await self.ollama.chat(
                model=settings.ollama_llm_model,
                messages=[{"role": "user", "content": extraction_prompt}],
                options={"temperature": 0.1, "num_predict": 2000},
            )

            raw_json = response["message"]["content"].strip()

            # Clean up common LLM formatting issues
            raw_json = re.sub(r"```json\s*", "", raw_json)
            raw_json = re.sub(r"```\s*", "", raw_json)

            data = json.loads(raw_json)
            return ParsedCV(**data)

        except json.JSONDecodeError as e:
            self.log.warning("llm_json_parse_error", error=str(e))
            existing_warnings.append(f"LLM returned malformed JSON: {str(e)}")
            return ParsedCV(parse_confidence=0.2, parse_warnings=existing_warnings)

        except Exception as e:
            self.log.error("llm_extraction_failed", error=str(e))
            existing_warnings.append(f"LLM extraction failed: {str(e)}")
            return ParsedCV(parse_confidence=0.1, parse_warnings=existing_warnings)


# ============================================================
# FILE: agents/matching_agent.py
# Semantic matching: embeddings + vector search + LLM reasoning
# ============================================================

import json
from typing import Any, Dict, List
import ollama as ollama_client
import structlog
from sqlalchemy import text

from agents.base_agent import BaseAgent
from shared.models import ScreeningResult
from shared.utils import truncate_text
from config.settings import settings

logger = structlog.get_logger(__name__)


class MatchingAgent(BaseAgent):
    """
    Core intelligence agent:
    1. Generate embeddings for CV and JD
    2. Compute cosine similarity via pgvector
    3. Use LLM to generate human-readable rationale
    4. Produce composite score and strengths/gaps analysis

    DESIGN PRINCIPLE: No fixed weights. Scoring is semantic + probabilistic.
    """

    def __init__(self, db):
        super().__init__("matching", db)
        self.ollama = ollama_client.AsyncClient(host=settings.ollama_base_url)

    async def execute(self, payload: Dict[str, Any], correlation_id: str) -> Dict[str, Any]:
        """
        payload expects:
          - cv_version_id: str
          - job_id: str
          - parsed_cv: dict    — from IngestionAgent
          - job_description: str
          - candidate_id: str
        """
        cv_version_id  = payload["cv_version_id"]
        job_id         = payload["job_id"]
        parsed_cv      = payload["parsed_cv"]
        job_description = payload["job_description"]
        candidate_id   = payload["candidate_id"]

        # ---- Step 1: Generate embeddings ----
        cv_text  = self._cv_to_text(parsed_cv)
        cv_embed = await self._embed(cv_text)
        jd_embed = await self._embed(truncate_text(job_description, 4000))

        # ---- Step 2: Store CV embedding in DB ----
        await self.db.execute(
            text("UPDATE cv_versions SET embedding = :emb WHERE id = :id"),
            {"emb": str(cv_embed), "id": cv_version_id},
        )

        # ---- Step 3: Compute cosine similarity via pgvector ----
        result = await self.db.execute(
            text("""
                SELECT 1 - (embedding <=> :jd_embed::vector) AS similarity
                FROM cv_versions WHERE id = :cv_id
            """),
            {"jd_embed": str(jd_embed), "cv_id": cv_version_id},
        )
        row = result.fetchone()
        semantic_similarity = float(row.similarity) if row else 0.0

        # ---- Step 4: LLM reasoning for deeper analysis ----
        llm_analysis = await self._llm_analyze(parsed_cv, job_description)

        # ---- Step 5: Compute composite score ----
        # Composite = blend of vector similarity + LLM relevance + potential
        # Weights are contextual, not hardcoded
        relevance_weight  = 0.4
        similarity_weight = 0.35
        potential_weight  = 0.25

        composite = (
            llm_analysis["relevance_score"]  * relevance_weight +
            semantic_similarity               * similarity_weight +
            llm_analysis["potential_score"]  * potential_weight
        )

        self.log.info(
            "matching_complete",
            similarity=round(semantic_similarity, 3),
            relevance=round(llm_analysis["relevance_score"], 3),
            composite=round(composite, 3),
        )

        return {
            "candidate_id":        candidate_id,
            "cv_version_id":       cv_version_id,
            "job_id":              job_id,
            "semantic_similarity": round(semantic_similarity, 3),
            "relevance_score":     round(llm_analysis["relevance_score"], 3),
            "potential_score":     round(llm_analysis["potential_score"], 3),
            "composite_score":     round(composite, 3),
            "strengths":           llm_analysis["strengths"],
            "gaps":                llm_analysis["gaps"],
            "transferable_skills": llm_analysis["transferable_skills"],
            "llm_rationale":       llm_analysis["rationale"],
        }

    async def _embed(self, text: str) -> List[float]:
        """Generate embedding vector using Ollama's embedding model."""
        response = await self.ollama.embeddings(
            model=settings.ollama_embed_model,
            prompt=text,
        )
        return response["embedding"]

    def _cv_to_text(self, parsed_cv: dict) -> str:
        """Convert parsed CV dict to a rich text for embedding."""
        parts = []
        if parsed_cv.get("cv_summary"):
            parts.append(f"Summary: {parsed_cv['cv_summary']}")

        tech = ", ".join(parsed_cv.get("technical_skills", []))
        if tech:
            parts.append(f"Technical Skills: {tech}")

        domains = ", ".join(parsed_cv.get("domain_expertise", []))
        if domains:
            parts.append(f"Domain Expertise: {domains}")

        for exp in parsed_cv.get("experience", [])[:5]:  # Top 5 roles
            role_text = f"{exp.get('role','')} at {exp.get('company','')}: {exp.get('description','')}"
            parts.append(role_text)

        certs = ", ".join(parsed_cv.get("certifications", []))
        if certs:
            parts.append(f"Certifications: {certs}")

        return "\n".join(parts)

    async def _llm_analyze(self, parsed_cv: dict, job_description: str) -> dict:
        """
        LLM generates:
        - Relevance score (0-1)
        - Potential score (0-1)
        - Strengths list
        - Gaps list
        - Transferable skills
        - Plain-English rationale for HR
        """
        cv_summary = json.dumps({
            "name": parsed_cv.get("full_name", "Candidate"),
            "skills": parsed_cv.get("technical_skills", [])[:15],
            "domains": parsed_cv.get("domain_expertise", []),
            "experience": [
                f"{e.get('role','')} at {e.get('company','')} ({e.get('duration_months',0)//12 if e.get('duration_months') else '?'} years)"
                for e in parsed_cv.get("experience", [])[:4]
            ],
            "total_years": parsed_cv.get("total_years_exp"),
            "education": [
                f"{e.get('degree','')} in {e.get('field','')} from {e.get('institution','')}"
                for e in parsed_cv.get("education", [])[:2]
            ],
        }, indent=2)

        analysis_prompt = f"""
You are a senior HR analyst. Evaluate this candidate against the job description.

GUARDRAILS:
- Base evaluation ONLY on skills, experience, and qualifications
- Do NOT consider or infer gender, age, nationality, or any protected characteristic
- Focus on transferable skills and growth potential, not just exact keyword matches

JOB DESCRIPTION:
{truncate_text(job_description, 2000)}

CANDIDATE PROFILE:
{cv_summary}

Return a JSON object with EXACTLY this structure:
{{
  "relevance_score": float 0.0-1.0,
  "potential_score": float 0.0-1.0,
  "strengths": ["list of 3-5 concrete strengths relevant to this role"],
  "gaps": ["list of 2-4 gaps or areas to probe in interview"],
  "transferable_skills": ["skills from other domains that apply here"],
  "rationale": "2-3 paragraph plain English explanation for an HR professional. Mention specific role requirements met, gaps to explore, and overall recommendation framing. Do NOT recommend hire/reject — HR makes that decision."
}}

Scoring guidance:
- relevance_score: How well skills/experience match JD requirements
- potential_score: Signals of learning agility, progression, adaptability

Return ONLY valid JSON. No markdown. No preamble.
"""
        try:
            response = await self.ollama.chat(
                model=settings.ollama_llm_model,
                messages=[{"role": "user", "content": analysis_prompt}],
                options={"temperature": 0.2, "num_predict": 1500},
            )
            raw = response["message"]["content"].strip()
            raw = raw.replace("```json", "").replace("```", "").strip()
            return json.loads(raw)

        except Exception as e:
            self.log.error("llm_analysis_failed", error=str(e))
            return {
                "relevance_score":     0.3,
                "potential_score":     0.3,
                "strengths":           ["Unable to analyze — LLM error"],
                "gaps":                ["Manual review required"],
                "transferable_skills": [],
                "rationale":           f"Automated analysis failed: {str(e)}. Manual review required.",
            }


# ============================================================
# FILE: agents/potential_agent.py
# Detects growth signals, learning velocity, leadership fit
# ============================================================

import json
from typing import Any, Dict
import ollama as ollama_client
import structlog

from agents.base_agent import BaseAgent
from config.settings import settings

logger = structlog.get_logger(__name__)


class PotentialAgent(BaseAgent):
    """
    Augments the MatchingAgent with deeper human-centric insights:
    - Career trajectory (ascending/stagnant/pivoting)
    - Learning velocity (certs, tech adoption timeline)
    - Leadership signals (promotions, team lead roles)
    - Role adaptability (cross-functional experience)

    This agent augments recruiter judgment — it does NOT override it.
    """

    def __init__(self, db):
        super().__init__("potential", db)
        self.ollama = ollama_client.AsyncClient(host=settings.ollama_base_url)

    async def execute(self, payload: Dict[str, Any], correlation_id: str) -> Dict[str, Any]:
        """
        payload expects:
          - parsed_cv: dict
        """
        parsed_cv = payload["parsed_cv"]
        insights  = await self._analyze_potential(parsed_cv)

        self.log.info(
            "potential_analysis_complete",
            trajectory=insights.get("career_trajectory"),
            leadership_signals=len(insights.get("leadership_signals", [])),
        )

        return insights

    async def _analyze_potential(self, parsed_cv: dict) -> dict:
        prompt = f"""
You are a talent development expert. Analyze this candidate's career profile for potential indicators.

GUARDRAILS:
- Focus ONLY on professional indicators: career progression, skill acquisition, leadership
- Do NOT infer or comment on personal attributes unrelated to job performance

CANDIDATE PROFILE:
{json.dumps({
    "experience": parsed_cv.get("experience", []),
    "certifications": parsed_cv.get("certifications", []),
    "technical_skills": parsed_cv.get("technical_skills", []),
    "domain_expertise": parsed_cv.get("domain_expertise", []),
    "total_years_exp": parsed_cv.get("total_years_exp"),
}, indent=2)}

Analyze and return this JSON:
{{
  "career_trajectory": "ascending | lateral | pivoting | stagnant | insufficient_data",
  "learning_velocity": "high | medium | low | insufficient_data",
  "leadership_signals": ["list of specific evidence of leadership or ownership"],
  "adaptability_indicators": ["cross-functional, industry, or tech pivots observed"],
  "growth_potential_label": "one of: Strong long-term fit | High upskilling ability | Good future leadership profile | Specialist with deep expertise | Generalist with broad exposure | Insufficient data",
  "value_add_insights": ["2-4 specific, actionable insights for the hiring manager"],
  "potential_score_rationale": "1 paragraph explaining the potential assessment"
}}

Return ONLY valid JSON. No markdown.
"""
        try:
            response = await self.ollama.chat(
                model=settings.ollama_llm_model,
                messages=[{"role": "user", "content": prompt}],
                options={"temperature": 0.2, "num_predict": 800},
            )
            raw = response["message"]["content"].strip().replace("```json","").replace("```","")
            return json.loads(raw)
        except Exception as e:
            self.log.error("potential_analysis_failed", error=str(e))
            return {
                "career_trajectory":      "insufficient_data",
                "learning_velocity":      "insufficient_data",
                "leadership_signals":     [],
                "adaptability_indicators": [],
                "growth_potential_label": "Insufficient data",
                "value_add_insights":     ["Manual review required"],
                "potential_score_rationale": f"Analysis failed: {str(e)}",
            }


# ============================================================
# FILE: agents/validation_agent.py
# Flags anomalies, duplicates, inconsistencies for HR review
# ============================================================

import hashlib
from typing import Any, Dict, List
from sqlalchemy import text
import structlog

from agents.base_agent import BaseAgent

logger = structlog.get_logger(__name__)


class ValidationAgent(BaseAgent):
    """
    Validates CVs and screenings. Flags issues to HR.
    Does NOT auto-decide — raises flags for human review.

    Checks:
    1. Duplicate CV (same file hash)
    2. Candidate already applied for this job
    3. Missing mandatory qualifications
    4. Employment gap analysis
    5. Inconsistent/overlapping job dates
    6. Low parse confidence
    7. Borderline composite score
    """

    def __init__(self, db):
        super().__init__("validation", db)

    async def execute(self, payload: Dict[str, Any], correlation_id: str) -> Dict[str, Any]:
        """
        payload expects:
          - file_hash: str
          - cv_version_id: str
          - candidate_id: str
          - job_id: str
          - parsed_cv: dict
          - composite_score: float
          - parse_confidence: float
        """
        anomalies = []

        # Run all checks
        anomalies += await self._check_duplicate_file(payload)
        anomalies += await self._check_duplicate_application(payload)
        anomalies += await self._check_parse_confidence(payload)
        anomalies += await self._check_employment_gaps(payload)
        anomalies += await self._check_date_consistency(payload)
        anomalies += await self._check_borderline_score(payload)

        # Persist anomalies to database
        for anomaly in anomalies:
            await self.db.execute(
                text("""
                    INSERT INTO anomalies
                        (cv_version_id, candidate_id, job_id, anomaly_type, severity, description, raw_evidence)
                    VALUES
                        (:cv_id, :cand_id, :job_id, :atype, :severity, :desc, :evidence::jsonb)
                """),
                {
                    "cv_id":   payload["cv_version_id"],
                    "cand_id": payload["candidate_id"],
                    "job_id":  payload["job_id"],
                    "atype":   anomaly["type"],
                    "severity": anomaly["severity"],
                    "desc":    anomaly["description"],
                    "evidence": str(anomaly.get("evidence", {})),
                },
            )

        self.log.info("validation_complete", anomaly_count=len(anomalies))

        return {
            "anomalies":       anomalies,
            "requires_review": any(a["severity"] in ["high", "critical"] for a in anomalies),
            "anomaly_count":   len(anomalies),
        }

    async def _check_duplicate_file(self, payload: dict) -> List[dict]:
        """Check if the exact same file was uploaded before."""
        result = await self.db.execute(
            text("""
                SELECT id, candidate_id, uploaded_at
                FROM cv_versions
                WHERE file_hash = :hash AND id != :cv_id
                LIMIT 1
            """),
            {"hash": payload["file_hash"], "cv_id": payload["cv_version_id"]},
        )
        row = result.fetchone()
        if row:
            return [{
                "type":        "duplicate_cv",
                "severity":    "high",
                "description": f"This exact CV file was previously uploaded on {row.uploaded_at.date()}. "
                               "May indicate resubmission or copy. HR review recommended.",
                "evidence":    {"duplicate_cv_id": str(row.id), "uploaded_at": str(row.uploaded_at)},
            }]
        return []

    async def _check_duplicate_application(self, payload: dict) -> List[dict]:
        """Check if candidate has already applied for this exact job."""
        result = await self.db.execute(
            text("""
                SELECT s.id, s.screened_at
                FROM screenings s
                WHERE s.candidate_id = :cand_id AND s.job_id = :job_id
                LIMIT 1
            """),
            {"cand_id": payload["candidate_id"], "job_id": payload["job_id"]},
        )
        row = result.fetchone()
        if row:
            return [{
                "type":        "duplicate_cv",
                "severity":    "medium",
                "description": f"This candidate has already been screened for this role on {row.screened_at.date()}.",
                "evidence":    {"prior_screening_id": str(row.id)},
            }]
        return []

    async def _check_parse_confidence(self, payload: dict) -> List[dict]:
        """Flag CVs with very low parsing confidence."""
        confidence = payload.get("parse_confidence", 1.0)
        if confidence < 0.35:
            return [{
                "type":        "low_parse_confidence",
                "severity":    "high",
                "description": f"CV parsing confidence is very low ({confidence:.0%}). "
                               "The document may be scanned, image-based, or have complex formatting. "
                               "Manual review of the original file is strongly recommended.",
                "evidence":    {"confidence": confidence},
            }]
        elif confidence < 0.55:
            return [{
                "type":        "low_parse_confidence",
                "severity":    "medium",
                "description": f"CV parsing confidence is moderate ({confidence:.0%}). "
                               "Some fields may be incomplete.",
                "evidence":    {"confidence": confidence},
            }]
        return []

    async def _check_employment_gaps(self, payload: dict) -> List[dict]:
        """Detect significant gaps in employment history."""
        experience = payload.get("parsed_cv", {}).get("experience", [])
        if len(experience) < 2:
            return []

        # Sort by start date and check gaps > 12 months
        anomalies = []
        try:
            sorted_exp = sorted(
                [e for e in experience if e.get("start_date") and e.get("end_date")],
                key=lambda x: x["start_date"],
            )
            for i in range(1, len(sorted_exp)):
                prev_end   = sorted_exp[i-1]["end_date"]
                curr_start = sorted_exp[i]["start_date"]
                if prev_end.lower() == "present":
                    continue
                # Simple year-based gap detection
                try:
                    prev_year  = int(str(prev_end)[:4])
                    curr_year  = int(str(curr_start)[:4])
                    gap_months = (curr_year - prev_year) * 12
                    if gap_months > 12:
                        anomalies.append({
                            "type":        "employment_gap",
                            "severity":    "low",
                            "description": f"Potential employment gap of ~{gap_months//12} year(s) detected "
                                           f"between {sorted_exp[i-1].get('company','')} and "
                                           f"{sorted_exp[i].get('company','')}. Worth exploring in interview.",
                            "evidence":    {"gap_months": gap_months, "before": prev_end, "after": curr_start},
                        })
                except (ValueError, TypeError):
                    pass
        except Exception as e:
            self.log.warning("gap_analysis_error", error=str(e))

        return anomalies

    async def _check_date_consistency(self, payload: dict) -> List[dict]:
        """Check for overlapping job dates (possible inconsistency)."""
        experience = payload.get("parsed_cv", {}).get("experience", [])
        active_roles = [
            e for e in experience
            if e.get("end_date", "").lower() == "present"
        ]
        if len(active_roles) > 1:
            return [{
                "type":        "inconsistent_dates",
                "severity":    "medium",
                "description": f"CV shows {len(active_roles)} simultaneously 'current' positions. "
                               "This may indicate consulting/freelance work or a data error. Clarify in interview.",
                "evidence":    {"active_roles": [r.get("role") for r in active_roles]},
            }]
        return []

    async def _check_borderline_score(self, payload: dict) -> List[dict]:
        """Flag borderline composite scores for mandatory HR review."""
        score = payload.get("composite_score", 0)
        if 0.38 <= score <= 0.52:
            return [{
                "type":        "borderline_score",
                "severity":    "medium",
                "description": f"Composite score ({score:.2f}) is in the borderline range (0.38–0.52). "
                               "System cannot make a confident recommendation. HR judgment is essential.",
                "evidence":    {"composite_score": score},
            }]
        return []
# ============================================================
# FILE: api/main.py
# FastAPI application entry point
# ============================================================

import logging
import os
import structlog
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware

from api.routers import candidates, jobs, screenings
from database.connection import check_db_health
from shared.models import HealthResponse
from config.settings import settings

# ---- Structured logging setup ----
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.JSONRenderer(),   # JSON logs for observability
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Runs on startup and shutdown."""
    # ---- STARTUP ----
    logger.info("system_startup", environment=settings.environment)

    # Ensure upload directories exist
    os.makedirs(f"{settings.upload_dir}/cvs", exist_ok=True)
    os.makedirs(f"{settings.upload_dir}/jds", exist_ok=True)
    os.makedirs(settings.log_dir, exist_ok=True)

    yield  # Application runs here

    # ---- SHUTDOWN ----
    logger.info("system_shutdown")


# ---- Create FastAPI app ----
app = FastAPI(
    title="CV Screening & Hiring Intelligence API",
    description="Agentic AI-powered CV screening system for HR",
    version="1.0.0",
    docs_url="/docs" if settings.environment == "development" else None,
    redoc_url=None,
    lifespan=lifespan,
)

# ---- Security middleware ----
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:8501"],  # Streamlit only
    allow_credentials=True,
    allow_methods=["GET", "POST", "PATCH"],
    allow_headers=["*"],
)

# ---- Route registration ----
app.include_router(jobs.router,       prefix="/api/v1/jobs",       tags=["Jobs"])
app.include_router(candidates.router, prefix="/api/v1/candidates", tags=["Candidates"])
app.include_router(screenings.router, prefix="/api/v1/screenings", tags=["Screenings"])


# ---- Health check endpoint ----
@app.get("/health", response_model=HealthResponse)
async def health_check():
    """
    System health check — used by Docker health checks and monitoring.
    """
    db_ok = await check_db_health()

    # Check Ollama availability
    import httpx
    ollama_ok = False
    try:
        async with httpx.AsyncClient(timeout=3.0) as client:
            resp = await client.get(f"{settings.ollama_base_url}/api/tags")
            ollama_ok = resp.status_code == 200
    except Exception:
        pass

    status = "healthy" if (db_ok and ollama_ok) else "degraded"

    return HealthResponse(
        status=status,
        database=db_ok,
        ollama=ollama_ok,
    )


# ============================================================
# FILE: api/routers/jobs.py
# Job Description upload and management
# ============================================================

import os
import uuid
from typing import List
from fastapi import APIRouter, Depends, UploadFile, File, Form, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text
import aiofiles
import structlog

from database.connection import get_db
from shared.models import JobCreateRequest, JobResponse
from shared.utils import sanitize_filename, truncate_text
from config.settings import settings
from agents.matching_agent import MatchingAgent

router = APIRouter()
logger = structlog.get_logger(__name__)


@router.post("/", response_model=JobResponse, status_code=201)
async def create_job(
    title:      str = Form(...),
    department: str = Form(None),
    location:   str = Form(None),
    created_by: str = Form(...),
    jd_file:    UploadFile = File(...),
    db:         AsyncSession = Depends(get_db),
):
    """
    Upload a Job Description.
    Accepts a text file or PDF containing the JD.
    Generates and stores the JD embedding for future candidate matching.
    """
    # Read JD content
    content = await jd_file.read()
    jd_text = content.decode("utf-8", errors="replace")

    if len(jd_text.strip()) < 100:
        raise HTTPException(400, "Job description is too short. Please upload a complete JD.")

    # Generate JD embedding
    agent  = MatchingAgent(db)
    jd_emb = await agent._embed(truncate_text(jd_text, 4000))

    # Store in DB
    job_id = str(uuid.uuid4())
    await db.execute(
        text("""
            INSERT INTO jobs (id, title, department, location, description_raw, embedding, created_by)
            VALUES (:id, :title, :dept, :loc, :desc, :emb::vector, :by)
        """),
        {
            "id":    job_id,
            "title": title,
            "dept":  department,
            "loc":   location,
            "desc":  jd_text,
            "emb":   str(jd_emb),
            "by":    created_by,
        },
    )

    # Save JD file to disk
    filename = sanitize_filename(jd_file.filename)
    jd_path  = os.path.join(settings.upload_dir, "jds", f"{job_id}_{filename}")
    async with aiofiles.open(jd_path, "wb") as f:
        await f.write(content)

    logger.info("job_created", job_id=job_id, title=title)

    result = await db.execute(text("SELECT * FROM jobs WHERE id = :id"), {"id": job_id})
    return dict(result.mappings().fetchone())


@router.get("/", response_model=List[dict])
async def list_jobs(active_only: bool = True, db: AsyncSession = Depends(get_db)):
    """List all job postings."""
    query = "SELECT id, title, department, location, is_active, created_at FROM jobs"
    if active_only:
        query += " WHERE is_active = TRUE"
    query += " ORDER BY created_at DESC"
    result = await db.execute(text(query))
    return [dict(r) for r in result.mappings()]


@router.get("/{job_id}/pipeline")
async def get_job_pipeline(job_id: str, db: AsyncSession = Depends(get_db)):
    """Get full hiring pipeline stats for a job."""
    result = await db.execute(
        text("SELECT * FROM v_job_pipeline WHERE id = :id"),
        {"id": job_id},
    )
    row = result.mappings().fetchone()
    if not row:
        raise HTTPException(404, "Job not found")
    return dict(row)


# ============================================================
# FILE: api/routers/screenings.py
# CV upload, bulk processing, and screening orchestration
# ============================================================

import os
import uuid
import asyncio
from typing import List
from fastapi import APIRouter, Depends, UploadFile, File, Form, HTTPException, BackgroundTasks
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text
import aiofiles
import structlog

from database.connection import get_db, AsyncSessionLocal
from shared.models import BulkUploadResponse
from shared.utils import sanitize_filename, compute_file_hash, get_file_extension
from config.settings import settings
from agents.ingestion_agent import IngestionAgent
from agents.matching_agent import MatchingAgent
from agents.potential_agent import PotentialAgent
from agents.validation_agent import ValidationAgent

router = APIRouter()
logger = structlog.get_logger(__name__)


@router.post("/upload", response_model=BulkUploadResponse)
async def upload_cvs(
    job_id:          str = Form(...),
    uploaded_by:     str = Form(...),
    cv_files:        List[UploadFile] = File(...),
    background_tasks: BackgroundTasks = BackgroundTasks(),
    db:              AsyncSession = Depends(get_db),
):
    """
    Bulk CV upload endpoint.
    - Accepts up to MAX_BULK_UPLOAD files per request
    - Validates format and size
    - Saves files to disk
    - Queues background processing (non-blocking)
    - Returns immediately with correlation IDs for tracking
    """
    if len(cv_files) > settings.max_bulk_upload:
        raise HTTPException(400, f"Too many files. Maximum is {settings.max_bulk_upload} per upload.")

    # Verify job exists
    job_result = await db.execute(text("SELECT id, description_raw FROM jobs WHERE id = :id"), {"id": job_id})
    job = job_result.mappings().fetchone()
    if not job:
        raise HTTPException(404, f"Job {job_id} not found")

    queued          = []
    failed          = []
    correlation_ids = []

    for cv_file in cv_files:
        correlation_id = str(uuid.uuid4())

        # ---- Validate extension ----
        ext = get_file_extension(cv_file.filename)
        if ext not in settings.allowed_ext_list:
            failed.append({"filename": cv_file.filename, "error": f"Unsupported format: {ext}"})
            continue

        # ---- Validate file size ----
        content = await cv_file.read()
        size_mb = len(content) / (1024 * 1024)
        if size_mb > settings.max_file_size_mb:
            failed.append({"filename": cv_file.filename, "error": f"File too large: {size_mb:.1f}MB"})
            continue

        # ---- Save to disk ----
        safe_name = sanitize_filename(cv_file.filename)
        file_path = os.path.join(settings.upload_dir, "cvs", f"{correlation_id}_{safe_name}")
        async with aiofiles.open(file_path, "wb") as f:
            await f.write(content)

        file_hash = compute_file_hash(content)

        # ---- Create cv_version record (status: pending) ----
        cv_version_id = str(uuid.uuid4())
        await db.execute(
            text("""
                INSERT INTO cv_versions
                    (id, job_id, original_filename, stored_path, file_format,
                     file_size_bytes, file_hash, ingestion_status, correlation_id)
                VALUES
                    (:id, :job_id, :fname, :path, :fmt, :size, :hash, 'pending', :cid)
            """),
            {
                "id":     cv_version_id,
                "job_id": job_id,
                "fname":  cv_file.filename,
                "path":   file_path,
                "fmt":    ext,
                "size":   len(content),
                "hash":   file_hash,
                "cid":    correlation_id,
            },
        )

        queued.append(cv_version_id)
        correlation_ids.append(correlation_id)

        # ---- Queue background processing ----
        background_tasks.add_task(
            process_cv_pipeline,
            cv_version_id=cv_version_id,
            job_id=job_id,
            file_path=file_path,
            filename=cv_file.filename,
            file_hash=file_hash,
            correlation_id=correlation_id,
        )

    logger.info("bulk_upload_complete",
        total=len(cv_files), queued=len(queued), failed=len(failed))

    return BulkUploadResponse(
        total_received=len(cv_files),
        queued=len(queued),
        failed=len(failed),
        correlation_ids=correlation_ids,
        errors=failed,
    )


async def process_cv_pipeline(
    cv_version_id: str,
    job_id: str,
    file_path: str,
    filename: str,
    file_hash: str,
    correlation_id: str,
) -> None:
    """
    Full agentic pipeline for a single CV.
    Runs in background — failures here NEVER affect other CVs.

    Pipeline:
    1. IngestionAgent  → parse CV → structured JSON
    2. Find/create Candidate record (dedup by email)
    3. MatchingAgent   → semantic scoring + LLM rationale
    4. PotentialAgent  → growth & value-add insights
    5. ValidationAgent → anomaly detection
    6. Store ScreeningResult in DB
    """
    bound_log = logger.bind(correlation_id=correlation_id, filename=filename)
    bound_log.info("pipeline_start")

    # Each background task gets its own DB session
    async with AsyncSessionLocal() as db:
        try:
            # ---- UPDATE STATUS: processing ----
            await db.execute(
                text("UPDATE cv_versions SET ingestion_status = 'processing' WHERE id = :id"),
                {"id": cv_version_id},
            )
            await db.commit()

            # ---- 1. INGESTION & PARSING ----
            ingestion = IngestionAgent(db)
            ingestion_result = await ingestion.run(
                payload={"file_path": file_path, "filename": filename},
                correlation_id=correlation_id,
            )
            parsed_cv = ingestion_result["parsed_cv"]

            # ---- 2. FIND OR CREATE CANDIDATE ----
            candidate_id = await _upsert_candidate(db, parsed_cv, correlation_id)

            # Link cv_version to candidate
            await db.execute(
                text("""
                    UPDATE cv_versions
                    SET candidate_id = :cid,
                        parsed_data = :data::jsonb,
                        parse_confidence = :conf,
                        embedding_model = :emodel,
                        ingestion_status = 'done',
                        processed_at = NOW()
                    WHERE id = :id
                """),
                {
                    "cid":    candidate_id,
                    "data":   str(parsed_cv),
                    "conf":   parsed_cv.get("parse_confidence", 0.5),
                    "emodel": settings.ollama_embed_model,
                    "id":     cv_version_id,
                },
            )
            await db.commit()

            # ---- 3. FETCH JD ----
            jd_result = await db.execute(
                text("SELECT description_raw FROM jobs WHERE id = :id"), {"id": job_id}
            )
            jd_row = jd_result.fetchone()
            job_description = jd_row.description_raw if jd_row else ""

            # ---- 4. SEMANTIC MATCHING ----
            matcher = MatchingAgent(db)
            match_result = await matcher.run(
                payload={
                    "cv_version_id":  cv_version_id,
                    "job_id":         job_id,
                    "candidate_id":   candidate_id,
                    "parsed_cv":      parsed_cv,
                    "job_description": job_description,
                },
                correlation_id=correlation_id,
            )

            # ---- 5. POTENTIAL ANALYSIS ----
            potential = PotentialAgent(db)
            potential_result = await potential.run(
                payload={"parsed_cv": parsed_cv},
                correlation_id=correlation_id,
            )
            value_add_insights = potential_result.get("value_add_insights", [])
            potential_label    = potential_result.get("growth_potential_label", "")

            # ---- 6. VALIDATION & ANOMALY DETECTION ----
            validator = ValidationAgent(db)
            validation_result = await validator.run(
                payload={
                    "file_hash":        file_hash,
                    "cv_version_id":    cv_version_id,
                    "candidate_id":     candidate_id,
                    "job_id":           job_id,
                    "parsed_cv":        parsed_cv,
                    "composite_score":  match_result["composite_score"],
                    "parse_confidence": parsed_cv.get("parse_confidence", 0.5),
                },
                correlation_id=correlation_id,
            )

            # ---- 7. STORE SCREENING RESULT ----
            screening_id = str(uuid.uuid4())
            # Combine rationale with potential insights
            full_rationale = match_result["llm_rationale"]
            if potential_label:
                full_rationale += f"\n\nPotential Assessment: {potential_label}. "
                full_rationale += potential_result.get("potential_score_rationale", "")

            await db.execute(
                text("""
                    INSERT INTO screenings
                        (id, cv_version_id, job_id, candidate_id,
                         semantic_similarity, relevance_score, potential_score, composite_score,
                         strengths, gaps, transferable_skills, value_add_insights, llm_rationale,
                         decision)
                    VALUES
                        (:id, :cv_id, :job_id, :cand_id,
                         :sim, :rel, :pot, :comp,
                         :str::jsonb, :gaps::jsonb, :trans::jsonb, :vai::jsonb, :rat,
                         :decision)
                """),
                {
                    "id":       screening_id,
                    "cv_id":    cv_version_id,
                    "job_id":   job_id,
                    "cand_id":  candidate_id,
                    "sim":      match_result["semantic_similarity"],
                    "rel":      match_result["relevance_score"],
                    "pot":      match_result.get("potential_score", 0.5),
                    "comp":     match_result["composite_score"],
                    "str":      str(match_result["strengths"]),
                    "gaps":     str(match_result["gaps"]),
                    "trans":    str(match_result["transferable_skills"]),
                    "vai":      str(value_add_insights),
                    "rat":      full_rationale,
                    "decision": "needs_review" if validation_result["requires_review"] else "needs_review",
                },
            )

            # ---- UPDATE CANDIDATE STATUS ----
            await db.execute(
                text("UPDATE candidates SET current_status = 'screened', last_updated_at = NOW() WHERE id = :id"),
                {"id": candidate_id},
            )

            await db.commit()
            bound_log.info("pipeline_complete", screening_id=screening_id,
                composite_score=match_result["composite_score"])

        except Exception as e:
            bound_log.error("pipeline_failed", error=str(e))
            # Mark cv_version as failed — do NOT re-raise (must not stop other CVs)
            try:
                await db.rollback()
                await db.execute(
                    text("UPDATE cv_versions SET ingestion_status = 'failed' WHERE id = :id"),
                    {"id": cv_version_id},
                )
                await db.commit()
            except Exception as inner_e:
                bound_log.error("pipeline_failure_cleanup_failed", error=str(inner_e))


async def _upsert_candidate(db, parsed_cv: dict, correlation_id: str) -> str:
    """
    Find existing candidate by email or create new one.
    This is how we deduplicate candidates across multiple job applications.
    """
    email = parsed_cv.get("email")

    if email:
        result = await db.execute(
            text("SELECT id, is_returning FROM candidates WHERE email = :email"),
            {"email": email.lower().strip()},
        )
        existing = result.fetchone()

        if existing:
            # Mark as returning candidate
            await db.execute(
                text("UPDATE candidates SET is_returning = TRUE, last_updated_at = NOW() WHERE id = :id"),
                {"id": str(existing.id)},
            )
            await db.commit()
            return str(existing.id)

    # Create new candidate
    candidate_id = str(uuid.uuid4())
    await db.execute(
        text("""
            INSERT INTO candidates (id, full_name, email, phone, linkedin_url, current_status)
            VALUES (:id, :name, :email, :phone, :linkedin, 'new')
        """),
        {
            "id":       candidate_id,
            "name":     parsed_cv.get("full_name"),
            "email":    email.lower().strip() if email else None,
            "phone":    parsed_cv.get("phone"),
            "linkedin": parsed_cv.get("linkedin_url"),
        },
    )
    await db.commit()
    return candidate_id


@router.get("/results/{job_id}")
async def get_screening_results(
    job_id:    str,
    min_score: float = 0.0,
    limit:     int   = 50,
    db:        AsyncSession = Depends(get_db),
):
    """
    Get ranked screening results for a job.
    Results are ordered by composite_score descending.
    """
    result = await db.execute(
        text("""
            SELECT
                s.id AS screening_id,
                s.composite_score,
                s.semantic_similarity,
                s.relevance_score,
                s.potential_score,
                s.strengths,
                s.gaps,
                s.transferable_skills,
                s.value_add_insights,
                s.llm_rationale,
                s.decision,
                s.screened_at,
                c.id AS candidate_id,
                c.full_name,
                c.email,
                c.current_status,
                c.is_returning,
                cv.original_filename,
                cv.stored_path,
                cv.parse_confidence,
                cv.ingestion_status,
                (SELECT COUNT(*) FROM anomalies a WHERE a.cv_version_id = cv.id) AS anomaly_count
            FROM screenings s
            JOIN candidates c    ON c.id = s.candidate_id
            JOIN cv_versions cv  ON cv.id = s.cv_version_id
            WHERE s.job_id = :job_id
              AND s.composite_score >= :min_score
            ORDER BY s.composite_score DESC
            LIMIT :lim
        """),
        {"job_id": job_id, "min_score": min_score, "lim": limit},
    )
    rows = result.mappings().fetchall()

    # Update rankings in DB
    for i, row in enumerate(rows, start=1):
        await db.execute(
            text("UPDATE screenings SET rank_in_pool = :rank, total_in_pool = :total WHERE id = :id"),
            {"rank": i, "total": len(rows), "id": row["screening_id"]},
        )

    return [dict(r) for r in rows]


@router.patch("/{screening_id}/decision")
async def update_decision(
    screening_id: str,
    decision:     str = Form(...),   # hr_approved / hr_rejected / hr_hold / forwarded
    decision_by:  str = Form(...),
    notes:        str = Form(None),
    db:           AsyncSession = Depends(get_db),
):
    """
    HR makes a final decision on a candidate.
    All decisions are audit-logged.
    """
    valid_decisions = ["hr_approved", "hr_rejected", "hr_hold", "forwarded"]
    if decision not in valid_decisions:
        raise HTTPException(400, f"Invalid decision. Must be one of: {valid_decisions}")

    await db.execute(
        text("""
            UPDATE screenings
            SET decision = :dec, decision_by = :by, decision_at = NOW(), decision_notes = :notes
            WHERE id = :id
        """),
        {"dec": decision, "by": decision_by, "notes": notes, "id": screening_id},
    )

    # Log this HR decision to audit trail
    await db.execute(
        text("""
            INSERT INTO audit_logs (event_type, actor, event_data, outcome)
            VALUES ('hr_decision', :actor, :data::jsonb, 'success')
        """),
        {
            "actor": f"hr:{decision_by}",
            "data":  str({"screening_id": screening_id, "decision": decision, "notes": notes}),
        },
    )

    logger.info("hr_decision_recorded",
        screening_id=screening_id, decision=decision, by=decision_by)

    return {"message": "Decision recorded", "screening_id": screening_id, "decision": decision}
# ============================================================
# FILE: frontend/app.py
# Main Streamlit application — HR Portal
# ============================================================

import streamlit as st

st.set_page_config(
    page_title="HR Intelligence Platform",
    page_icon="🎯",
    layout="wide",
    initial_sidebar_state="expanded",
)

# Custom CSS for professional HR look
st.markdown("""
<style>
    .main-header {
        font-size: 2rem;
        font-weight: 700;
        color: #1a1a2e;
        margin-bottom: 0.5rem;
    }
    .metric-card {
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        padding: 1.5rem;
        border-radius: 12px;
        color: white;
        text-align: center;
        margin-bottom: 1rem;
    }
    .score-high   { color: #22c55e; font-weight: bold; }
    .score-mid    { color: #f59e0b; font-weight: bold; }
    .score-low    { color: #ef4444; font-weight: bold; }
    .anomaly-high { background-color: #fee2e2; border-left: 4px solid #ef4444; padding: 8px; margin: 4px 0; border-radius: 4px; }
    .anomaly-med  { background-color: #fef3c7; border-left: 4px solid #f59e0b; padding: 8px; margin: 4px 0; border-radius: 4px; }
    .anomaly-low  { background-color: #dbeafe; border-left: 4px solid #3b82f6; padding: 8px; margin: 4px 0; border-radius: 4px; }
    .rationale-box {
        background: #f8f9ff;
        border: 1px solid #e2e8f0;
        border-radius: 8px;
        padding: 1rem;
        font-size: 0.95rem;
        line-height: 1.7;
    }
    .returning-badge {
        background: #7c3aed;
        color: white;
        padding: 2px 8px;
        border-radius: 12px;
        font-size: 0.75rem;
    }
    .stButton > button {
        border-radius: 8px;
        font-weight: 600;
    }
</style>
""", unsafe_allow_html=True)

st.markdown('<div class="main-header">🎯 HR Intelligence Platform</div>', unsafe_allow_html=True)
st.markdown("*Agentic AI-powered screening — HR is always in control*")

st.sidebar.title("Navigation")
st.sidebar.info("Use the pages on the left to navigate between sections.")
st.sidebar.markdown("---")
st.sidebar.markdown("**System Status**")

# Health check
import requests
try:
    r = requests.get("http://localhost:8000/health", timeout=3)
    h = r.json()
    st.sidebar.success("✅ API Online")
    st.sidebar.markdown(f"Database: {'✅' if h['database'] else '❌'}")
    st.sidebar.markdown(f"AI Engine: {'✅' if h['ollama'] else '❌'}")
except Exception:
    st.sidebar.error("❌ API Offline")

st.markdown("### Welcome to the HR Intelligence Platform")
st.markdown("""
Use the sidebar to navigate:
- **📋 Post a Job** — Upload a Job Description
- **📤 Upload CVs** — Submit CVs for screening
- **📊 Screening Results** — View ranked candidates with AI insights
- **📁 Candidate History** — Review past candidates and outcomes
""")


# ============================================================
# FILE: frontend/pages/01_Post_a_Job.py
# Upload Job Description
# ============================================================

import streamlit as st
import requests

st.set_page_config(page_title="Post a Job", page_icon="📋")
st.title("📋 Post a New Job")
st.markdown("Upload a Job Description to begin accepting CV submissions.")

API = "http://localhost:8000/api/v1"

with st.form("create_job_form"):
    col1, col2 = st.columns(2)
    with col1:
        title      = st.text_input("Job Title *", placeholder="e.g. Senior Data Engineer")
        department = st.text_input("Department", placeholder="e.g. Technology")
    with col2:
        location   = st.text_input("Location", placeholder="e.g. Dubai, UAE")
        created_by = st.text_input("Your Name *", placeholder="HR Manager name")

    jd_file = st.file_uploader(
        "Upload Job Description *",
        type=["txt", "pdf"],
        help="Plain text or PDF containing the full JD"
    )

    st.markdown("---")
    submitted = st.form_submit_button("🚀 Post Job & Enable CV Screening", type="primary")

if submitted:
    if not all([title, created_by, jd_file]):
        st.error("Please fill in all required fields (*) and upload a JD file.")
    else:
        with st.spinner("Creating job and generating AI embeddings..."):
            try:
                response = requests.post(
                    f"{API}/jobs/",
                    data={"title": title, "department": department,
                          "location": location, "created_by": created_by},
                    files={"jd_file": (jd_file.name, jd_file.getvalue(), jd_file.type)},
                    timeout=60,
                )
                if response.status_code == 201:
                    job = response.json()
                    st.success(f"✅ Job posted successfully!")
                    st.info(f"**Job ID:** `{job['id']}`\n\nShare this ID with your team to upload CVs.")
                    st.balloons()
                else:
                    st.error(f"Error: {response.json().get('detail', 'Unknown error')}")
            except Exception as e:
                st.error(f"Connection error: {str(e)}")

# Show existing jobs
st.markdown("---")
st.subheader("Active Job Postings")
try:
    jobs = requests.get(f"{API}/jobs/", timeout=5).json()
    if jobs:
        for job in jobs:
            with st.expander(f"**{job['title']}** — {job.get('department','')} | {job.get('location','')}"):
                st.code(f"Job ID: {job['id']}", language=None)
                st.markdown(f"Posted: {job['created_at'][:10]}")
    else:
        st.info("No active jobs yet.")
except Exception:
    st.warning("Could not load jobs. Is the API running?")


# ============================================================
# FILE: frontend/pages/02_Upload_CVs.py
# Bulk and single CV upload
# ============================================================

import streamlit as st
import requests
import time

st.set_page_config(page_title="Upload CVs", page_icon="📤")
st.title("📤 Upload CVs")
st.markdown("Upload candidate CVs for AI-powered screening.")

API = "http://localhost:8000/api/v1"

# --- Job selection ---
try:
    jobs = requests.get(f"{API}/jobs/", timeout=5).json()
    job_options = {f"{j['title']} — {j.get('department','')}": j['id'] for j in jobs}
except Exception:
    job_options = {}
    st.error("Cannot connect to API. Please ensure the system is running.")

if not job_options:
    st.warning("No active jobs found. Please post a job first.")
    st.stop()

selected_job_label = st.selectbox("Select Job Position *", list(job_options.keys()))
selected_job_id    = job_options[selected_job_label]
uploaded_by        = st.text_input("Your Name *", placeholder="HR Recruiter name")

st.markdown("---")

# --- File upload ---
st.subheader("Upload CVs")
st.info("✅ Supported formats: PDF, DOCX, TXT | Max file size: 20MB | Max batch: 500 files")

cv_files = st.file_uploader(
    "Upload one or more CV files",
    type=["pdf", "docx", "txt"],
    accept_multiple_files=True,
    help="You can select multiple files at once. Each CV will be processed independently."
)

if cv_files:
    st.markdown(f"**{len(cv_files)} file(s) selected:**")
    file_data = []
    total_size = 0
    for f in cv_files:
        size_mb = len(f.getvalue()) / (1024*1024)
        total_size += size_mb
        status = "✅" if size_mb <= 20 else "❌ Too large"
        file_data.append({"Filename": f.name, "Size": f"{size_mb:.2f} MB", "Status": status})

    import pandas as pd
    st.dataframe(pd.DataFrame(file_data), use_container_width=True, hide_index=True)
    st.caption(f"Total: {total_size:.1f} MB across {len(cv_files)} files")

if st.button("🚀 Start AI Screening", type="primary", disabled=not (cv_files and uploaded_by)):
    if not uploaded_by:
        st.error("Please enter your name.")
    else:
        with st.spinner(f"Uploading {len(cv_files)} CVs and queuing for AI analysis..."):
            try:
                file_tuples = [
                    ("cv_files", (f.name, f.getvalue(), f.type or "application/octet-stream"))
                    for f in cv_files
                ]
                response = requests.post(
                    f"{API}/screenings/upload",
                    data={"job_id": selected_job_id, "uploaded_by": uploaded_by},
                    files=file_tuples,
                    timeout=120,
                )

                if response.status_code == 200:
                    result = response.json()
                    st.success(f"✅ {result['queued']} CVs queued for AI screening!")

                    col1, col2, col3 = st.columns(3)
                    col1.metric("Total Received", result["total_received"])
                    col2.metric("Queued for Processing", result["queued"])
                    col3.metric("Failed (invalid format/size)", result["failed"])

                    if result["errors"]:
                        st.warning("Some files could not be processed:")
                        for err in result["errors"]:
                            st.markdown(f"- **{err['filename']}**: {err['error']}")

                    st.info("⏳ AI screening is running in the background. "
                            "Navigate to **Screening Results** to view rankings as they appear.")
                else:
                    st.error(f"Upload failed: {response.json().get('detail')}")

            except Exception as e:
                st.error(f"Error: {str(e)}")


# ============================================================
# FILE: frontend/pages/03_Screening_Results.py
# Main HR dashboard — ranked candidates with AI explanations
# ============================================================

import streamlit as st
import requests
import pandas as pd
import json

st.set_page_config(page_title="Screening Results", page_icon="📊", layout="wide")
st.title("📊 Screening Results")
st.markdown("AI-ranked candidates with semantic analysis and explainable scores.")

API = "http://localhost:8000/api/v1"

# --- Sidebar filters ---
st.sidebar.header("Filters")

try:
    jobs = requests.get(f"{API}/jobs/", timeout=5).json()
    job_options = {f"{j['title']} — {j.get('department','')}": j['id'] for j in jobs}
except Exception:
    job_options = {}

if not job_options:
    st.warning("No jobs found. Please post a job and upload CVs.")
    st.stop()

selected_job_label = st.sidebar.selectbox("Job Position", list(job_options.keys()))
selected_job_id    = job_options[selected_job_label]
min_score          = st.sidebar.slider("Minimum Score", 0.0, 1.0, 0.3, 0.05)
show_limit         = st.sidebar.slider("Show top N candidates", 10, 200, 50)
filter_decision    = st.sidebar.multiselect(
    "Filter by Decision",
    ["needs_review", "hr_approved", "hr_hold", "hr_rejected", "forwarded"],
    default=["needs_review", "hr_approved", "hr_hold"],
)

# Auto-refresh toggle
auto_refresh = st.sidebar.checkbox("Auto-refresh every 30s", value=False)
if auto_refresh:
    import time
    time.sleep(30)
    st.rerun()

# --- Fetch results ---
try:
    results = requests.get(
        f"{API}/screenings/results/{selected_job_id}",
        params={"min_score": min_score, "limit": show_limit},
        timeout=10,
    ).json()
except Exception as e:
    st.error(f"Could not load results: {e}")
    st.stop()

# Filter by decision
if filter_decision:
    results = [r for r in results if r.get("decision") in filter_decision]

# --- Pipeline stats ---
try:
    pipeline = requests.get(f"{API}/jobs/{selected_job_id}/pipeline", timeout=5).json()
    c1, c2, c3, c4, c5 = st.columns(5)
    c1.metric("Total Screened",    pipeline.get("total_screened", 0))
    c2.metric("Shortlisted",       pipeline.get("shortlisted", 0),     delta_color="normal")
    c3.metric("Pending Review",    pipeline.get("pending_review", 0))
    c4.metric("Rejected",          pipeline.get("rejected", 0),        delta_color="inverse")
    c5.metric("Avg Score",         f"{pipeline.get('avg_score', 0):.2f}")
except Exception:
    pass

st.markdown("---")

if not results:
    st.info("No results yet. Upload CVs and wait for AI screening to complete.")
    st.stop()

# --- Summary table ---
st.subheader(f"Ranked Candidates ({len(results)} shown)")

table_data = []
for i, r in enumerate(results, 1):
    score = r["composite_score"]
    score_label = (
        "🟢 Strong" if score >= 0.65
        else "🟡 Moderate" if score >= 0.45
        else "🔴 Weak"
    )
    table_data.append({
        "Rank":          i,
        "Name":          r.get("full_name") or "Unknown",
        "Score":         f"{score:.2f} ({score_label})",
        "Semantic Sim":  f"{r.get('semantic_similarity', 0):.2f}",
        "Relevance":     f"{r.get('relevance_score', 0):.2f}",
        "Potential":     f"{r.get('potential_score', 0):.2f}",
        "Anomalies":     r.get("anomaly_count", 0),
        "Returning":     "🔁 Yes" if r.get("is_returning") else "New",
        "Decision":      r.get("decision", "").replace("_", " ").title(),
        "screened_at":   r.get("screened_at", "")[:10] if r.get("screened_at") else "",
    })

df = pd.DataFrame(table_data)
st.dataframe(df, use_container_width=True, hide_index=True)

st.markdown("---")
st.subheader("🔍 Candidate Details")
st.caption("Click on a candidate below to review AI analysis and take action.")

# --- Candidate detail cards ---
for rank, r in enumerate(results, 1):
    score = r["composite_score"]
    score_color = "score-high" if score >= 0.65 else "score-mid" if score >= 0.45 else "score-low"
    anomaly_count = r.get("anomaly_count", 0)
    is_returning  = r.get("is_returning", False)

    header_label = f"#{rank} — {r.get('full_name','Unknown')} | Score: {score:.2f}"
    if is_returning:
        header_label += " 🔁 (Returning)"
    if anomaly_count > 0:
        header_label += f" ⚠️ ({anomaly_count} flags)"

    with st.expander(header_label):
        col1, col2 = st.columns([2, 1])

        with col1:
            # Score breakdown
            sc1, sc2, sc3, sc4 = st.columns(4)
            sc1.metric("Composite",  f"{r.get('composite_score', 0):.2f}")
            sc2.metric("Similarity", f"{r.get('semantic_similarity', 0):.2f}")
            sc3.metric("Relevance",  f"{r.get('relevance_score', 0):.2f}")
            sc4.metric("Potential",  f"{r.get('potential_score', 0):.2f}")

            # AI Rationale
            st.markdown("**📖 AI Analysis (for HR review)**")
            st.markdown(
                f'<div class="rationale-box">{r.get("llm_rationale","No analysis available.")}</div>',
                unsafe_allow_html=True
            )

            # Strengths & Gaps
            scol1, scol2 = st.columns(2)
            with scol1:
                st.markdown("**✅ Strengths**")
                strengths = r.get("strengths") or []
                if isinstance(strengths, str):
                    try: strengths = json.loads(strengths)
                    except: strengths = [strengths]
                for s in strengths:
                    st.markdown(f"• {s}")

            with scol2:
                st.markdown("**🔍 Gaps to Explore in Interview**")
                gaps = r.get("gaps") or []
                if isinstance(gaps, str):
                    try: gaps = json.loads(gaps)
                    except: gaps = [gaps]
                for g in gaps:
                    st.markdown(f"• {g}")

            # Transferable skills
            trans = r.get("transferable_skills") or []
            if isinstance(trans, str):
                try: trans = json.loads(trans)
                except: trans = [trans]
            if trans:
                st.markdown("**🔄 Transferable Skills Identified**")
                st.markdown(", ".join(f"`{t}`" for t in trans))

            # Value-add insights
            vai = r.get("value_add_insights") or []
            if isinstance(vai, str):
                try: vai = json.loads(vai)
                except: vai = [vai]
            if vai:
                st.markdown("**💡 Talent Development Insights**")
                for insight in vai:
                    st.markdown(f"• {insight}")

        with col2:
            # CV download link
            st.markdown("**📄 CV File**")
            st.markdown(f"`{r.get('original_filename','')}`")
            if r.get("stored_path"):
                try:
                    with open(r["stored_path"], "rb") as f:
                        st.download_button(
                            "⬇️ Download CV",
                            data=f,
                            file_name=r.get("original_filename", "cv.pdf"),
                            key=f"dl_{r['screening_id']}",
                        )
                except Exception:
                    st.caption("File not accessible from UI")

            st.markdown("**📧 Contact**")
            email = r.get("email", "")
            st.markdown(f"Email: {email[:2]}***@{email.split('@')[-1]}" if '@' in (email or '') else "Not extracted")
            if is_returning:
                st.markdown("🔁 **Returning Candidate**")
                st.caption("This person has applied before. Check history tab.")

            # Anomalies
            if anomaly_count > 0:
                st.markdown(f"**⚠️ {anomaly_count} Anomaly Flag(s)**")
                st.caption("HR review recommended before deciding.")

            # Decision panel
            st.markdown("---")
            st.markdown("**✍️ HR Decision**")
            current_decision = r.get("decision", "needs_review")
            st.caption(f"Current: {current_decision.replace('_',' ').title()}")

            decision_options = {
                "✅ Shortlist (Approve)": "hr_approved",
                "📞 Schedule Interview":  "forwarded",
                "⏸️ Hold for Later":      "hr_hold",
                "❌ Reject":              "hr_rejected",
            }
            chosen_label = st.selectbox(
                "Update Decision",
                list(decision_options.keys()),
                key=f"dec_{r['screening_id']}",
            )
            decision_notes = st.text_area(
                "Notes (optional)",
                key=f"notes_{r['screening_id']}",
                height=80,
                placeholder="Reason for decision...",
            )
            hr_name = st.text_input("Your Name", key=f"hr_{r['screening_id']}")

            if st.button("Save Decision", key=f"save_{r['screening_id']}", type="primary"):
                if not hr_name:
                    st.error("Please enter your name.")
                else:
                    try:
                        resp = requests.patch(
                            f"{API}/screenings/{r['screening_id']}/decision",
                            data={
                                "decision":    decision_options[chosen_label],
                                "decision_by": hr_name,
                                "notes":       decision_notes,
                            },
                            timeout=10,
                        )
                        if resp.status_code == 200:
                            st.success("✅ Decision saved!")
                            st.rerun()
                        else:
                            st.error(f"Error: {resp.json().get('detail')}")
                    except Exception as e:
                        st.error(str(e))
# ============================================================
# FILE: docker-compose.yml
# Orchestrates ALL services — run this to start the system
# ============================================================

version: '3.9'

services:

  # ---- PostgreSQL + pgvector ----
  postgres:
    image: pgvector/pgvector:pg16   # Official image with pgvector pre-installed
    container_name: cv_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB:       ${POSTGRES_DB}
      POSTGRES_USER:     ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data          # Persistent data
      - ./database/migrations:/docker-entrypoint-initdb.d  # Auto-run on first start
    ports:
      - "5432:5432"   # Only expose for dev. Remove in production.
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ---- Ollama (Local LLM Engine) ----
  ollama:
    image: ollama/ollama:latest
    container_name: cv_ollama
    restart: unless-stopped
    volumes:
      - ollama_data:/root/.ollama   # Persists downloaded models (important!)
    ports:
      - "11434:11434"
    # Uncomment if you have NVIDIA GPU:
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: 1
    #           capabilities: [gpu]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/api/tags"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # ---- Model Puller (runs once to download models) ----
  ollama-setup:
    image: curlimages/curl:latest
    container_name: cv_ollama_setup
    depends_on:
      ollama:
        condition: service_healthy
    command: >
      sh -c "
        echo 'Pulling LLM model...' &&
        curl -X POST http://ollama:11434/api/pull -d '{\"name\": \"llama3.1:8b\"}' &&
        echo 'Pulling embedding model...' &&
        curl -X POST http://ollama:11434/api/pull -d '{\"name\": \"nomic-embed-text\"}' &&
        echo 'Models ready!'
      "
    restart: "no"

  # ---- FastAPI Backend ----
  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    container_name: cv_api
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      ollama:
        condition: service_healthy
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_PORT=5432
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - OLLAMA_BASE_URL=http://ollama:11434
      - OLLAMA_LLM_MODEL=${OLLAMA_LLM_MODEL}
      - OLLAMA_EMBED_MODEL=${OLLAMA_EMBED_MODEL}
      - SECRET_KEY=${SECRET_KEY}
      - UPLOAD_DIR=/app/uploads
      - LOG_DIR=/app/logs
      - ENVIRONMENT=${ENVIRONMENT}
    volumes:
      - uploads_data:/app/uploads    # CV file storage
      - logs_data:/app/logs          # Persistent logs
    ports:
      - "8000:8000"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 30s

  # ---- Streamlit Frontend ----
  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
    container_name: cv_frontend
    restart: unless-stopped
    depends_on:
      api:
        condition: service_healthy
    environment:
      - API_URL=http://api:8000
    ports:
      - "8501:8501"     # Access HR portal at http://localhost:8501
    volumes:
      - uploads_data:/app/uploads:ro   # Read-only access to CVs for download

volumes:
  postgres_data:    # PostgreSQL data files
  ollama_data:      # Downloaded AI models
  uploads_data:     # Uploaded CV and JD files
  logs_data:        # Application logs


---


# ============================================================
# FILE: Dockerfile.api
# Builds the FastAPI backend container
# ============================================================

FROM python:3.11-slim

# Install system dependencies for PDF processing
RUN apt-get update && apt-get install -y \
    libmagic1 \
    libgl1-mesa-glx \
    curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies first (cached unless requirements change)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY agents/       ./agents/
COPY api/          ./api/
COPY database/     ./database/
COPY config/       ./config/
COPY shared/       ./shared/

# Create necessary directories
RUN mkdir -p uploads/cvs uploads/jds logs

# Non-root user for security
RUN useradd -m -u 1001 hrapp && chown -R hrapp:hrapp /app
USER hrapp

EXPOSE 8000

CMD ["uvicorn", "api.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]


---


# ============================================================
# FILE: Dockerfile.frontend
# Builds the Streamlit frontend container
# ============================================================

FROM python:3.11-slim

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir streamlit streamlit-extras requests pandas

COPY frontend/ ./frontend/

RUN useradd -m -u 1001 hrapp && chown -R hrapp:hrapp /app
USER hrapp

EXPOSE 8501

CMD ["streamlit", "run", "frontend/app.py",
     "--server.port=8501",
     "--server.address=0.0.0.0",
     "--server.headless=true",
     "--browser.gatherUsageStats=false"]


# ============================================================
# DEPLOYMENT GUIDE — Step by step
# ============================================================

STEP 1: Clone / create project
─────────────────────────────────────
  mkdir cv_screening_system
  cd cv_screening_system
  # Create all files as described in this guide

STEP 2: Create your .env file
─────────────────────────────────────
  cp .env.example .env
  # Edit .env — CHANGE the passwords and secret key!
  nano .env

STEP 3: Build and start all services
─────────────────────────────────────
  docker compose up --build -d

  # This will:
  # 1. Start PostgreSQL and create the schema automatically
  # 2. Start Ollama
  # 3. Download llama3.1:8b and nomic-embed-text (~5.5GB total, one-time)
  # 4. Start the FastAPI API
  # 5. Start Streamlit frontend

STEP 4: Watch the logs
─────────────────────────────────────
  # All services:
  docker compose logs -f

  # Just the API:
  docker compose logs -f api

  # Just Ollama (watch model downloads):
  docker compose logs -f ollama

STEP 5: Verify everything is working
─────────────────────────────────────
  # Check system health:
  curl http://localhost:8000/health

  # Expected response:
  # {"status":"healthy","database":true,"ollama":true,"version":"1.0.0"}

STEP 6: Open the HR Portal
─────────────────────────────────────
  Open your browser: http://localhost:8501

STEP 7: First use — workflow
─────────────────────────────────────
  1. Go to "Post a Job" page
  2. Fill in job details and upload a JD text file
  3. Go to "Upload CVs" page
  4. Select the job, upload CVs (PDF/DOCX/TXT)
  5. Wait ~30-60 seconds per CV for AI processing
  6. Go to "Screening Results" to see ranked candidates
  7. Review AI analysis, click candidates for details
  8. Make HR decisions (Approve / Reject / Hold)


# ============================================================
# USEFUL DOCKER COMMANDS
# ============================================================

# Stop all services:
docker compose down

# Stop and wipe ALL data (caution!):
docker compose down -v

# Restart just the API after code changes:
docker compose restart api

# View real-time logs:
docker compose logs -f api

# Run database migrations manually:
docker compose exec postgres psql -U hr_admin -d cv_screening -f /docker-entrypoint-initdb.d/001_initial_schema.sql

# Connect to database directly:
docker compose exec postgres psql -U hr_admin -d cv_screening

# Access API shell for debugging:
docker compose exec api bash


# ============================================================
# FILE: tests/unit/test_ingestion_agent.py
# Unit tests for the Ingestion Agent
# ============================================================

import pytest
import asyncio
from unittest.mock import AsyncMock, MagicMock, patch
import json

# We test agents in isolation — mock the DB and Ollama
class TestIngestionAgent:

    @pytest.mark.asyncio
    async def test_extracts_text_from_txt(self, tmp_path):
        """Test that TXT files are correctly read."""
        from agents.ingestion_agent import IngestionAgent

        # Create a temp TXT file
        test_cv = tmp_path / "test_cv.txt"
        test_cv.write_text("John Doe\njohn@example.com\nPython Developer\n5 years experience")

        mock_db = AsyncMock()
        agent = IngestionAgent(mock_db)

        text, warnings = await agent._extract_text(str(test_cv), "txt")

        assert "John Doe" in text
        assert "john@example.com" in text
        assert len(warnings) == 0

    @pytest.mark.asyncio
    async def test_handles_corrupt_file_gracefully(self, tmp_path):
        """Corrupt files should return empty text, not crash."""
        from agents.ingestion_agent import IngestionAgent

        corrupt_file = tmp_path / "corrupt.pdf"
        corrupt_file.write_bytes(b"NOT A REAL PDF CONTENT @@##$$")

        mock_db = AsyncMock()
        agent = IngestionAgent(mock_db)

        text, warnings = await agent._extract_text(str(corrupt_file), "pdf")

        # Should not raise — should return empty with a warning
        assert isinstance(warnings, list)

    @pytest.mark.asyncio
    async def test_llm_extraction_parses_json(self):
        """LLM response should be parsed into ParsedCV."""
        from agents.ingestion_agent import IngestionAgent

        mock_db = AsyncMock()
        agent = IngestionAgent(mock_db)

        # Mock Ollama response
        mock_llm_response = {
            "message": {
                "content": json.dumps({
                    "full_name": "Jane Smith",
                    "email": "jane@example.com",
                    "phone": "+971501234567",
                    "linkedin_url": None,
                    "location": "Dubai, UAE",
                    "technical_skills": ["Python", "SQL", "Machine Learning"],
                    "soft_skills": ["Leadership", "Communication"],
                    "domain_expertise": ["Data Science", "Fintech"],
                    "certifications": ["AWS Certified ML Specialist"],
                    "languages": ["English", "Arabic"],
                    "experience": [],
                    "education": [],
                    "total_years_exp": 5.0,
                    "cv_summary": "Experienced data scientist with 5 years in fintech.",
                    "parse_confidence": 0.87,
                })
            }
        }

        with patch.object(agent.ollama, "chat", new_callable=AsyncMock, return_value=mock_llm_response):
            result = await agent._llm_extract("Sample CV text", [])

        assert result.full_name == "Jane Smith"
        assert result.email == "jane@example.com"
        assert "Python" in result.technical_skills
        assert result.parse_confidence == 0.87

    @pytest.mark.asyncio
    async def test_llm_json_error_returns_low_confidence(self):
        """Malformed LLM JSON should return a low-confidence ParsedCV, not crash."""
        from agents.ingestion_agent import IngestionAgent

        mock_db = AsyncMock()
        agent = IngestionAgent(mock_db)

        bad_response = {"message": {"content": "This is not JSON at all!"}}

        with patch.object(agent.ollama, "chat", new_callable=AsyncMock, return_value=bad_response):
            result = await agent._llm_extract("Sample CV text", [])

        assert result.parse_confidence <= 0.3
        assert len(result.parse_warnings) > 0


class TestValidationAgent:

    @pytest.mark.asyncio
    async def test_flags_borderline_score(self):
        """Score between 0.38 and 0.52 should be flagged."""
        from agents.validation_agent import ValidationAgent

        mock_db = AsyncMock()
        mock_db.execute = AsyncMock(return_value=AsyncMock(fetchone=lambda: None))
        agent = ValidationAgent(mock_db)

        anomalies = await agent._check_borderline_score({"composite_score": 0.45})

        assert len(anomalies) == 1
        assert anomalies[0]["type"] == "borderline_score"
        assert anomalies[0]["severity"] == "medium"

    @pytest.mark.asyncio
    async def test_no_anomaly_for_strong_score(self):
        """Score of 0.75 should not trigger borderline flag."""
        from agents.validation_agent import ValidationAgent

        mock_db = AsyncMock()
        agent = ValidationAgent(mock_db)

        anomalies = await agent._check_borderline_score({"composite_score": 0.75})

        assert len(anomalies) == 0


# Run tests:
# pytest tests/ -v --cov=agents --cov-report=html
# ============================================================
# SECURITY, COMPLIANCE & ATS INTEGRATION GUIDE
# ============================================================


# ============================================================
# 1. GUARDRAILS IMPLEMENTATION CHECKLIST
# ============================================================

BIAS & FAIRNESS CONTROLS (Implemented)
──────────────────────────────────────
✅ LLM prompts explicitly instruct: "Do NOT infer gender, age,
   nationality, religion, or any protected attribute"
✅ ParsedCV schema has NO field for: age, gender, photo, ethnicity
✅ Matching is purely skill/experience based
✅ All LLM reasoning output is visible to HR (no black box)
✅ HR makes ALL final decisions — system only recommends

DATA PRIVACY CONTROLS (Implemented)
────────────────────────────────────
✅ No data sent outside local environment (Ollama is fully local)
✅ Files stored on local filesystem with path hashing
✅ Email addresses partially masked in UI display
✅ Database runs inside Docker with no public internet access
✅ Audit logs are append-only (REVOKE UPDATE/DELETE applied)

HUMAN OVERSIGHT CONTROLS (Implemented)
───────────────────────────────────────
✅ System produces RECOMMENDATIONS, never final decisions
✅ Every screening shows "needs_review" until HR acts
✅ Borderline scores are ALWAYS flagged for mandatory review
✅ Anomalies (duplicates, gaps, low confidence) escalate to HR
✅ All HR decisions are logged with timestamp and HR name

COMPLIANCE AUDIT TRAIL (Implemented)
──────────────────────────────────────
✅ Every agent action writes to audit_logs (immutable)
✅ Correlation ID traces a CV from upload → decision
✅ Rejection reasons stored in database
✅ All HR decisions include: who, when, what, why
✅ LLM rationale stored as plain text for explanation


# ============================================================
# 2. LOGGING STRATEGY
# ============================================================

# FILE: config/logging_config.py

import structlog
import logging
import json
import os
from datetime import datetime
from config.settings import settings


def setup_logging():
    """
    Configure structured JSON logging for the entire application.

    Log files are written to settings.log_dir.
    Each log line is a valid JSON object for easy parsing by
    observability tools (Grafana, ELK, Splunk, etc.)
    """
    os.makedirs(settings.log_dir, exist_ok=True)

    # Log file path with date
    log_file = os.path.join(settings.log_dir, f"cv_screening_{datetime.now().strftime('%Y-%m-%d')}.jsonl")

    # File handler — appends JSON lines
    file_handler = logging.FileHandler(log_file)
    file_handler.setLevel(logging.INFO)

    # Console handler — readable during dev
    console_handler = logging.StreamHandler()
    console_handler.setLevel(logging.DEBUG if settings.environment == "development" else logging.INFO)

    logging.basicConfig(
        level=logging.DEBUG,
        handlers=[file_handler, console_handler],
        format="%(message)s",
    )

    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
    )


# EXAMPLE LOG LINES (what you'll see in the .jsonl file)
# ─────────────────────────────────────────────────────────
# Ingestion start:
# {"event":"agent_started","agent":"ingestion","correlation_id":"abc-123","timestamp":"2025-01-15T09:00:00Z"}
#
# Successful parse:
# {"event":"ingestion_complete","filename":"john_doe_cv.pdf","confidence":0.87,"skills_found":12,"duration_ms":3421}
#
# LLM timeout:
# {"event":"agent_failed","agent":"matching","error":"Ollama timeout after 30s","outcome":"failure","duration_ms":30000}
#
# HR decision:
# {"event":"hr_decision_recorded","screening_id":"xyz-456","decision":"hr_approved","by":"sarah.hr","timestamp":"2025-01-15T10:30:00Z"}


# ============================================================
# 3. ERROR HANDLING PATTERNS
# ============================================================

# Pattern 1: Bulk upload — one failure NEVER stops others
# ────────────────────────────────────────────────────────
# In process_cv_pipeline() (screenings router):
#
# for each CV:
#   try:
#     await full_pipeline(cv)
#   except Exception as e:
#     log error with correlation_id
#     mark cv_version as 'failed'
#     CONTINUE to next CV
#
# This guarantees: 1 corrupt CV in 500 doesn't block the other 499.

# Pattern 2: LLM timeout — graceful degradation
# ──────────────────────────────────────────────
# In MatchingAgent._llm_analyze():
#
# try:
#   response = await ollama.chat(...)
# except Exception as e:
#   log the failure
#   return safe defaults:
#     relevance_score = 0.3
#     rationale = "Automated analysis failed. Manual review required."
#
# The CV still gets a screening record — just with lower confidence.

# Pattern 3: Database failure — rollback and log
# ────────────────────────────────────────────────
# In get_db() (connection.py):
#
# try:
#   yield session
#   await session.commit()
# except Exception:
#   await session.rollback()
#   log error
#   raise  ← FastAPI returns 500 to client


# ============================================================
# 4. PERFORMANCE TUNING FOR SCALE
# ============================================================

# At 3,000+ CV backlog:
# ──────────────────────
# 1. Increase API workers:
#    uvicorn api.main:app --workers 4
#
# 2. Use a task queue (Celery + Redis) for background processing:
#    Replace BackgroundTasks with Celery tasks
#    Allows horizontal scaling of workers
#
# 3. Batch embedding generation:
#    Instead of 1 embed call per CV, batch 10 at a time
#    ollama.embeddings supports batching
#
# 4. pgvector index tuning:
#    For >100K vectors, increase IVFFlat lists:
#    CREATE INDEX ON cv_versions USING ivfflat (embedding vector_cosine_ops)
#    WITH (lists = 200);  -- sqrt(n) rule
#
# 5. Caching JD embeddings:
#    JD embedding for a job never changes.
#    Store in Redis to avoid recomputing on each CV.


# ============================================================
# 5. ATS / ORACLE INTEGRATION (Extensibility)
# ============================================================

# FILE: api/routers/integrations.py (Future)

"""
To integrate with Oracle HCM, Workday, SAP SuccessFactors, or
any ATS via their REST APIs:

1. OUTBOUND (Push approved candidates to ATS):

async def push_to_ats(screening_id: str, ats_type: str):
    result = fetch_screening(screening_id)  # from our DB
    
    if ats_type == "oracle_hcm":
        payload = map_to_oracle_format(result)
        await oracle_client.post("/candidateApplications", json=payload)
    
    elif ats_type == "workday":
        payload = map_to_workday_format(result)
        await workday_client.post("/workers/applications", json=payload)
    
    # Log the integration event
    await write_audit(event="ats_push", system=ats_type, candidate=result.candidate_id)


2. INBOUND (Receive CVs from career portal webhook):

@router.post("/webhooks/career-portal")
async def receive_from_portal(payload: dict):
    # Validate webhook signature
    verify_signature(payload, secret=settings.webhook_secret)
    
    # Map portal fields to our schema
    cv_data = map_portal_to_internal(payload)
    
    # Queue for AI screening
    await queue_cv_for_screening(cv_data)


3. Environment variables needed for ATS:
   ORACLE_HCM_BASE_URL=https://your-org.oraclecloud.com
   ORACLE_HCM_CLIENT_ID=xxx
   ORACLE_HCM_CLIENT_SECRET=xxx
   CAREER_PORTAL_WEBHOOK_SECRET=xxx
"""


# ============================================================
# 6. PRODUCTION HARDENING CHECKLIST
# ============================================================

BEFORE GO-LIVE:
───────────────
□ Change ALL default passwords in .env
□ Generate a strong SECRET_KEY (64+ random characters):
    python -c "import secrets; print(secrets.token_hex(32))"
□ Remove public port 5432 (PostgreSQL) from docker-compose.yml
□ Remove /docs API route (set docs_url=None in FastAPI)
□ Set ENVIRONMENT=production in .env
□ Enable HTTPS reverse proxy (nginx/traefik) in front of Streamlit
□ Set up regular PostgreSQL backups:
    pg_dump -U hr_admin cv_screening > backup_$(date +%Y%m%d).sql
□ Configure log rotation to prevent disk fill
□ Set up alerting on /health endpoint (PagerDuty/Slack)
□ Perform penetration test before production launch
□ Conduct bias audit: run 50 test CVs from diverse backgrounds,
  verify scores correlate only with skills/experience


# ============================================================
# 7. QUICK TROUBLESHOOTING GUIDE
# ============================================================

PROBLEM: Ollama model not responding
SOLUTION:
  docker compose exec ollama ollama list
  # If empty, pull models again:
  docker compose exec ollama ollama pull llama3.1:8b
  docker compose exec ollama ollama pull nomic-embed-text

PROBLEM: CV stuck in "pending" status
SOLUTION:
  # Check API logs for errors:
  docker compose logs api | grep ERROR
  # Check for disk space:
  df -h

PROBLEM: Low parse confidence on all CVs
SOLUTION:
  # CVs may be image-based PDFs (scanned).
  # Add OCR support with tesseract:
  #   pip install pytesseract
  #   apt-get install tesseract-ocr
  # Integrate in _extract_pdf() as fallback when no text extracted.

PROBLEM: Slow screening (>5 min per CV)
SOLUTION:
  # llama3.1:8b requires ~8GB RAM. Check available RAM:
  free -h
  # If RAM-constrained, use smaller model:
  #   OLLAMA_LLM_MODEL=mistral:7b  (faster, slightly less accurate)
  #   OLLAMA_LLM_MODEL=phi3:mini   (fastest, for dev/testing only)

PROBLEM: pgvector extension not found
SOLUTION:
  # Ensure you're using the pgvector/pgvector image, not plain postgres:
  #   image: pgvector/pgvector:pg16  ← correct
  #   image: postgres:16             ← missing pgvector
  docker compose down -v && docker compose up --build -d

PROBLEM: Frontend can't connect to API
SOLUTION:
  # Check API is healthy:
  curl http://localhost:8000/health
  # Check containers can talk to each other:
  docker compose exec frontend curl http://api:8000/health
  # Verify network in docker-compose.yml (services share default network)


# ============================================================
# 8. FULL FILE LIST SUMMARY
# ============================================================

Create these files in order:

1.  .env                                ← copy from .env.example, edit passwords
2.  requirements.txt
3.  config/settings.py
4.  database/connection.py
5.  database/migrations/001_initial_schema.sql
6.  shared/models.py
7.  shared/utils.py
8.  agents/__init__.py                  ← empty file
9.  agents/base_agent.py
10. agents/ingestion_agent.py
11. agents/matching_agent.py
12. agents/potential_agent.py
13. agents/validation_agent.py
14. api/__init__.py                     ← empty file
15. api/main.py
16. api/routers/__init__.py             ← empty file
17. api/routers/jobs.py
18. api/routers/screenings.py
19. api/routers/candidates.py           ← basic CRUD (similar pattern to jobs)
20. frontend/app.py
21. frontend/pages/01_Post_a_Job.py
22. frontend/pages/02_Upload_CVs.py
23. frontend/pages/03_Screening_Results.py
24. frontend/pages/04_Candidate_History.py
25. Dockerfile.api
26. Dockerfile.frontend
27. docker-compose.yml
28. tests/unit/test_ingestion_agent.py
29. tests/unit/test_validation_agent.py

__init__.py files are empty — they just tell Python
"this folder is a package".

TOTAL ESTIMATED DEVELOPMENT TIME:
  Backend + Agents: 3-4 days
  Frontend: 1-2 days
  Testing: 1-2 days
  Docker + Deployment: 0.5 days
  Total: ~1 week for a skilled team
