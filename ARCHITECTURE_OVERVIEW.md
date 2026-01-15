# Pulsepoint System Architecture Overview

Complete system design showing all components, data flows, and integration points.

---

## Table of Contents

1. [System Architecture Diagram](#system-architecture-diagram)
2. [Component Descriptions](#component-descriptions)
3. [Data Flow](#data-flow)
4. [Technology Stack](#technology-stack)
5. [Deployment Architecture](#deployment-architecture)
6. [Multi-Tenancy Model](#multi-tenancy-model)
7. [Observability Stack](#observability-stack)
8. [Integration Points](#integration-points)

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        EXTERNAL SOURCES                         │
├─────────────────────────────────────────────────────────────────┤
│  Shopify    Meta Ads    TikTok Ads    Google Ads   Claude AI    │
└────┬────────────┬────────────┬──────────┬──────────┬────────────┘
     │            │            │          │          │
     │   OAuth & API Keys      │          │          │
     │            │            │          │          │
     ▼            ▼            ▼          ▼          ▼
┌─────────────────────────────────────────────────────────────────┐
│                  PULSEPOINT MCP SERVER (Backend)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  FastAPI Application Layer                              │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  Routes:                                                 │   │
│  │  ├─ /api/brands          (Brand management)             │   │
│  │  ├─ /api/questions       (Q&A & AI analysis)            │   │
│  │  ├─ /api/recommendations (AI recommendations)           │   │
│  │  ├─ /api/sync            (Data sync triggers)           │   │
│  │  ├─ /api/admin           (Admin endpoints)              │   │
│  │  ├─ /api/analytics       (Metrics & dashboards)         │   │
│  │  └─ /webhooks/n8n/*      (n8n integration)              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                  │                                │
│  ┌──────────────────────────────┴──────────────────────────┐   │
│  │                  Middleware Stack                       │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  • CORS Middleware                                       │   │
│  │  • TenantContextMiddleware (extract tenant from JWT)     │   │
│  │  • AuthMiddleware (verify JWT token)                    │   │
│  │  • ErrorHandlerMiddleware                               │   │
│  │  • RequestTracingMiddleware (X-Request-ID)              │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                  │                                │
│  ┌──────────────────────────────┴──────────────────────────┐   │
│  │  Service Layer (Business Logic)                         │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  • AttributionService      (Order → Campaign mapping)   │   │
│  │  • AIInsightsService       (Claude API integration)      │   │
│  │  • RecommendationEngine    (Generate recommendations)    │   │
│  │  • InventoryService        (Stock level analysis)        │   │
│  │  • AuthService             (JWT & RBAC)                 │   │
│  │  • ObservabilityService    (Event tracking)              │   │
│  │  • TokenTrackerService     (AI cost tracking)            │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                  │                                │
│  ┌──────────────────────────────┴──────────────────────────┐   │
│  │  Integration Layer (External APIs)                      │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  • ShopifyService         (Products, Orders, Inventory) │   │
│  │  • MetaAdsService         (Campaigns, Performance)       │   │
│  │  • TikTokAdsService       (Campaign metrics)             │   │
│  │  • GoogleAdsService       (Campaign performance)         │   │
│  │  • AnthropicService       (Claude AI)                    │   │
│  │  • BigQueryService        (Analytics backend)            │   │
│  │  • PubSubService          (Event publishing)             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                  │                                │
│  ┌──────────────────────────────┴──────────────────────────┐   │
│  │  Data Access Layer (Database & Cache)                   │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  • SQLAlchemy ORM (async queries)                        │   │
│  │  • RLS Policies (Row-Level Security)                     │   │
│  │  • Redis Caching (1-hour TTL)                            │   │
│  │  • Connection Pooling                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                  │                                │
└──────────────────────────────────┼──────────────────────────────┘
                                   │
                ┌──────────────────┼──────────────────┐
                │                  │                  │
                ▼                  ▼                  ▼
    ┌──────────────────┐  ┌───────────────┐  ┌──────────────────┐
    │  PostgreSQL      │  │  Redis        │  │  Google Cloud    │
    │  (Primary Store) │  │  (Cache)      │  │  (Analytics)     │
    ├──────────────────┤  ├───────────────┤  ├──────────────────┤
    │ • Tenants        │  │ Cached:       │  │ • BigQuery       │
    │ • Brands         │  │ • Tenant data │  │ • Pub/Sub        │
    │ • Users          │  │ • Brand       │  │ • Cloud Storage  │
    │ • Orders         │  │   summaries   │  │ • Cloud Run      │
    │ • AI Tokens      │  │ • Query       │  └──────────────────┘
    │ • Events         │  │   results     │
    │ (RLS enabled)    │  └───────────────┘
    └──────────────────┘
```

---

## Component Descriptions

### Frontend Layer

**Web Dashboard** (React)

- Real-time metrics display
- Question form for AI analysis
- Recommendation viewer
- Alert center

### API Layer

**FastAPI Application**

- RESTful endpoints
- WebSocket support (future)
- Request validation (Pydantic)
- Automatic API documentation

### Service Layer

**Business Logic Services:**

1. **AttributionService**

   - Maps orders to ad campaigns
   - Last-touch and multi-touch attribution models
   - Revenue attribution by channel

2. **AIInsightsService**

   - Sends metrics to Claude
   - Processes AI-generated insights
   - Caches analysis results

3. **RecommendationEngine**

   - Generates budget recommendations
   - Identifies optimization opportunities
   - Prioritizes actions by ROI

4. **ObservabilityService**

   - Tracks all API requests
   - Monitors token usage
   - Records accuracy feedback
   - Publishes to PostgreSQL, Pub/Sub, BigQuery

5. **TokenTrackerService**
   - Monitors Claude API token usage
   - Calculates costs ($3/M input, $15/M output)
   - Tracks per-tenant usage
   - Generates cost alerts

### Integration Layer

**External API Clients:**

| Service    | Purpose                | Method       | Rate Limit      |
| ---------- | ---------------------- | ------------ | --------------- |
| Shopify    | Fetch products, orders | OAuth 2.0    | 40 req/sec      |
| Meta Ads   | Campaign performance   | Access Token | 200 req/hour    |
| TikTok Ads | Campaign metrics       | OAuth 2.0    | 100 req/hour    |
| Google Ads | Ad performance         | OAuth 2.0    | 1000 req/day    |
| Claude     | AI analysis            | API Key      | Depends on plan |

### Data Layer

**PostgreSQL Database:**

- Multi-tenant with RLS
- ACID transactions
- Full-text search on brands
- 90-day event retention

**Redis Cache:**

- Session management
- Query result caching
- Rate limiting counters
- Tenant context cache (1-hour TTL)

**Google Cloud:**

- BigQuery: 7-year analytics retention
- Pub/Sub: Real-time event streaming
- Cloud Run: Serverless hosting
- Cloud Storage: Backup storage

---

## Data Flow

### Request Flow

```
1. Client Request (with JWT)
   ↓
2. CORS Middleware
   ↓
3. Authentication Middleware (verify JWT)
   ↓
4. TenantContext Middleware (extract tenant_id)
   ↓
5. Route Handler
   ├─ Check Redis cache
   ├─ Query database (RLS filters automatically)
   ├─ Call external APIs if needed
   ├─ Process with AI if needed
   └─ Store in cache
   ↓
6. ObservabilityMiddleware (record event)
   ├─ Store in PostgreSQL
   ├─ Publish to Pub/Sub
   └─ Stream to BigQuery
   ↓
7. Response to Client
```

### Async Job Flow

```
n8n Cron Trigger (Every Hour)
   ↓
n8n HTTP Request → Pulsepoint API
   ↓
Backend Job Handler
   ├─ ShopifyService.sync_products()
   ├─ MetaAdsService.fetch_metrics()
   ├─ TikTokAdsService.fetch_metrics()
   └─ GoogleAdsService.fetch_metrics()
   ↓
AttributionService.match_orders_to_campaigns()
   ↓
Store in PostgreSQL
   ↓
Trigger n8n Webhook (Data Available)
   ↓
n8n Workflow
   ├─ Generate AI Analysis
   ├─ Send Slack Alert
   └─ Store in BigQuery
```

### Event Tracking Flow

```
API Request/Response
   ↓
ObservabilityService.track_event()
   ↓
Create StructuredEvent (with tokens, duration, cost)
   ↓
MultiDestinationEventPublisher
   ├─ PostgreSQL (compliance storage)
   ├─ Pub/Sub (real-time alerts)
   └─ BigQuery (analytics)
```

---

## Technology Stack

### Backend

```
Language:           Python 3.11+
Web Framework:      FastAPI 0.104+
ASGI Server:        Uvicorn 0.24+
ORM:                SQLAlchemy 2.0+
Database Driver:    asyncpg (async PostgreSQL)
Database:           PostgreSQL 15+
Cache:              Redis 7+
Task Queue:         n8n (external orchestration)
```

### Integrations

```
Authentication:     JWT (PyJWT)
Encryption:         cryptography library
AI:                 Anthropic Claude API
Analytics:          Google BigQuery
Event Stream:       Google Pub/Sub
Cloud Hosting:      Google Cloud Run
Container:          Docker + Docker Compose
```

### Frontend

```
Framework:          React 18+
State Management:   Redux or Context API
Styling:            Tailwind CSS
HTTP Client:        Axios
```

### Testing

```
Unit Tests:         pytest + pytest-asyncio
Integration:        pytest with fixtures
Load Testing:       locust
```

### Development

```
Code Formatting:    black
Linting:           ruff
Type Checking:      mypy
Pre-commit Hooks:   pre-commit
Git Workflow:       GitHub + GitHub Actions
```

---

## Deployment Architecture

### Development Environment

```
Local Machine
├── Python venv
├── PostgreSQL (Docker)
├── Redis (Docker)
├── FastAPI (hot reload)
├── n8n (Docker)
└── Frontend (npm dev server)
```

### Production Environment (GCP)

```
GCP Project
├── Cloud Run
│   ├── Pulsepoint API (containerized)
│   ├── Scheduler (n8n alternative)
│   └── Background Jobs
│
├── Cloud SQL
│   └── PostgreSQL (managed, replicated)
│
├── Memorystore
│   └── Redis (managed)
│
├── BigQuery
│   ├── Observability events
│   ├── Analytics tables
│   └── Tenant-specific datasets
│
├── Pub/Sub
│   ├── Event topics
│   ├── Alert subscriptions
│   └── n8n webhooks
│
├── Cloud Storage
│   ├── Backups
│   └── Exports
│
└── Cloud Monitoring
    ├── Logs
    ├── Metrics
    └── Dashboards
```

### Docker Compose Stack (Local + Staging)

```
Services:
├── postgres:15-alpine
├── redis:7-alpine
├── pulsepoint-api (FastAPI)
├── n8n (Workflow engine)
└── nginx (reverse proxy)
```

---

## Multi-Tenancy Model

### Isolation Levels

**Level 1: JWT Context**

```
Token contains: {tenant_id, user_id, role}
Validated on every request
```

**Level 2: Application-Level**

```
TenantContextMiddleware
- Extracts tenant_id from JWT
- Sets as ContextVar for thread-safety
- Available in all request handlers
```

**Level 3: Database-Level (RLS)**

```
PostgreSQL Row-Level Security
- Policies enforce: tenant_id = current_setting('app.tenant_id')
- Cannot be bypassed by SQL injection
- Redundant with application-level checks
```

**Level 4: Cache-Level**

```
Redis keys include tenant_id
- Format: "data:{tenant_id}:{resource_id}"
- Prevents cross-tenant cache hits
```

### Tenant Data Storage

```
PostgreSQL (Primary)
├── Shared tables (tenants, users)
└── Tenant-isolated tables (brands, orders, events) with RLS

BigQuery (Analytics)
├── Per-tenant datasets: dataset_tenant_{tenant_id}
├── 7-year retention
└── Cross-dataset queries restricted by IAM

Redis (Cache)
├── Namespaced keys by tenant
└── 1-hour TTL
```

---

## Observability Stack

### Metrics Collected

**Performance Metrics:**

- Request duration (p50, p95, p99)
- API latency by endpoint
- Database query time
- Cache hit ratio

**Business Metrics:**

- Questions asked per tenant
- Recommendations generated
- Accuracy scores
- Cost per analysis

**System Metrics:**

- CPU & memory usage
- Database connection pool
- Redis memory usage
- API error rates

### Monitoring Dashboards

**Real-time Metrics:**

```
BigQuery Query → Google Data Studio Dashboard
- Daily costs by tenant
- API latency trends
- Accuracy improvements over time
- Revenue impact of recommendations
```

**Alerting:**

```
Pub/Sub Subscriber → Alert Handler
- High cost alerts (>$1000/day)
- Error rate alerts (>5%)
- Latency alerts (>5s)
- Accuracy drops (<80%)
```

### Log Aggregation

```
Application Logs
├── FastAPI structured logging
├── Service logs
├── Database query logs (slow queries)
└── Integration logs (API calls)

Centralized to:
└── Google Cloud Logging → Log Viewer
```

---

## Integration Points

### n8n Integration

**Webhook from n8n → Pulsepoint:**

```
POST /webhooks/n8n/data-sync
{
  "tenant_id": "uuid",
  "brand_id": "uuid",
  "sync_type": "shopify",
  "records_count": 100,
  "timestamp": "2024-01-14T10:00:00Z"
}
```

**Webhook from Pulsepoint → n8n:**

```
POST {N8N_WEBHOOK_URL}/recommendations-ready
{
  "tenant_id": "uuid",
  "recommendations": [...],
  "urgent": false
}
```

### Event Streaming

**Pub/Sub Topics:**

```
projects/{project}/topics/pulsepoint-events
├── Subscription: pulsepoint-alerts
├── Subscription: pulsepoint-analytics
└── Subscription: n8n-workflow-triggers
```

### BigQuery Integration

**Per-tenant datasets:**

```
pulsepoint-prod
├── dataset_tenant_abc123
│   ├── observability_events
│   ├── ai_token_usage
│   └── recommendations
├── dataset_tenant_def456
└── ...
```

---

## Security Architecture

### Authentication

```
User Login
  ↓
Generate JWT (HS256)
  ├─ sub: user_id
  ├─ tenant_id: tenant_id
  ├─ role: admin|manager|user
  └─ exp: expires in 24 hours
  ↓
Include in Authorization: Bearer header
  ↓
Verified on every request
```

### Encryption

```
API Keys (at rest)
├─ Encrypted in PostgreSQL
├─ Decrypted only when needed
└─ Rotated regularly

JWT Secrets
├─ Strong random values
├─ Different per environment
└─ Stored in Google Secret Manager
```

### Network Security

```
Cloud Run (TLS)
  ├─ Automatic HTTPS
  ├─ Certificate managed by GCP
  └─ Enforced TLS 1.2+

Database (Private)
  ├─ Private IP (no public access)
  ├─ IAM roles required
  └─ Encrypted backups

Redis (Private)
  ├─ Private IP
  ├─ IAM roles required
  └─ Encrypted in transit
```

---

## Performance Optimization

### Caching Strategy

```
Cache Hierarchy:
1. Redis (1-hour TTL)
   - Tenant summaries
   - Query results
   - API responses

2. BigQuery (incremental export)
   - Historical data
   - Analytics dashboards

3. PostgreSQL (indexes)
   - Composite indexes on (tenant_id, brand_id)
   - Full-text search on content
```

### Database Optimization

```
Connection Pooling:
- Min connections: 5
- Max connections: 20
- Idle timeout: 300s

Query Optimization:
- Prepared statements
- Batch inserts
- Pagination (limit 100)
- Indexed columns
```

### API Rate Limiting

```
Per-tenant limits:
- 60 requests/minute
- 1000 requests/hour
- 10000 requests/day

Enforced via Redis counters
with exponential backoff
```

---

## Disaster Recovery

### Backup Strategy

```
PostgreSQL:
- Automated daily backups (GCP Cloud SQL)
- Point-in-time recovery (7 days)
- Cross-region replicas

BigQuery:
- No delete (append-only)
- 7-year retention
- Cross-region tables

Redis:
- Session-only (non-critical)
- Recreated from source on failure
```

### Failover Procedures

```
Database Failover:
- Automatic via Cloud SQL
- Failover time: ~2 minutes
- Connection string unchanged

API Failover:
- Multiple Cloud Run instances
- Load balancer (Cloud Load Balancing)
- Automatic restart on failure
```

---

## Compliance & Audit

### Data Privacy

```
GDPR Compliance:
- Encryption at rest and in transit
- Data retention policies
- User deletion capabilities
- Audit logs

Data Residency:
- All data in US region
- No cross-border transfers (without consent)
- Regional backups
```

### Audit Logging

```
Tracked Events:
- User login/logout
- Data access (what, when, who)
- Configuration changes
- Admin actions

Stored in:
- PostgreSQL (90 days)
- BigQuery (7 years)
- Cloud Audit Logs (indefinite)
```

---

**Last Updated:** January 2026 | **Version:** 1.0
