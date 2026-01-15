AI Control Tower for Shopify: Detailed Step-by-Step Explanation
Table of Contents

Phase 1: Foundation & Data Infrastructure
Phase 2: Data Integration & ETL Pipelines
Phase 3: Data Unification & Analytics Engine
Phase 4: AI Intelligence Layer
Phase 5: API & Frontend
Phase 6: Deployment & Operations
Phase 7: Testing & Quality Assurance
Phase 8: Go-to-Market Preparation


Phase 1: Foundation & Data Infrastructure
Step 1.1: Set Up Core Architecture
What it does:
This step establishes the technical foundation for the entire application. Think of it as building the skeleton and nervous system of your platform.
Why it matters:

Backend framework: FastAPI or Django provides the web server that handles all incoming requests from users and external services
Database choice: PostgreSQL stores transactional data (orders, products, campaigns) that need fast writes and updates
BigQuery: Handles analytics queries on massive datasets without slowing down your main database
Redis: Acts as a fast memory cache to speed up repeated queries and manage job queues
Celery/Cloud Tasks: Runs background jobs (like syncing data from Shopify) without blocking user requests

Real-world analogy:
Like building a house, you need the foundation (database), plumbing (data pipelines), electrical system (API framework), and HVAC (caching/background jobs) before you can add furniture.
What happens:
User Request → FastAPI → PostgreSQL (for immediate data)
                      → BigQuery (for analytics)
                      → Redis (for cached results)

Background: Celery Worker → Runs ETL jobs every hour

Step 1.2: Design Database Schema
What it does:
Creates the blueprint for how all your data is organized and connected. Every piece of information (brands, products, orders, ads, inventory) gets its own table with defined relationships.
Why it matters:
A well-designed schema is crucial because:

Prevents data duplication: Each piece of info stored once
Ensures data integrity: Relationships (foreign keys) prevent orphaned records
Enables complex queries: Properly linked tables allow you to answer questions like "Which campaigns drove sales for low-stock products in California?"
Scales efficiently: Good design means fast queries even with millions of records

Key Tables Explained:
brands table:

Stores each Shopify store/client
Primary identifier for all other data
Example: "Acme Candles", "Urban Wellness Co."

integrations table:

Stores API tokens for Shopify, Meta, TikTok, Google
Tracks connection status and last sync time
Critical for OAuth authentication

products table:

Master catalog of all SKUs
Includes cost, price, margin calculations
Links to inventory and orders

inventory_levels table:

Current stock by product AND location
Updated in real-time from Shopify
Key for inventory-aware recommendations

orders & order_line_items tables:

Every purchase from Shopify
Includes customer location (for geo analysis)
Links to products and fulfillment costs

ad_campaigns & ad_performance tables:

Campaign data from Meta/TikTok/Google
Daily performance metrics by campaign and geo
The "front-end" that connects to inventory "back-end"

BigQuery tables:

unified_performance: Pre-joined data combining ads + sales + inventory
ai_insights: Cached AI summaries to avoid regenerating them

What happens:
Database Schema Design Process:
1. Identify entities (brands, products, orders, campaigns)
2. Define relationships (one brand has many products)
3. Add indexes for fast lookups (product_id, order_date)
4. Partition large tables by date for query performance

Phase 2: Data Integration & ETL Pipelines
Step 2.1: Shopify Integration
What it does:
Connects to Shopify's API to automatically pull product catalog, orders, and inventory data into your database.
Why it matters:
Shopify is the "source of truth" for revenue. Without this integration, you can't:

Attribute orders to ad campaigns
Know what's actually in stock
Calculate true ROAS (Return on Ad Spend)

How it works:
ShopifyService class:
This is a Python wrapper around Shopify's API that handles authentication and data fetching.
sync_products():

Fetches all products and their variants
Extracts SKU, price, cost (from metafields)
Returns structured product data ready for database insertion

sync_orders():

Pulls orders from the last 90 days (configurable)
Includes line items (what was purchased)
Captures customer location (country, state, ZIP) for geo analysis
Critical: This location data is how you match orders to regional ad performance

sync_inventory_levels():

Gets current stock quantities by warehouse/location
Shows available vs. reserved inventory
Updates every hour to catch stockouts quickly

ETL Job (sync_shopify_data):
This Celery task runs automatically on a schedule (e.g., every hour):
1. Authenticate with Shopify using stored OAuth token
2. Fetch new/updated products → Upsert to products table
3. Fetch new orders → Upsert to orders table
4. Fetch current inventory → Update inventory_levels table
5. Mark integration.last_synced_at = now()
What happens:
Shopify Store → API Request → ShopifyService
                            → Transform data format
                            → Upsert to PostgreSQL
                            → Log success/failure
Error handling:

If API token expires: Trigger OAuth refresh flow
If rate limit hit: Exponential backoff and retry
If product deleted in Shopify: Mark as inactive, don't delete (preserve historical data)


Step 2.2: Meta Ads Integration
What it does:
Pulls advertising performance data from Facebook/Instagram Ads Manager, including spend, revenue, and geographic breakdown.
Why it matters:
Meta often represents 40-60% of a DTC brand's ad spend. You need this data to:

Calculate ROAS by campaign, region, and product
Detect when performance drops
Make budget reallocation recommendations

How it works:
MetaAdsService class:
Uses Facebook's official facebook_business Python SDK to authenticate and fetch data.
get_campaign_performance():
The core method that fetches daily performance metrics:
Key parameters:

time_range: Date window (e.g., last 7 days)
level: 'campaign': Aggregates data by campaign (could also do ad set or ad level)
breakdowns: ['country', 'region']: Splits performance by geography

Fields fetched:

spend: How much was spent on ads
impressions: How many times ads were shown
clicks: Click-through rate
actions: Contains conversions (purchases, add-to-carts, etc.)
action_values: Contains revenue from conversions

The tricky part - extracting conversions:
Meta returns actions as a nested array:
json"actions": [
  {"action_type": "link_click", "value": 145},
  {"action_type": "purchase", "value": 23},
  {"action_type": "add_to_cart", "value": 67}
]
```

Your code needs to find `action_type: "purchase"` and extract the value.

**Geographic breakdown:**
Each row of data includes:
- `country`: "United States"
- `region`: "California"

This allows you to see that Meta spent $5,000 in California with 3.2x ROAS, but $2,000 in Texas with only 1.8x ROAS.

**What happens:**
```
Meta Ads Manager → API Request with date range + breakdowns
                 → Returns nested JSON with performance data
                 → Extract conversions and revenue from actions
                 → Structure into flat table format
                 → Insert into ad_performance table
Scheduling:
This runs daily at 2am to fetch yesterday's finalized data.

Step 2.3: TikTok & Google Ads Integration
What it does:
Same concept as Meta, but for TikTok and Google ad platforms. Each has its own API structure and authentication.
TikTok Ads Service:
Why TikTok is different:

Uses Marketing API with different endpoint structure
Requires separate OAuth flow
Data format differs from Meta (but you normalize it)

Key method: get_campaign_performance():
pythonpayload = {
    "report_type": "BASIC",
    "data_level": "AUCTION_CAMPAIGN",  # Campaign-level data
    "dimensions": ["campaign_id", "country_code", "province_name"],
    "metrics": ["spend", "conversion", "total_purchase_value"]
}
What's happening:

You request a report for specific date range
TikTok returns campaign performance split by geography
You transform their field names to match your standard schema

Google Ads Service:
Why Google is different:

Uses Google Ads Query Language (GAQL), similar to SQL
More complex authentication (OAuth + developer token)
Provides extremely granular data (keyword-level)

Key method: get_campaign_performance():
pythonquery = """
    SELECT campaign.id, campaign.name, metrics.cost_micros, 
           metrics.conversions, segments.geo_target_country
    FROM campaign
    WHERE segments.date BETWEEN '2024-01-01' AND '2024-01-07'
"""
```

**Cost in micros:**
Google returns costs in micros (millionths), so $100 = 100,000,000 micros. You must divide by 1,000,000.

**What happens across all platforms:**
```
For each platform (Meta, TikTok, Google):
  1. Authenticate using stored token
  2. Fetch campaign performance for date range
  3. Transform to standard format:
     - platform_campaign_id
     - spend, impressions, clicks, conversions, revenue
     - country, state/region
  4. Insert into ad_performance table with platform='meta'|'tiktok'|'google'
  5. Handle errors and log sync status
Result:
You now have a unified ad_performance table with data from all platforms in the same structure, ready for analysis.

Phase 3: Data Unification & Analytics Engine
Step 3.1: Build Unified Data Model
What it does:
Combines data from Shopify (orders) with data from ad platforms (campaigns) to answer: "Which ads drove which sales?"
Why it matters:
This is the CORE value proposition. Without attribution, you can't:

Know true ROAS (each platform self-reports, but we need Shopify truth)
Recommend budget changes
Connect ad performance to inventory levels

The Attribution Challenge:
When someone buys from your Shopify store, Shopify knows:

Order ID: 12345
Order date: 2024-01-15 10:30am
Total: $85
Location: Austin, TX
Products: SKU-A (qty 2), SKU-B (qty 1)

But Shopify DOESN'T definitively know which ad campaign caused this purchase.
Meanwhile, Meta reports:

Campaign "Winter Sale - Texas" spent $500 on 2024-01-15
Got 25 conversions (purchases)
Generated $2,100 in revenue (self-reported via Meta pixel)

Your job: Connect the dots.
AttributionService class:
build_unified_performance() method:
Step 1: Get Shopify orders
sqlSELECT order_id, order_date, total_amount, 
       shipping_country, shipping_state, 
       product_ids, customer_id
FROM orders
WHERE order_date BETWEEN start_date AND end_date
Step 2: Get ad performance from all platforms
sqlSELECT platform, campaign_id, ad_date, 
       spend, conversions, revenue,
       country, state
FROM ad_performance
WHERE ad_date BETWEEN start_date AND end_date
Step 3: Attribution logic (_attribute_orders_to_campaigns)
Last-touch attribution (simplified model):
For each order:

Look back 7 days from order date
Find ad campaigns that:

Ran during that window
Match the customer's geographic location
Promoted the same products (if data available)


Assign the order to the most recent matching campaign (last-touch)

pythonFor order placed on 2024-01-15 in Austin, TX:
  - Look for campaigns from 2024-01-08 to 2024-01-15
  - Filter to campaigns targeting Texas
  - Pick the last one the customer likely saw
  - Attribute this order to that campaign
Step 4: Enrich with inventory data
Now join with inventory:
pythonunified_df = attributed_orders.merge(
    inventory_levels,
    on=['product_id', 'location'],
    how='left'
)
This adds:

Stock levels at time of sale
Days of cover (how long stock would last at current sales rate)
Margin % and fulfillment cost

Step 5: Calculate derived metrics
pythonunified_df['roas'] = revenue / spend
unified_df['profit'] = revenue - spend - cogs - fulfillment_cost
unified_df['roi'] = profit / spend
```

**Step 6: Load to BigQuery**

The final unified dataset goes to BigQuery's `unified_performance` table, which looks like:

| date | brand_id | channel | campaign_id | geo_state | product_id | spend | revenue | roas | stock_level | days_of_cover |
|------|----------|---------|-------------|-----------|------------|-------|---------|------|-------------|---------------|
| 2024-01-15 | brand_1 | meta | camp_123 | TX | SKU-A | 20.00 | 85.00 | 4.25 | 150 | 12 |

**What happens:**
```
Shopify Orders + Ad Platform Data + Inventory Levels
                    ↓
         Attribution Algorithm
                    ↓
      Unified Performance Table
                    ↓
    Ready for Analytics & AI

Step 3.2: Create Analytics Views & Calculations
What it does:
Builds pre-calculated analytics that answer common business questions without slow queries.
InventoryIntelligenceService class:
calculate_days_of_cover():
What it calculates:
Days of Cover (DOC) = How many days current inventory will last at current sales velocity
Why it matters:

DOC < 14 days = Low stock, risk of stockout
DOC 14-90 days = Healthy
DOC > 90 days = Overstock, tied up capital

How it works:
Step 1: Calculate sales velocity
sql-- For each product + location, find average daily sales
SELECT product_id, location_id, AVG(daily_quantity) as avg_daily_sales
FROM (
    -- Get daily sales for last 30 days
    SELECT product_id, location_id, DATE(order_date), SUM(quantity)
    FROM orders
    WHERE order_date >= CURRENT_DATE - 30 days
    GROUP BY product_id, location_id, date
)
GROUP BY product_id, location_id
Step 2: Calculate days of cover
sqldays_of_cover = current_inventory / avg_daily_sales

Example:
- Current stock: 210 units
- Avg daily sales: 15 units/day
- Days of cover: 210 / 15 = 14 days
detect_stock_issues():
Uses the DOC calculation to categorize problems:
pythonissues = {
    'stockouts': products where quantity = 0,
    'low_stock': products where DOC < 14 days,
    'overstock': products where DOC > 90 days
}
Why this matters for recommendations:
The AI will later say: "Reduce spend on SKU-A in Texas—you only have 8 days of cover and you're spending heavily there."
calculate_inventory_ad_alignment():
What it does:
Compares how much you're spending on ads for each product vs. how much inventory you have.
The query:
sqlSELECT 
    product_id,
    SUM(ad_spend) as total_ad_spend,
    SUM(inventory) as total_inventory,
    AVG(days_of_cover) as avg_doc,
    ad_spend / total_ad_spend as spend_share,
    ROAS
FROM unified_performance
GROUP BY product_id
Then categorize mismatches:
pythonIf DOC < 14 and spend_share > 15%:
    → "overspending_low_stock"

If DOC > 90 and spend_share < 5%:
    → "underspending_overstock"

If inventory = 0 and ad_spend > 0:
    → "spending_on_stockout" (the worst!)
```

**What happens:**
```
Inventory Data + Sales Velocity + Ad Spend
                    ↓
          Analytics Calculations
                    ↓
        Mismatch Detection
                    ↓
     Recommendations Input
Example output:
ProductAd SpendInventoryDOCMismatch TypeSKU-A$5,000 (35%)120 units8 daysoverspending_low_stockSKU-B$500 (7%)1,800 units120 daysunderspending_overstockSKU-C$1,2000 units0 daysspending_on_stockout
This data feeds directly into AI recommendations.

Phase 4: AI Intelligence Layer
Step 4.1: Build AI Explanation Engine
What it does:
Uses Claude (Anthropic's AI) to analyze your data and generate human-readable insights and recommendations.
Why it matters:
Data alone doesn't help users. They need:

Plain-language explanations of what happened
Specific actions to take
Reasoning behind each recommendation

AIInsightsService class:
generate_weekly_summary():
Input:

Performance data (spend, revenue, ROAS by channel/geo)
Inventory data (stock levels, mismatches)

What it does:
Step 1: Prepare context
The _prepare_context() method structures your raw data into a format Claude can understand:
json{
  "performance": {
    "current_week": {
      "total_spend": 18500,
      "total_revenue": 72000,
      "blended_roas": 3.9,
      "by_channel": {
        "meta": {"spend": 11000, "revenue": 46200, "roas": 4.2},
        "tiktok": {"spend": 5000, "revenue": 10500, "roas": 2.1},
        "google": {"spend": 2500, "revenue": 8750, "roas": 3.5}
      }
    },
    "previous_week": { /* same structure */ }
  },
  "inventory": {
    "misalignments": [
      {
        "product": "SKU-A",
        "issue": "overspending_low_stock",
        "doc": 8,
        "ad_spend": 5000,
        "geo": "CA, TX"
      }
    ]
  }
}
```

**Step 2: Create AI prompt**

You send Claude a structured prompt:
```
You are an AI advisor for a Shopify DTC brand. Analyze this data:

PERFORMANCE DATA:
[JSON data above]

Generate a summary with:
1. Overall performance summary
2. Channel breakdown (what's working/not working)
3. Geographic insights
4. Inventory-ad alignment issues
5. 3-5 specific recommendations

Return as JSON.
Step 3: Claude analyzes and responds
Claude examines the data and generates:
json{
  "summary": "Last week, you spent $18,500 on ads and generated $72,000 in Shopify revenue at a blended 3.9× ROAS. Meta drove 61% of revenue with stable 4.2× ROAS. Google improved from 3.0× to 3.5×. TikTok spend increased 28%, but ROAS fell from 2.8× to 2.1×, especially in California and Texas where conversion rates dropped.",
  
  "channel_performance": {
    "meta": "Stable performer at 4.2x ROAS, representing your strongest channel...",
    "tiktok": "ROAS declined 25% week-over-week, driven by increased spend in low-converting regions...",
    "google": "Showing improvement, worth testing additional budget..."
  },
  
  "inventory_insights": "You are heavily promoting SKU-A on TikTok in CA and TX, but inventory in those regions is low (8 days cover) and fulfillment costs are high. Meanwhile, SKUs B and C are overstocked in the Midwest with better margins and minimal ad support.",
  
  "recommendations": [
    {
      "action": "Cut TikTok prospecting spend on SKU-A in CA/TX by 20%",
      "reason": "Low ROAS (2.1x) + low inventory (8 days) + high fulfillment costs",
      "expected_impact": "Avoid stockouts and improve blended ROAS by 0.3x",
      "priority": "high"
    },
    {
      "action": "Increase budget by $1,500 for SKUs B and C in Midwest states",
      "reason": "Strong historical ROAS (4.8x) + 120 days inventory + better margins",
      "expected_impact": "Capture $7,200 in incremental revenue at profitable ROAS",
      "priority": "high"
    }
  ]
}
Step 4: Store in database
This AI response is cached in the ai_insights table so you don't regenerate it every time the user refreshes the page.
detect_alerts():
What it does:
Automatically detects when something needs immediate attention.
Alert triggers:
ROAS drop alert:
pythonif current_roas < historical_avg * 0.85:  # 15% drop
    trigger_alert("roas_drop", channel_data)
Inventory-ad mismatch alert:
pythonif spending_on_stockout or overspending_low_stock:
    trigger_alert("inventory_mismatch", mismatch_data)
```

**For each alert, Claude generates an explanation:**
```
Prompt: "Explain why TikTok ROAS dropped from 2.8x to 2.1x in CA/TX"

Claude Response: "The decline is primarily driven by a new broad prospecting campaign that launched 4 days ago, targeting a cold audience. Additionally, you're promoting SKU-A heavily in these regions where inventory is critically low (8 days cover) and fulfillment costs are 30% higher than average. This creates poor unit economics even when conversions occur."
answer_ad_hoc_question():
What it does:
Lets users ask questions in natural language and get AI-powered answers.
Example:
User asks: "Why did my revenue drop this week?"
Step 1: Retrieve relevant context
pythoncontext_data = {
    'revenue_this_week': 72000,
    'revenue_last_week': 85000,
    'channel_breakdown': {...},
    'top_products': {...}
}
```

**Step 2: Send to Claude**
```
Question: "Why did my revenue drop this week?"
Context: [JSON data]

Provide a specific answer with numbers.
```

**Step 3: Claude responds**
"Your revenue dropped by $13,000 (15.3%) compared to last week. The primary driver was a 35% reduction in Google Shopping spend, which typically generates high-intent traffic at 4.5x ROAS. Additionally, your best-selling product (SKU-A) went out of stock in California on Tuesday, representing approximately $8,000 in lost revenue."

**What happens:**
```
User Question → Retrieve Relevant Data → Send to Claude API
                                       → Parse Response
                                       → Return to User

Step 4.2: Build Recommendation Engine
What it does:
Generates specific, actionable recommendations based on performance + inventory data, without relying solely on AI (combines rules-based logic with AI explanations).
RecommendationEngine class:
generate_budget_recommendations():
The goal:
Suggest specific budget changes that will improve overall profitability by considering:

ROAS performance
Inventory levels
Margins
Geographic factors

How it works:
Step 1: Get performance matrix
sqlSELECT channel, geo_state, product_id, 
       spend, revenue, roas,
       stock_level, days_of_cover, margin_pct
FROM unified_performance
WHERE date >= CURRENT_DATE - 30
Step 2: Apply recommendation rules
Rule 1: Reduce spend on underperformers with low stock
pythonFor each product/channel/geo combination:
    if ROAS < 2.5 
       AND days_of_cover < 14 
       AND spend_share > 10%:
        
        recommendation = {
            'type': 'reduce_spend',
            'channel': 'TikTok',
            'geo': 'California',
            'product': 'SKU-A',
            'current_spend': 5000,
            'recommended_change': -1000,  # 20% reduction
            'reason': 'ROAS of 2.1x below target while inventory critically low (8 days)',
            'expected_outcome': 'Avoid stockouts, improve blended ROAS'
        }
Why this matters:
You're spending $5,000/month on ads in California for a product that:

Only has 8 days of inventory left
Is generating low ROAS (2.1x)
Has high fulfillment costs in that region

The recommendation: Cut that spend by $1,000 (20%) to avoid:

Running out of stock (bad customer experience)
Paying high costs to fulfill orders that aren't profitable
Wasting budget that could go to better opportunities

Rule 2: Increase spend on high performers with good inventory
pythonFor each product/channel/geo combination:
    if ROAS > 4.0 
       AND days_of_cover > 30 
       AND spend_share < 10%:
        
        recommendation = {
            'type': 'increase_spend',
            'channel': 'Meta',
            'geo': 'Midwest',
            'product': 'SKU-B',
            'current_spend': 1000,
            'recommended_change': +500,  # 50% increase
            'reason': 'Strong ROAS of 4.8x with healthy inventory (65 days)',
            'expected_outcome': 'Capture $2,400 incremental revenue at profitable ROAS'
        }
The math:

Current spend: $1,000/month → Revenue: $4,800 (4.8x ROAS)
Proposed spend: $1,500/month → Expected revenue: $7,200
Incremental revenue: $2,400
Stock: 65 days of cover (plenty to fulfill)

Rule 3: Launch promotions for overstock
pythonFor products with days_of_cover > 90:
    
    recommendation = {
        'type': 'promotional_campaign',
        'product': 'SKU-C',
        'geo': 'Midwest',
        'reason': 'Excess inventory (120 days cover)',
        'suggested_action': 'Launch limited-time 15-20% discount bundle',
        'expected_outcome': 'Accelerate sell-through, free up $12,000 in tied capital'
    }
Step 3: Prioritize recommendations
Scoring logic:
pythonpriority_score = 0

# Impact size
if abs(budget_change) > $1000: score += 3
if abs(budget_change) > $500: score += 2

# Type (preventing waste > capturing opportunity)
if type == 'reduce_spend': score += 2  # Stop the bleeding
if type == 'increase_spend': score += 1  # Grow what works

# Sort by score, return top 5
```

**What happens:**
```
Performance Data + Inventory Data
            ↓
    Apply Business Rules
            ↓
    Generate Recommendations
            ↓
    Score & Prioritize
            ↓
    Return Top 5 Actions
Example output:
json[
  {
    "priority": "high",
    "type": "reduce_spend",
    "action": "Cut TikTok spend on SKU-A in CA/TX by $1,000",
    "current_spend": 5000,
    "new_spend": 4000,
    "reason": "Low ROAS (2.1x) + critically low stock (8 days) + high fulfillment costs",
    "expected_impact": "Avoid stockouts, improve ROAS by 0.3x"
  },
  {
    "priority": "high",
    "type": "increase_spend",
    "action": "Increase Meta budget for SKU-B in Midwest by $500",
    "current_spend": 1000,
    "new_spend": 1500,
    "reason": "Strong ROAS (4.8x) + healthy stock (65 days) + better margins",
    "expected_impact": "Generate $2,400 incremental revenue"
  },
  {
    "priority": "medium",
    "type": "promotion",
    "action": "Launch clearance bundle for SKUs C, D, E",
    "reason": "Excess inventory (120+ days cover) tying up $15,000 capital",
    "expected_impact": "Clear slow-movers, improve cash flow"
  }
]
```

These recommendations appear in the dashboard with "Accept", "Dismiss", or "See Details" buttons.

---

## Phase 5: API & Frontend

### Step 5.1: Build REST API

**What it does:**
Creates the HTTP endpoints that the frontend (website/app) calls to get data and trigger actions.

**Why it matters:**
The API is the bridge between your backend logic and the user interface. It needs to:
- Respond quickly (< 500ms for most requests)
- Return data in a consistent format
- Handle errors gracefully
- Support both brands and agencies

**Main API Routes:**

**`GET /api/brands/{brand_id}/dashboard`**

**What it does:**
Returns everything needed for the main dashboard screen.

**Request:**
```
GET /api/brands/brand_123/dashboard
Processing flow:
python1. Fetch performance data (last 7 days vs previous 7 days)
2. Fetch inventory alignment data
3. Generate or retrieve cached AI weekly summary
4. Generate budget recommendations
5. Combine all data into single response
Response:
json{
  "summary": {
    "summary": "Last week you spent $18,500...",
    "channel_performance": {
      "meta": "Meta drove 61% of revenue...",
      "tiktok": "TikTok ROAS declined...",
      "google": "Google improved..."
    },
    "inventory_insights": "You are heavily promoting SKU-A..."
  },
  "recommendations": [
    {
      "priority": "high",
      "action": "Cut TikTok spend by $1,000...",
      "reason": "Low ROAS + low stock...",
      "expected_impact": "Avoid stockouts..."
    }
  ],
  "performance_snapshot": {
    "total_spend": 18500,
    "total_revenue": 72000,
    "blended_roas": 3.9,
    "by_channel": {
      "meta": {"spend": 11000, "revenue": 46200, "roas": 4.2},
      "tiktok": {"spend": 5000, "revenue": 10500, "roas": 2.1},
      "google": {"spend": 2500, "revenue": 8750, "roas": 3.5}
    }
  }
}
Frontend uses this to:

Display weekly summary text
Show channel performance cards
Render recommendation cards
Plot ROAS trends

GET /api/brands/{brand_id}/alerts
What it does:
Returns urgent alerts that need immediate attention.
Response:
json{
  "alerts": [
    {
      "type": "roas_drop",
      "severity": "high",
      "channel": "tiktok",
      "title": "TikTok ROAS dropped 25% in CA/TX",
      "explanation": "The decline is driven by new broad prospecting campaign...",
      "recommended_action": "Reduce or pause Campaign ID 789 in CA/TX"
    },
    {
      "type": "stockout_risk",
      "severity": "critical",
      "product": "SKU-A",
      "title": "SKU-A will stock out in 3 days",
      "explanation": "Current sales velocity is 15 units/day with 45 units remaining...",
      "recommended_action": "Pause all ads promoting SKU-A immediately"
    }
  ]
}
POST /api/brands/{brand_id}/ask
What it does:
Natural language Q&A interface.
Request:
json{
  "question": "Which products performed best last month?"
}
Processing:
python1. Retrieve relevant data (top products by revenue/profit)
2. Send question + context to Claude API
3. Return AI-generated answer
Response:
json{
  "question": "Which products performed best last month?",
  "answer": "Your top 3 products by revenue last month were:\n\n1. **SKU-A** - Generated $24,500 in revenue at 4.2x ROAS, primarily through Meta campaigns in Texas and California.\n\n2. **SKU-B** - Generated $18,200 in revenue at 4.8x ROAS, best performer in Midwest regions via Google Shopping.\n\n3. **SKU-C** - Generated $12,800 in revenue at 3.1x ROAS, though inventory is concerning (120 days of cover suggests overstock).\n\nRecommendation: Focus additional budget on SKU-B given its strong ROAS and healthy inventory position."
}
```

**`GET /api/brands/{brand_id}/drilldown/channel/{channel}`**

**What it does:**
Deep dive into specific channel performance.

**Request:**
```
GET /api/brands/brand_123/drilldown/channel/meta
Response:
json{
  "channel": "meta",
  "performance": {
    "total_spend": 45000,
    "total_revenue": 189000,
    "roas": 4.2,
    "by_geo": {
      "CA": {"spend": 18000, "revenue": 72000, "roas": 4.0},
      "TX": {"spend": 12000, "revenue": 54000, "roas": 4.5},
      "NY": {"spend": 9000, "revenue": 40500, "roas": 4.5}
    }
  },
  "top_campaigns": [
    {"id": "camp_1", "name": "Winter Sale - TX", "spend": 5000, "roas": 5.2},
    {"id": "camp_2", "name": "Retargeting - All", "spend": 4500, "roas": 6.1}
  ],
  "inventory_alignment": {
    "products_promoted": ["SKU-A", "SKU-B", "SKU-C"],
    "mismatches": [
      {"product": "SKU-A", "issue": "low_stock", "days_cover": 12}
    ]
  },
  "recommendations": [
    "Shift 20% of CA budget to TX where ROAS is stronger",
    "Increase retargeting budget by $1,000 given 6.1x performance"
  ]
}
```

**`GET /api/agency/overview`**

**What it does:**
Shows all brands managed by an agency, sorted by urgency.

**Request:**
```
GET /api/agency/overview?agency_id=agency_456
Response:
json{
  "brands": [
    {
      "brand_id": "brand_123",
      "brand_name": "Acme Candles",
      "health_score": 65,
      "alerts": 3,
      "opportunities": 1,
      "status": "needs_attention",
      "top_issue": "TikTok ROAS dropped 25%, spending on low-stock SKU"
    },
    {
      "brand_id": "brand_456",
      "brand_name": "Urban Wellness",
      "health_score": 88,
      "alerts": 0,
      "opportunities": 2,
      "status": "opportunity",
      "top_issue": "Strong inventory + ROAS in Midwest, room to scale"
    },
    {
      "brand_id": "brand_789",
      "brand_name": "Pet Paradise",
      "health_score": 92,
      "alerts": 0,
      "opportunities": 0,
      "status": "stable",
      "top_issue": null
    }
  ]
}
```

**Frontend displays this as a priority list:**
- Red flag brands appear at top (needs attention)
- Green flag brands in middle (opportunities)
- Gray brands at bottom (stable, no action needed)

**What happens:**
```
User clicks button → HTTP request to API endpoint
                   → API fetches/calculates data
                   → Returns JSON response
                   → Frontend renders UI

Step 5.2: Build Frontend UI
What it does:
Creates the visual interface users interact with—the website or app they see in their browser.
Dashboard Component:
What it displays:
1. AI Weekly Summary Card
jsx<Card className="summary-card">
  <h2>Weekly Summary</h2>
  <p>Last week you spent $18,500 on ads...</p>
  
  <MetricGrid>
    <Metric label="Total Spend" value="$18,500" />
    <Metric label="Revenue" value="$72,000" />
    <Metric label="ROAS" value="3.9x" trend="+0.2" />
  </MetricGrid>
</Card>
What happens:

User opens dashboard
Component calls fetchDashboard() API
Displays AI-generated summary in plain English
Shows key metrics with visual indicators (↑ green for improvements, ↓ red for declines)

2. Channel Performance Tabs
jsx<Tabs>
  <Tab label="Meta">
    <p>Meta drove 61% of revenue with stable 4.2× ROAS...</p>
    <BarChart data={metaPerformanceByGeo} />
  </Tab>
  <Tab label="TikTok">
    <p>TikTok ROAS fell from 2.8× to 2.1×...</p>
    <BarChart data={tiktokPerformanceByGeo} />
  </Tab>
</Tabs>
What happens:

User clicks between tabs to see channel-specific insights
Each tab shows AI explanation + data visualization
Charts use Recharts library to plot spend/revenue/ROAS by geography

3. Inventory & Ad Alignment Section
This is the KEY differentiator—showing front + back connection:
jsx<Card className="inventory-insights">
  <h3>📦 Inventory & Ad Alignment</h3>
  <p>You are heavily promoting SKU-A on TikTok in CA/TX, 
     but inventory is low (8 days cover)...</p>
  
  <InventoryAlignmentChart 
    data={[
      {product: "SKU-A", spend: 5000, stock: 120, doc: 8, status: "overspending"},
      {product: "SKU-B", spend: 500, stock: 1950, doc: 130, status: "underspending"}
    ]} 
  />
</Card>
Visual representation:

Bubble chart where:

X-axis = Ad spend
Y-axis = Days of cover
Bubble size = Revenue
Color = Status (red = problem, green = opportunity, gray = aligned)



4. Recommendations List
The most important UI component:
jsx<RecommendationCard 
  recommendation={{
    type: "reduce_spend",
    priority: "high",
    action: "Cut TikTok spend on SKU-A in CA/TX by $1,000",
    reason: "Low ROAS (2.1x) + low stock (8 days) + high fulfillment costs",
    expected_impact: "Avoid stockouts, improve blended ROAS by 0.3x",
    recommended_change: -1000
  }}
  onAccept={handleAccept}
  onDismiss={handleDismiss}
/>
What happens:

Each recommendation displays as a card
Priority badge (High/Medium/Low) with color coding
"Accept" button → Applies the recommendation (adjusts budgets in ad platform)
"Dismiss" button → Hides this recommendation
"See details" → Shows full data behind the recommendation

5. Natural Language Q&A
jsx<QuestionInput brandId={brandId} />

// Inside the component:
<input 
  placeholder="E.g., Which regions are performing best this month?"
  value={question}
  onChange={e => setQuestion(e.target.value)}
/>
<Button onClick={handleAsk}>Ask</Button>

{answer && (
  <div className="answer-box">
    <p>{answer}</p>
  </div>
)}
User experience:

User types: "Why did my revenue drop?"
Clicks "Ask" button
Loading spinner shows "Thinking..."
AI answer appears: "Your revenue dropped by $13,000 primarily because..."

Agency Overview Component:
For agencies managing multiple brands:
jsx<AgencyOverview>
  <BrandCard 
    brand={{
      name: "Acme Candles",
      health_score: 65,
      alerts: 3,
      status: "needs_attention"
    }}
  />
  <BrandCard 
    brand={{
      name: "Urban Wellness",
      health_score: 88,
      alerts: 0,
      status: "opportunity"
    }}
  />
</AgencyOverview>
```

**Visual hierarchy:**
- Brands sorted by urgency (alerts first)
- Color-coded status badges
- Quick metrics at a glance
- Click card → Go to that brand's detailed dashboard

**What happens across the entire UI:**
```
User Interaction → API Call → Backend Processing
                             → Database Query
                             → AI Analysis (if needed)
                             → JSON Response
                             → Frontend Rendering
                             → User sees results
Performance optimization:

Initial page load: Fetch cached summary (fast)
Drill-downs: Fetch on-demand (lazy loading)
Real-time updates: WebSocket for live alerts
Charts: Render client-side with Recharts (no server load)


Phase 6: Deployment & Operations
Step 6.1: Deploy to Production
What it does:
Takes your application from your laptop to live servers accessible on the internet.
Why it matters:
Development environments are forgiving; production is not. You need:

Reliability: 99.9% uptime
Scalability: Handle 1 user or 10,000 users
Security: Protect sensitive data (OAuth tokens, customer info)
Monitoring: Know immediately if something breaks

Docker Configuration:
What Docker does:
Packages your entire application (code + dependencies) into a container that runs identically anywhere.
Dockerfile explained:
dockerfileFROM python:3.11-slim
# Starts with Python 3.11 base image (small, secure)

WORKDIR /app
# Sets working directory inside container

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
# Installs all Python dependencies (FastAPI, pandas, etc.)

COPY backend/ ./backend/
COPY .env .
# Copies your application code

CMD ["uvicorn", "backend.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
# Starts the web server when container runs
Docker Compose (for local development):
What it does:
Runs multiple containers together (API + Database + Redis + Background Worker).
yamlservices:
  api:
    # Your main web application
    ports: "8000:8000"  # Accessible at localhost:8000
  
  db:
    # PostgreSQL database
    # Data persists in docker volume
  
  redis:
    # Cache and job queue
  
  worker:
    # Celery worker for background jobs
    # Runs ETL tasks without blocking API
To run locally:
bashdocker-compose up
# Starts all services
# Visit http://localhost:8000 to see your app
GCP Cloud Run Deployment:
What Cloud Run does:

Hosts your containerized app
Auto-scales based on traffic (0 users = $0 cost, 1000 users = auto-scales)
Handles HTTPS/SSL automatically
Global CDN for fast response times

Deployment process:
Step 1: Build container image
bashgcloud builds submit --tag gcr.io/your-project/shopify-control-tower
This:

Uploads your code to Google Cloud
Builds Docker container
Stores in Google Container Registry

Step 2: Deploy to Cloud Run
bashgcloud run deploy shopify-control-tower \
  --image gcr.io/your-project/shopify-control-tower \
  --platform managed \
  --region us-central1 \
  --set-env-vars DATABASE_URL=$DATABASE_URL
What happens:

Cloud Run downloads your container
Starts instances (servers running your app)
Assigns public URL: https://shopify-control-tower-abc123.run.app
Routes traffic with load balancing

Environment variables (critical):
bashDATABASE_URL=postgresql://user:pass@db-host:5432/db
SHOPIFY_CLIENT_ID=abc123...
SHOPIFY_CLIENT_SECRET=xyz789...
META_APP_ID=...
TIKTOK_APP_KEY=...
ANTHROPIC_API_KEY=sk-ant-...
```

These contain sensitive credentials and must be:
- Stored in Google Secret Manager (not in code)
- Injected at runtime
- Different for dev/staging/production

**What happens when a user visits your app:**
```
User Request → Cloud Run Load Balancer
            → Routes to available container instance
            → Container runs your FastAPI app
            → Queries PostgreSQL/BigQuery
            → Returns response
            → Cloud Run scales up/down based on traffic
Database setup:
bash# PostgreSQL on Cloud SQL (managed database)
gcloud sql instances create shopify-db \
  --database-version=POSTGRES_15 \
  --tier=db-custom-2-8192 \
  --region=us-central1

# Create database
gcloud sql databases create shopify_control --instance=shopify-db
What happens:

Managed PostgreSQL instance (Google handles backups, updates, scaling)
Automatic daily backups
Point-in-time recovery
High availability with failover


Step 6.2: Set Up Monitoring
What it does:
Tracks system health, performance, and errors so you know immediately when something goes wrong.
Why it matters:
In production, you can't just print() debug statements. You need:

Performance metrics: How fast are API responses?
Error tracking: What exceptions are occurring?
Usage metrics: How many API calls? How much does AI cost?
Alerts: Wake you up if the site goes down

MetricsCollector class:
record_api_latency():
What it does:
Tracks how long each API endpoint takes to respond.
python# In your API routes
start_time = time.time()
result = process_dashboard_data(brand_id)
duration_ms = (time.time() - start_time) * 1000

metrics.record_api_latency("/api/dashboard", duration_ms)
```

**This creates a time-series metric:**
```
Timestamp: 2024-01-15 14:30:00
Endpoint: /api/dashboard
Duration: 450ms
In Google Cloud Monitoring:

View graphs of response times over time
Set alerts if p95 latency > 1000ms
Identify slow endpoints that need optimization

record_ai_token_usage():
What it does:
Tracks AI API usage for cost monitoring.
pythonresponse = claude_api.generate_summary(data)
tokens_used = response.usage.total_tokens

metrics.record_ai_token_usage("claude-sonnet-4", tokens_used)
Why this matters:
AI API costs can explode if not monitored:

Claude Sonnet: ~$3 per million input tokens
If you generate summaries for 1000 brands daily
Each summary = 5000 tokens average
Daily cost = $15
Monthly cost = $450

Monitoring prevents:

Unexpected bills
Token leaks (infinite loops calling AI)
Inefficient prompts (using 10k tokens when 2k would work)

Additional monitoring (using Google Cloud Monitoring):
Error tracking:
pythonfrom google.cloud import error_reporting

error_client = error_reporting.Client()

try:
    result = process_data()
except Exception as e:
    error_client.report_exception()
    # Logs full stack trace to Google Cloud
    # Sends alert if error rate spikes
Application logs:
pythonimport logging
logging.info(f"Processing brand {brand_id}")
logging.warning(f"ROAS below threshold for {campaign_id}")
logging.error(f"Failed to sync Shopify: {error}")
Logs appear in Cloud Logging:

Searchable (find all errors for brand_123)
Filterable (show only warnings)
Exportable (send to BigQuery for analysis)

Uptime monitoring:
bash# Cloud Monitoring uptime check
gcloud monitoring uptime-checks create https-check \
  --display-name="API Health Check" \
  --resource-type=uptime-url \
  --monitored-resource-url=https://your-app.run.app/health
```

**What happens:**
- Pings your `/health` endpoint every 1 minute
- If 3 consecutive failures → Triggers alert
- Sends email/SMS to on-call engineer

**Alerting policies:**

**Example: High error rate alert**
```
IF error_rate > 5% for 5 minutes
THEN send email to team@company.com
AND page on-call engineer
```

**Example: High AI cost alert**
```
IF daily_ai_tokens > 50M
THEN send Slack notification
AND email finance team
```

**Dashboards:**

Create real-time dashboards showing:
- Requests per second
- Average response time
- Error rate
- Database connection pool usage
- AI token consumption
- Cost metrics

**What happens in production:**
```
Your App Runs → Emits metrics/logs
             → Cloud Monitoring collects
             → Stores in time-series database
             → Displays in dashboards
             → Triggers alerts if thresholds exceeded
             → On-call engineer investigates

Phase 7: Testing & Quality Assurance
Step 7.1: Unit Tests
What it does:
Tests individual functions in isolation to ensure they work correctly.
Why it matters:

Catch bugs before deploying to production
Ensure changes don't break existing functionality
Document expected behavior
Enable confident refactoring

test_attribution_service.py:
What it tests:
test_attribution_logic():
pythondef test_attribution_logic():
    """Test that orders are correctly matched to campaigns"""
    
    # Create mock data
    orders = pd.DataFrame([
        {
            'order_id': 'ord_1',
            'order_date': datetime(2024, 1, 15, 10, 30),
            'country': 'US',
            'state': 'TX',
            'total_amount': 85.00
        }
    ])
    
    campaigns = pd.DataFrame([
        {
            'campaign_id': 'camp_1',
            'platform': 'meta',
            'date': datetime(2024, 1, 14),
            'country': 'US',
            'state': 'TX',
            'spend': 100.00,
            'conversions': 5
        }
    ])
    
    # Run attribution
    attribution_service = AttributionService(mock_bq_client)
    result = attribution_service._attribute_orders_to_campaigns(orders, campaigns)
    
    # Assertions (what we expect to be true)
    assert len(result) == 1  # One order should be attributed
    assert result.iloc[0]['campaign_id'] == 'camp_1'  # To the correct campaign
    assert result.iloc[0]['revenue'] == 85.00  # With correct revenue
What this catches:

Orders not being matched when they should be
Orders matched to wrong campaigns
Attribution window logic errors
Geo-matching failures

test_inventory_alignment():
pythondef test_inventory_alignment():
    """Test that inventory-ad mismatches are correctly detected"""
    
    inventory_service = InventoryIntelligenceService(mock_db)
    
    # Run alignment calculation
    alignment = inventory_service.calculate_inventory_ad_alignment('brand_123')
    
    # Verify structure
    assert 'mismatch_type' in alignment.columns
    
    # Verify categories are correct
    assert alignment['mismatch_type'].isin([
        'overspending_low_stock',
        'underspending_overstock',
        'spending_on_stockout',
        'aligned'
    ]).all()
    
    # Verify logic
    low_stock_overspend = alignment[
        (alignment['days_of_cover'] < 14) & 
        (alignment['spend_share'] > 0.15)
    ]
    assert (low_stock_overspend['mismatch_type'] == 'overspending_low_stock').all()
What this catches:

Mismatch categorization errors
Calculation bugs (wrong thresholds)
Missing data handling

Running tests:
bashpytest tests/test_attribution_service.py -v
```

**Output:**
```
test_attribution_logic PASSED
test_inventory_alignment PASSED
test_days_of_cover_calculation PASSED
test_stockout_detection PASSED

4 passed in 2.3s

Step 7.2: Integration Tests
What it does:
Tests entire workflows end-to-end, including external services (AI API, databases).
Why it matters:
Unit tests verify individual functions work, but integration tests verify the entire system works together.
test_ai_insights.py:
test_weekly_summary_generation():
pythondef test_weekly_summary_generation():
    """Test AI summary generation end-to-end"""
    
    # Set up test environment
    ai_service = AIInsightsService(test_api_key)
    
    # Load test data (realistic sample data)
    performance_data = {
        'current_week': {
            'total_spend': 18500,
            'total_revenue': 72000,
            'blended_roas': 3.9
        },
        'by_channel': {
            'meta': {'spend': 11000, 'revenue': 46200, 'roas': 4.2},
            'tiktok': {'spend': 5000, 'revenue': 10500, 'roas': 2.1}
        }
    }
    
    inventory_data = {
        'stock_issues': [
            {'product': 'SKU-A', 'days_of_cover': 8, 'issue': 'low_stock'}
        ],
        'misalignments': [
            {'product': 'SKU-A', 'type': 'overspending_low_stock'}
        ]
    }
    
    # Generate summary
    summary = ai_service.generate_weekly_summary(
        'brand_test',
        performance_data,
        inventory_data
    )
    
    # Verify response structure
    assert 'summary' in summary
    assert 'channel_performance' in summary
    assert 'recommendations' in summary
    
    # Verify recommendations have required fields
    assert len(summary['recommendations']) > 0
    for rec in summary['recommendations']:
        assert 'action' in rec
        assert 'reason' in rec
        assert 'expected_impact' in rec
        assert 'priority' in rec
    
    # Verify AI understood the data
    assert 'SKU-A' in summary['inventory_insights']  # Mentioned problem product
    assert '2.1' in summary['channel_performance']['tiktok']  # Mentioned low ROAS
What this catches:

AI API connection issues
Malformed prompts
Response parsing errors
Missing data in AI context
AI hallucinations (making up data)

test_end_to_end_workflow():
pythondef test_end_to_end_workflow():
    """Test complete workflow: data sync → analysis → recommendations"""
    
    # Step 1: Sync Shopify data
    sync_result = sync_shopify_data('brand_test')
    assert sync_result['status'] == 'success'
    
    # Step 2: Verify data loaded
    products = db.query(Product).filter_by(brand_id='brand_test').count()
    assert products > 0
    
    # Step 3: Run analytics
    attribution_service = AttributionService(bq_client)
    unified_data = attribution_service.build_unified_performance('brand_test')
    assert len(unified_data) > 0
    
    # Step 4: Generate recommendations
    rec_engine = RecommendationEngine(db, ai_service)
    recommendations = rec_engine.generate_budget_recommendations('brand_test')
    assert len(recommendations) > 0
    
    # Step 5: Verify recommendation quality
    for rec in recommendations:
        assert rec['priority'] in ['high', 'medium', 'low']
        assert rec['recommended_change'] != 0  # Should suggest actual changes
What this catches:

Data pipeline failures
Integration between services
Database transaction issues
End-to-end timing problems

Performance tests:
pythondef test_dashboard_response_time():
    """Ensure dashboard API responds within 500ms"""
    
    start = time.time()
    response = client.get(f"/api/brands/brand_123/dashboard")
    duration = time.time() - start
    
    assert response.status_code == 200
    assert duration < 0.5  # Must respond in < 500ms
What this catches:

Slow queries
N+1 query problems
Missing database indexes

Running integration tests:
bashpytest tests/integration/ -v --slow
CI/CD Integration:
yaml# GitHub Actions workflow
on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: |
          docker-compose up -d  # Start test environment
          pytest tests/
          docker-compose down
      
      - name: Deploy if tests pass
        if: github.ref == 'refs/heads/main'
        run: gcloud run deploy ...
```

**What happens:**
```
Code Push → GitHub Actions triggered
          → Runs all unit tests
          → Runs integration tests
          → If all pass → Deploy to staging
          → If staging works → Deploy to production
          → If any fail → Block deployment, notify team

Phase 8: Go-to-Market Preparation
Step 8.1: Create Onboarding Flow
What it does:
Guides new users through connecting their Shopify store and ad accounts.
Why it matters:
First impression is critical. A smooth onboarding:

Increases conversion (free trial → paid customer)
Reduces support tickets
Gets users to "aha moment" faster

Shopify OAuth Flow:
What OAuth does:
Allows your app to access Shopify data without seeing the user's password. Instead, Shopify gives you an access token.
Step 1: User clicks "Connect Shopify"
python@router.get("/oauth/shopify")
async def shopify_oauth_init(shop: str):
    """Redirect user to Shopify authorization page"""
    
    scopes = "read_products,read_orders,read_inventory,read_analytics"
    # What data you're requesting access to
    
    redirect_uri = f"{BASE_URL}/oauth/shopify/callback"
    # Where Shopify sends user after they approve
    
    auth_url = f"https://{shop}/admin/oauth/authorize?" \
               f"client_id={SHOPIFY_CLIENT_ID}&" \
               f"scope={scopes}&" \
               f"redirect_uri={redirect_uri}"
    
    return {"auth_url": auth_url}
```

**What happens:**
```
User enters store URL: acme-candles.myshopify.com
                    → Your app constructs OAuth URL
                    → Redirects user to Shopify
                    → Shopify shows: "Shopify Control Tower wants to access..."
                    → User clicks "Approve"
```

**Step 2: Shopify redirects back with authorization code**
```
https://your-app.com/oauth/shopify/callback?shop=acme-candles.myshopify.com&code=abc123xyz
Step 3: Exchange code for access token
python@router.get("/oauth/shopify/callback")
async def shopify_oauth_callback(shop: str, code: str):
    """Handle Shopify OAuth callback"""
    
    # Exchange code for access token
    token_response = requests.post(
        f"https://{shop}/admin/oauth/access_token",
        json={
            "client_id": SHOPIFY_CLIENT_ID,
            "client_secret": SHOPIFY_CLIENT_SECRET,
            "code": code
        }
    )
    
    access_token = token_response.json()["access_token"]
    # This is what you need to make API calls
    
    # Create brand record
    brand = Brand(
        brand_name=shop.replace('.myshopify.com', ''),
        shopify_store_url=shop
    )
    db.add(brand)
    
    # Store integration
    integration = Integration(
        brand_id=brand.brand_id,
        platform='shopify',
        access_token=access_token,
        is_active=True
    )
    db.add(integration)
    db.commit()
    
    # Trigger initial data sync
    sync_shopify_data.delay(brand.brand_id)
    # Background job starts pulling products, orders, inventory
    
    return redirect("/dashboard?brand_id={brand.brand_id}")
Step 4: Show onboarding progress
jsx// Frontend onboarding wizard
<OnboardingWizard>
  <Step 1: Connect Shopify>
    ✓ Connected: acme-candles.myshopify.com
    Syncing data... (2,345 products, 15,234 orders)
  </Step>
  
  <Step 2: Connect Ad Accounts>
    [ ] Meta Ads
    [ ] TikTok Ads
    [ ] Google Ads
  </Step>
  
  <Step 3: Review & Launch>
    Preview your first dashboard →
  </Step>
</OnboardingWizard>
Meta/TikTok/Google OAuth:
Same pattern, different endpoints:
Meta:
pythonauth_url = f"https://www.facebook.com/v18.0/dialog/oauth?" \
           f"client_id={META_APP_ID}&" \
           f"redirect_uri={CALLBACK_URL}&" \
           f"scope=ads_read,ads_management"
TikTok:
pythonauth_url = f"https://business-api.tiktok.com/portal/auth?" \
           f"app_id={TIKTOK_APP_ID}&" \
           f"redirect_uri={CALLBACK_URL}"
Google:
pythonauth_url = google_auth_client.get_authorization_url(
    scopes=['https://www.googleapis.com/auth/adwords']
)
```

**What happens during onboarding:**
```
User signs up → Connect Shopify → OAuth flow
                               → Access token stored
                               → Background sync starts
                               
              → Connect Meta → OAuth flow
                            → Access token stored
                            
              → Connect TikTok → OAuth flow
                              → Access token stored
                              
              → All connected → Dashboard ready
                              → First AI summary generated
                              → User sees insights

Step 8.2: Documentation
What it does:
Provides comprehensive documentation for developers integrating with your API and users learning the product.
Why it matters:
Good docs:

Enable self-service (fewer support tickets)
Accelerate partner integrations
Improve developer experience
Build trust with technical users

API Documentation (OpenAPI/Swagger):
What OpenAPI does:
Auto-generates interactive API documentation from your code.
pythonfrom fastapi import FastAPI
from fastapi.openapi.utils import get_openapi

app = FastAPI(
    title="Shopify Control Tower API",
    description="AI-powered multi-channel ad performance and inventory intelligence",
    version="1.0.0"
)

# Your API routes automatically documented
@router.get(
    "/api/brands/{brand_id}/dashboard",
    summary="Get brand dashboard",
    description="Returns weekly summary, recommendations, and performance metrics",
    response_model=DashboardResponse
)
async def get_dashboard(brand_id: str):
    """
    Retrieve complete dashboard data for a brand.
    
    Args:
        brand_id: Unique brand identifier
    
    Returns:
        Dashboard object containing:
        - AI-generated weekly summary
        - Channel performance breakdown
        - Inventory alignment insights
        - Prioritized recommendations
    
    Example response:
```json
    {
      "summary": {
        "summary": "Last week you spent $18,500...",
        "channel_performance": {...}
      },
      "recommendations": [...]
    }
```
    """
    return get_dashboard_data(brand_id)
```

**Accessing the docs:**
```
https://your-api.com/docs
Interactive features:

See all endpoints
View request/response schemas
Try API calls directly from browser
See example responses
Download OpenAPI spec

User Documentation:
Getting Started Guide:
markdown# Getting Started with Shopify Control Tower

## 1. Create Your Account
Visit https://app.shopifycontroltower.com and sign up.

## 2. Connect Your Shopify Store
Click "Connect Shopify" and authorize access to:
- Products & Inventory
- Orders
- Customer data

## 3. Connect Ad Accounts
Connect your advertising platforms:

### Meta (Facebook/Instagram)
1. Click "Connect Meta"
2. Select your ad account
3. Grant permissions

### TikTok
1. Click "Connect TikTok"
2. Select advertiser account
3. Grant permissions

## 4. View Your Dashboard
After connections complete, your dashboard will show:
- Weekly AI summary
- Channel performance
- Inventory insights
- Recommendations

## 5. Take Action
Review recommendations and click "Accept" to:
- Adjust ad budgets
- Launch promotions
- Optimize inventory
Feature Documentation:
markdown# Understanding Recommendations

## What Are Recommendations?
AI-powered suggestions to improve ad performance and inventory efficiency.

## Types of Recommendations

### Reduce Spend
**When shown:** Low ROAS + low inventory
**Example:** "Cut TikTok spend on SKU-A in CA by $1,000"
**Why:** Prevents stockouts and wasted ad spend

### Increase Spend
**When shown:** High ROAS + good inventory
**Example:** "Increase Meta budget for SKU-B in Midwest by $500"
**Why:** Captures incremental profitable revenue

### Promotional Campaign
**When shown:** Excess inventory
**Example:** "Launch 15% discount for SKU-C"
**Why:** Clears overstock, improves cash flow

## How to Accept Recommendations
1. Review the recommendation card
2. Click "See details" to view supporting data
3. Click "Accept" to apply
4. System adjusts budgets automatically

## Recommendation Priority
- **High:** Urgent action needed (prevent stockout, stop waste)
- **Medium:** Good opportunity (optimize performance)
- **Low:** Fine-tuning (incremental improvements)
API Reference:
markdown# API Reference

## Authentication
All API requests require authentication via Bearer token:
```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
     https://api.shopifycontroltower.com/api/brands/123/dashboard
```

## Endpoints

### GET /api/brands/{brand_id}/dashboard
Returns complete dashboard data.

**Parameters:**
- `brand_id` (path, required): Brand identifier

**Response:**
```json
{
  "summary": {...},
  "recommendations": [...],
  "performance_snapshot": {...}
}
```

### POST /api/brands/{brand_id}/ask
Natural language question answering.

**Request body:**
```json
{
  "question": "Which products performed best last month?"
}
```

**Response:**
```json
{
  "question": "...",
  "answer": "Your top 3 products were..."
}
```
```

**What happens:**
```
Developer visits docs → Sees API reference
                     → Tries example requests
                     → Integrates successfully
                     
User visits docs → Learns features
                → Understands recommendations
                → Uses product effectively

Summary: What Each Phase Accomplishes
Phase 1: Foundation
Builds the technical infrastructure—databases, APIs, and architecture.
Phase 2: Data Integration
Connects to Shopify, Meta, TikTok, Google to pull in data.
Phase 3: Data Unification
Combines data sources, attributes orders to campaigns, calculates metrics.
Phase 4: AI Intelligence
Uses Claude to analyze data and generate insights/recommendations.
Phase 5: API & Frontend
Creates the interface users interact with—dashboards, charts, Q&A.
Phase 6: Deployment
Takes the app live on cloud infrastructure with monitoring.
Phase 7: Testing
Ensures everything works correctly before reaching customers.
Phase 8: Go-to-Market
Enables users to onboard smoothly and learn the product.
End result: A production-ready AI Control Tower that unifies front-end (ads) with back-end (inventory) to help Shopify merchants sell more profitably.