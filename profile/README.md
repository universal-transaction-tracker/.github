# Universal Transaction Tracker

> **A bank-agnostic personal finance engine that transforms bank email alerts into a unified, interactive financial dashboard — supporting any card, checking account, or local bank.**

---

## Table of Contents

- Overview
- System Architecture
- Data Flow
- Repositories
  - email-parser
  - query-chart-bridge
  - dashboard
- Infrastructure & Deployment
- Authentication & Security
- Banks & Transaction Types Supported
- Tag Filter System
- Caching Strategy
- Getting Started
- Configuration Reference

---

## Overview

Universal Transaction Tracker is a self-hosted, privacy-first financial aggregation system. Rather than connecting to bank APIs (which require open banking agreements), it intercepts the transaction alert emails that banks already send, parses them with configurable regex templates, and stores structured transaction data in a MySQL database. A Go API service then queries that database and feeds Chart.js-compatible payloads to a Vue 3 single-page dashboard.

**Key design principles:**
- **Bank-agnostic**: Any bank that sends email alerts can be supported by adding a YAML rule.
- **Multi-user**: A single deployment serves multiple users — each user's transactions are scoped to their email address.
- **Zero proprietary SDKs**: No bank API keys or screen scraping. Works with standard email delivery infrastructure.
- **Self-hosted on Kubernetes**: All three services ship as minimal Docker images with example Kubernetes manifests.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              INGESTION LAYER                                    │
│                                                                                 │
│  ┌─────────────┐   email    ┌──────────────┐  webhook   ┌──────────────────┐   │
│  │  Your Bank  │──────────► │   Mailgun    │──────────► │  email-parser    │   │
│  │  (UOB/DBS)  │  alert     │  (routing)   │  POST      │  (Go / Gin)      │   │
│  └─────────────┘            └──────────────┘            │  :8083/webhook   │   │
│                                                          └────────┬─────────┘   │
│                                                                   │ GORM        │
│                                                                   ▼ AutoMigrate │
└───────────────────────────────────────────────────────────────────┼─────────────┘
                                                                    │
┌───────────────────────────────────────────────────────────────────▼─────────────┐
│                              DATA LAYER                                         │
│                                                                                 │
│              ┌──────────────────────────────────────────┐                      │
│              │            MySQL Database                 │                      │
│              │         (email_parser_db)                 │                      │
│              │                                           │                      │
│              │  email_transactions  │  tag_filters       │                      │
│              │  user_settings       │                    │                      │
│              └──────────────────────────────────────────┘                      │
│                                    ▲                                            │
│                              ┌─────┘                                            │
└──────────────────────────────┼──────────────────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────────────────┐
│                              QUERY LAYER                                        │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                    query-chart-bridge  (Go / Gin)                         │  │
│  │                         :8085                                             │  │
│  │                                                                           │  │
│  │  POST /charts/get       POST /charts/list      POST /values/get           │  │
│  │  POST /values/table     GET  /tags/list        POST /tags/filters/save    │  │
│  │                                                                           │  │
│  │  ┌──────────────────┐   ┌──────────────────┐                             │  │
│  │  │  Redis Cache     │   │  Cloudflare JWT  │                             │  │
│  │  │  (optional)      │   │  Auth middleware │                             │  │
│  │  └──────────────────┘   └──────────────────┘                             │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
                               │  Chart.js JSON
┌──────────────────────────────▼──────────────────────────────────────────────────┐
│                          PRESENTATION LAYER                                     │
│                                                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                    dashboard  (Vue 3 + TypeScript)                        │  │
│  │                    served by Nginx  :80                                   │  │
│  │                                                                           │  │
│  │  Swipeable dashboards  │  Date controllers  │  Tag controllers            │  │
│  │  12 chart types        │  URL state sync    │  User settings              │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │          Cloudflare Zero Trust Tunnel (single domain)                   │    │
│  │                                                                         │    │
│  │   Browser ──► Cloudflare Access (Google OAuth) ──► cloudflared pod     │    │
│  │                                                          │              │    │
│  │                               /api/* ───────────────────┤              │    │
│  │                               /*    ───────────────────►│              │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow

```
 1. Bank sends email alert to user inbox
        │
        ▼
 2. Mailgun route intercepts email,
    POSTs multipart/urlencoded form to
    https://your-domain.com/webhook
        │
        ▼
 3. email-parser receives webhook
    ┌───────────────────────────────────────────────────────────┐
    │  a. Extract subject from form values                      │
    │  b. Look up subject in config.yml parser.subjects         │
    │  c. Run regex patterns against email body (best-effort)   │
    │  d. Map named capture groups to EmailTransaction fields   │
    │     (currency, quantity, source, recipient, timestamp...) │
    │  e. Set EmailReceiver = Mailgun recipient field           │
    │     (this is the per-user key)                            │
    └───────────────────────────────────────────────────────────┘
        │
        ▼
 4. INSERT INTO email_transactions (via GORM AutoMigrate)
        │
        ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ (async, later) ─ ─ ─ ─ ─ ─ ─ ─
        │
        ▼
 5. User opens https://app.yourdomain.com
    Cloudflare Access challenges → Google OAuth → CF_Authorization cookie set
        │
        ▼
 6. Dashboard boots, fetches /app-config.json (injected via K8s ConfigMap)
    Loads dashboard definitions: panels, SQL queries, chart types, controllers
        │
        ▼
 7. For each visible dashboard tab, POST /api/charts/list
    {
      "panels": [ { "id": "...", "query": "SELECT ... FROM $BASE ..." } ],
      "controllers": [
        { "type": "month", "value": "2026-06" },
        { "type": "tag",   "name": "food", "value": 1, "args": { "filters": [...] } }
      ]
    }
        │
        ▼
 8. query-chart-bridge resolves controllers:
    ┌──────────────────────────────────────────────────────────────────────┐
    │  a. Inject mandatory user isolation condition:                       │
    │     LOWER(email_origin_receiver) = LOWER('<user_email>')             │
    │  b. Month controller → datetime BETWEEN '2026-06-01' AND '2026-06-30'│
    │  c. Tag controller (val=1) → (bank='DBS-SG' AND recipient REGEXP...) │
    │  d. Build base CTE:                                                  │
    │     WITH base AS (SELECT * FROM email_transactions WHERE <all conds>)│
    │  e. Replace $BASE in every panel query with `base`                   │
    │  f. Execute queries, format → Chart.js labels + datasets             │
    │  g. Redis cache hit check (if cacheMode != 0)                        │
    └──────────────────────────────────────────────────────────────────────┘
        │
        ▼
 9. Dashboard renders Chart.js panels (bar/line/pie/table/etc.)
    Tab state, dates, tag toggles persisted in URL query params
```

---

## Repositories

### email-parser

| | |
|---|---|
| **Language** | Go 1.21+ |
| **Framework** | Gin |
| **Database** | MySQL (GORM, AutoMigrate) |
| **Port** | 8083 |
| **Image** | GitHub Container Registry (ghcr.io) |

The ingestion service. Exposes a single webhook endpoint that Mailgun calls with the email metadata. Matches the email subject to a YAML-configured rule, applies one or more regex patterns to the body, and persists the parsed transaction.

**Key endpoints:**
```
POST /webhook     — Mailgun webhook receiver (form/multipart/JSON)
GET  /health      — Liveness probe
GET  /metrics     — Prometheus metrics (webhook_requests_total, http_request_duration_seconds)
GET  /debug       — Uptime + goroutine count
```

**Transaction model (MySQL table `email_transactions`):**

| Column | Type | Description |
|---|---|---|
| `id` | uint PK | Auto-increment |
| `ref_id` | varchar(128) | Bank transaction ref (or SHA hash) |
| `quantity` | bigint | Amount in smallest currency unit (e.g. cents) |
| `currency` | varchar(10) | ISO currency code (SGD, USD, …) |
| `recipient` | varchar(255) | Payee / merchant |
| `source` | varchar(255) | Account / card used |
| `transaction_timestamp` | varchar(100) | Raw timestamp string from email |
| `transaction_type` | varchar(255) | e.g. "UOB-SG Card", "DBS-SG Paynow" |
| `bank` | varchar(255) | e.g. "UOB-SG", "DBS-SG" |
| `email_origin_receiver` | varchar(255) | `To:` field (original inbox) |
| `email_receiver` | varchar(255) | Mailgun `recipient` — **user key** |
| `email_sender` | varchar(255) | Bank sender address |
| `datetime` | datetime | Parsed email `Date:` header |
| `inbound` | bool | true = money received |
| `properties` | longtext | Reserved JSON blob |

**Parsing pipeline:**

```
Email subject
    │
    ▼  Look up in config.yml parser.subjects
TypeSettings { type, bank, patterns[], anti-patterns[], samples[] }
    │
    ├─ anti-patterns → if any match → ErrIgnore (skip silently)
    │
    └─ patterns (ordered) → first match extracts named groups:
         currency, quantity, source, recipient,
         transaction_timestamp, direction, transaction_ref,
         skip_reason
```

**Adding a new bank / rule** — edit `config/config.yml`:
```yaml
parser:
  subjects:
    "Your Bank Alert Subject":
      type: "MYBANK-SG Card"
      bank: "MYBANK-SG"
      patterns:
        - |-
          (?P<currency>[A-Z]+) (?P<quantity>[0-9.,]+) charged to (?P<source>.+?) at (?P<recipient>.+?)\.
      samples:
        - |
          SGD 12.34 charged to My Card ending 1234 at STARBUCKS.
```

---

### query-chart-bridge

| | |
|---|---|
| **Language** | Go 1.21+ |
| **Framework** | Gin |
| **Database** | MySQL (database/sql + GORM for tag tables) |
| **Cache** | Redis |
| **Port** | 8085 |
| **Image** | GitHub Container Registry (ghcr.io) |

The data layer bridge. Accepts SQL queries in the request body, applies controller-generated WHERE clauses, executes against MySQL, and returns Chart.js-ready JSON. It also manages the per-user tag filter system.

**Endpoints:**

```
POST /charts/get          Single panel query → ChartData
POST /charts/list         Batch panels + controllers → multiple ChartData
POST /values/get          Single-column query → flat value array
POST /values/table        Any query → 2D table (first row = headers)

GET  /tags/list           List all tag names for the authenticated user
POST /tags/create         Create a new tag (sentinel row)
POST /tags/delete         Delete a tag (requires all filters deleted first)
POST /tags/rename         Rename a tag across all its filters

GET  /tags/filters/list         List all filters for a tag
POST /tags/filters/save         Upsert a named filter onto a tag
POST /tags/filters/delete       Delete a specific filter from a tag
GET  /tags/filters/field_options  Column names available for filtering
GET  /tags/filters/value_options  Distinct values for given columns

GET  /health              DB ping check
GET  /metrics             DB pool stats
```

**Controller types:**

| Type | Effect |
|---|---|
| `day` | `datetime BETWEEN '<date> 00:00:00' AND '<date> 23:59:59'` |
| `month` | `datetime BETWEEN '<year-month>-01 00:00:00' AND '<year-month>-last 23:59:59'` |
| `year` | `datetime BETWEEN '<year>-01-01 00:00:00' AND '<year>-12-31 23:59:59'` |
| `tag` | SQL conditions built from stored `tag_filters` rows (value: 1=include, -1=exclude, 0=off) |

**CTE injection pattern:**

All panel queries use `$BASE` as a placeholder. The bridge replaces it with a filtered CTE:

```sql
-- Panel query (from app-config.json):
SELECT bank, SUM(quantity) FROM $BASE GROUP BY bank

-- After controller injection (month=2026-06, user=alice@example.com):
WITH base AS (
  SELECT * FROM email_transactions
  WHERE LOWER(email_origin_receiver) = LOWER('alice@example.com')
    AND datetime BETWEEN '2026-06-01 00:00:00' AND '2026-06-30 23:59:59'
)
SELECT bank, SUM(quantity) FROM base GROUP BY bank
```

**Chart response format:**

```json
{
  "id": "my-panel",
  "chartData": {
    "labels": ["UOB-SG", "DBS-SG"],
    "datasets": [{
      "label": "SUM(quantity)",
      "data": [12345, 67890],
      "backgroundColor": "rgba(75,192,192,0.8)",
      "borderColor": "#4bc0c0",
      "fill": true
    }]
  }
}
```

---

### dashboard

| | |
|---|---|
| **Language** | TypeScript + Vue 3 (Composition API) |
| **Build** | Vite |
| **Charts** | Chart.js via vue-chartjs |
| **Server** | Nginx (Alpine) |
| **Port** | 80 (container) |
| **Image** | GitHub Container Registry (ghcr.io) |

The single-page application. Dashboards are defined in `app-config.json` (injected into the container via Kubernetes ConfigMap), making the Docker image fully environment-agnostic.

**UI layout:**

```
┌────────────────────────────────────────────────────────┐
│  [Tab 1]  [Tab 2]  [Tab 3]  ...                   [⚙] │  ← tab bar
├────────────────────────────────────────────────────────┤
│  Dashboard Title                                        │
│                                                         │
│  [Month: Jun 2026 ◄►]  [Tag: food ⊘ ✓ ✗]  ...        │  ← controllers
│                                                         │
│  ┌──────────────────────┐  ┌──────────────────────┐    │
│  │     Bar Chart        │  │     Pie Chart        │    │  ← panels
│  └──────────────────────┘  └──────────────────────┘    │
│  ┌────────────────────────────────────────────────┐     │
│  │               Transaction Table                │     │
│  └────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────┘
```

**Supported chart types:** `bar`, `line`, `area`, `pie`, `doughnut`, `radar`, `polararea`, `scatter`, `bubble`, `table`, `list`, `treemap`

**`app-config.json` structure (injected via ConfigMap):**

```json
{
  "API_URL": "https://app.yourdomain.com/api",
  "dashboards": [
    {
      "id": "monthly",
      "title": "Monthly – $month",
      "headerTitle": "Spending – $month",
      "req": {
        "panels": [
          {
            "id": "by-bank",
            "title": "By Bank",
            "chartType": "bar",
            "query": "SELECT bank, SUM(quantity)/100 FROM $BASE GROUP BY bank"
          },
          {
            "id": "txn-table",
            "title": "All Transactions",
            "chartType": "table",
            "query": "SELECT datetime, recipient, bank, quantity/100 AS sgd FROM $BASE ORDER BY datetime DESC"
          }
        ],
        "controllers": [
          { "type": "month" },
          { "type": "tag" }
        ]
      }
    }
  ]
}
```

**URL state persistence:**

Every tab switch, date change, and tag toggle is reflected in the URL query string, enabling bookmarking and sharing of exact dashboard states:
```
https://app.yourdomain.com/?tab=monthly&monthly,month=2026-05&tag_food=1&tag_transport=-1
```

---

## Infrastructure & Deployment

```
Kubernetes Cluster
├── email-parser Deployment       (image: ghcr.io/universal-transaction-tracker/email-parser)
│   └── ConfigMap / Secret → config.yml (db creds, parser rules)
│
├── query-chart-bridge Deployment (image: ghcr.io/universal-transaction-tracker/query-chart-bridge)
│   └── ConfigMap / Secret → config.yaml (db creds, cloudflare AUD, redis)
│
├── dashboard Deployment          (image: ghcr.io/universal-transaction-tracker/dashboard)
│   └── ConfigMap → app-config.json (API_URL, dashboards definitions)
│
├── MySQL StatefulSet             (or external managed DB)
├── Redis Deployment              (optional, for query caching)
│
└── cloudflared Deployment        (Cloudflare tunnel agent)
    └── routes:
        /api/* → query-chart-bridge Service :8085
        /*     → dashboard Service :80
```

**CI/CD:** Each repo has a GitHub Actions workflow (`.github/workflows/docker-publish.yml`) that builds and pushes images to `ghcr.io` on push to the default branch.

---

## Authentication & Security

```
Browser
  │
  ▼  HTTPS: https://app.yourdomain.com
Cloudflare Edge
  │  Cloudflare Access policy: Google OAuth
  │  On success: sets CF_Authorization cookie (HTTP-only, Secure)
  │
  ▼  Cloudflare Tunnel protocol
cloudflared pod
  │
  ├─ /api/* ──► query-chart-bridge
  │              │  Extracts Cf-Access-Jwt-Assertion header
  │              │  Validates RS256 signature against Cloudflare JWKS
  │              │  Validates AUD claim
  │              │  Sets gin context: user_email = JWT email claim
  │              │  Every SQL query prepended with:
  │              │    LOWER(email_origin_receiver) = LOWER('<user_email>')
  │              └─ (enforced=false → dev mode bypasses JWT check)
  │
  └─ /*   ──► dashboard (static Nginx)
               No auth logic needed — Cloudflare handles it at the edge
               Cookie is sent automatically by browser on every /api/* call
```

**Per-user data isolation** is enforced at the SQL layer inside `query-chart-bridge`. Every query execution injects a mandatory `email_origin_receiver` filter using the validated JWT email. This means even if a malicious request bypasses the JWT (in non-enforced mode), it can only read rows tied to the `test_email` config value.

---

## Banks & Transaction Types Supported

| Bank | Transaction Type | Email Subject |
|---|---|---|
| **UOB-SG** | Card debit | `UOB - Transaction Alert` |
| **UOB-SG** | PayNow / NETS QR outgoing | `UOB Personal Internet Banking Notification Alerts` |
| **UOB-SG** | NETS QR payment | `UOB - NETS QR payment made` |
| **UOB-SG** | Overseas QR | `UOB-Overseas QR payment made` |
| **UOB-SG** | Scheduled FAST transfer | `UOB-Scheduled FAST Transfer Status` |
| **UOB-SG** | Scheduled PayNow | `UOB-Scheduled Paynow Transfer Status` |
| **UOB-SG** | PayNow incoming | `UOB-PayNow transfer received` |
| **UOB-SG** | GIRO / fund transfer (debit/credit) | `UOB Transaction Alert` |
| **UOB-SG** | ATM withdrawal | `UOB-ATM Domestic Trns` |
| **UOB-SG** | Reversal | `Your transaction has been reversed` |
| **DBS-SG** | Card debit | `Card Transaction Alert` |
| **DBS-SG** | PayLah Scan & Pay | `Transaction Alert` |
| **DBS-SG** | PayNow / transfers | `Transaction Alerts`, `iBanking Alerts` |
| **DBS-SG** | PayNow incoming | `digibank Alerts - You've received a transfer` |
| **DBS-SG** | NETS Scan & Pay | `digibank Alert - Successful NETS Scan & Pay` |

Any bank that sends structured email notifications can be added by writing a YAML rule with a regex pattern.

---

## Tag Filter System

Tags are user-defined named sets of SQL filter conditions stored in the `tag_filters` MySQL table. They allow dashboard panels to segment transaction data without hardcoding SQL in the dashboard config.

**Schema:**

```
tag_filters (username, tag_name, filter_name, field_values JSON)
```

A sentinel row `(username, tag_name, '-', {})` represents the tag's existence. Each real filter row stores field–value pairs that translate to SQL conditions.

**String conditions support regex syntax:**

| Filter value | SQL generated |
|---|---|
| `"DBS-SG"` | `bank = 'DBS-SG'` |
| `"McD.*\|KFC"` | `recipient REGEXP '^McD.*\|KFC$'` |
| `1` (numeric) | `inbound = 1` |
| `true` (bool) | `inbound = 1` |

**Tag controller tri-state:**

| Value | Effect |
|---|---|
| `0` | Off — no filter applied |
| `1` | Include — only rows matching any tag filter |
| `-1` | Exclude — rows NOT matching any tag filter |

**Example flow:**

```
User creates tag "food":
  filter "fast-food": { bank: "DBS-SG", recipient: "McD.*|KFC|Burger King" }
  filter "coffee":    { bank: "UOB-SG", recipient: "Starbucks.*|%Coffee%" }

Dashboard request with tag "food" = 1:
  → SQL: ((bank = 'DBS-SG' AND recipient REGEXP '^McD.*|KFC|Burger King$')
          OR
          (bank = 'UOB-SG' AND recipient REGEXP '^Starbucks.*|%Coffee%$'))
```

---

## Caching Strategy

`query-chart-bridge` uses Redis for query result caching. Cache mode is set per-panel in the request, or automatically overridden by date controllers.

| `cache` value | TTL | When to use |
|---|---|---|
| `0` (default) | No cache | Always-fresh queries (current day) |
| `1` | 1 minute | Current month data (data can change) |
| `-1` | 30 days | Historical months (data is immutable) |

The cache key is `{cacheMode}/x/'{endpoint}'/{sha256(normalized_sql)}`. SQL normalization uppercases keywords and collapses whitespace so functionally identical queries share cache entries.

---

## Getting Started

### Prerequisites

- Docker + Kubernetes cluster (or local k3s/minikube)
- A Mailgun account with a receiving route pointing to your `email-parser` webhook URL
- Bank email alerts forwarding (or filtering) to the Mailgun-monitored inbox
- (Optional) A Cloudflare Zero Trust account for the single-domain authenticated deployment

### Local Development

```bash
# 1. Start MySQL and Redis
docker run -d --name mysql -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=email_parser_db -p 3306:3306 mysql:8
docker run -d --name redis -p 6379:6379 redis:alpine

# 2. email-parser
cd email-parser
go run . --config-path config/config.yml
# Webhook available at http://localhost:8083/webhook

# 3. query-chart-bridge
cd query-chart-bridge
go run .
# API available at http://localhost:8085

# 4. dashboard
cd dashboard
npm install
npm run dev
# Dashboard at http://localhost:5173
```

### Kubernetes Deployment

Each repo contains a `k8s-example.yaml` / `example-k8s.yaml` with a ready-to-adapt `Deployment + Service + ConfigMap`. Steps:

```bash
# 1. Create the namespace and secrets
kubectl create namespace tracker
kubectl -n tracker create secret generic db-creds \
  --from-literal=password=your_db_password

# 2. Deploy in order: database → email-parser → query-chart-bridge → dashboard
kubectl apply -n tracker -f email-parser/example-k8s.yaml
kubectl apply -n tracker -f query-chart-bridge/k8s-example.yaml
kubectl apply -n tracker -f dashboard/k8s-example.yaml

# 3. Create the Cloudflare tunnel and configure path-based routing:
#   /api/* → query-chart-bridge service :8085
#   /*     → dashboard service :80
```

---

## Configuration Reference

### email-parser (`config/config.yml`)

```yaml
port: 8083
db:
  username: email_parser
  password: <password>
  host: mysql
  port: 3306
  name: email_parser_db
parser:
  omitted-chars: ["*", "[", "]", "|", ...]   # chars stripped from email body before regex
  inbound-keywords:                           # values that mark a transaction as inbound
    received: receive
  subjects:
    "<Exact Email Subject>":
      type: "<Transaction Type Label>"
      bank: "<Bank Label>"
      patterns:
        - |-
          (?P<currency>[A-Z]+) (?P<quantity>[0-9.,]+) ...
      anti-patterns:
        - <regex to skip this subject entirely>
      samples:
        - |
          Example email body for documentation and testing
```

### query-chart-bridge (`config.yaml`)

```yaml
database:
  user: email_parser_reader
  password: <password>
  host: mysql
  port: 3306
  name: email_parser_db
redis:
  host: redis
  port: 6379
  password: ""
  db: 0
server:
  port: 8085
  debug: false
cloudflare:
  team_domain: "https://your-team.cloudflareaccess.com"
  aud: "<Application Audience Tag>"
  enforced: true
  test_email: dev@example.com       # used when enforced=false
controllers:
  base-target: email_transactions   # the root MySQL table
```

### dashboard (`public/app-config.json` / K8s ConfigMap)

```json
{
  "API_URL": "https://app.yourdomain.com/api",
  "loadTestData": false,
  "dashboards": [
    {
      "id": "<unique-id>",
      "title": "Dashboard Title ($month)",
      "headerTitle": "Header ($month)",
      "req": {
        "panels": [
          {
            "id": "<panel-id>",
            "title": "Panel Title",
            "chartType": "bar | line | pie | doughnut | table | ...",
            "query": "SELECT col1, col2 FROM $BASE ...",
            "color_distribution": "" | "by-value"
          }
        ],
        "controllers": [
          { "type": "day" },
          { "type": "month" },
          { "type": "year" },
          { "type": "tag" }
        ]
      }
    }
  ]
}
```

---
