# MCP (Model Context Protocol) Design - Pulsepoint AI Control Tower

## Executive Summary

This document outlines a production-grade MCP server architecture for Pulsepoint—an AI-powered platform that unifies e-commerce data (Shopify) with multi-channel advertising (Meta, TikTok, Google) and inventory intelligence. The design emphasizes **multi-tenant data isolation**, **comprehensive observability**, **security**, and **scalability**.

---

## 1. Architecture Overview

### 1.1 MCP Server Components

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Applications                      │
│              (Web UI, Mobile App, Partner APIs)                  │
└────────────────────────┬────────────────────────────────────────┘
                         │
                    HTTP/WebSocket
                         │
         ┌───────────────▼──────────────────┐
         │   MCP Server (FastAPI/uvicorn)   │
         │   - Request Router               │
         │   - Authentication/Authorization │
         │   - Multi-tenancy Middleware     │
         └───────────────┬──────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
    ┌────────┐     ┌──────────┐     ┌─────────────┐
    │ Auth   │     │ Insights │     │  Data Flow  │
    │Service │     │ Service  │     │  Orchestr.  │
    └────────┘     └──────────┘     └─────────────┘
        │                │                │
        ▼                ▼                ▼
    ┌──────────────────────────────────────────────┐
    │         Service Layer (Business Logic)       │
    │  - Attribution Service                       │
    │  - AI Insights Service                       │
    │  - Recommendation Engine                     │
    │  - Inventory Intelligence Service            │
    │  - Observability Service                     │
    └──────────────────────────────────────────────┘
        │                                │
        ▼                                ▼
    ┌──────────────────────────────────────────────┐
    │           Data Access Layer (DAL)            │
    │  - PostgreSQL (Transactional)               │
    │  - BigQuery (Analytics)                      │
    │  - Redis Cache (Session/Results)            │
    │  - Elasticsearch (Logs/Search)              │
    └──────────────────────────────────────────────┘
        │         │             │            │
        ▼         ▼             ▼            ▼
    Postgres   BigQuery      Redis        Elasticsearch
```

---

## 2. Multi-Tenancy Architecture

### 2.1 Tenant Isolation Strategy

**Requirement:** No tenant can see or access another tenant's data.

#### 2.1.1 Tenant Context Management

```python
# Core tenant context middleware
from contextvars import ContextVar
from typing import Optional

# Thread-safe context variable for tenant ID
current_tenant_id: ContextVar[Optional[str]] = ContextVar('tenant_id', default=None)
current_user_id: ContextVar[Optional[str]] = ContextVar('user_id', default=None)
current_brand_id: ContextVar[Optional[str]] = ContextVar('brand_id', default=None)

class TenantContextMiddleware:
    """
    Extracts tenant/user context from JWT token.
    Validates JWT signature and applies context to all requests.
    """

    async def __call__(self, request: Request, call_next):
        # Extract JWT from Authorization header
        auth_header = request.headers.get("Authorization")

        if not auth_header:
            raise HTTPException(status_code=401, detail="Missing auth token")

        try:
            token = auth_header.split(" ")[1]
            payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])

            tenant_id = payload.get("tenant_id")
            user_id = payload.get("user_id")
            brand_id = payload.get("brand_id")

            # Validate tenant exists
            if not await validate_tenant(tenant_id):
                raise HTTPException(status_code=403, detail="Invalid tenant")

            # Set context variables
            current_tenant_id.set(tenant_id)
            current_user_id.set(user_id)
            current_brand_id.set(brand_id)

        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail="Invalid token")

        response = await call_next(request)
        return response
```

#### 2.1.2 Database Row-Level Security (RLS)

```sql
-- PostgreSQL Row Level Security Policy
-- Ensures users can only see data belonging to their tenant

-- Create RLS policies
ALTER TABLE brands ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE ad_performance ENABLE ROW LEVEL SECURITY;
ALTER TABLE ai_insights ENABLE ROW LEVEL SECURITY;
ALTER TABLE integrations ENABLE ROW LEVEL SECURITY;

-- Policy for brands table
CREATE POLICY brands_tenant_isolation ON brands
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid)
    WITH CHECK (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Policy for orders table
CREATE POLICY orders_tenant_isolation ON orders
    USING (
        brand_id IN (
            SELECT brand_id FROM brands
            WHERE tenant_id = current_setting('app.current_tenant_id')::uuid
        )
    );

-- Similar policies for all other tables...

-- Enforce on every query
SET app.current_tenant_id = '550e8400-e29b-41d4-a716-446655440000';
```

#### 2.1.3 Schema-Based Multi-Tenancy (Hybrid Approach)

For extreme isolation, use separate PostgreSQL schemas per tenant:

```python
# At runtime, dynamically switch schemas based on tenant
class TenantAwareDatabase:
    def __init__(self, base_url: str):
        self.base_url = base_url

    def get_session(self, tenant_id: str):
        """Get database session with tenant schema"""
        schema_name = f"tenant_{tenant_id}"

        # Set search_path for this connection
        session = Session(self.engine)
        session.execute(f"SET search_path TO {schema_name}")
        return session
```

#### 2.1.4 BigQuery Multi-Tenancy

BigQuery doesn't support RLS natively, so use dataset-level access control:

```python
class BigQueryTenantManager:
    """Manages BigQuery datasets per tenant"""

    def __init__(self, project_id: str):
        self.bq_client = bigquery.Client(project=project_id)

    def get_dataset_for_tenant(self, tenant_id: str) -> str:
        """
        Returns dataset name for tenant.
        Dataset: pulsepoint_tenant_{tenant_id}
        """
        dataset_id = f"pulsepoint_tenant_{tenant_id}"

        # Verify dataset exists and user has access
        dataset = self.bq_client.get_dataset(dataset_id)
        return dataset_id

    def ensure_tenant_dataset_exists(self, tenant_id: str):
        """Creates isolated BigQuery dataset for new tenant"""
        dataset_id = f"pulsepoint_tenant_{tenant_id}"
        dataset = bigquery.Dataset(f"{self.bq_client.project}.{dataset_id}")
        dataset.location = "US"

        # Create with user/service account as owner
        dataset = self.bq_client.create_dataset(dataset)

        # Grant access only to tenant's service account
        self.grant_dataset_access(dataset_id, tenant_id)

    def grant_dataset_access(self, dataset_id: str, tenant_id: str):
        """Grant dataset access to tenant's service account"""
        policy = self.bq_client.get_iam_policy(dataset_id)

        service_account = f"tenant-{tenant_id}@project.iam.gserviceaccount.com"
        policy.grant_role("roles/bigquery.dataEditor", f"serviceAccount:{service_account}")

        self.bq_client.set_iam_policy(dataset_id, policy)
```

#### 2.1.5 Redis Cache Isolation

```python
class TenantAwareRedisCache:
    """Isolate cache keys by tenant"""

    def __init__(self, redis_url: str):
        self.redis = redis.from_url(redis_url)

    def get_cache_key(self, key: str, tenant_id: str) -> str:
        """Namespace all cache keys with tenant ID"""
        return f"tenant:{tenant_id}:{key}"

    async def get(self, key: str, tenant_id: str) -> Optional[str]:
        """Get cached value for specific tenant"""
        cache_key = self.get_cache_key(key, tenant_id)
        return self.redis.get(cache_key)

    async def set(self, key: str, value: str, tenant_id: str, ttl: int = 3600):
        """Set cached value, scoped to tenant"""
        cache_key = self.get_cache_key(key, tenant_id)
        self.redis.setex(cache_key, ttl, value)

    async def flush_tenant_cache(self, tenant_id: str):
        """Clear all cache for a tenant (e.g., on data update)"""
        pattern = f"tenant:{tenant_id}:*"
        keys = self.redis.keys(pattern)
        if keys:
            self.redis.delete(*keys)
```

### 2.2 Data Access Control

#### 2.2.1 API-Level Enforcement

```python
from fastapi import Depends, HTTPException
from typing import Annotated

async def verify_brand_access(brand_id: str, current_user = Depends(get_current_user)):
    """
    Verify user has access to this brand.
    Prevents: User A accessing Brand B's data.
    """

    # Check if brand belongs to user's tenant
    brand = await db.query(Brand).filter(
        Brand.brand_id == brand_id,
        Brand.tenant_id == current_user.tenant_id
    ).first()

    if not brand:
        raise HTTPException(
            status_code=403,
            detail="Access denied. Brand does not exist or belongs to another tenant."
        )

    return brand


@router.get("/api/brands/{brand_id}/dashboard")
async def get_dashboard(
    brand_id: str,
    brand: Brand = Depends(verify_brand_access)
):
    """
    Get brand dashboard.
    - Automatically filtered to current tenant
    - No cross-tenant data exposure
    """

    # At this point, brand is guaranteed to belong to current user's tenant
    dashboard_data = await get_dashboard_data(brand_id)
    return dashboard_data
```

#### 2.2.2 Query-Level Isolation

```python
class TenantAwareRepository:
    """Base repository class ensuring tenant isolation in all queries"""

    async def find_by_id(self, id: str):
        """Find record, but only if it belongs to current tenant"""

        tenant_id = current_tenant_id.get()

        # Automatically filter by tenant
        query = self.model.query.filter(
            self.model.id == id,
            self.model.tenant_id == tenant_id
        )

        return await query.first()

    async def find_all(self, filters: dict = None):
        """List all records, automatically filtered by tenant"""

        tenant_id = current_tenant_id.get()

        query = self.model.query.filter(self.model.tenant_id == tenant_id)

        # Apply additional filters
        if filters:
            for key, value in filters.items():
                query = query.filter(getattr(self.model, key) == value)

        return await query.all()
```

---

## 3. Comprehensive Observability

### 3.1 Observability Strategy

**Goal:** Complete visibility into:

- Who asked what questions
- How many tokens consumed
- Model accuracy/performance
- Latency and throughput
- Error rates and types
- Cost per operation

### 3.2 Event-Driven Observability

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
from typing import Dict, Any, Optional

class EventType(str, Enum):
    """All observable events in the system"""

    # User events
    USER_AUTHENTICATED = "user.authenticated"
    USER_QUESTION_ASKED = "user.question_asked"
    USER_ACCEPTED_RECOMMENDATION = "user.accepted_recommendation"

    # API events
    API_REQUEST = "api.request"
    API_RESPONSE = "api.response"
    API_ERROR = "api.error"

    # AI events
    AI_SUMMARY_GENERATED = "ai.summary_generated"
    AI_ALERT_DETECTED = "ai.alert_detected"
    AI_QUESTION_ANSWERED = "ai.question_answered"

    # Data events
    DATA_SYNC_STARTED = "data.sync_started"
    DATA_SYNC_COMPLETED = "data.sync_completed"
    DATA_SYNC_FAILED = "data.sync_failed"

    # Integration events
    INTEGRATION_CONNECTED = "integration.connected"
    INTEGRATION_DISCONNECTED = "integration.disconnected"
    INTEGRATION_FAILED = "integration.failed"


@dataclass
class ObservabilityEvent:
    """Structured event for all observability tracking"""

    # Core identifiers
    tenant_id: str
    brand_id: Optional[str]
    user_id: Optional[str]

    # Event details
    event_type: EventType
    timestamp: datetime

    # Context
    session_id: str
    request_id: str  # For tracing entire request flow

    # Performance metrics
    duration_ms: Optional[float] = None

    # Resource tracking
    tokens_used: Optional[int] = None
    cost_usd: Optional[float] = None

    # Result
    status: str = "success"  # success, failure, partial
    error: Optional[str] = None

    # AI-specific metrics
    model_used: Optional[str] = None
    temperature: Optional[float] = None

    # Data quality metrics
    accuracy_score: Optional[float] = None
    confidence_score: Optional[float] = None

    # Metadata
    metadata: Dict[str, Any] = None


class ObservabilityService:
    """Central service for event tracking and observability"""

    def __init__(self, bq_client, pubsub_client):
        self.bq_client = bq_client
        self.pubsub_client = pubsub_client

    async def track_event(self, event: ObservabilityEvent):
        """
        Track an event.
        1. Publish to Pub/Sub (real-time alerts)
        2. Write to BigQuery (analytics)
        3. Update metrics in monitoring system
        """

        # 1. Real-time: Publish to Pub/Sub for immediate processing
        await self.publish_to_pubsub(event)

        # 2. Analytics: Write to BigQuery for historical analysis
        await self.write_to_bigquery(event)

        # 3. Metrics: Update Prometheus/Stackdriver counters
        await self.update_metrics(event)

    async def publish_to_pubsub(self, event: ObservabilityEvent):
        """Publish to Pub/Sub for real-time alerts"""

        topic_name = f"projects/{PROJECT_ID}/topics/pulsepoint-events"

        message_json = json.dumps({
            "tenant_id": event.tenant_id,
            "event_type": event.event_type.value,
            "timestamp": event.timestamp.isoformat(),
            "tokens_used": event.tokens_used,
            "cost_usd": event.cost_usd,
            "status": event.status
        })

        future = self.pubsub_client.publish(topic_name, message_json.encode())
        await future  # Wait for publish to complete

    async def write_to_bigquery(self, event: ObservabilityEvent):
        """Write event to BigQuery for analytics"""

        table_id = f"{self.bq_client.project}.pulsepoint.observability_events"

        rows_to_insert = [{
            "tenant_id": event.tenant_id,
            "brand_id": event.brand_id,
            "user_id": event.user_id,
            "event_type": event.event_type.value,
            "timestamp": event.timestamp.isoformat(),
            "session_id": event.session_id,
            "request_id": event.request_id,
            "duration_ms": event.duration_ms,
            "tokens_used": event.tokens_used,
            "cost_usd": event.cost_usd,
            "status": event.status,
            "error": event.error,
            "model_used": event.model_used,
            "accuracy_score": event.accuracy_score
        }]

        errors = self.bq_client.insert_rows_json(table_id, rows_to_insert)

        if errors:
            logger.error(f"BigQuery insert errors: {errors}")

    async def update_metrics(self, event: ObservabilityEvent):
        """Update Prometheus/Stackdriver metrics"""

        # Track API latency
        if event.event_type == EventType.API_RESPONSE:
            api_latency.labels(
                endpoint=event.metadata.get("endpoint"),
                method=event.metadata.get("method")
            ).observe(event.duration_ms / 1000)

        # Track AI token usage
        if event.tokens_used:
            ai_tokens_counter.labels(
                model=event.model_used or "unknown",
                tenant=event.tenant_id
            ).inc(event.tokens_used)

            ai_cost_counter.labels(
                model=event.model_used or "unknown",
                tenant=event.tenant_id
            ).inc(event.cost_usd or 0)

        # Track errors
        if event.status == "failure":
            error_counter.labels(
                event_type=event.event_type.value,
                error_type=event.error
            ).inc()
```

### 3.3 Question & Accuracy Tracking

```python
class AIQuestionTracker:
    """Track all AI questions, responses, and accuracy"""

    async def track_question(
        self,
        question: str,
        brand_id: str,
        tenant_id: str,
        user_id: str
    ) -> str:
        """
        Create a question tracking record.
        Returns question_id for later linking with answer.
        """

        question_record = AIQuestion(
            question_id=str(uuid4()),
            tenant_id=tenant_id,
            brand_id=brand_id,
            user_id=user_id,
            question_text=question,
            asked_at=datetime.utcnow(),
            question_category=self.categorize_question(question)
        )

        db.add(question_record)
        await db.commit()

        return question_record.question_id

    async def track_answer(
        self,
        question_id: str,
        answer_text: str,
        model_used: str,
        tokens_used: int,
        generation_time_ms: float
    ):
        """Record AI's response"""

        answer_record = AIAnswer(
            answer_id=str(uuid4()),
            question_id=question_id,
            answer_text=answer_text,
            model=model_used,
            tokens_consumed=tokens_used,
            generation_time_ms=generation_time_ms,
            generated_at=datetime.utcnow()
        )

        db.add(answer_record)
        await db.commit()

    async def track_feedback(
        self,
        question_id: str,
        answer_id: str,
        user_rating: int,  # 1-5 stars
        accuracy_comment: Optional[str],
        was_actionable: bool,
        user_id: str
    ):
        """Track user feedback for accuracy scoring"""

        feedback_record = AIFeedback(
            feedback_id=str(uuid4()),
            question_id=question_id,
            answer_id=answer_id,
            rating=user_rating,
            accuracy_comment=accuracy_comment,
            was_actionable=was_actionable,
            provided_by=user_id,
            provided_at=datetime.utcnow()
        )

        db.add(feedback_record)
        await db.commit()

        # Recalculate accuracy metrics
        await self.update_accuracy_metrics(question_id)

    async def update_accuracy_metrics(self, question_id: str):
        """Recalculate accuracy based on feedback"""

        feedbacks = db.query(AIFeedback).filter(
            AIFeedback.question_id == question_id
        ).all()

        avg_rating = sum(f.rating for f in feedbacks) / len(feedbacks)
        actionable_count = sum(1 for f in feedbacks if f.was_actionable)
        actionable_rate = actionable_count / len(feedbacks)

        question_record = db.query(AIQuestion).filter(
            AIQuestion.question_id == question_id
        ).first()

        question_record.accuracy_score = avg_rating / 5.0  # Normalize to 0-1
        question_record.actionability_score = actionable_rate

        await db.commit()
```

### 3.4 Token & Cost Tracking

```python
class TokenTracker:
    """Track AI token usage and costs across tenant"""

    # Token pricing (Claude 3 Sonnet as example)
    PRICING = {
        "claude-3-sonnet": {
            "input": 0.003 / 1_000_000,  # $3 per million input tokens
            "output": 0.015 / 1_000_000   # $15 per million output tokens
        }
    }

    async def track_ai_usage(
        self,
        tenant_id: str,
        brand_id: str,
        model: str,
        input_tokens: int,
        output_tokens: int,
        operation: str
    ):
        """Track AI API usage"""

        # Calculate cost
        pricing = self.PRICING.get(model, self.PRICING["claude-3-sonnet"])
        input_cost = input_tokens * pricing["input"]
        output_cost = output_tokens * pricing["output"]
        total_cost = input_cost + output_cost

        # Record in database
        usage_record = AITokenUsage(
            usage_id=str(uuid4()),
            tenant_id=tenant_id,
            brand_id=brand_id,
            model=model,
            operation=operation,
            input_tokens=input_tokens,
            output_tokens=output_tokens,
            total_tokens=input_tokens + output_tokens,
            input_cost_usd=input_cost,
            output_cost_usd=output_cost,
            total_cost_usd=total_cost,
            recorded_at=datetime.utcnow()
        )

        db.add(usage_record)

        # Update running totals
        tenant_usage = db.query(TenantTokenUsage).filter(
            TenantTokenUsage.tenant_id == tenant_id,
            TenantTokenUsage.date == datetime.utcnow().date()
        ).first()

        if tenant_usage:
            tenant_usage.total_tokens += input_tokens + output_tokens
            tenant_usage.total_cost_usd += total_cost
        else:
            tenant_usage = TenantTokenUsage(
                tenant_id=tenant_id,
                date=datetime.utcnow().date(),
                total_tokens=input_tokens + output_tokens,
                total_cost_usd=total_cost
            )
            db.add(tenant_usage)

        await db.commit()

        # Track in metrics
        ai_tokens_by_tenant.labels(tenant=tenant_id).inc(input_tokens + output_tokens)
        ai_cost_by_tenant.labels(tenant=tenant_id).inc(total_cost)

    async def get_usage_report(
        self,
        tenant_id: str,
        start_date: date,
        end_date: date
    ) -> Dict:
        """Get usage report for billing/monitoring"""

        usages = db.query(AITokenUsage).filter(
            AITokenUsage.tenant_id == tenant_id,
            AITokenUsage.recorded_at >= start_date,
            AITokenUsage.recorded_at <= end_date
        ).all()

        return {
            "total_tokens": sum(u.total_tokens for u in usages),
            "total_cost_usd": sum(u.total_cost_usd for u in usages),
            "by_model": self._group_by_model(usages),
            "by_operation": self._group_by_operation(usages),
            "by_day": self._group_by_day(usages)
        }
```

### 3.5 Dashboards & Alerting

**BigQuery Queries for Observability:**

```sql
-- Top questions asked by tenant
SELECT
    tenant_id,
    question_category,
    COUNT(*) as question_count,
    AVG(accuracy_score) as avg_accuracy
FROM pulsepoint.ai_questions
WHERE date(asked_at) >= CURRENT_DATE() - 30
GROUP BY tenant_id, question_category
ORDER BY question_count DESC;

-- Token usage by tenant (daily)
SELECT
    tenant_id,
    DATE(recorded_at) as date,
    SUM(total_tokens) as total_tokens,
    SUM(total_cost_usd) as cost_usd,
    AVG(total_cost_usd) as avg_cost_per_usage
FROM pulsepoint.ai_token_usage
WHERE DATE(recorded_at) >= CURRENT_DATE() - 30
GROUP BY tenant_id, date
ORDER BY date DESC;

-- Accuracy metrics
SELECT
    model,
    COUNT(*) as total_questions,
    AVG(accuracy_score) as avg_accuracy,
    AVG(actionability_score) as avg_actionability,
    COUNTIF(rating >= 4) / COUNT(*) as satisfaction_rate
FROM pulsepoint.ai_questions q
JOIN pulsepoint.ai_feedback f ON q.question_id = f.question_id
WHERE DATE(q.asked_at) >= CURRENT_DATE() - 30
GROUP BY model;

-- API latency by endpoint
SELECT
    endpoint,
    COUNT(*) as request_count,
    AVG(duration_ms) as avg_latency_ms,
    PERCENTILE_CONT(duration_ms, 0.95) OVER () as p95_latency,
    PERCENTILE_CONT(duration_ms, 0.99) OVER () as p99_latency
FROM pulsepoint.observability_events
WHERE event_type = 'api.response'
  AND DATE(timestamp) >= CURRENT_DATE() - 1
GROUP BY endpoint
ORDER BY p99_latency DESC;
```

**Alerting Rules (Prometheus/Stackdriver):**

```yaml
# Alert: High token usage (cost spike)
alert: HighAITokenUsage
expr: rate(ai_tokens_consumed_total[1h]) > 1000000
for: 15m
labels:
  severity: warning
annotations:
  summary: "Tenant {{ $labels.tenant }} using {{ $value }} tokens/hour"

# Alert: Accuracy drop
alert: LowAccuracyScore
expr: ai_question_accuracy_score < 0.7
for: 1h
labels:
  severity: warning
annotations:
  summary: "AI accuracy dropped below 70%"

# Alert: High error rate
alert: HighErrorRate
expr: rate(api_errors_total[5m]) > 0.05
for: 5m
labels:
  severity: critical
annotations:
  summary: "Error rate > 5% for 5 minutes"
```

---

## 4. Security Architecture

### 4.1 Authentication & Authorization

#### 4.1.1 JWT-Based Authentication

```python
from jose import JWTError, jwt
from datetime import datetime, timedelta
from typing import Optional

class AuthService:
    """Secure JWT authentication"""

    def __init__(self, secret_key: str, algorithm: str = "HS256"):
        self.secret_key = secret_key
        self.algorithm = algorithm

    def create_access_token(
        self,
        tenant_id: str,
        user_id: str,
        brand_id: Optional[str] = None,
        expires_delta: Optional[timedelta] = None
    ) -> str:
        """Create JWT token with tenant/user context"""

        if expires_delta is None:
            expires_delta = timedelta(hours=8)

        expire = datetime.utcnow() + expires_delta

        payload = {
            "tenant_id": tenant_id,
            "user_id": user_id,
            "brand_id": brand_id,
            "exp": expire
        }

        encoded_jwt = jwt.encode(
            payload,
            self.secret_key,
            algorithm=self.algorithm
        )

        return encoded_jwt

    def verify_token(self, token: str) -> dict:
        """Verify and decode JWT token"""

        try:
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=[self.algorithm]
            )

            tenant_id = payload.get("tenant_id")
            user_id = payload.get("user_id")

            if not tenant_id or not user_id:
                raise HTTPException(status_code=401, detail="Invalid token")

            return payload

        except JWTError:
            raise HTTPException(status_code=401, detail="Invalid token")
```

#### 4.1.2 Role-Based Access Control (RBAC)

```python
from enum import Enum

class Role(str, Enum):
    """Tenant role hierarchy"""
    ADMIN = "admin"           # Full access, user management
    MANAGER = "manager"       # Data access, view recommendations
    ANALYST = "analyst"       # Read-only data access
    API_CLIENT = "api_client" # Programmatic access only


class Permission(str, Enum):
    """Granular permissions"""
    # Brands
    BRAND_READ = "brand:read"
    BRAND_WRITE = "brand:write"
    BRAND_DELETE = "brand:delete"

    # Data
    DATA_READ = "data:read"
    DATA_EXPORT = "data:export"

    # AI
    AI_ASK = "ai:ask"
    AI_VIEW_HISTORY = "ai:view_history"

    # Integrations
    INTEGRATION_CONNECT = "integration:connect"
    INTEGRATION_DISCONNECT = "integration:disconnect"

    # Users
    USER_MANAGE = "user:manage"
    ROLE_ASSIGN = "role:assign"

    # Admin
    ADMIN_SETTINGS = "admin:settings"
    BILLING_VIEW = "billing:view"


ROLE_PERMISSIONS = {
    Role.ADMIN: [
        Permission.BRAND_READ,
        Permission.BRAND_WRITE,
        Permission.BRAND_DELETE,
        Permission.DATA_READ,
        Permission.DATA_EXPORT,
        Permission.AI_ASK,
        Permission.AI_VIEW_HISTORY,
        Permission.INTEGRATION_CONNECT,
        Permission.INTEGRATION_DISCONNECT,
        Permission.USER_MANAGE,
        Permission.ROLE_ASSIGN,
        Permission.ADMIN_SETTINGS,
        Permission.BILLING_VIEW
    ],
    Role.MANAGER: [
        Permission.BRAND_READ,
        Permission.BRAND_WRITE,
        Permission.DATA_READ,
        Permission.DATA_EXPORT,
        Permission.AI_ASK,
        Permission.AI_VIEW_HISTORY,
        Permission.INTEGRATION_CONNECT
    ],
    Role.ANALYST: [
        Permission.BRAND_READ,
        Permission.DATA_READ,
        Permission.DATA_EXPORT,
        Permission.AI_ASK,
        Permission.AI_VIEW_HISTORY
    ],
    Role.API_CLIENT: [
        Permission.DATA_READ,
        Permission.AI_ASK
    ]
}


def require_permission(required_permission: Permission):
    """Dependency for protecting endpoints"""

    async def verify_permission(current_user = Depends(get_current_user)):

        user_role = current_user.role
        allowed_permissions = ROLE_PERMISSIONS.get(user_role, [])

        if required_permission not in allowed_permissions:
            raise HTTPException(
                status_code=403,
                detail=f"Permission denied. Requires {required_permission}"
            )

        return current_user

    return verify_permission


# Usage in endpoint
@router.post("/api/brands/{brand_id}/integrations")
async def connect_integration(
    brand_id: str,
    integration_data: IntegrationCreate,
    current_user = Depends(require_permission(Permission.INTEGRATION_CONNECT))
):
    """Only users with INTEGRATION_CONNECT permission can call this"""
    # ...
```

### 4.2 Data Encryption

#### 4.2.1 Encryption at Rest

```python
from cryptography.fernet import Fernet
import os

class EncryptionService:
    """Encrypt sensitive data at rest"""

    def __init__(self):
        # Get encryption key from environment (should be in Secret Manager)
        self.key = os.getenv("DATA_ENCRYPTION_KEY").encode()
        self.cipher = Fernet(self.key)

    def encrypt(self, plaintext: str) -> str:
        """Encrypt sensitive data"""
        encrypted = self.cipher.encrypt(plaintext.encode())
        return encrypted.decode()

    def decrypt(self, ciphertext: str) -> str:
        """Decrypt sensitive data"""
        decrypted = self.cipher.decrypt(ciphertext.encode())
        return decrypted.decode()


# Usage: Encrypt OAuth tokens in database
class Integration(Base):
    """Store OAuth tokens encrypted"""

    __tablename__ = "integrations"

    integration_id: str = Column(String, primary_key=True)
    tenant_id: str = Column(String, index=True)
    platform: str = Column(String)
    access_token: str = Column(String)  # Encrypted in database
    refresh_token: str = Column(String)  # Encrypted in database

    encryption_service = EncryptionService()

    def save_tokens(self, access_token: str, refresh_token: str):
        """Save tokens encrypted"""
        self.access_token = self.encryption_service.encrypt(access_token)
        self.refresh_token = self.encryption_service.encrypt(refresh_token)

    def get_access_token(self) -> str:
        """Retrieve and decrypt token"""
        return self.encryption_service.decrypt(self.access_token)

    def get_refresh_token(self) -> str:
        """Retrieve and decrypt token"""
        return self.encryption_service.decrypt(self.refresh_token)
```

#### 4.2.2 Encryption in Transit

```python
# All API endpoints must use HTTPS/TLS 1.3+
# Configure in FastAPI:

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from starlette.middleware.https import HTTPSMiddleware

app = FastAPI()

# Force HTTPS in production
if not DEBUG:
    app.add_middleware(HTTPSMiddleware, enforce=True)

# Enforce HSTS header
@app.middleware("http")
async def add_security_headers(request: Request, call_next):
    response = await call_next(request)

    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"
    response.headers["X-XSS-Protection"] = "1; mode=block"

    return response
```

### 4.3 API Security

#### 4.3.1 Rate Limiting (Prevent Abuse)

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@router.get("/api/brands/{brand_id}/dashboard")
@limiter.limit("100/minute")  # 100 requests per minute per IP
async def get_dashboard(request: Request, brand_id: str):
    """API rate limited to prevent abuse"""
    # ...

# Tenant-based rate limiting (stricter per tenant)
@router.post("/api/brands/{brand_id}/ask")
@limiter.limit("20/minute")  # Q&A limited to 20/minute (AI cost control)
async def ask_question(request: Request, brand_id: str, question: str):
    """Stricter limit on AI questions to control costs"""
    # ...
```

#### 4.3.2 Input Validation & Sanitization

```python
from pydantic import BaseModel, validator, constr
import bleach

class AskQuestionRequest(BaseModel):
    question: constr(min_length=1, max_length=500)  # Length validation

    @validator('question')
    def sanitize_question(cls, v):
        """Remove potentially malicious characters"""
        # Allow only safe HTML tags
        allowed_tags = []
        cleaned = bleach.clean(v, tags=allowed_tags, strip=True)
        return cleaned.strip()

# Usage
@router.post("/api/brands/{brand_id}/ask")
async def ask_question(brand_id: str, request: AskQuestionRequest):
    """Question is automatically validated and sanitized"""
    # request.question is safe to use
```

#### 4.3.3 SQL Injection Prevention

```python
# Always use parameterized queries
# DON'T DO THIS:
# query = f"SELECT * FROM orders WHERE brand_id = '{brand_id}'"

# DO THIS:
from sqlalchemy import text

query = text("""
    SELECT * FROM orders
    WHERE brand_id = :brand_id
    AND tenant_id = :tenant_id
""")

result = db.execute(query, {"brand_id": brand_id, "tenant_id": tenant_id})
```

---

## 5. API Design

### 5.1 MCP Server Endpoints

```python
# 1. Question Answering (Core MCP Resource)
POST /mcp/v1/brands/{brand_id}/ask
  Input:  { question: str }
  Output: { answer: str, sources: [], tokens_used: int, confidence: float }
  Observability: Track question, answer, accuracy

# 2. Get Dashboard Summary
GET /mcp/v1/brands/{brand_id}/dashboard
  Output: DashboardSummary (AI summary, recommendations, metrics)
  Cache: Cached for 1 hour per brand

# 3. Get Alerts
GET /mcp/v1/brands/{brand_id}/alerts
  Output: List[Alert] (sorted by severity and urgency)
  Real-time: Pub/Sub based alerts

# 4. Get Recommendations
GET /mcp/v1/brands/{brand_id}/recommendations
  Query Params: limit, sort_by (priority, impact)
  Output: List[Recommendation]

# 5. Accept Recommendation
POST /mcp/v1/brands/{brand_id}/recommendations/{rec_id}/accept
  Input: { action: str }
  Output: { status: str, applied_at: timestamp }

# 6. Data Export
GET /mcp/v1/brands/{brand_id}/export
  Query Params: format (csv, json), date_range, metrics
  Output: File or URL to download
  Observability: Track data exports, PII handling

# 7. Integration Management
POST /mcp/v1/brands/{brand_id}/integrations
  Input: { platform: str, oauth_code: str }
  Output: { status: str, syncing: bool }

DELETE /mcp/v1/brands/{brand_id}/integrations/{platform}
  Output: { status: str }

# 8. Admin: Tenant Usage
GET /mcp/admin/v1/tenants/{tenant_id}/usage
  Output: { total_questions, total_tokens, total_cost, by_date }
  Permission: BILLING_VIEW

# 9. Admin: Tenant Health
GET /mcp/admin/v1/tenants/{tenant_id}/health
  Output: {
    error_rate,
    avg_latency_ms,
    active_users,
    last_sync_status
  }
```

---

## 6. High-Level Data Flow

```
┌──────────────┐
│  User/Client │
└───────┬──────┘
        │ HTTPS Request
        ▼
┌──────────────────────────────────────┐
│      MCP Server (FastAPI)            │
│  - Request Router                    │
│  - Auth Middleware                   │
│  - Tenant Context (TLS)              │
│  - Rate Limiting                     │
└────────┬─────────────────────────────┘
         │
         ├─ Query tenant context
         │
         ▼
    ┌────────────────────────────────────┐
    │    Tenant Data Access Layer        │
    │  - RLS Policies (PostgreSQL)       │
    │  - Schema Isolation (if used)      │
    │  - Dataset Access (BigQuery)       │
    │  - Cache Namespace (Redis)         │
    └────────┬──────────┬────────────────┘
             │          │
             ▼          ▼
      ┌─────────────────────────┐
      │   PostgreSQL (Primary)  │ ◄─ Brand-specific data
      │   BigQuery (Analytics)  │ ◄─ Metrics & analysis
      │   Redis (Cache)         │ ◄─ Session & results
      │   Elasticsearch (Logs)  │ ◄─ Search & debugging
      └────────┬────────────────┘
               │
               ▼
    ┌──────────────────────────────────────┐
    │     Observability Event Emission     │
    │  - Create ObservabilityEvent         │
    │  - Publish to Pub/Sub (real-time)   │
    │  - Write to BigQuery (analytics)    │
    │  - Update Metrics                   │
    └──────────┬──────────────────────────┘
               │
               ├─ Real-time alerts
               ├─ Cost tracking
               └─ Accuracy scoring
```

---

## 7. Database Schema for Multi-Tenancy

### 7.1 Core Tables

```sql
-- Tenants (isolated organizations)
CREATE TABLE tenants (
    tenant_id UUID PRIMARY KEY,
    company_name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    subscription_tier ENUM ('starter', 'pro', 'enterprise'),
    data_retention_days INT DEFAULT 90,
    UNIQUE(company_name)
);

-- Users within tenant
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    email VARCHAR(255) NOT NULL,
    hashed_password VARCHAR(255) NOT NULL,
    role ENUM ('admin', 'manager', 'analyst', 'api_client'),
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    UNIQUE(tenant_id, email)
);

-- Brands managed by tenant
CREATE TABLE brands (
    brand_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    brand_name VARCHAR(255) NOT NULL,
    shopify_store_url VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    UNIQUE(tenant_id, brand_name)
);

-- OAuth integrations (encrypted)
CREATE TABLE integrations (
    integration_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    brand_id UUID NOT NULL,
    platform ENUM ('shopify', 'meta', 'tiktok', 'google'),
    access_token_encrypted TEXT NOT NULL,  -- Encrypted
    refresh_token_encrypted TEXT,          -- Encrypted
    token_expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    last_synced_at TIMESTAMP,
    sync_error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    FOREIGN KEY (brand_id) REFERENCES brands(brand_id),
    UNIQUE(brand_id, platform)
);

-- Observability events (for analytics)
CREATE TABLE observability_events (
    event_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    brand_id UUID,
    user_id UUID,
    event_type VARCHAR(50) NOT NULL,  -- 'api.request', 'ai.question', etc.
    timestamp TIMESTAMP DEFAULT NOW(),
    session_id VARCHAR(100),
    request_id VARCHAR(100) UNIQUE,
    duration_ms FLOAT,
    tokens_used INT,
    cost_usd DECIMAL(10, 6),
    status VARCHAR(20),  -- 'success', 'failure', 'partial'
    error_message TEXT,
    model_used VARCHAR(100),
    accuracy_score FLOAT,
    metadata JSONB,
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    INDEX (tenant_id, timestamp),
    INDEX (event_type, timestamp)
);

-- AI questions tracking
CREATE TABLE ai_questions (
    question_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    brand_id UUID NOT NULL,
    user_id UUID NOT NULL,
    question_text TEXT NOT NULL,
    question_category VARCHAR(50),
    asked_at TIMESTAMP DEFAULT NOW(),
    accuracy_score FLOAT,  -- Set after feedback
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    FOREIGN KEY (brand_id) REFERENCES brands(brand_id),
    INDEX (tenant_id, asked_at)
);

-- AI answers tracking
CREATE TABLE ai_answers (
    answer_id UUID PRIMARY KEY,
    question_id UUID NOT NULL,
    answer_text TEXT NOT NULL,
    model VARCHAR(100),
    tokens_consumed INT,
    generation_time_ms FLOAT,
    generated_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (question_id) REFERENCES ai_questions(question_id)
);

-- User feedback on answers (for accuracy scoring)
CREATE TABLE ai_feedback (
    feedback_id UUID PRIMARY KEY,
    question_id UUID NOT NULL,
    answer_id UUID NOT NULL,
    rating INT CHECK (rating >= 1 AND rating <= 5),
    accuracy_comment TEXT,
    was_actionable BOOLEAN,
    provided_by UUID NOT NULL,
    provided_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (question_id) REFERENCES ai_questions(question_id),
    FOREIGN KEY (answer_id) REFERENCES ai_answers(answer_id),
    INDEX (question_id)
);

-- AI token usage tracking (for billing)
CREATE TABLE ai_token_usage (
    usage_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    brand_id UUID,
    model VARCHAR(100),
    operation VARCHAR(100),  -- 'dashboard_summary', 'ask_question', etc.
    input_tokens INT,
    output_tokens INT,
    total_tokens INT,
    input_cost_usd DECIMAL(12, 8),
    output_cost_usd DECIMAL(12, 8),
    total_cost_usd DECIMAL(12, 8),
    recorded_at TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    INDEX (tenant_id, DATE(recorded_at))
);

-- Daily tenant token usage (for dashboard)
CREATE TABLE tenant_token_usage (
    usage_id UUID PRIMARY KEY,
    tenant_id UUID NOT NULL,
    date DATE NOT NULL,
    total_tokens INT,
    total_cost_usd DECIMAL(12, 2),
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    UNIQUE(tenant_id, date)
);
```

---

## 8. Production Deployment Checklist

### 8.1 Security

- [ ] All secrets in Google Secret Manager
- [ ] HTTPS/TLS 1.3 enforced
- [ ] Database encryption at rest enabled
- [ ] Row-Level Security policies applied
- [ ] API keys rotated monthly
- [ ] Firewall rules whitelist only needed services
- [ ] VPC endpoint for databases (no internet exposure)

### 8.2 Data Protection

- [ ] Automated daily backups (Point-in-time recovery)
- [ ] Backup encryption enabled
- [ ] Data deletion policy (30/90/180 day retention)
- [ ] OAuth token refresh automated
- [ ] PII anonymization for logs

### 8.3 Monitoring & Alerting

- [ ] Uptime monitoring (99.9% SLA target)
- [ ] Error rate alerting (>5% triggers page)
- [ ] Token usage alerting (cost control)
- [ ] Accuracy score tracking
- [ ] Performance dashboards deployed

### 8.4 Compliance

- [ ] SOC 2 controls documented
- [ ] Data residency compliance (GDPR, CCPA)
- [ ] API rate limits enforced
- [ ] Audit logging enabled
- [ ] Terms of Service reviewed

---

## Summary Table

| Component         | Solution                                            | Benefit                                |
| ----------------- | --------------------------------------------------- | -------------------------------------- |
| Multi-Tenancy     | Row-Level Security + Schema Isolation + JWT Context | Complete data isolation, secure        |
| Observability     | Pub/Sub + BigQuery + Prometheus                     | Real-time alerts + historical analysis |
| Token Tracking    | OpenAI SDK integration + DB logging                 | Full cost visibility and control       |
| Question Tracking | Question → Answer → Feedback workflow               | Accuracy measurement and improvement   |
| Security          | JWT + RBAC + Encryption + Rate Limiting             | Production-ready protection            |
| Scalability       | Cloud Run (auto-scale) + BigQuery + Redis           | Handles growth without manual scaling  |
| Reliability       | Backups + monitoring + alerting                     | 99.9% uptime SLA                       |
