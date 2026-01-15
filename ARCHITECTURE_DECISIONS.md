# Pulsepoint: Key Design Decisions & Architectural Patterns

## Executive Summary

This document outlines the critical architectural decisions made for Pulsepoint MCP server, with emphasis on **production-readiness**, **multi-tenancy**, and **observability**.

---

## 1. Multi-Tenancy Approach

### Decision: Hybrid Isolation Strategy

We implement **4-layer tenant isolation** for maximum security and flexibility:

| Layer        | Implementation                      | Guarantee                                               |
| ------------ | ----------------------------------- | ------------------------------------------------------- |
| **Database** | PostgreSQL Row-Level Security (RLS) | Queries automatically filtered by tenant_id             |
| **Schema**   | Optional schema-per-tenant (hybrid) | Can upgrade to extreme isolation if needed              |
| **BigQuery** | Separate dataset per tenant         | IAM controls access at dataset level                    |
| **Cache**    | Namespace all keys with tenant_id   | Cache::get(key, tenant_id) prevents cross-contamination |

### Why This Approach?

**Pros:**

- ✅ Single application server (cheaper, simpler operations)
- ✅ Shared infrastructure reduces DevOps complexity
- ✅ Easy to add schema isolation later (zero code changes)
- ✅ RLS provides guaranteed database-level protection (even if app code has bugs)
- ✅ Scales to thousands of tenants

**Cons:**

- ⚠️ Data breach could expose all tenants (mitigated by encryption at rest)
- ⚠️ Noisy neighbor problem if one tenant uses all resources

### Alternative Considered

**Full Isolation:** Separate Cloud Run instance, database, BigQuery dataset per tenant

- **Why rejected:** Costs ∝ number of tenants, operational overhead increases exponentially

### Verification Strategy

```python
# Automatic tenant enforcement in every query
async def get_brand(brand_id: str):
    tenant_id = get_current_tenant_id()  # From JWT context

    # This will return NULL if brand doesn't belong to tenant
    # because RLS policy prevents access
    brand = await db.query(Brand).filter(
        Brand.brand_id == brand_id,
        Brand.tenant_id == tenant_id  # Double-check at app level
    ).first()

    if not brand:
        raise HTTPException(403, "Access denied")

    return brand
```

**Defense in depth:** Both RLS AND app-level checks.

---

## 2. Observability Architecture

### Decision: Event-Driven Observability

Every observable action generates an event published to **3 systems simultaneously**:

```
┌─────────────┐
│ AI Question │
└──────┬──────┘
       │
       ├─→ PostgreSQL (local storage)
       │   └─ Insert into observability_events
       │   └─ Used for: debugging, user queries
       │
       ├─→ Pub/Sub (real-time alerts)
       │   └─ Subscribed by: alert service
       │   └─ Latency: < 1 second
       │   └─ Used for: alerting on anomalies
       │
       └─→ BigQuery (historical analytics)
           └─ Batch inserts every 10s
           └─ Used for: dashboards, trend analysis
```

### Why This Approach?

**Observability Layers:**

1. **PostgreSQL** (transactional database)

   - Source of truth for events
   - Can query immediately after event
   - Used for: "Show me last 10 questions"
   - Retention: 90 days

2. **Pub/Sub** (event stream)

   - Real-time processing
   - Triggers alerts immediately
   - Example: Alert if token spend > $100/day
   - Latency: 1-2 seconds

3. **BigQuery** (analytics)
   - Unlimited scale (100M events/day)
   - SQL queries across all data
   - Example: "Average accuracy by model over 90 days"
   - Retention: 7 years (for compliance)

### What We Track

#### Every API Request

```python
{
    "request_id": "req_123",
    "endpoint": "/api/brands/{id}/dashboard",
    "method": "GET",
    "status_code": 200,
    "duration_ms": 450,
    "tenant_id": "t_456",
    "user_id": "u_789"
}
```

#### Every AI Question

```python
{
    "question_id": "q_123",
    "question": "Which products performed best?",
    "tenant_id": "t_456",
    "user_id": "u_789",
    "model": "claude-3-sonnet",
    "input_tokens": 5000,
    "output_tokens": 1200,
    "generation_time_ms": 1420,
    "cost_usd": 0.0234
}
```

#### User Feedback → Accuracy Scoring

```python
{
    "question_id": "q_123",
    "user_rating": 5,  # 1-5 stars
    "was_actionable": true,
    "accuracy_comment": "Perfect, exactly what I needed"
}
# Aggregated: avg_accuracy = 4.8/5 = 96%
```

#### Real-Time Alerts

```python
# When metric exceeds threshold:
if daily_token_spend > $1000:
    send_alert(
        severity="warning",
        message=f"Tenant {tenant_id} spent ${daily_token_spend} on tokens today"
    )
```

### Queries Enabled

**1. Cost Analysis**

```sql
SELECT
    tenant_id,
    DATE(recorded_at) as date,
    SUM(tokens_used) as tokens,
    SUM(cost_usd) as cost
FROM observability_events
WHERE event_type = 'ai.answer_generated'
GROUP BY tenant_id, date
ORDER BY cost DESC;
```

**Result:** "Tenant X spent $15 today on AI"

**2. Accuracy Trending**

```sql
SELECT
    model,
    DATE(asked_at) as date,
    AVG(accuracy_score) as avg_accuracy
FROM ai_questions q
JOIN ai_feedback f ON q.question_id = f.question_id
GROUP BY model, date
ORDER BY date DESC;
```

**Result:** "Claude accuracy improved from 0.85 to 0.92 over past week"

**3. Latency Analysis**

```sql
SELECT
    endpoint,
    PERCENTILE_CONT(duration_ms, 0.95) as p95_latency,
    PERCENTILE_CONT(duration_ms, 0.99) as p99_latency
FROM observability_events
WHERE event_type = 'api.response'
GROUP BY endpoint;
```

**Result:** "Dashboard endpoint has p95 latency of 850ms"

---

## 3. Authentication & Authorization

### Decision: JWT with Tenant Context

Every API request includes a JWT token with embedded tenant/user context:

```python
# JWT Payload
{
    "tenant_id": "t_550e8400-e29b-41d4-a716",
    "user_id": "u_f47ac10b-58cc-4372-a567",
    "brand_id": "b_550e8400-e29b-41d4-a716",  # Optional, current brand
    "role": "manager",
    "email": "alice@acme.com",
    "exp": 1705334400,  # Expires in 8 hours
    "iat": 1705305600
}
```

### JWT Flow

```
1. User logs in
   POST /api/auth/login
   → Verify password hash
   → Generate JWT with tenant context
   → Return JWT token

2. User makes API call
   GET /api/brands/{id}/dashboard
   Authorization: Bearer JWT_TOKEN

3. MCP server validates JWT
   → Check signature (SECRET_KEY)
   → Check expiration
   → Extract tenant_id from payload
   → Set context variables

4. Request proceeds with tenant context
   All queries automatically filtered by tenant_id
```

### Role-Based Access Control (RBAC)

```python
ROLE_PERMISSIONS = {
    Role.ADMIN: [
        Permission.BRAND_READ,
        Permission.BRAND_WRITE,
        Permission.USER_MANAGE,
        Permission.BILLING_VIEW,
        Permission.ADMIN_SETTINGS
    ],
    Role.MANAGER: [
        Permission.BRAND_READ,
        Permission.BRAND_WRITE,
        Permission.DATA_READ,
        Permission.INTEGRATION_CONNECT
    ],
    Role.ANALYST: [
        Permission.BRAND_READ,
        Permission.DATA_READ,
        Permission.DATA_EXPORT
    ]
}
```

### Why JWT + RBAC?

**Pros:**

- ✅ Stateless (no session storage required)
- ✅ Scales horizontally (any server can validate)
- ✅ Tenant context embedded (no extra DB query needed)
- ✅ Granular permissions (can enforce at endpoint level)
- ✅ Short-lived tokens (8 hour expiration limits damage if leaked)

**Cons:**

- ⚠️ Token revocation requires additional mechanism (Redis blacklist)
- ⚠️ If leaked, attacker has access until expiration

### Token Refresh Strategy

```python
# Long-lived Refresh Token (30 days)
refresh_token = create_token(user_id, expires_in=30*24*3600)

# Short-lived Access Token (8 hours)
access_token = create_token(user_id, expires_in=8*3600)

# Client flow:
# 1. Get access_token + refresh_token at login
# 2. Use access_token for API calls (8 hours)
# 3. When access_token expires → use refresh_token to get new access_token
# 4. Keep using same refresh_token for 30 days
```

---

## 4. Data Flow & Async Processing

### Decision: Async-First Architecture

Critical insight: **Not all data needs real-time updates**

```
Synchronous (Real-time):
├─ User authentication
├─ Dashboard queries (cached)
├─ AI question answering
└─ API responses (< 500ms target)

Asynchronous (Background):
├─ Data syncing (Shopify, Meta, TikTok, Google) → Every hour
├─ BigQuery data loading → Every 10 minutes
├─ AI insight generation → Daily
├─ Observability event batching → Every 10 seconds
└─ Email notifications → On-demand
```

### Celery Job Architecture

```python
# Background job: Sync Shopify data
@celery_app.task(bind=True, max_retries=3)
def sync_shopify_data(self, brand_id: str):
    """
    Async job runs without blocking API.
    If it fails, Celery retries automatically.
    """
    try:
        shopify_service = ShopifyService()

        # Fetch data
        products = shopify_service.sync_products(brand_id)
        orders = shopify_service.sync_orders(brand_id)
        inventory = shopify_service.sync_inventory(brand_id)

        # Load to databases
        load_to_postgres(products, orders, inventory)
        load_to_bigquery(products, orders, inventory)

        # Update status
        mark_integration_synced(brand_id, "shopify")

    except Exception as exc:
        # Retry with exponential backoff
        self.retry(exc=exc, countdown=60 * (2 ** self.request.retries))

# Scheduled: Run every hour
schedule = {
    'sync-shopify': {
        'task': 'backend.jobs.sync_shopify_data',
        'schedule': crontab(minute=0),  # Every hour
    }
}
```

### Why Async?

**Reason 1: Data freshness vs. responsiveness**

- User doesn't need data updated every second
- Syncing Shopify takes 5-30 seconds
- If synchronous, user would wait 30s for page load
- If async, user sees results immediately (cached) + data updates in background

**Reason 2: Fault tolerance**

- If Shopify API is down, background job fails
- Celery retries automatically (exponential backoff)
- API stays responsive (user still sees last known data)

**Reason 3: Cost optimization**

- Don't waste expensive AI API calls on every page load
- Generate summaries once per day, cache results
- Save 80% on AI costs

---

## 5. Caching Strategy

### Decision: Multi-Level Cache

```
┌──────────────────────────────────────┐
│ User Request                         │
└────────────┬─────────────────────────┘
             │
             ▼
    ┌─────────────────┐
    │ Level 1: Browser│ (client-side, handled by frontend)
    │ TTL: 5 minutes  │
    └────────┬────────┘
             │ (if miss)
             ▼
    ┌────────────────────┐
    │ Level 2: Redis     │
    │ TTL: 1 hour        │
    │ Key: tenant:brand  │
    └────────┬───────────┘
             │ (if miss)
             ▼
    ┌──────────────────────┐
    │ Level 3: PostgreSQL  │ (regenerate or compute)
    │ TTL: persistent      │
    └──────────────────────┘
```

### What Gets Cached?

```python
CACHE_STRATEGY = {
    "dashboard_summary": {
        "ttl": 3600,  # 1 hour
        "invalidate_on": ["data_sync_completed"]
    },
    "alerts": {
        "ttl": 300,  # 5 minutes (more frequent)
        "invalidate_on": ["alert_detected"]
    },
    "recommendations": {
        "ttl": 3600,  # 1 hour
        "invalidate_on": ["data_updated"]
    },
    "user_preferences": {
        "ttl": 86400,  # 1 day
        "invalidate_on": ["user_settings_changed"]
    }
}
```

### Cache Invalidation Pattern

```python
async def invalidate_tenant_cache(tenant_id: str, pattern: str = None):
    """Clear cache when data changes"""

    if pattern:
        # Invalidate specific pattern
        cache_key = f"tenant:{tenant_id}:{pattern}"
        await redis.delete(cache_key)
    else:
        # Invalidate all for tenant
        keys = await redis.keys(f"tenant:{tenant_id}:*")
        if keys:
            await redis.delete(*keys)

# Usage in Shopify sync job
async def sync_shopify_data(brand_id: str):
    # ... sync logic ...

    # Invalidate cache after sync
    tenant_id = await get_brand_tenant(brand_id)
    await invalidate_tenant_cache(tenant_id, "*dashboard*")
    await invalidate_tenant_cache(tenant_id, "*recommendations*")
```

---

## 6. Error Handling & Resilience

### Decision: Graceful Degradation

System should never crash, should degrade gracefully:

```
Scenario 1: Shopify API down
├─ Background job fails
├─ Retry exponentially (1min, 2min, 4min, ...)
├─ User still sees: cached data from last sync
├─ Alert: "Data is 3 hours old"
└─ Dashboard still works

Scenario 2: Claude API overloaded
├─ Timeout error (30 seconds)
├─ Return cached summary from yesterday
├─ User sees: summary + "Generated 1 day ago"
└─ Dashboard still works

Scenario 3: BigQuery unavailable
├─ Fall back to PostgreSQL data
├─ Lose advanced analytics, but reports still work
└─ Dashboard partially works

Scenario 4: Database connection lost
├─ Health check fails
├─ Cloud Run marks instance as unhealthy
├─ Load balancer routes traffic to healthy instance
└─ System recovers automatically
```

### Implementation Pattern

```python
async def get_dashboard(brand_id: str):
    try:
        # Try fresh
        dashboard = await generate_fresh_dashboard(brand_id)
        return dashboard

    except TimeoutError:
        # Fall back to cache
        logger.warning(f"Dashboard generation timed out, using cache")
        cached = await get_cached_dashboard(brand_id)

        if cached:
            # Add warning to response
            cached["warning"] = "Dashboard generated 1 hour ago"
            return cached
        else:
            raise HTTPException(503, "Service temporarily unavailable")

    except Exception as e:
        logger.error(f"Dashboard error: {e}")
        raise HTTPException(500, "Internal server error")
```

---

## 7. Scalability Strategy

### Horizontal Scaling

```
┌─────────────────────────────┐
│   Load Balancer (Cloud LB)  │ (auto-scales)
└──────────┬──────────────────┘
           │
    ┌──────┼──────┐
    │      │      │
    ▼      ▼      ▼
   Pod1   Pod2   Pod3  (Cloud Run instances)
   (API)  (API)  (API)
    │      │      │
    └──────┼──────┘
           │
        ┌──┴──┐
        │     │
        ▼     ▼
    PostgreSQL  BigQuery
    (RDS)       (Managed)
```

**Auto-scaling Rules:**

- Min instances: 2 (always-on, fast response)
- Max instances: 100 (sudden traffic spike)
- Scale up if: CPU > 70% OR requests > 100/s
- Scale down if: CPU < 30% for 5 minutes

### Database Scaling

```
PostgreSQL:
├─ Tier: db-custom-4-16GB (16 CPU, 64GB RAM)
├─ Replication: Multi-region for HA
├─ Backups: Daily + point-in-time recovery
└─ Capacity: Supports 10,000 req/s

BigQuery:
├─ Auto-scaling queries (no manual tuning)
├─ 100M rows/second scan speed
├─ Unlimited storage (on-demand pricing)
└─ Capacity: Supports millions of analytics queries
```

---

## 8. Security Hardening

### Defense in Depth

```
Layer 1: Network
├─ HTTPS/TLS 1.3 enforced
├─ VPC endpoint for databases (no internet exposure)
└─ Firewall rules: whitelist only needed services

Layer 2: Authentication
├─ JWT with secret key
├─ 8-hour token expiration
└─ Token revocation via Redis blacklist

Layer 3: Authorization
├─ RBAC (role-based permissions)
├─ Row-Level Security (tenant isolation)
└─ API-level permission checks

Layer 4: Data
├─ Encryption at rest (database + backups)
├─ Encryption in transit (TLS)
├─ PII anonymization in logs
└─ Secrets in Secret Manager (never in code)

Layer 5: Application
├─ Input validation (Pydantic models)
├─ SQL injection prevention (parameterized queries)
├─ Rate limiting (prevent brute force)
└─ CORS: whitelist specific origins only
```

### Token Encryption

```python
class Integration(Base):
    # OAuth tokens encrypted in database
    access_token_encrypted: str  # "gAAAAABl8X1q2_j..."
    refresh_token_encrypted: str

    def save_tokens(self, access_token: str, refresh_token: str):
        """Encrypt before storing"""
        cipher = Fernet(ENCRYPTION_KEY)
        self.access_token_encrypted = cipher.encrypt(access_token.encode())
        self.refresh_token_encrypted = cipher.encrypt(refresh_token.encode())

    def get_access_token(self) -> str:
        """Decrypt when retrieving"""
        cipher = Fernet(ENCRYPTION_KEY)
        decrypted = cipher.decrypt(self.access_token_encrypted.encode())
        return decrypted.decode()
```

---

## 9. Monitoring & SLA

### SLA Targets

```
API Availability:     99.9% (< 43 minutes downtime/month)
Dashboard Response:   p95 < 500ms
Question Response:    p95 < 3000ms (AI latency)
Data Freshness:       < 1 hour (last sync)
Accuracy Score:       > 85% (user ratings)
```

### Monitoring Metrics

```
Real-time Dashboards:
├─ Requests/second (traffic)
├─ Error rate % (health)
├─ API latency p95/p99 (performance)
├─ Token usage/day (cost)
└─ Data sync status (freshness)

Daily Reports:
├─ Questions asked (usage)
├─ Accuracy trending (quality)
├─ Cost per tenant (billing)
├─ Failed integrations (support)
└─ Security incidents (compliance)
```

### Alert Thresholds

```yaml
Alerts:
  - Error Rate > 5%: page on-call engineer
  - API Latency p95 > 1000ms: send warning
  - Token cost > $1000/day: notify finance
  - Data sync failure > 6 hours: escalate to support
  - Accuracy score < 70%: investigate model
```

---

## 10. Compliance & Data Governance

### Data Retention Policy

```
PostgreSQL (operational data):
├─ Retention: 90 days (real-time queries)
├─ After 90 days: Archive to BigQuery
└─ Reason: Limit database size, cost

BigQuery (analytics data):
├─ Retention: 7 years (regulatory requirement)
├─ Partition by date (efficient deletion)
└─ Reason: GDPR/CCPA audit trail

Logs (system events):
├─ Retention: 30 days hot (Stackdriver)
├─ Archive to Cloud Storage after 30 days
└─ Reason: Compliance, cost optimization
```

### GDPR Compliance

```
Right to Access:
├─ User can export all personal data via API
└─ Endpoint: GET /api/users/me/export

Right to Deletion:
├─ Cascade delete: user → questions → feedback
├─ Anonymize in logs (remove user_id)
└─ BigQuery: POLICY TABLE (compliance audit)

Data Portability:
├─ Export in standard formats (CSV, JSON)
└─ No proprietary formats
```

---

## Summary Table

| Decision      | Choice                           | Rationale                         |
| ------------- | -------------------------------- | --------------------------------- |
| Multi-Tenancy | Hybrid (RLS + optional schema)   | Balance cost with security        |
| Observability | Event-driven (3 systems)         | Real-time alerts + analytics      |
| Auth          | JWT + RBAC                       | Stateless, scalable               |
| Data Sync     | Async (Celery)                   | Don't block API, retry on failure |
| Caching       | Multi-level (Redis + PostgreSQL) | Fast response, fault-tolerant     |
| Errors        | Graceful degradation             | User always has something to see  |
| Scaling       | Horizontal (Cloud Run)           | Auto-scale based on demand        |
| Security      | Defense in depth                 | Multiple layers catch breaches    |
| Monitoring    | Real-time + scheduled reports    | Proactive incident detection      |
| Compliance    | GDPR + retention policies        | Legal + audit trail               |

---

## Questions to Consider

1. **What if a tenant tries to access another tenant's data?**

   - Answer: RLS policy at database level prevents it. API-level check is defense in depth.

2. **How do we prevent a noisy neighbor from consuming all resources?**

   - Answer: Per-tenant rate limits + quota system. Can upgrade to separate instances if needed.

3. **What happens if the AI API goes down?**

   - Answer: Return cached summary from yesterday. User sees "Generated 1 day ago" warning.

4. **How do we monitor accuracy over time?**

   - Answer: Track user feedback (1-5 stars) on every answer. Aggregate by model in BigQuery.

5. **How much does this cost to run?**
   - Answer: ~$2-3k/month for small scale (< 100 tenants). Scales linearly with usage due to pay-as-you-go pricing.
