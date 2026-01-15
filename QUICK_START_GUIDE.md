# Pulsepoint Quick Start Guide

Get Pulsepoint running in 30 minutes with this accelerated setup guide.

---

## Prerequisites (5 minutes)

Ensure you have installed:

- Python 3.11+ (`python --version`)
- PostgreSQL 15+ (`psql --version`)
- Redis 7+ (`redis-cli --version`)
- Docker & Docker Compose (`docker --version`)
- Git (`git --version`)

---

## Local Development Setup (25 minutes)

### Step 1: Clone & Setup (3 minutes)

```bash
# Clone or create project directory
cd ~/projects
mkdir pulsepoint && cd pulsepoint

# Initialize git (if new project)
git init

# Create Python virtual environment
python -m venv venv
source venv/bin/activate  # macOS/Linux
# OR: venv\Scripts\activate  # Windows PowerShell

# Verify Python
python --version
```

### Step 2: Install Dependencies (2 minutes)

```bash
# Upgrade pip
pip install --upgrade pip

# Create requirements.txt (use from README.md or IMPLEMENTATION_GUIDE.md)
# Then install
pip install -r requirements.txt

# Verify installations
python -c "import fastapi; import sqlalchemy; print('✓ Dependencies OK')"
```

### Step 3: Database Setup (5 minutes)

```bash
# Start PostgreSQL (if using Docker)
docker run --name postgres-pulsepoint \
  -e POSTGRES_USER=pulsepoint \
  -e POSTGRES_PASSWORD=password123 \
  -e POSTGRES_DB=pulsepoint \
  -p 5432:5432 \
  -d postgres:15

# OR start Redis with Docker
docker run --name redis-pulsepoint \
  -p 6379:6379 \
  -d redis:7

# Create and configure database
createdb pulsepoint
createuser pulsepoint

# Enable PostgreSQL extensions
psql pulsepoint << 'EOF'
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
EOF
```

### Step 4: Environment Configuration (2 minutes)

```bash
# Create .env file
cat > .env << 'EOF'
DATABASE_URL=postgresql+asyncpg://pulsepoint:password123@localhost:5432/pulsepoint
REDIS_URL=redis://localhost:6379/0

# Add your API keys here
ANTHROPIC_API_KEY=sk-ant-...
SHOPIFY_API_KEY=...
META_ACCESS_TOKEN=...

# JWT
JWT_SECRET=your-secret-change-in-production
JWT_ALGORITHM=HS256

SERVER_HOST=0.0.0.0
SERVER_PORT=8000
ENVIRONMENT=development
LOG_LEVEL=INFO
EOF

# IMPORTANT: Add your actual API keys to .env
```

### Step 5: Initialize Database (5 minutes)

```bash
# Copy and run initial migration (or use Alembic)
python scripts/setup_db.py
# This creates tables and RLS policies
```

### Step 6: Start the Server (3 minutes)

```bash
# Terminal 1: Start API
python -m uvicorn backend.api.main:app --reload

# Terminal 2: Start n8n (optional)
docker run -p 5678:5678 n8nio/n8n

# Terminal 3: Start Redis (if needed)
redis-server
```

### Step 7: Verify Everything Works (3 minutes)

```bash
# Test API
curl http://localhost:8000/health
# Expected: {"status":"healthy","service":"pulsepoint-mcp","version":"1.0.0"}

# Access Swagger UI
# Open: http://localhost:8000/docs

# Access n8n
# Open: http://localhost:5678
```

---

## Your First API Call (5 minutes)

### 1. Create a Tenant & User

```bash
# Create JWT token (in Python)
python << 'EOF'
import jwt
import os
from datetime import datetime, timedelta

tenant_id = "550e8400-e29b-41d4-a716-446655440000"
user_id = "650e8400-e29b-41d4-a716-446655440001"

payload = {
    "sub": user_id,
    "tenant_id": tenant_id,
    "role": "admin",
    "exp": datetime.utcnow() + timedelta(hours=24)
}

token = jwt.encode(
    payload,
    "your-secret-change-in-production",
    algorithm="HS256"
)

print(f"Token: {token}")
EOF
```

### 2. Make an API Request

```bash
# Use the token from above
TOKEN="eyJhbGc..."

# Ask a question
curl -X POST http://localhost:8000/api/questions \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "What is my total ad spend this month?",
    "brand_id": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

### 3. View Response

You'll get:

```json
{
  "answer": "Your total ad spend this month is $4,250.00 across all platforms...",
  "tokens_used": 512,
  "cost_usd": 0.02,
  "sources": ["shopify", "meta", "tiktok"],
  "confidence": 0.95
}
```

---

## n8n Quick Setup (10 minutes)

### 1. Start n8n

```bash
docker run -it -p 5678:5678 \
  -e N8N_HOST=localhost \
  -e N8N_PROTOCOL=http \
  n8nio/n8n
```

Open: http://localhost:5678

### 2. Create First Workflow

In n8n UI:

1. Click "New Workflow"
2. Add **Cron Trigger** → Set to "Every 1 hour"
3. Add **HTTP Request** node
   - URL: `http://localhost:8000/api/sync`
   - Method: POST
   - Headers: Add Bearer token
4. Add **Slack** node (optional for alerts)
5. Save & Activate

### 3. Webhook Integration

```bash
# Create webhook in n8n to receive Pulsepoint events
# In API route:
@app.post("/webhooks/n8n/sync-complete")
async def n8n_webhook(request: Request):
    payload = await request.json()
    # Process sync results
    return {"status": "processed"}
```

---

## Common Tasks

### Create a Brand

```bash
curl -X POST http://localhost:8000/api/brands \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Ecommerce Store",
    "shopify_store_url": "mystore.myshopify.com",
    "description": "Online storefront"
  }'
```

### Get Brand Analytics

```bash
curl http://localhost:8000/api/brands/brand-id/analytics \
  -H "Authorization: Bearer $TOKEN"
```

### Generate Recommendations

```bash
curl -X POST http://localhost:8000/api/recommendations \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "brand_id": "brand-id",
    "analysis_type": "ad_spend_optimization"
  }'
```

---

## Troubleshooting

### "Connection refused" on port 8000

```bash
# Check if process running
lsof -i :8000
# Kill if needed
kill -9 <PID>
```

### "PostgreSQL connection error"

```bash
# Check PostgreSQL is running
psql -U pulsepoint -d pulsepoint -c "SELECT 1"
# Restart if needed
docker restart postgres-pulsepoint
```

### "JWT token invalid"

```bash
# Make sure JWT_SECRET in .env matches the secret used to create token
# Regenerate token with correct secret
```

### "n8n can't reach Pulsepoint"

```bash
# In n8n, use: http://host.docker.internal:8000
# Instead of: http://localhost:8000
```

### "Tenant context not found"

```bash
# Verify Authorization header is set: Bearer <token>
# Token must contain tenant_id claim
```

---

## What's Next?

1. **Read the docs**: Check [README.md](README.md) for full feature overview
2. **Detailed implementation**: See [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md) for complete code
3. **Multi-tenancy details**: Read [DETAILED_APPROACH.md](DETAILED_APPROACH.md)
4. **Step-by-step guide**: Follow [COMPLETE_DEVELOPMENT_GUIDE.md](COMPLETE_DEVELOPMENT_GUIDE.md)
5. **API Reference**: Check `/docs` in running API

---

## Development Tips

### Hot Reload

```bash
# FastAPI auto-reloads on file changes
python -m uvicorn backend.api.main:app --reload
```

### Database Debugging

```bash
# Connect to PostgreSQL
psql pulsepoint

# View all tables
\dt

# Check RLS policies
\dp brands

# Query data (set tenant context first)
SET app.tenant_id = '550e8400-e29b-41d4-a716-446655440000';
SELECT * FROM brands;
```

### View Logs

```bash
# Terminal logs show request details
# Search for ERROR or WARNING entries
# Set LOG_LEVEL=DEBUG for verbose output

# Change in .env:
LOG_LEVEL=DEBUG
```

### Test with Swagger UI

1. Open http://localhost:8000/docs
2. Click "Authorize" button
3. Paste your JWT token
4. Try out endpoints directly in UI

---

## Performance Tips

1. **Use connection pooling**: Set `pool_size=20` in SQLAlchemy
2. **Enable Redis caching**: Set cache TTLs appropriately
3. **Batch API calls**: Use bulk operations when possible
4. **Monitor queries**: Enable SQL query logging in dev

```python
# In alembic env.py
engine = create_async_engine(
    database_url,
    echo=False,  # Set to True to see SQL queries
    pool_size=20,
    max_overflow=10
)
```

---

## Production Checklist

Before deploying:

- [ ] Change JWT_SECRET to strong random value
- [ ] Enable HTTPS/TLS
- [ ] Set up CORS correctly
- [ ] Enable rate limiting
- [ ] Configure proper logging
- [ ] Setup monitoring and alerts
- [ ] Backup PostgreSQL regularly
- [ ] Test disaster recovery
- [ ] Security audit completed
- [ ] Load testing passed

---

## Getting Help

- **Error messages**: Check logs first (`docker logs pulsepoint-api`)
- **API issues**: Test endpoints in Swagger UI at /docs
- **Database problems**: Query logs in PostgreSQL
- **n8n workflows**: Check execution logs in n8n UI

---

## Quick Reference

| Component  | URL                        | Port |
| ---------- | -------------------------- | ---- |
| FastAPI    | http://localhost:8000      | 8000 |
| Swagger UI | http://localhost:8000/docs | -    |
| PostgreSQL | localhost                  | 5432 |
| Redis      | localhost                  | 6379 |
| n8n        | http://localhost:5678      | 5678 |

| Command                                           | Purpose              |
| ------------------------------------------------- | -------------------- |
| `python -m uvicorn backend.api.main:app --reload` | Start API dev server |
| `psql pulsepoint`                                 | Connect to database  |
| `redis-cli`                                       | Connect to Redis     |
| `pytest backend/tests -v`                         | Run tests            |
| `alembic upgrade head`                            | Apply migrations     |

---

**You're ready to develop!** 🚀

Start with the [README.md](README.md) for features overview, then refer to specific guides as needed.
