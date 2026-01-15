# Pulsepoint Platform Blueprint

## Complete Architecture, Structure & Data Flow Documentation

**Purpose:** This document serves as the foundational blueprint for the Pulsepoint platform. It defines the overall structure, business logic, folder organization, and how data flows through the system. This is the "before implementation" guide for the team.

---

## Table of Contents

1. [Platform Overview](#platform-overview)
2. [Business Logic & Core Concepts](#business-logic--core-concepts)
3. [Platform Architecture](#platform-architecture)
4. [Directory & Folder Structure](#directory--folder-structure)
5. [Data Flow Architecture](#data-flow-architecture)
6. [Key Features by Domain](#key-features-by-domain)
7. [Integration Points](#integration-points)
8. [Multi-Tenancy Model](#multi-tenancy-model)
9. [Observability & Monitoring](#observability--monitoring)

---

## Platform Overview

### What is Pulsepoint?

Pulsepoint is an **AI-powered e-commerce control tower** that unifies inventory management with multi-channel advertising. It serves as a bridge between two critical business operations:

- **Backend:** Shopify inventory management (what you have in stock)
- **Frontend:** Multi-channel advertising (Meta, TikTok, Google Ads)
- **Intelligence:** Claude AI generating recommendations to optimize both

### Core Value Proposition

Pulsepoint solves the fundamental disconnect in e-commerce: **advertising teams and inventory teams operate in silos.**

| Problem                                                    | Solution                                            | Benefit                                     |
| ---------------------------------------------------------- | --------------------------------------------------- | ------------------------------------------- |
| Ad campaigns spend budget without knowing inventory levels | Pulsepoint unifies both data sources                | Prevent stockouts while maximizing ad spend |
| Can't attribute orders to specific campaigns               | Order attribution service connects purchases to ads | Accurate ROAS calculation per campaign      |
| No single source of truth for profitability                | Multi-data fusion with AI analysis                  | Know which channels are actually profitable |
| Manual decisions on inventory vs ad spend                  | AI recommendations engine                           | Data-driven optimization 24/7               |
| Black box: Can't explain why recommendations happen        | Full observability & audit trails                   | Complete transparency for compliance        |

### Target Users

1. **E-commerce Store Owners** - Shopify store operators with multiple ad channels
2. **Marketing Teams** - Running campaigns across Meta, TikTok, Google
3. **Operations Teams** - Managing inventory and fulfillment
4. **Finance Teams** - Analyzing ROI and profitability per channel

---

## Business Logic & Core Concepts

### 1. Core Business Domains

#### Domain 1: Brand Management

- **Purpose:** Manage Shopify store connections (OAuth setup, credentials storage)
- **Scope:** Each "brand" is one Shopify store (one company can have multiple stores)
- **Key Activities:**
  - Connect/disconnect Shopify stores
  - Manage API credentials securely
  - Configure brand preferences and settings

#### Domain 2: Product & Inventory

- **Purpose:** Real-time visibility into what products exist and stock levels
- **Scope:** Product catalog with pricing, cost, and inventory across locations
- **Key Activities:**
  - Sync product catalog from Shopify
  - Track inventory levels by warehouse/location
  - Monitor low-stock products
  - Calculate margins and profitability

#### Domain 3: Orders & Attribution

- **Purpose:** Track customer purchases and attribute them to campaigns
- **Scope:** Every order linked to the campaign that drove it
- **Key Activities:**
  - Ingest orders from Shopify
  - Match orders to ad campaigns using UTM parameters/pixel data
  - Calculate order value, profit, and customer data
  - Enable performance analysis per channel

#### Domain 4: Advertising & Performance

- **Purpose:** Unified view of ad spend and results across all channels
- **Scope:** Campaigns from Meta, TikTok, Google Ads normalized into single format
- **Key Activities:**
  - Sync campaign performance data daily
  - Track impressions, clicks, spend, conversions
  - Calculate metrics (CPC, CTR, ROAS)
  - Analyze performance by product, geography, audience

#### Domain 5: AI Insights & Recommendations

- **Purpose:** Generate intelligent analysis and recommendations using Claude AI
- **Scope:** Answer business questions and suggest optimization actions
- **Key Activities:**
  - Process user questions about business performance
  - Generate insights by combining multiple data sources
  - Create actionable recommendations
  - Track token usage for billing

#### Domain 6: Observability & Analytics

- **Purpose:** Track all system operations for monitoring, cost tracking, and compliance
- **Scope:** Every API call, AI question, and data event logged
- **Key Activities:**
  - Record who did what and when
  - Calculate AI costs and token usage
  - Generate performance dashboards
  - Alert on anomalies

---

### 2. Key Business Processes

#### Process 1: Data Ingestion Pipeline

**Trigger:** Scheduled job (every 1-4 hours) or manual sync request

1. Pull product catalog from Shopify → Store in PostgreSQL
2. Pull orders from Shopify → Store in PostgreSQL + attribute to campaigns
3. Pull campaigns from Meta/TikTok/Google → Store in PostgreSQL
4. Pull performance data from all ad platforms → Store in PostgreSQL
5. Aggregate data → Push to BigQuery for analytics
6. Record all activities → Store in observability events

**Result:** Unified, up-to-date view of inventory and advertising

#### Process 2: User Question → AI Answer

**Trigger:** User asks a question via API or frontend

1. User provides question (e.g., "Which campaigns are driving sales for low-stock items?")
2. System extracts tenant context from JWT token
3. Query relevant data from PostgreSQL (orders, campaigns, inventory)
4. Send context + question to Claude AI
5. Claude generates answer with reasoning
6. Store observability event (tokens used, cost, latency)
7. Return answer to user

**Result:** Intelligent, data-driven insights

#### Process 3: Recommendation Generation

**Trigger:** Daily batch job or on-demand request

1. Analyze inventory levels (what's low? what's overstocked?)
2. Analyze ad performance (which campaigns are profitable? which are wasting budget?)
3. Cross-reference: Match inventory status with campaign performance
4. Generate recommendations (e.g., "Reduce spend on Campaign X - targeting low-stock items")
5. Score recommendations by confidence and impact
6. Store in database for display and audit trail
7. (Optional) Trigger n8n workflow to take action

**Result:** Actionable recommendations with clear reasoning

---

## Platform Architecture

### System-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXTERNAL SERVICES                             │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────┐    │
│  │ Shopify      │ │ Meta Ads     │ │ Claude AI            │    │
│  │ (Products,   │ │ (Campaigns,  │ │ (Intelligence)       │    │
│  │ Orders)      │ │ Performance) │ │                      │    │
│  └──────────────┘ └──────────────┘ └──────────────────────┘    │
│  ┌──────────────┐ ┌──────────────┐                              │
│  │ TikTok Ads   │ │ Google Ads   │                              │
│  │ (Campaigns)  │ │ (Campaigns)  │                              │
│  └──────────────┘ └──────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
                              │
                    OAuth & API Integration
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────┐
│                   PULSEPOINT BACKEND (MCP Server)                │
│                    Single FastAPI Application                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  API Layer - Request Routing & Validation                 │  │
│  │  Endpoints: /api/brands, /questions, /recommendations     │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Middleware Layer - Security & Context                    │  │
│  │  • Authentication (JWT verification)                      │  │
│  │  • Tenant Context (extract from token)                    │  │
│  │  • Authorization (RBAC checks)                            │  │
│  │  • Request Tracing                                        │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Service Layer - Business Logic                           │  │
│  │  • AttributionService      (Orders → Campaigns)           │  │
│  │  • AIInsightsService       (Claude integration)            │  │
│  │  • RecommendationEngine    (Smart suggestions)            │  │
│  │  • InventoryService        (Stock analysis)               │  │
│  │  • ObservabilityService    (Event tracking)                │  │
│  │  • AuthService             (JWT & permissions)            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Integration Layer - External APIs                        │  │
│  │  • ShopifyService          (Sync products/orders)         │  │
│  │  • MetaAdsService          (Sync campaigns)               │  │
│  │  • TikTokAdsService        (Sync campaigns)               │  │
│  │  • GoogleAdsService        (Sync campaigns)               │  │
│  │  • AnthropicService        (Claude AI calls)              │  │
│  │  • BigQueryService         (Analytics queries)            │  │
│  │  • PubSubService           (Event publishing)             │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  Data Access Layer (DAL) - Database Operations            │  │
│  │  • Repository Pattern for all data queries                │  │
│  │  • Connection pooling & optimization                      │  │
│  │  • Caching strategy (Redis)                               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
        │                          │                        │
        ▼                          ▼                        ▼
   ┌──────────────┐          ┌──────────────┐       ┌──────────────┐
   │ PostgreSQL   │          │   Redis      │       │   BigQuery   │
   │ (Transactional          │   (Cache)    │       │  (Analytics) │
   │  Storage)    │          │              │       │              │
   └──────────────┘          └──────────────┘       └──────────────┘
        │                                                   │
        │                                      Google Cloud
        │                                    Pub/Sub Topic
        │                                                   │
        └───────────────────────────────────────────────────┘
```

### Architectural Layers Explained

#### 1. API Layer (Request Routing)

- **Responsibility:** Accept HTTP requests, validate input, route to services
- **Characteristics:**
  - RESTful endpoints following standard patterns
  - Input validation using Pydantic models
  - Response serialization (JSON)
  - Error handling (400, 403, 404, 500 responses)
- **Key Endpoints:**
  - `POST /api/brands` - Create/register brand
  - `POST /api/questions` - Ask an AI question
  - `GET /api/recommendations` - Get recommendations
  - `POST /api/sync` - Trigger data sync

#### 2. Middleware Layer (Security & Context)

- **Responsibility:** Enforce security policies and extract request context
- **Characteristics:**
  - JWT token validation
  - Extract tenant ID from token (tenant context)
  - Role-based access control (RBAC)
  - Request tracing (unique request ID)
  - CORS policy enforcement
- **Security Flow:**
  - Client sends JWT in Authorization header
  - Middleware validates signature
  - Extract tenant_id, user_id, permissions
  - Set context variables for entire request
  - Pass to next layer

#### 3. Service Layer (Business Logic)

- **Responsibility:** Implement business rules and coordinate operations
- **Characteristics:**
  - **Stateless** (no per-request state)
  - **Composable** (services call other services)
  - **Testable** (business logic independent of HTTP)
  - **Observable** (all actions logged)
- **Key Services:**
  - **AttributionService:** Matches orders to campaigns using UTM/pixel data
  - **AIInsightsService:** Calls Claude AI with tenant-filtered context
  - **RecommendationEngine:** Generates suggestions based on inventory & performance
  - **InventoryService:** Analyzes stock levels and trends
  - **AuthService:** JWT token creation/validation and role checks
  - **ObservabilityService:** Records all events for audit & analytics

#### 4. Integration Layer (External APIs)

- **Responsibility:** Handle OAuth flow and communicate with external platforms
- **Characteristics:**
  - Secure credential storage (encrypted in database)
  - Retry logic and error handling
  - Rate limit respect (avoid throttling)
  - Data normalization (different APIs → common format)
- **Key Integrations:**
  - **Shopify:** Products, orders, inventory via REST API
  - **Meta/TikTok/Google:** Campaign performance via their respective APIs
  - **Claude AI:** Questions → Answers via Anthropic API
  - **BigQuery:** Bulk analytics queries
  - **Pub/Sub:** Publish observability events

#### 5. Data Access Layer (Database Operations)

- **Responsibility:** Isolate database operations and optimize queries
- **Characteristics:**
  - Repository pattern (one class per entity type)
  - Connection pooling (efficient resource use)
  - Query optimization (indexes, filtering early)
  - Tenant isolation enforcement (RLS or WHERE clauses)
  - Caching strategy (Redis for frequently accessed data)
- **Operations:**
  - CRUD (Create, Read, Update, Delete) on all entities
  - Complex queries (joins, aggregations)
  - Batch operations (bulk insert/update)
  - Transaction management (ACID compliance)

---

## Directory & Folder Structure

### Project Root Structure

```
pulsepoint/
│
├── backend/                          # Backend application
│   ├── api/                          # API layer (HTTP endpoints)
│   │   ├── __init__.py
│   │   ├── main.py                  # FastAPI app initialization
│   │   ├── routes/                  # API endpoint files (one per domain)
│   │   │   ├── auth.py              # Authentication endpoints
│   │   │   ├── brands.py            # Brand CRUD operations
│   │   │   ├── products.py          # Product management
│   │   │   ├── orders.py            # Order management
│   │   │   ├── campaigns.py         # Campaign management
│   │   │   ├── questions.py         # Q&A endpoints
│   │   │   ├── recommendations.py   # Recommendation endpoints
│   │   │   ├── integrations.py      # OAuth flow endpoints
│   │   │   ├── analytics.py         # Analytics & dashboards
│   │   │   └── admin.py             # Admin endpoints
│   │   │
│   │   └── middleware/              # Request processing middleware
│   │       ├── __init__.py
│   │       ├── tenant_context.py    # Extract tenant from JWT
│   │       ├── auth.py              # JWT validation & token verification
│   │       ├── error_handlers.py    # Global exception handling
│   │       ├── request_tracing.py   # X-Request-ID propagation
│   │       └── cors.py              # CORS policy enforcement
│   │
│   ├── services/                    # Service layer (business logic)
│   │   ├── __init__.py
│   │   ├── auth_service.py          # JWT creation, role validation
│   │   ├── brand_service.py         # Brand operations
│   │   ├── product_service.py       # Product catalog operations
│   │   ├── order_service.py         # Order processing
│   │   ├── attribution_service.py   # Order → Campaign matching
│   │   ├── inventory_service.py     # Stock analysis & alerts
│   │   ├── campaign_service.py      # Campaign aggregation
│   │   ├── ai_insights_service.py   # Claude AI integration
│   │   ├── recommendation_engine.py # Recommendation generation
│   │   ├── observability_service.py # Event tracking & logging
│   │   ├── token_tracker_service.py # AI token usage & billing
│   │   └── notification_service.py  # Email/Slack alerts
│   │
│   ├── integrations/                # External API integrations
│   │   ├── __init__.py
│   │   ├── shopify_service.py       # Shopify OAuth & sync
│   │   ├── meta_ads_service.py      # Meta Business API
│   │   ├── tiktok_ads_service.py    # TikTok Ads API
│   │   ├── google_ads_service.py    # Google Ads API
│   │   ├── anthropic_service.py     # Claude API wrapper
│   │   ├── bigquery_service.py      # BigQuery analytics
│   │   └── pubsub_service.py        # Pub/Sub events
│   │
│   ├── models/                      # Data models
│   │   ├── __init__.py
│   │   ├── database.py              # SQLAlchemy ORM models
│   │   ├── schemas.py               # Pydantic request/response models
│   │   ├── constants.py             # Enums and constants
│   │   └── types.py                 # Custom type definitions
│   │
│   ├── database/                    # Database setup & migrations
│   │   ├── __init__.py
│   │   ├── connection.py            # DB connection setup
│   │   ├── queries.py               # Common SQL queries
│   │   └── migrations/              # Alembic migration files
│   │       ├── versions/            # Individual migration scripts
│   │       ├── env.py               # Alembic environment config
│   │       └── script.py.mako       # Migration template
│   │
│   ├── utils/                       # Utility functions
│   │   ├── __init__.py
│   │   ├── encryption.py            # Token/credential encryption
│   │   ├── cache.py                 # Redis cache operations
│   │   ├── logging.py               # Structured logging
│   │   ├── validators.py            # Input validation helpers
│   │   ├── date_utils.py            # Date/time helpers
│   │   └── currency.py              # Currency conversion helpers
│   │
│   ├── jobs/                        # Background jobs
│   │   ├── __init__.py
│   │   ├── sync_shopify.py          # Sync products/orders from Shopify
│   │   ├── sync_meta.py             # Sync Meta Ads campaigns
│   │   ├── sync_tiktok.py           # Sync TikTok Ads campaigns
│   │   ├── sync_google.py           # Sync Google Ads campaigns
│   │   ├── generate_insights.py     # Generate daily insights
│   │   ├── process_recommendations.py # Process and rank recommendations
│   │   └── cleanup_jobs.py          # Data archival & cleanup
│   │
│   ├── config.py                    # Environment configuration
│   └── dependencies.py              # Dependency injection setup
│
├── frontend/                        # Frontend application (React/Vue)
│   ├── public/
│   ├── src/
│   │   ├── components/
│   │   │   ├── Dashboard.tsx        # Main dashboard view
│   │   │   ├── QuestionForm.tsx     # Q&A interface
│   │   │   ├── Recommendations.tsx  # Recommendation display
│   │   │   ├── Charts.tsx           # Analytics charts
│   │   │   └── Layout.tsx           # App shell
│   │   ├── pages/
│   │   │   ├── Home.tsx
│   │   │   ├── Dashboard.tsx
│   │   │   ├── Brands.tsx
│   │   │   ├── Settings.tsx
│   │   │   └── Admin.tsx
│   │   ├── services/
│   │   │   └── api.ts               # API client (calls backend)
│   │   └── App.tsx
│   └── package.json
│
├── scripts/                         # Setup & maintenance scripts
│   ├── setup_db.py                  # Create schemas, tables, RLS policies
│   ├── seed_data.py                 # Populate test data
│   ├── migrate_bigquery.py          # Setup BigQuery datasets
│   ├── deploy.sh                    # Production deployment script
│   └── migrate_production.py        # Production data migration
│
├── tests/                           # Test suite
│   ├── unit/                        # Unit tests (business logic)
│   ├── integration/                 # Integration tests (services + DB)
│   ├── e2e/                         # End-to-end tests (full flow)
│   └── fixtures/                    # Test data & mocks
│
├── docker/                          # Container configuration
│   ├── Dockerfile                   # Backend image
│   ├── docker-compose.yml           # Local development stack
│   └── nginx.conf                   # Reverse proxy config
│
├── docs/                            # Project documentation
│   ├── API.md                       # API reference
│   ├── DATABASE_SCHEMA.md           # Database design
│   ├── DEPLOYMENT.md                # Deployment guide
│   └── TROUBLESHOOTING.md           # Common issues
│
├── .env.example                     # Environment template
├── requirements.txt                 # Python dependencies
├── README.md                        # Project overview
├── .gitignore                       # Git ignore rules
└── docker-compose.yml               # Development environment
```

### Naming Conventions

**Files & Folders:**

- Use `snake_case` for Python files: `attribution_service.py`
- Use `PascalCase` for classes: `class AttributionService`
- Use `camelCase` for functions/variables: `def get_brand_inventory()`

**API Routes:**

- Use plural nouns: `/api/brands`, `/api/products`, `/api/orders`
- Use HTTP verbs: `GET /api/brands`, `POST /api/brands`, `PUT /api/brands/{id}`
- Use nesting for relationships: `GET /api/brands/{id}/products`

**Database:**

- Use `snake_case` for table names: `ad_campaigns`, `order_line_items`
- Use `snake_case` for column names: `created_at`, `tenant_id`
- Foreign keys: `{entity}_id` (e.g., `brand_id`, `product_id`)

---

## Data Flow Architecture

### End-to-End Data Flow Overview

```
Step 1: DATA INGESTION
─────────────────────
Trigger: Scheduled job (hourly) or manual request

    External APIs               Pulsepoint Backend              Storage
    ─────────────               ──────────────────              ───────

    Shopify API ──────────→ ShopifyService.sync_products() ──→ PostgreSQL
    (products)             (OAuth → REST call)                (products table)

    Shopify API ──────────→ ShopifyService.sync_orders() ────→ PostgreSQL
    (orders)               (OAuth → REST call)                (orders table)

    Meta API ─────────────→ MetaAdsService.sync_campaigns() ─→ PostgreSQL
    (campaigns)            (Bearer token → REST call)         (campaigns table)

    TikTok API ────────────→ TikTokAdsService.sync() ────────→ PostgreSQL
    (campaigns)            (OAuth → API v2)                   (campaigns table)

    Google Ads API ────────→ GoogleAdsService.sync() ────────→ PostgreSQL
    (campaigns)            (Service account → gRPC)           (campaigns table)


Step 2: DATA PROCESSING & ENRICHMENT
─────────────────────────────────────
    PostgreSQL                Service Layer              PostgreSQL
    ──────────                ─────────────              ──────────

    Raw Orders ──────────────→ AttributionService ─────→ Orders.campaign_id
    Campaign Data               (UTM parsing,
    Customer Data               Pixel matching)


    Products ─────────────────→ InventoryService ──────→ Inventory Alerts
    Inventory Levels           (Stock analysis,         (Low stock warnings,
                                Trends)                  Overstock alerts)


Step 3: AGGREGATION & ANALYTICS
────────────────────────────────
    PostgreSQL                                BigQuery
    ──────────                                ────────

    Products ──────┐
    Orders     ───┤──→ BigQueryService ─────→ unified_performance (table)
    Campaigns  ───┤    (Transform &            • product_id
    Inventory  ───┘    Aggregate)              • campaign_id
                                               • order_count
                                               • revenue
                                               • cost
                                               • roas


    This allows queries like:
    "Which campaigns are driving sales for low-stock items?"


Step 4: AI INSIGHTS & RECOMMENDATIONS
──────────────────────────────────────
    User Request               Pulsepoint Backend              Claude AI
    ────────────               ──────────────────              ────────

    "What should I            1. Extract tenant_id (from JWT)
    do with low               2. Query inventory: SELECT * FROM products
    stock items?"                 WHERE stock < min_threshold
                                  AND tenant_id = {tenant_id}
                              3. Query campaigns: SELECT * FROM campaigns
                                 WHERE tenant_id = {tenant_id}
                              4. Build prompt with context
                              5. Call Claude API ──────────→ Claude AI
                                                             (generates answer)
                              6. Return answer ←──────────← Claude response


Step 5: OBSERVABILITY & TRACKING
─────────────────────────────────
    Every Action               ObservabilityService          Multiple Storage
    ────────────               ──────────────────            ──────────────────

    API Call ──────────────→ Record event ─────────────→ PostgreSQL
    (request, response,        (request_id,               observability_events
    latency)                    timestamp,
                                endpoint,
                                status)

                           ─────────────→ Pub/Sub Topic
                                         (real-time stream)

                           ─────────────→ BigQuery
                                         (historical analytics)

    AI Question ──────────→ Record tokens ────────────→ PostgreSQL
    (question, answer,        (input_tokens,             ai_token_usage
    tokens used)               output_tokens,
                                model,
                                cost_usd)

                           ─────────────→ Pub/Sub Topic
                                         (for billing pipeline)

                           ─────────────→ BigQuery
                                         (cost analytics)
```

### Detailed Data Flow for Key Processes

#### Process 1: Product Sync Flow

```
START (Scheduled Job)
    │
    ├─→ Read: Shopify API configuration (stored in integrations table)
    │
    ├─→ Authenticate: Use refresh_token to get access_token
    │
    ├─→ Call Shopify API: GET /products.json?limit=250&status=active
    │
    ├─→ Transform: Normalize Shopify product format to internal schema
    │   {
    │     shopify_product_id: "123456",
    │     sku: "PRODUCT-001",
    │     name: "Blue Widget",
    │     cost: 15.00,
    │     price: 49.99,
    │     currency: "USD"
    │   }
    │
    ├─→ Upsert: Insert/update in products table
    │   INSERT INTO products (tenant_id, shopify_product_id, sku, ...)
    │   ON CONFLICT (tenant_id, shopify_product_id) DO UPDATE SET ...
    │
    ├─→ Sync variants: For each product, sync inventory levels
    │   Variant 1 → Location 1: 50 units
    │   Variant 1 → Location 2: 30 units
    │   Variant 2 → Location 1: 0 units
    │
    ├─→ Record observation: Log sync event to observability
    │   ObservabilityService.log_event(
    │     event_type="shopify.sync",
    │     status="success",
    │     product_count=1250
    │   )
    │
    └─→ END (Success or error logged)
```

#### Process 2: Order Attribution Flow

```
START (New Order Webhook from Shopify)
    │
    ├─→ Parse webhook: Extract order data
    │   • Order ID: ORD-123456
    │   • Customer email: customer@example.com
    │   • UTM parameters: ?utm_source=meta&utm_campaign=summer_sale
    │   • Pixel data: fbp, ttclid cookies
    │   • Purchase amount: $150.00
    │   • Products: [Product A, Product B]
    │
    ├─→ Extract campaign: AttributionService.attribute_order()
    │   • Check UTM source=meta → query Meta Ads API for campaign "summer_sale"
    │   • Check fbp cookie → query Meta Business SDK for matching campaign
    │   • Check ttclid → query TikTok conversion tracking
    │   • Score multiple potential matches, pick highest confidence
    │
    ├─→ Update order: Link to campaign
    │   UPDATE orders SET campaign_id=CAM-789 WHERE order_id=ORD-123456
    │
    ├─→ Update metrics: Record in daily performance
    │   UPDATE campaign_daily_performance
    │   SET conversions=conversions+1, revenue=revenue+150.00
    │   WHERE campaign_id=CAM-789 AND date=TODAY()
    │
    ├─→ Check alerts: InventoryService.check_low_stock()
    │   For each product in order:
    │     SELECT current_stock FROM inventory WHERE product_id=?
    │     IF stock < minimum_threshold:
    │       → Send alert to user
    │
    ├─→ Record observation: Log order event
    │   ObservabilityService.log_event(
    │     event_type="order.attributed",
    │     status="success",
    │     order_id=ORD-123456,
    │     campaign_id=CAM-789,
    │     confidence=0.95
    │   )
    │
    └─→ END (Order fully attributed)
```

#### Process 3: AI Question Answer Flow

```
START (User asks question)
    │
    ├─→ Receive request: "Which campaigns are most profitable?"
    │
    ├─→ Extract context from JWT:
    │   • tenant_id = T-001
    │   • user_id = U-123
    │   • user_role = manager
    │
    ├─→ Permission check: AuthService.check_permission()
    │   • Does user have "read:analytics" permission? YES
    │
    ├─→ Build context data: Query database for relevant info
    │
    │   SELECT c.campaign_id, c.name, c.platform,
    │          SUM(o.amount) as revenue,
    │          SUM(cdc.spend) as ad_spend,
    │          SUM(o.amount) - SUM(cdc.spend) as profit
    │   FROM campaigns c
    │   LEFT JOIN orders o ON c.campaign_id = o.campaign_id
    │   LEFT JOIN campaign_daily_cost cdc ON c.campaign_id = cdc.campaign_id
    │   WHERE c.tenant_id = 'T-001'
    │   GROUP BY c.campaign_id
    │   ORDER BY profit DESC
    │
    ├─→ Prepare prompt for Claude:
    │   """
    │   You are an e-commerce analyst. Here is campaign performance data:
    │
    │   Campaign 1: Summer Sale - Revenue: $25,000, Ad Spend: $5,000, Profit: $20,000
    │   Campaign 2: Flash Deal - Revenue: $12,000, Ad Spend: $8,000, Profit: $4,000
    │
    │   Question: Which campaigns are most profitable?
    │   """
    │
    ├─→ Call Claude API:
    │   APIService.call_claude(
    │     messages=[{"role": "user", "content": prompt}],
    │     max_tokens=1000
    │   )
    │
    ├─→ Claude responds:
    │   "Campaign 'Summer Sale' is significantly more profitable with a 4:1 ROI.
    │    It generated $20,000 in profit from $5,000 in ad spend, while 'Flash Deal'
    │    only achieved a 0.5:1 ROI. I recommend increasing budget allocation to
    │    Summer Sale and optimizing or pausing Flash Deal."
    │
    ├─→ Track tokens:
    │   TokenTrackerService.record_usage(
    │     question_id=Q-456,
    │     input_tokens=250,
    │     output_tokens=180,
    │     model="claude-3-sonnet",
    │     cost_usd=0.00234
    │   )
    │
    ├─→ Store response:
    │   INSERT INTO ai_questions (question_id, tenant_id, question_text, answer, ...)
    │
    ├─→ Record observation:
    │   ObservabilityService.log_event(
    │     event_type="ai.question_answered",
    │     status="success",
    │     question_id=Q-456,
    │     tokens_used=430,
    │     latency_ms=1200
    │   )
    │
    ├─→ Return to user:
    │   {
    │     "question": "Which campaigns are most profitable?",
    │     "answer": "[Claude's analysis]",
    │     "confidence": 0.92,
    │     "data_sources": ["campaign_performance", "order_data"]
    │   }
    │
    └─→ END (User gets answer)
```

---

## Key Features by Domain

### Domain 1: Brand Management

**What it does:**

- Connect Shopify stores to Pulsepoint
- Securely store OAuth credentials
- Manage multiple stores under one tenant
- Configure brand-specific settings

**Key Features:**

- Shopify OAuth flow (secure connect/disconnect)
- Encrypted credential storage
- Connection health monitoring
- API key rotation
- Admin access controls

**Data Stored:**

- Brand name and metadata
- Shopify API credentials (encrypted)
- Connected ad account IDs (Meta, TikTok, Google)
- Brand preferences and settings

---

### Domain 2: Product & Inventory

**What it does:**

- Sync product catalog from Shopify
- Track real-time inventory levels
- Monitor low-stock situations
- Analyze inventory trends

**Key Features:**

- Automated product sync (every hour)
- Inventory level tracking by location
- Low-stock alerts
- Inventory forecasting
- Historical inventory trends

**Data Stored:**

- Product details (SKU, name, cost, price)
- Inventory levels per location
- Inventory alerts and thresholds
- Historical inventory snapshots

---

### Domain 3: Orders & Attribution

**What it does:**

- Import orders from Shopify
- Link orders to the campaigns that drove them
- Calculate order profitability
- Enable order-level analysis

**Key Features:**

- Multi-touch attribution (UTM, pixel, email)
- Automatic order sync
- Attribution confidence scoring
- Customer data enrichment
- Order-to-campaign linking

**Data Stored:**

- Orders with customer information
- Order line items (products, quantities, prices)
- Campaign attribution (with confidence score)
- Fulfillment and shipping data

---

### Domain 4: Advertising & Performance

**What it does:**

- Unify ad performance across all channels
- Normalize data from different ad platforms
- Track campaign-level metrics
- Enable cross-channel analysis

**Key Features:**

- Multi-channel campaign sync (Meta, TikTok, Google)
- Daily performance aggregation
- Metric normalization (different platforms → common format)
- ROAS calculation per campaign
- Spend and performance trending

**Data Stored:**

- Campaigns (normalized across platforms)
- Daily campaign performance metrics
- Ad spend tracking
- Conversions by channel

---

### Domain 5: AI Insights & Recommendations

**What it does:**

- Answer business questions using AI
- Generate actionable recommendations
- Analyze complex business scenarios
- Provide explanations for insights

**Key Features:**

- Natural language Q&A
- Multi-data source context (inventory + ads)
- Intelligent recommendations
- Confidence scoring
- Audit trail of all AI interactions

**Data Stored:**

- Questions asked (with responses)
- Token usage per question
- AI cost tracking
- User feedback on answers

---

### Domain 6: Observability & Analytics

**What it does:**

- Track all system operations
- Monitor performance metrics
- Generate analytics dashboards
- Enable compliance auditing

**Key Features:**

- Comprehensive event logging
- Real-time dashboards
- Alert triggers
- Cost analysis by tenant
- Compliance reporting

**Data Stored:**

- All API requests and responses
- AI token usage and costs
- Performance metrics (latency, errors)
- User activity logs

---

## Integration Points

### How Different Components Connect

```
Component Communication Pattern:

┌─────────────┐
│ API Routes  │ Receives HTTP request → Validates → Calls services
│             │
└──────┬──────┘
       │
       ├─→ AuthService
       │   ├─→ JWT validation
       │   └─→ Permission checking
       │
       ├─→ Brand Service
       │   ├─→ Brand repository (database)
       │   └─→ Shopify integration service
       │
       ├─→ Attribution Service
       │   ├─→ Order repository
       │   ├─→ Campaign repository
       │   ├─→ Meta Ads integration
       │   └─→ TikTok integration
       │
       ├─→ AI Insights Service
       │   ├─→ Product/Inventory repository
       │   ├─→ Campaign repository
       │   ├─→ Anthropic integration
       │   └─→ Token tracker service
       │
       └─→ Observability Service
           ├─→ Event storage (PostgreSQL)
           ├─→ Pub/Sub publishing
           └─→ BigQuery insertion
```

### External Service Dependencies

**Synchronous (Blocking) Calls:**

- Claude AI (user waiting for answer)
- Database queries (need data to proceed)
- JWT validation (security check)

**Asynchronous (Background) Calls:**

- Shopify product sync (background job)
- BigQuery analytics (eventual consistency)
- Email/Slack notifications (user doesn't wait)
- Pub/Sub event publishing (fire-and-forget)

---

## Multi-Tenancy Model

### Tenant Isolation Strategy

Pulsepoint implements **4-layer isolation** to ensure complete data separation:

#### Layer 1: Database Row-Level Security (RLS)

- **How:** PostgreSQL RLS policies automatically filter queries by tenant
- **Benefit:** Database-level protection (even if app code has bugs)
- **Example:**
  ```
  SELECT * FROM products
  → Automatically becomes:
  SELECT * FROM products WHERE tenant_id = current_tenant
  ```

#### Layer 2: JWT Context

- **How:** Extract tenant_id from JWT token on every request
- **Benefit:** Know who is requesting what
- **Example:**
  ```
  Token payload: {tenant_id: "T-001", user_id: "U-123", role: "manager"}
  → Set context variables: current_tenant_id = "T-001"
  ```

#### Layer 3: BigQuery Datasets

- **How:** Each tenant gets their own BigQuery dataset
- **Benefit:** Isolated analytics, billing per tenant
- **Example:**
  ```
  Tenant A → dataset "pulsepoint_tenant_t001"
  Tenant B → dataset "pulsepoint_tenant_t002"
  ```

#### Layer 4: Redis Key Namespacing

- **How:** All cache keys prefixed with tenant ID
- **Benefit:** No cache cross-contamination
- **Example:**
  ```
  Key format: "tenant:{tenant_id}:{entity}:{id}"
  Tenant A cache: "tenant:t001:products:123"
  Tenant B cache: "tenant:t002:products:123" (different key, no collision)
  ```

### Tenant Data Boundaries

```
Tenant A                          Tenant B
─────────                         ─────────
Brands: [Store A]                 Brands: [Store B]
Products: [A1, A2, A3]            Products: [B1, B2]
Orders: [O1-100]                  Orders: [O101-200]
Campaigns: [Meta, Google]         Campaigns: [Meta, TikTok]

↓ Cannot See Each Other's Data ↓

Even if User A queries database directly:
SELECT * FROM products
→ Only sees [A1, A2, A3]
→ Cannot see [B1, B2] due to RLS policy

Even if cache is accessed:
GET "tenant:a:products:123" → Returns data for Tenant A
GET "tenant:a:products:124" → Returns NULL (B's product)
```

---

## Observability & Monitoring

### What Gets Observed

```
Category          Events Tracked              Storage              Purpose
────────          ──────────────              ───────              ────────

API Operations    • Request received          PostgreSQL           Debugging
                  • Response sent             (90-day retention)
                  • Latency (ms)
                  • Status code
                  • User/tenant

AI Operations     • Question asked            PostgreSQL           Usage analytics
                  • Answer generated          Pub/Sub (real-time)  Billing
                  • Tokens consumed          BigQuery (7-year)
                  • Cost (USD)
                  • Model used

Business Events   • Product added/updated    PostgreSQL            Compliance
                  • Order received           BigQuery             Auditing
                  • Campaign created
                  • Permission granted

Errors            • Exception type           PostgreSQL           Alerting
                  • Stack trace              BigQuery             Monitoring
                  • Timestamp
                  • Affected user/tenant

Inventory         • Stock level change       PostgreSQL           Analytics
                  • Low stock alert          BigQuery             Forecasting
                  • Reorder triggered
```

### Event Flow Example

```
Action: User asks AI question
    │
    ├─→ Event 1: api.request_received
    │   {timestamp, request_id, endpoint, user_id, tenant_id}
    │
    ├─→ Event 2: ai.question_submitted
    │   {question_id, question_text, tenant_id}
    │
    ├─→ Event 3: ai.claude_called
    │   {question_id, prompt_tokens, model}
    │
    ├─→ Event 4: ai.claude_response
    │   {question_id, completion_tokens, latency_ms}
    │
    ├─→ Event 5: ai.token_usage_tracked
    │   {question_id, total_tokens, cost_usd, tenant_id}
    │
    ├─→ Event 6: api.response_sent
    │   {request_id, status_code, latency_ms, response_size}
    │
    └─→ Parallel: All events published to:
        • PostgreSQL (immediate storage)
        • Pub/Sub (real-time processing)
        • BigQuery (eventual analytics)
```

---

## Summary: Blueprint for Implementation

### Before Writing Code, Team Should Understand:

1. **Architecture:** Single FastAPI server with 5 layers (API → Middleware → Services → Integration → DAL)

2. **Structure:** 10 backend modules organized by responsibility (api, services, integrations, models, database, etc.)

3. **Data Flow:** Data moves from external APIs → PostgreSQL → BigQuery, with observability tracking at every step

4. **Multi-Tenancy:** 4-layer isolation ensures complete data separation at JWT, database, analytics, and cache levels

5. **Business Logic:**

   - Domain 1: Brand connection management
   - Domain 2: Product & inventory tracking
   - Domain 3: Order attribution
   - Domain 4: Ad performance aggregation
   - Domain 5: AI insights
   - Domain 6: Observability

6. **Key Processes:**

   - Data Ingestion: External APIs → PostgreSQL (hourly)
   - Data Enrichment: Attribution, inventory analysis (real-time)
   - Analytics: PostgreSQL → BigQuery (async)
   - AI Insights: Query data + call Claude + track tokens
   - Observability: Log every action for compliance

7. **Integration Points:** Services call each other in a coordinated way; external APIs only called via integration layer

This blueprint provides the foundation. Implementation will follow this structure exactly, with each team member working on their assigned domain/module.
