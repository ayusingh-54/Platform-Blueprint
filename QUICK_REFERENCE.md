# Pulsepoint MCP: Quick Reference & Deployment Checklist

## Quick Start (5 minutes)

```bash
# 1. Clone and setup
git clone https://github.com/pulsepoint/mcp.git
cd pulsepoint
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 2. Configure environment
cp .env.example .env
# Edit .env with your API keys

# 3. Run locally
docker-compose up
open http://localhost:3000  # Frontend
open http://localhost:8000/docs  # API docs

# 4. Test
pytest tests/
```

---

## Multi-Tenancy Checklist

### ✅ Development

- [ ] Create `.env` with `SECRET_KEY` for JWT signing
- [ ] Run `migrations/setup_rls.sql` to enable RLS policies
- [ ] Test: Try accessing Brand A data as User B → Should get 403
- [ ] Verify: JWT token includes tenant_id
- [ ] Check: Context variables set properly in middleware

### ✅ Database

- [ ] PostgreSQL RLS enabled on all tables
- [ ] unique(tenant_id, email) constraint on users table
- [ ] All queries include `WHERE tenant_id = current_tenant_id`
- [ ] BigQuery dataset created per tenant (pulsepoint*tenant*{id})
- [ ] IAM policies grant access only to tenant's service account

### ✅ Testing

```bash
# Test 1: RLS Enforcement
curl -H "Authorization: Bearer USER_A_TOKEN" \
     https://api/brands/{BRAND_ID_B}
# Expected: 403 Forbidden

# Test 2: Tenant Isolation
SELECT * FROM brands WHERE tenant_id != current_setting('app.current_tenant_id')
# Expected: 0 rows

# Test 3: Cache Isolation
redis> KEYS tenant:*
# Expected: Only keys for current tenant
```

### ✅ Production

- [ ] All secrets in Google Secret Manager (not in code)
- [ ] RLS policies enforced at database level
- [ ] JWT secret rotated quarterly
- [ ] Audit logging enabled for admin operations
- [ ] Monitoring alert: Multiple tenants accessing same brand

---

## Observability Checklist

### ✅ Development

- [ ] ObservabilityService initialized in FastAPI
- [ ] Every API endpoint tracks request/response
- [ ] Every AI question logged with tokens
- [ ] Error tracking in place

### ✅ Queries Ready

```sql
-- Cost by tenant (daily)
SELECT tenant_id, DATE(recorded_at), SUM(cost_usd)
FROM ai_token_usage GROUP BY tenant_id, date;

-- Accuracy by model
SELECT model, AVG(accuracy_score)
FROM ai_questions JOIN ai_feedback
GROUP BY model;

-- Latency by endpoint
SELECT endpoint, PERCENTILE_CONT(duration_ms, 0.95)
FROM observability_events WHERE event_type = 'api.response'
GROUP BY endpoint;
```

### ✅ Dashboards Deployed

- [ ] Grafana dashboard for real-time metrics
- [ ] BigQuery saved queries for daily reports
- [ ] Alert rules configured in Stackdriver
- [ ] Email notifications working

### ✅ Cost Tracking

- [ ] Token usage tracked per tenant
- [ ] AI cost calculated and billed correctly
- [ ] Rate limiting enforced to prevent runaway costs
- [ ] Budget alerts set (e.g., > $500/month)

### ✅ Accuracy Scoring

- [ ] User feedback collection working
- [ ] Accuracy score calculated from feedback
- [ ] Trending tracked over time
- [ ] Low accuracy alerts (<70%) working

---

## Security Checklist

### ✅ Authentication

- [ ] JWT tokens have 8-hour expiration
- [ ] Secret key is truly random and rotated
- [ ] Token refresh logic working
- [ ] Expired token rejection working
- [ ] /health endpoint doesn't require auth

### ✅ Authorization

- [ ] RBAC roles defined (admin, manager, analyst)
- [ ] Permission checks on sensitive endpoints
- [ ] User can't escalate own permissions
- [ ] Admin audit log of permission changes

### ✅ Encryption

- [ ] OAuth tokens encrypted in database
- [ ] Encryption key in Secret Manager (never in code)
- [ ] TLS 1.3 enforced for all connections
- [ ] Database encryption at rest enabled

### ✅ Input Validation

- [ ] Pydantic models validate all inputs
- [ ] SQL injection tests passing
- [ ] XSS prevention in place
- [ ] Rate limiting enforced (100 req/min default)

### ✅ Secrets Management

- [ ] Zero secrets in .git
- [ ] .env in .gitignore
- [ ] All secrets in Secret Manager
- [ ] Rotation scheduled quarterly

---

## Deployment Checklist

### ✅ Local Development

```bash
docker-compose up
# Services running:
# - API: localhost:8000
# - Frontend: localhost:3000
# - PostgreSQL: localhost:5432
# - Redis: localhost:6379
# - BigQuery: Emulator (optional)
```

### ✅ Staging

```bash
gcloud run deploy pulsepoint-staging \
  --image gcr.io/PROJECT/pulsepoint:staging \
  --region us-central1 \
  --set-env-vars DATABASE_URL=$STAGING_DB_URL
```

### ✅ Production

```bash
# 1. Build image
gcloud builds submit --tag gcr.io/PROJECT/pulsepoint:latest

# 2. Deploy
gcloud run deploy pulsepoint \
  --image gcr.io/PROJECT/pulsepoint:latest \
  --region us-central1 \
  --min-instances 2 \
  --max-instances 100 \
  --memory 2Gi \
  --cpu 2 \
  --timeout 300 \
  --allow-unauthenticated

# 3. Configure load balancer
gcloud compute backend-services create pulsepoint-backend \
  --global \
  --protocol HTTP2

# 4. Setup monitoring
gcloud monitoring dashboards create --config-from-file monitoring/dashboards.yaml
```

### ✅ High Availability

- [ ] Multi-region failover configured
- [ ] Auto-scaling enabled (min=2, max=100)
- [ ] Database backups: daily + point-in-time
- [ ] Health checks: 30s interval
- [ ] Load balancer: round-robin across instances

### ✅ Monitoring in Production

- [ ] Uptime monitoring (target 99.9%)
- [ ] Alert: Error rate > 5%
- [ ] Alert: API latency p95 > 1000ms
- [ ] Alert: Token cost > $1000/day
- [ ] Alert: Data sync failure > 6 hours

---

## API Endpoint Testing

### Test: Authentication

```bash
# 1. Login
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"secret"}'

# Expected Response:
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}

# 2. Use token for API calls
export TOKEN="eyJhbGc..."

curl -H "Authorization: Bearer $TOKEN" \
     http://localhost:8000/api/brands

# Expected: List of brands for authenticated user's tenant
```

### Test: Ask Question

```bash
curl -X POST http://localhost:8000/api/brands/BRAND_ID/ask \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "Which products performed best last month?"
  }'

# Expected Response:
{
  "question_id": "q_123...",
  "answer": "Your top 3 products were...",
  "tokens_used": 1250,
  "generation_time_ms": 1420,
  "confidence_score": 0.95
}
```

### Test: Get Dashboard

```bash
curl -H "Authorization: Bearer $TOKEN" \
     http://localhost:8000/api/brands/BRAND_ID/dashboard

# Expected Response:
{
  "summary": {
    "summary": "Last week you spent...",
    "channel_performance": {...},
    "inventory_insights": "..."
  },
  "recommendations": [...],
  "performance_snapshot": {...}
}
```

### Test: Multi-Tenancy Isolation

```bash
# User A tries to access Brand B data (different tenant)
curl -H "Authorization: Bearer USER_A_TOKEN" \
     http://localhost:8000/api/brands/BRAND_ID_B

# Expected: 403 Forbidden
{
  "detail": "Access denied. Brand does not exist or belongs to another tenant."
}
```

---

## Troubleshooting Guide

### Issue: "Invalid token" Error

**Cause:** JWT secret mismatch or token expired

**Fix:**

```bash
# Verify SECRET_KEY in .env matches what's used to sign tokens
echo $SECRET_KEY

# Verify token expiration
python -c "
import jwt
token = 'your_token_here'
payload = jwt.decode(token, open('.env').read().split('=')[1].strip(), algorithms=['HS256'])
print(payload)
"
```

### Issue: "Access denied" for Brand User Should Access

**Cause:** Tenant context not set correctly

**Debug:**

```python
# In middleware, add logging
print(f"Tenant ID: {current_tenant_id.get()}")
print(f"User ID: {current_user_id.get()}")
print(f"JWT payload: {payload}")
```

### Issue: High Token Costs

**Cause:** Too many AI questions or inefficient prompts

**Analysis:**

```sql
SELECT operation, COUNT(*), AVG(total_tokens), SUM(total_cost_usd)
FROM ai_token_usage
WHERE DATE(recorded_at) = CURRENT_DATE()
GROUP BY operation
ORDER BY total_cost_usd DESC;
```

**Fix:**

- Add rate limiting on /api/ask endpoint
- Cache dashboard summaries longer
- Optimize prompts to use fewer tokens

### Issue: Slow API Response

**Cause:** Database query or BigQuery timeout

**Debug:**

```sql
-- Find slow queries
SELECT endpoint, duration_ms, COUNT(*)
FROM observability_events
WHERE event_type = 'api.response'
  AND duration_ms > 1000
GROUP BY endpoint, duration_ms
ORDER BY duration_ms DESC;
```

**Fix:**

- Add database indexes
- Implement query caching
- Increase Cloud Run memory/CPU

### Issue: Data Not Syncing from Shopify

**Cause:** OAuth token expired or API error

**Check:**

```sql
SELECT brand_id, platform, last_synced_at, sync_error_message
FROM integrations
WHERE brand_id = 'BRAND_ID'
  AND last_synced_at < NOW() - INTERVAL '1 hour';
```

**Fix:**

```python
# Manually retry sync
from backend.jobs.sync_shopify import sync_shopify_data
sync_shopify_data.delay(brand_id)

# Check token expiration
SELECT token_expires_at FROM integrations WHERE brand_id = 'BRAND_ID'

# If expired, trigger OAuth refresh
refresh_integration_token(brand_id, 'shopify')
```

---

## Monitoring Queries

### Daily Cost Report

```sql
SELECT
  tenant_id,
  DATE(recorded_at) as date,
  SUM(total_tokens) as tokens,
  SUM(total_cost_usd) as cost,
  COUNT(*) as num_operations
FROM ai_token_usage
WHERE DATE(recorded_at) >= CURRENT_DATE() - 30
GROUP BY tenant_id, date
ORDER BY date DESC, cost DESC
LIMIT 1000
```

### Performance Report

```sql
SELECT
  endpoint,
  COUNT(*) as requests,
  AVG(duration_ms) as avg_latency,
  PERCENTILE_CONT(duration_ms, 0.95) OVER () as p95_latency,
  PERCENTILE_CONT(duration_ms, 0.99) OVER () as p99_latency,
  COUNTIF(status='failure') as errors
FROM observability_events
WHERE event_type = 'api.response'
  AND DATE(timestamp) = CURRENT_DATE()
GROUP BY endpoint
ORDER BY p99_latency DESC
```

### Accuracy Report

```sql
SELECT
  model,
  COUNT(*) as questions,
  AVG(accuracy_score) as avg_accuracy,
  COUNTIF(accuracy_score >= 4)/COUNT(*) as satisfaction_rate,
  COUNTIF(was_actionable)/COUNT(*) as actionability_rate
FROM ai_questions q
JOIN ai_feedback f ON q.question_id = f.question_id
WHERE DATE(q.asked_at) >= CURRENT_DATE() - 30
GROUP BY model
ORDER BY avg_accuracy DESC
```

---

## Key Metrics to Track

| Metric                | Target     | Check Frequency |
| --------------------- | ---------- | --------------- |
| API Availability      | 99.9%      | Daily           |
| Dashboard Latency p95 | < 500ms    | Real-time       |
| Question Response p95 | < 3000ms   | Real-time       |
| AI Accuracy Score     | > 85%      | Daily           |
| Data Freshness        | < 1 hour   | Real-time       |
| Token Cost / Tenant   | < $100/day | Daily           |
| Error Rate            | < 0.5%     | Real-time       |
| Failed Syncs          | 0          | Hourly          |

---

## Quick Commands

```bash
# View logs
gcloud run logs read pulsepoint --limit 100

# SSH into Cloud Run instance (debug)
gcloud compute ssh INSTANCE --zone us-central1-a

# Scale manually
gcloud run services update-traffic pulsepoint --to-revisions LATEST=100

# View metrics
gcloud monitoring time-series list \
  --filter='resource.type="cloud_run_revision"'

# Execute database query
psql $DATABASE_URL -c "SELECT * FROM brands LIMIT 10;"

# Backup database
gcloud sql backups create --instance=pulsepoint-db

# View BigQuery storage
bq show --format=prettyjson pulsepoint_tenant_123

# Restart service
gcloud run services delete pulsepoint && gcloud run deploy pulsepoint ...
```

---

## Production Readiness Scorecard

| Component         | Status | Evidence                                         |
| ----------------- | ------ | ------------------------------------------------ |
| Multi-Tenancy     | ✅     | RLS policies enforced, tested isolation          |
| Observability     | ✅     | Events tracked in 3 systems, dashboards deployed |
| Security          | ✅     | JWT auth, encryption, rate limiting              |
| High Availability | ✅     | Auto-scaling, backups, failover                  |
| Monitoring        | ✅     | Real-time dashboards + alerts                    |
| Documentation     | ✅     | This document + code comments                    |
| Testing           | ✅     | Unit + integration + e2e tests                   |
| Deployment        | ✅     | Terraform / Docker / Cloud Run                   |

---

## Support

- 📧 **Email:** engineering@pulsepoint.com
- 💬 **Discord:** https://discord.gg/pulsepoint
- 🐛 **Issues:** https://github.com/pulsepoint/mcp/issues
- 📖 **Docs:** https://docs.pulsepoint.com
