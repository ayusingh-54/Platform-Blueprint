# Pulsepoint Complete Development Guide - All Steps Detailed

## Executive Summary


This guide walks through building Pulsepoint from scratch, step-by-step, with complete code and configuration for every phase. It includes all 7 phases with decision points and validation steps.

---

## Table of Contents

- [Phase 1: Local Environment Setup](#phase-1-local-environment-setup)
- [Phase 2: Database & Core Backend](#phase-2-database--core-backend)
- [Phase 3: API Development](#phase-3-api-development)
- [Phase 4: Data Integration Services](#phase-4-data-integration-services)
- [Phase 5: AI Integration & Intelligence](#phase-5-ai-integration--intelligence)
- [Phase 6: Multi-Tenant Architecture](#phase-6-multi-tenant-architecture)
- [Phase 7: n8n Workflow Automation](#phase-7-n8n-workflow-automation)
- [Phase 8: Observability & Monitoring](#phase-8-observability--monitoring)
- [Phase 9: Testing](#phase-9-testing)
- [Phase 10: Deployment & Production](#phase-10-deployment--production)

---

## Phase 1: Local Environment Setup

### 1.1 Prerequisites Check

Verify you have:

- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- Docker & Docker Compose
- Git
- Node.js 18+ (for frontend)

### 1.2 Initial Project Creation

```bash
# Create project directory
mkdir ~/projects/pulsepoint
cd ~/projects/pulsepoint

# Initialize Git
git init
git config user.name "Your Name"
git config user.email "your@email.com"

# Create main branch
git checkout -b main

# Create .gitignore
cat > .gitignore << 'EOF'
venv/
__pycache__/
*.pyc
.env
.env.local
.DS_Store
node_modules/
dist/
build/
*.egg-info/
.pytest_cache/
.coverage
htmlcov/
.vscode/
.idea/
*.log
.n8n/
EOF

git add .gitignore
git commit -m "Initial commit: Add gitignore"
```

### 1.3 Python Virtual Environment

```bash
# Create virtual environment
python -m venv venv

# Activate
source venv/bin/activate  # macOS/Linux
# OR
venv\Scripts\activate  # Windows

# Verify Python
python --version  # Should be 3.11+

# Upgrade pip
pip install --upgrade pip setuptools wheel
```

### 1.4 Dependencies Installation

Create `requirements.txt`:

```
# Web Framework
fastapi==0.104.1
uvicorn[standard]==0.24.0
starlette==0.27.0

# Database
sqlalchemy==2.0.23
asyncpg==0.29.0
alembic==1.12.1
psycopg2-binary==2.9.9

# Data Validation
pydantic==2.5.0
pydantic-settings==2.1.0

# Authentication
python-jose==3.3.0
python-multipart==0.0.6
bcrypt==4.1.1
PyJWT==2.8.1

# Environment
python-dotenv==1.0.0

# Async & Concurrency
aioredis==2.0.1
asyncio==3.4.3
httpx==0.25.0
requests==2.31.0

# Google Cloud
google-cloud-bigquery==3.13.1
google-cloud-pubsub==2.18.4
google-cloud-storage==2.10.0
google-cloud-logging==3.8.0

# AI & LLM
anthropic==0.7.0

# Security
cryptography==41.0.7

# Utilities
python-dateutil==2.8.2
pytz==2023.3

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-cov==4.1.0
httpx==0.25.0

# Development
black==23.11.0
ruff==0.1.6
mypy==1.7.1
isort==5.12.0
pre-commit==3.5.0

# Logging & Monitoring
python-json-logger==2.0.7

# Email
python-multipart==0.0.6
```

Install:

```bash
pip install -r requirements.txt
```

### 1.5 Environment Configuration

Create `.env.example`:

```bash
cat > .env.example << 'EOF'
# ===== DATABASE =====
DATABASE_URL=postgresql+asyncpg://pulsepoint:password123@localhost:5432/pulsepoint
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=pulsepoint
DATABASE_PASSWORD=password123
DATABASE_NAME=pulsepoint

# ===== REDIS =====
REDIS_URL=redis://localhost:6379/0
REDIS_HOST=localhost
REDIS_PORT=6379

# ===== GOOGLE CLOUD =====
GCP_PROJECT_ID=pulsepoint-prod
GOOGLE_APPLICATION_CREDENTIALS=/path/to/gcp-credentials.json

# ===== API KEYS =====
SHOPIFY_API_KEY=your-shopify-api-key
SHOPIFY_API_SECRET=your-shopify-api-secret
SHOPIFY_SHOP_NAME=your-store.myshopify.com

META_ACCESS_TOKEN=your-meta-business-token
META_BUSINESS_ACCOUNT_ID=123456789

TIKTOK_ACCESS_TOKEN=your-tiktok-token
TIKTOK_BUSINESS_ACCOUNT_ID=your-account-id

GOOGLE_ADS_DEVELOPER_TOKEN=your-google-token
GOOGLE_ADS_CLIENT_ID=your-client-id
GOOGLE_ADS_CLIENT_SECRET=your-client-secret

ANTHROPIC_API_KEY=sk-ant-your-claude-key

# ===== JWT CONFIGURATION =====
JWT_SECRET=your-super-secret-key-change-in-production
JWT_ALGORITHM=HS256
JWT_EXPIRY_HOURS=24

# ===== SERVER CONFIG =====
SERVER_HOST=0.0.0.0
SERVER_PORT=8000
ENVIRONMENT=development
LOG_LEVEL=INFO

# ===== CORS =====
CORS_ORIGINS=http://localhost:3000,http://localhost:5678

# ===== n8n =====
N8N_URL=http://localhost:5678
N8N_WEBHOOK_URL=http://localhost:5678/webhook
N8N_API_KEY=your-n8n-api-key

# ===== EMAIL =====
SMTP_SERVER=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password

# ===== ALERTS =====
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# ===== RATE LIMITING =====
RATE_LIMIT_PER_MINUTE=60
RATE_LIMIT_PER_HOUR=1000
EOF

cp .env.example .env
```

Edit `.env` with your actual credentials.

### 1.6 Folder Structure Creation

```bash
# Create all directories
mkdir -p backend/{api,services,integrations,models,database,utils,jobs,middleware}
mkdir -p backend/api/{routes,middleware,schemas}
mkdir -p backend/database/{migrations,seeds}
mkdir -p backend/tests/{unit,integration,e2e}
mkdir -p frontend/src/{components,pages,services,styles,utils}
mkdir -p frontend/public
mkdir -p scripts
mkdir -p docs
mkdir -p n8n-workflows
mkdir -p deployment/{docker,k8s,terraform}

# Create __init__.py files
touch backend/__init__.py
touch backend/api/__init__.py
touch backend/services/__init__.py
touch backend/integrations/__init__.py
touch backend/models/__init__.py
touch backend/database/__init__.py
touch backend/utils/__init__.py
touch backend/jobs/__init__.py
touch backend/tests/__init__.py
```

### 1.7 Git First Commit

```bash
git add -A
git commit -m "Phase 1: Initialize project structure and dependencies"
```

---

## Phase 2: Database & Core Backend

### 2.1 PostgreSQL Setup

```bash
# Create PostgreSQL database
createdb pulsepoint

# Create admin user
createuser pulsepoint
psql -d pulsepoint << 'EOF'
ALTER USER pulsepoint WITH PASSWORD 'password123';
GRANT ALL PRIVILEGES ON DATABASE pulsepoint TO pulsepoint;
ALTER USER pulsepoint CREATEDB;
EOF

# Enable extensions
psql -d pulsepoint << 'EOF'
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "btree_gin";
EOF
```

### 2.2 SQLAlchemy Models

Create `backend/models/database.py`:

```python
from sqlalchemy import Column, String, UUID, DateTime, Boolean, Enum, Integer, Float, ForeignKey, Text, JSON
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
from sqlalchemy.orm import declarative_base, relationship
from datetime import datetime
import uuid
import enum

Base = declarative_base()

class TenantStatus(str, enum.Enum):
    ACTIVE = "active"
    SUSPENDED = "suspended"
    DELETED = "deleted"

class UserRole(str, enum.Enum):
    ADMIN = "admin"
    MANAGER = "manager"
    USER = "user"

class Tenant(Base):
    """Multi-tenant organization"""
    __tablename__ = "tenants"

    tenant_id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(255), nullable=False, unique=True)
    status = Column(String(20), default="active")
    subscription_tier = Column(String(50), default="basic")  # basic, pro, enterprise
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    users = relationship("User", back_populates="tenant", cascade="all, delete-orphan")
    brands = relationship("Brand", back_populates="tenant", cascade="all, delete-orphan")

class User(Base):
    """Users within a tenant"""
    __tablename__ = "users"

    user_id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.tenant_id"), nullable=False)
    email = Column(String(255), nullable=False)
    password_hash = Column(String(255), nullable=False)
    full_name = Column(String(255))
    role = Column(String(20), default="user")
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    tenant = relationship("Tenant", back_populates="users")

class Brand(Base):
    """Brands owned by tenants"""
    __tablename__ = "brands"

    brand_id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.tenant_id"), nullable=False)
    name = Column(String(255), nullable=False)
    description = Column(Text)
    shopify_store_url = Column(String(255))
    status = Column(String(20), default="active")
    metadata = Column(JSON, default={})
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Relationships
    tenant = relationship("Tenant", back_populates="brands")

class AITokenUsage(Base):
    """Track AI API token consumption"""
    __tablename__ = "ai_token_usage"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.tenant_id"), nullable=False)
    brand_id = Column(UUID(as_uuid=True), ForeignKey("brands.brand_id"))
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.user_id"))

    model = Column(String(100), default="claude-3-5-sonnet-20241022")
    operation = Column(String(100))  # ai_generation, analysis, recommendation
    input_tokens = Column(Integer, default=0)
    output_tokens = Column(Integer, default=0)
    total_tokens = Column(Integer, default=0)

    input_cost_usd = Column(Float, default=0.0)
    output_cost_usd = Column(Float, default=0.0)
    total_cost_usd = Column(Float, default=0.0)

    recorded_at = Column(DateTime, default=datetime.utcnow)
    created_at = Column(DateTime, default=datetime.utcnow)

class ObservabilityEvent(Base):
    """Event tracking for observability"""
    __tablename__ = "observability_events"

    event_id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    request_id = Column(String(255))
    tenant_id = Column(UUID(as_uuid=True), ForeignKey("tenants.tenant_id"), nullable=False)
    user_id = Column(UUID(as_uuid=True), ForeignKey("users.user_id"))

    category = Column(String(50))  # api.lifecycle, ai.operations, billing, etc.
    event_type = Column(String(100))
    severity = Column(String(20))  # debug, info, warning, error, critical
    status = Column(String(20))  # success, failure, partial

    duration_ms = Column(Float, default=0.0)
    cpu_time_ms = Column(Float, default=0.0)

    tokens_input = Column(Integer, default=0)
    tokens_output = Column(Integer, default=0)
    tokens_used = Column(Integer, default=0)
    cost_usd = Column(Float, default=0.0)

    accuracy_score = Column(Float)  # 0-1
    was_cached = Column(Boolean, default=False)
    fallback_used = Column(String(100))

    error_code = Column(String(50))
    error_message = Column(Text)

    metadata = Column(JSON, default={})
    timestamp = Column(DateTime, default=datetime.utcnow)
    created_at = Column(DateTime, default=datetime.utcnow)

# Database connection setup
async def init_db(database_url: str):
    engine = create_async_engine(
        database_url,
        echo=True,  # Set to False in production
        future=True,
        pool_pre_ping=True
    )

    # Create tables
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

    # Create session factory
    async_session_maker = async_sessionmaker(
        engine,
        class_=AsyncSession,
        expire_on_commit=False
    )

    return engine, async_session_maker
```

### 2.3 Alembic Migration Setup

```bash
# Initialize Alembic
alembic init backend/database/migrations

# Update alembic.ini to point to correct sqlalchemy.url
# Also update backend/database/migrations/env.py to use async
```

Create initial migration:

```bash
alembic revision --autogenerate -m "Initial schema"
alembic upgrade head
```

---

## Phase 3: API Development

### 3.1 FastAPI Main Application

Create `backend/api/main.py`:

```python
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
import logging
import os

# Setup logging
logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"))
logger = logging.getLogger(__name__)

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown events"""
    logger.info("🚀 Starting Pulsepoint MCP Server")
    logger.info(f"Environment: {os.getenv('ENVIRONMENT')}")

    yield

    logger.info("🛑 Shutting down Pulsepoint MCP Server")

# Create FastAPI app
app = FastAPI(
    title="Pulsepoint MCP Server",
    description="AI-powered e-commerce control tower",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    openapi_url="/openapi.json",
    lifespan=lifespan
)

# CORS Configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=os.getenv("CORS_ORIGINS", "http://localhost:3000").split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Health check endpoint
@app.get("/health")
async def health_check():
    return {
        "status": "healthy",
        "service": "pulsepoint-mcp",
        "version": "1.0.0"
    }

@app.get("/")
async def root():
    return {
        "name": "Pulsepoint MCP Server",
        "description": "AI-powered e-commerce control tower",
        "docs": "/docs"
    }

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        app,
        host=os.getenv("SERVER_HOST", "0.0.0.0"),
        port=int(os.getenv("SERVER_PORT", 8000)),
        reload=os.getenv("ENVIRONMENT") == "development"
    )
```

### 3.2 Run FastAPI Server

```bash
# Install uvicorn if not already
pip install uvicorn

# Run development server
python -m uvicorn backend.api.main:app --reload

# Access http://localhost:8000/docs for Swagger UI
```

---

## Phase 4: Data Integration Services

### 4.1 Shopify Integration

Create `backend/integrations/shopify_service.py`:

```python
import httpx
import logging
from typing import List, Dict

logger = logging.getLogger(__name__)

class ShopifyService:
    def __init__(self, api_key: str, api_secret: str, shop_name: str):
        self.api_key = api_key
        self.api_secret = api_secret
        self.shop_name = shop_name
        self.base_url = f"https://{shop_name}.myshopify.com/admin/api/2024-01"

    async def get_products(self, limit: int = 100) -> List[Dict]:
        """Fetch products from Shopify"""
        url = f"{self.base_url}/products.json"
        headers = {"X-Shopify-Access-Token": self.api_key}
        params = {"limit": limit}

        try:
            async with httpx.AsyncClient(timeout=30.0) as client:
                response = await client.get(url, headers=headers, params=params)
                response.raise_for_status()
                data = response.json()
                logger.info(f"Fetched {len(data['products'])} products from Shopify")
                return data["products"]
        except httpx.HTTPError as e:
            logger.error(f"Shopify API error: {e}")
            raise

    async def get_orders(self, status: str = "any", limit: int = 50) -> List[Dict]:
        """Fetch orders from Shopify"""
        url = f"{self.base_url}/orders.json"
        headers = {"X-Shopify-Access-Token": self.api_key}
        params = {"status": status, "limit": limit}

        try:
            async with httpx.AsyncClient(timeout=30.0) as client:
                response = await client.get(url, headers=headers, params=params)
                response.raise_for_status()
                data = response.json()
                logger.info(f"Fetched {len(data['orders'])} orders from Shopify")
                return data["orders"]
        except httpx.HTTPError as e:
            logger.error(f"Shopify API error: {e}")
            raise
```

---

## Phase 5: AI Integration & Intelligence

### 5.1 Claude Integration

Create `backend/integrations/anthropic_service.py`:

```python
import json
from anthropic import Anthropic
import logging

logger = logging.getLogger(__name__)

class AnthropicService:
    def __init__(self, api_key: str):
        self.client = Anthropic(api_key=api_key)
        self.model = "claude-3-5-sonnet-20241022"

    async def analyze_metrics(self, data: dict, question: str) -> dict:
        """Analyze metrics using Claude"""

        context = f"""
        You are an expert e-commerce analyst. Analyze the following data and answer the question.

        **Data:**
        {json.dumps(data, indent=2, default=str)}

        **Question:** {question}

        Provide:
        1. Key findings
        2. Actionable insights
        3. Recommendations

        Format as JSON with keys: findings, insights, recommendations
        """

        message = self.client.messages.create(
            model=self.model,
            max_tokens=1024,
            messages=[
                {"role": "user", "content": context}
            ]
        )

        response_text = message.content[0].text

        try:
            # Try to parse JSON response
            result = json.loads(response_text)
        except json.JSONDecodeError:
            # Fallback to plain text response
            result = {"response": response_text}

        # Track token usage
        usage = {
            "input_tokens": message.usage.input_tokens,
            "output_tokens": message.usage.output_tokens,
            "total_tokens": message.usage.input_tokens + message.usage.output_tokens
        }

        return {
            "analysis": result,
            "tokens": usage
        }
```

---

## Phase 6: Multi-Tenant Architecture

### 6.1 Tenant Context Middleware

Create `backend/api/middleware/tenant_context.py`:

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import JSONResponse
import jwt
from contextvars import ContextVar
import logging
import os

logger = logging.getLogger(__name__)

# Thread-safe context variables
current_tenant_id = ContextVar('tenant_id', default=None)
current_user_id = ContextVar('user_id', default=None)
current_user_role = ContextVar('user_role', default=None)

class TenantContextMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Skip middleware for health checks and docs
        if request.url.path in ["/health", "/docs", "/openapi.json", "/redoc"]:
            return await call_next(request)

        # Extract Authorization header
        auth_header = request.headers.get("Authorization")
        if not auth_header:
            return JSONResponse(
                status_code=401,
                content={"error": "Missing authorization header"}
            )

        try:
            # Extract token
            token = auth_header.replace("Bearer ", "")

            # Decode JWT
            payload = jwt.decode(
                token,
                os.getenv("JWT_SECRET"),
                algorithms=[os.getenv("JWT_ALGORITHM", "HS256")]
            )

            tenant_id = payload.get("tenant_id")
            user_id = payload.get("sub")
            user_role = payload.get("role", "user")

            if not tenant_id:
                return JSONResponse(
                    status_code=403,
                    content={"error": "Tenant not found in token"}
                )

            # Set context variables
            current_tenant_id.set(tenant_id)
            current_user_id.set(user_id)
            current_user_role.set(user_role)

            # Store in request state for database queries
            request.state.tenant_id = tenant_id
            request.state.user_id = user_id
            request.state.user_role = user_role

            logger.info(f"Request from tenant: {tenant_id}, user: {user_id}")

        except jwt.ExpiredSignatureError:
            return JSONResponse(
                status_code=401,
                content={"error": "Token expired"}
            )
        except jwt.InvalidTokenError as e:
            return JSONResponse(
                status_code=401,
                content={"error": f"Invalid token: {str(e)}"}
            )

        # Call next middleware
        response = await call_next(request)
        return response

def get_current_tenant_id() -> str:
    """Get current tenant ID from context"""
    return current_tenant_id.get()

def get_current_user_id() -> str:
    """Get current user ID from context"""
    return current_user_id.get()
```

### 6.2 RLS Policies Setup

Create `scripts/setup_rls.py`:

```python
import asyncio
from sqlalchemy import text
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
import os

async def setup_rls():
    """Setup Row-Level Security policies"""

    database_url = os.getenv("DATABASE_URL")
    engine = create_async_engine(database_url)

    async with engine.begin() as conn:
        logger.info("Setting up RLS policies...")

        # Enable RLS on all tables
        await conn.execute(text("ALTER TABLE tenants ENABLE ROW LEVEL SECURITY"))
        await conn.execute(text("ALTER TABLE users ENABLE ROW LEVEL SECURITY"))
        await conn.execute(text("ALTER TABLE brands ENABLE ROW LEVEL SECURITY"))
        await conn.execute(text("ALTER TABLE ai_token_usage ENABLE ROW LEVEL SECURITY"))
        await conn.execute(text("ALTER TABLE observability_events ENABLE ROW LEVEL SECURITY"))

        # Create policies for brands table
        await conn.execute(text("""
            DROP POLICY IF EXISTS brand_isolation ON brands;
            CREATE POLICY brand_isolation ON brands
            USING (tenant_id = current_setting('app.tenant_id')::uuid)
            WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
        """))

        # Create policies for ai_token_usage
        await conn.execute(text("""
            DROP POLICY IF EXISTS ai_token_isolation ON ai_token_usage;
            CREATE POLICY ai_token_isolation ON ai_token_usage
            USING (tenant_id = current_setting('app.tenant_id')::uuid);
        """))

        await conn.commit()
        logger.info("✓ RLS policies created successfully")

asyncio.run(setup_rls())
```

Run:

```bash
python scripts/setup_rls.py
```

---

## Phase 7: n8n Workflow Automation

### 7.1 n8n Setup with Docker

```bash
# Create n8n data directory
mkdir -p ~/.n8n

# Run n8n container
docker run -it --rm \
  -p 5678:5678 \
  -e N8N_HOST=localhost \
  -e N8N_PROTOCOL=http \
  -e WEBHOOK_URL=http://localhost:5678/webhook \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n:latest
```

Access: http://localhost:5678

### 7.2 n8n Webhook Integration

Create `backend/api/routes/webhooks.py`:

```python
from fastapi import APIRouter, Request, Depends
from backend.api.middleware.tenant_context import get_current_tenant_id
import logging

router = APIRouter()
logger = logging.getLogger(__name__)

@router.post("/webhooks/n8n/data-sync")
async def handle_n8n_data_sync(request: Request):
    """Webhook endpoint for n8n data sync completion"""

    payload = await request.json()

    logger.info(f"Received n8n sync webhook: {payload}")

    tenant_id = payload.get("tenant_id")
    brand_id = payload.get("brand_id")
    sync_type = payload.get("sync_type")  # shopify, meta, tiktok
    records_count = payload.get("records_count", 0)

    # Process the sync data
    # TODO: Update database, trigger recommendations

    return {
        "status": "processed",
        "tenant_id": tenant_id,
        "brand_id": brand_id,
        "sync_type": sync_type,
        "records_processed": records_count
    }

@router.post("/webhooks/n8n/ai-analysis")
async def handle_n8n_ai_analysis(request: Request):
    """Webhook for AI analysis results"""

    payload = await request.json()

    logger.info("Received AI analysis webhook")

    # TODO: Store analysis results

    return {"status": "stored"}
```

---

## Phase 8: Observability & Monitoring

### 8.1 Observability Service

Create `backend/services/observability_service.py`:

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
import uuid
from sqlalchemy.ext.asyncio import AsyncSession
from backend.models.database import ObservabilityEvent

class EventCategory(str, Enum):
    API_LIFECYCLE = "api.lifecycle"
    AI_OPERATIONS = "ai.operations"
    BILLING = "billing"
    SECURITY = "security"
    DATA_SYNC = "data.sync"
    SYSTEM = "system"

class EventSeverity(str, Enum):
    DEBUG = "debug"
    INFO = "info"
    WARNING = "warning"
    ERROR = "error"
    CRITICAL = "critical"

@dataclass
class StructuredEvent:
    tenant_id: str
    category: EventCategory
    event_type: str
    severity: EventSeverity

    event_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    request_id: str = None
    user_id: str = None
    timestamp: datetime = field(default_factory=datetime.utcnow)
    duration_ms: float = 0.0
    cpu_time_ms: float = 0.0
    tokens_input: int = 0
    tokens_output: int = 0
    tokens_used: int = 0
    cost_usd: float = 0.0
    status: str = "success"
    error_code: str = None
    error_message: str = None

class ObservabilityService:
    def __init__(self, db_session: AsyncSession):
        self.db_session = db_session

    async def track_event(self, event: StructuredEvent):
        """Store event in database"""

        db_event = ObservabilityEvent(
            event_id=event.event_id,
            request_id=event.request_id,
            tenant_id=event.tenant_id,
            user_id=event.user_id,
            category=event.category.value,
            event_type=event.event_type,
            severity=event.severity.value,
            status=event.status,
            duration_ms=event.duration_ms,
            cpu_time_ms=event.cpu_time_ms,
            tokens_input=event.tokens_input,
            tokens_output=event.tokens_output,
            tokens_used=event.tokens_used,
            cost_usd=event.cost_usd,
            error_code=event.error_code,
            error_message=event.error_message
        )

        self.db_session.add(db_event)
        await self.db_session.commit()

        return db_event
```

---

## Phase 9: Testing

### 9.1 Unit Tests

Create `backend/tests/unit/test_shopify_service.py`:

```python
import pytest
from unittest.mock import AsyncMock, patch
from backend.integrations.shopify_service import ShopifyService

@pytest.mark.asyncio
async def test_get_products():
    service = ShopifyService(
        api_key="test-key",
        api_secret="test-secret",
        shop_name="test-shop"
    )

    with patch('httpx.AsyncClient.get') as mock_get:
        mock_response = AsyncMock()
        mock_response.json.return_value = {
            "products": [
                {"id": 1, "title": "Product 1"},
                {"id": 2, "title": "Product 2"}
            ]
        }
        mock_get.return_value = mock_response

        products = await service.get_products()

        assert len(products) == 2
        assert products[0]["title"] == "Product 1"
```

Run tests:

```bash
pytest backend/tests/unit -v
pytest backend/tests/integration -v
```

---

## Phase 10: Deployment & Production

### 10.1 Docker Containerization

Create `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Run migrations
RUN alembic upgrade head

# Expose port
EXPOSE 8000

# Start server
CMD ["uvicorn", "backend.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### 10.2 Docker Compose Full Stack

Create `docker-compose.yml`:

```yaml
version: "3.8"

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: pulsepoint
      POSTGRES_PASSWORD: password123
      POSTGRES_DB: pulsepoint
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U pulsepoint"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  pulsepoint-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://pulsepoint:password123@postgres:5432/pulsepoint
      REDIS_URL: redis://redis:6379/0
      ENVIRONMENT: production
      LOG_LEVEL: INFO
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./backend:/app/backend

  n8n:
    image: n8nio/n8n:latest
    ports:
      - "5678:5678"
    environment:
      N8N_HOST: localhost
      N8N_PROTOCOL: http
      WEBHOOK_URL: http://localhost:5678/webhook
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  postgres_data:
  redis_data:
  n8n_data:
```

Deploy:

```bash
docker-compose up -d
```

### 10.3 GCP Cloud Run Deployment

```bash
# Build and push image
docker build -t gcr.io/PROJECT_ID/pulsepoint:latest .
docker push gcr.io/PROJECT_ID/pulsepoint:latest

# Deploy to Cloud Run
gcloud run deploy pulsepoint-api \
  --image gcr.io/PROJECT_ID/pulsepoint:latest \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars=DATABASE_URL=$DATABASE_URL,REDIS_URL=$REDIS_URL
```

---

## Validation Checkpoints

After completing each phase, verify:

### Phase 1 ✓

- Virtual environment active
- All dependencies installed
- .env configured

### Phase 2 ✓

- PostgreSQL running
- Models created
- Alembic migrations applied

### Phase 3 ✓

- FastAPI server starts
- /health endpoint responds
- Swagger docs accessible at /docs

### Phase 4 ✓

- Shopify API integration works
- Can fetch products and orders
- Error handling in place

### Phase 5 ✓

- Claude integration working
- Token tracking implemented
- Analysis endpoint responds

### Phase 6 ✓

- JWT tokens generated
- Tenant context extracted
- RLS policies active

### Phase 7 ✓

- n8n running
- Webhooks configured
- Data flowing from n8n to Pulsepoint

### Phase 8 ✓

- Events tracked in database
- BigQuery receiving events
- Dashboards updated

### Phase 9 ✓

- Unit tests pass
- Integration tests pass
- Coverage > 80%

### Phase 10 ✓

- Docker image builds
- Container runs successfully
- Cloud Run deployment working

---

## Troubleshooting Guide

### "ModuleNotFoundError: No module named 'backend'"

**Solution:** Add project root to PYTHONPATH

```bash
export PYTHONPATH="${PYTHONPATH}:$(pwd)"
```

### "psycopg2 compilation error"

**Solution:** Install binary version

```bash
pip install psycopg2-binary
```

### "JWT token invalid"

**Solution:** Verify JWT_SECRET matches in .env and token creation

### "RLS policy denies access"

**Solution:** Ensure `SET app.tenant_id` is executed before queries

### "n8n can't reach Pulsepoint API"

**Solution:** Use `http://host.docker.internal:8000` in n8n instead of `localhost`

---

## Git Commit Strategy

After each phase, commit:

```bash
git add -A
git commit -m "Phase X: [description]"
```

Keep commits organized and descriptive for easier rollback.

---

## Next Steps After Deployment

1. **Monitor**: Set up CloudWatch/Prometheus dashboards
2. **Scale**: Configure auto-scaling policies
3. **Security**: Implement API key rotation
4. **Compliance**: Audit logs and data access
5. **Documentation**: Write runbooks for operations team

---

This guide should be updated as the project evolves. Keep documentation in sync with code changes.
