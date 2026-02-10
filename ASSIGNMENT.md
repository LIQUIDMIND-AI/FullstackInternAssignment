# 🚀 The Assignment: "TradeBoard — Export Operations Command Center"

## Deadline: February 13th, 2026 (End of Day)
## Evaluation Call: February 14th, 2026 (30-minute call)
## You MAY use AI tools — but read the [note at the bottom](#-a-note-on-using-ai).

---

## The Problem

An Indian exporter manages **200+ shipments/month** across 15 countries. Today, their workflow looks like this:

- Shipment data lives in **Excel sheets** emailed between teams
- The CHA (Customs House Agent) sends updates via **WhatsApp**
- Payment status is tracked in a **separate bank portal**
- Nobody knows which shipments are stuck, which payments are overdue, or which documents are missing — until someone manually checks
- The Operations Head starts every Monday with **2 hours of "where are we?"** meetings because there's no single source of truth

**Build TradeBoard** — a full-stack web application that serves as the **single command center** for an exporter's operations team.

---

## Day-by-Day Suggested Breakdown

| Day | Focus | Hours |
|-----|-------|-------|
| Day 1 | Understand the domain, data modeling, database setup, seed data | ~5 hrs |
| Day 2 | Backend APIs + business logic (status engine, alerts, filters) | ~5 hrs |
| Day 3 | Frontend — dashboard, shipment list, shipment detail | ~5 hrs |
| Day 4 | Create shipment form, connect everything, deploy, DESIGN_DECISIONS.md | ~5 hrs |

This is a suggestion, not a requirement. Work however you want.

---

## The Data Model

Design your database around these entities. Use **any database** — SQLite is perfectly fine. Postgres, MySQL, MongoDB — whatever you're comfortable with. **Justify your choice in DESIGN_DECISIONS.md.**

### Entity 1: `Buyer` (8-10 rows in seed data)

> **Schema definition:** [`schemas/buyer_schema.csv`](schemas/buyer_schema.csv)

### Entity 2: `Product` (12-15 rows in seed data)

> **Schema definition:** [`schemas/product_schema.csv`](schemas/product_schema.csv)

### Entity 3: `Shipment` (Core entity — 80-100 rows in seed data)

> **Schema definition:** [`schemas/shipment_schema.csv`](schemas/shipment_schema.csv)

### Entity 4: `Document`

Each shipment needs multiple documents. Track what's done and what's missing.

> **Schema definition:** [`schemas/document_schema.csv`](schemas/document_schema.csv)

### Entity 5: `ShipmentEvent` (Activity log)

Every status change or update gets logged automatically.

> **Schema definition:** [`schemas/shipment_event_schema.csv`](schemas/shipment_event_schema.csv)

---

## Shipment Status Machine

Shipments flow through these stages. **Your backend must enforce valid transitions only.**

```
ORDER_RECEIVED
    ↓
DOCUMENTS_IN_PROGRESS
    ↓
DOCUMENTS_READY
    ↓
CUSTOMS_FILED
    ↓
CUSTOMS_CLEARED ←───── CUSTOMS_HELD (can go back to CUSTOMS_FILED)
    ↓                        ↑
SHIPPED                 (from CUSTOMS_FILED)
    ↓
IN_TRANSIT
    ↓
ARRIVED
    ↓
DELIVERED
    ↓
COMPLETED
```

Also: any shipment can be moved to `CANCELLED` from any state.

**If someone tries to skip a step** (e.g., ORDER_RECEIVED → SHIPPED), the API must return a **400 error** with a message like:
> `"Invalid transition: cannot move from ORDER_RECEIVED to SHIPPED. Allowed next states: DOCUMENTS_IN_PROGRESS, CANCELLED"`

This is an important backend test. Don't skip it.

---

## Seed Data Requirements

Write a seed script that fills the database with realistic data. The data should tell a story — when the Operations Head opens the dashboard, they should immediately see a mix of healthy shipments and problems.

**Include these realistic problems in your seed data:**

> **Full problem definitions:** [`schemas/seed_data_problems.csv`](schemas/seed_data_problems.csv)

| Problem | How Many | Why It Matters |
|---------|----------|---------------|
| Overdue payments | 5-6 shipments | `days_to_payment` way past `payment_terms`, `payment_status = overdue` |
| Stuck in customs | 3-4 shipments | `status = CUSTOMS_HELD` for 5+ days, `customs_status = rejected` or `under_query` |
| Missing documents | 6-8 shipments | Status is advanced (CUSTOMS_FILED or beyond) but some documents still `not_started` or `draft` |
| Departure soon, not ready | 3-4 shipments | `estimated_departure` within 3 days but status is still DOCUMENTS_IN_PROGRESS or earlier |
| Arrival overdue | 2-3 shipments | Past `estimated_arrival` + 7 days but still showing IN_TRANSIT |

---

## Backend Requirements

### Tech: Python 3.11+, FastAPI (preferred) or Flask

### API Endpoints:

> **Full API specifications:** [`schemas/api_endpoints.csv`](schemas/api_endpoints.csv)

**Shipments**
- `GET /api/shipments` - List shipments with filters + pagination
- `GET /api/shipments/{id}` - Full shipment detail (include documents + events)
- `POST /api/shipments` - Create new shipment
- `PATCH /api/shipments/{id}` - Update shipment fields
- `POST /api/shipments/{id}/status` - Change shipment status (state machine enforced)

**Documents**
- `GET /api/shipments/{id}/documents` - List documents for a shipment
- `PATCH /api/documents/{id}` - Update document status

**Dashboard**
- `GET /api/dashboard/summary` - Aggregated numbers for dashboard cards
- `GET /api/dashboard/alerts` - List of action items / problems

**Analytics**
- `GET /api/analytics/shipments-by-status` - Count of shipments per status (for chart)
- `GET /api/analytics/monthly-trend` - Shipment count + total value per month (for chart)

**Filters on `GET /api/shipments`** (implement at least these):
- `status` (single or multi)
- `buyer_id`
- `payment_status`
- `customs_status`
- `date_from`, `date_to`
- `search` (text search across shipment_number, buyer name, product description)
- `page`, `page_size`
- `sort_by`, `sort_order`

### What We Expect From the Backend:

1. **State machine enforcement** — invalid transitions → 400 error with helpful message
2. **Automatic event logging** — when status changes or documents update, a `ShipmentEvent` is created automatically by the backend, not manually by the frontend
3. **Alert logic** — `/api/dashboard/alerts` should compute and return:
   - Payments overdue beyond terms + 30 days
   - Shipments in CUSTOMS_HELD for 5+ days
   - Shipments with departure in ≤ 3 days but missing documents
   - Shipments past estimated arrival + 7 days still IN_TRANSIT
4. **Input validation** — proper Pydantic/schema validation, meaningful error messages
5. **Proper HTTP status codes** — 201 for creation, 400 for bad input, 404 for not found

---

## Frontend Requirements

### Tech: React + TypeScript. Use any UI library you like — Shadcn, Ant Design, MUI, Chakra, Mantine, plain Tailwind. Your choice.

### Page 1: Dashboard

The Operations Head's Monday morning view.

**Top row — Summary cards:**
- Total active shipments (not COMPLETED/CANCELLED)
- Shipments requiring action (alerts count)
- Total FOB value in pipeline (sum of active shipments)
- Overdue payments (count + total value)

**Alerts panel:**
A list of problems, color-coded by severity. Each one clickable → goes to that shipment's detail page.
- 🔴 Critical: Overdue payments, customs rejections
- 🟡 Warning: Missing documents with departure approaching, customs held
- 🟢 Info: Arrivals pending confirmation

**At least 2 charts:**
- Shipments by status (bar chart or donut)
- Monthly trend — shipment count or value over last 3 months (line or bar chart)

Use any charting library: Recharts, Chart.js, Nivo, Victory — whatever you prefer.

### Page 2: Shipments List

A filterable, sortable data table.

- Filter dropdowns/inputs for: status, buyer, payment status, customs status, date range, search
- Sortable columns
- Pagination
- Status shown as **colored badges**
- Payment status with color indicator (🟢 received, 🟡 partial/pending, 🔴 overdue)
- Click any row → navigate to shipment detail page

### Page 3: Shipment Detail

Everything about one shipment.

**Section 1 — Header:**
Shipment number, buyer name, country, status badge, key dates

**Section 2 — Shipment Info:**
All fields, grouped logically (not a giant unformatted list):
- Trade details (product, quantity, price, FOB, incoterm)
- Logistics (ports, shipping line, container, vessel, transit)
- Financial (payment terms, payment status, drawback)

**Section 3 — Documents Checklist:**
Visual list of all 7 document types with status:
- ✅ Tax Invoice — Verified
- ✅ Packing List — Ready
- ⬜ Bill of Lading — Not Started
- 🔄 Shipping Bill — Draft
- ...

Allow changing document status via dropdown or buttons on this page.

**Section 4 — Timeline:**
Activity log showing shipment history, most recent first:
```
📄 Feb 14 — Certificate of Origin status → submitted
🔄 Feb 13 — Status changed: ORDER_RECEIVED → DOCUMENTS_IN_PROGRESS
📄 Feb 12 — Packing List status → ready
📄 Feb 11 — Tax Invoice status → ready
📦 Feb 10 — Shipment created
```

**Section 5 — Status Action:**
A button or dropdown to advance to the next valid status. Only show valid transitions. When clicked, the backend validates and the timeline updates.

### Page 4: Create Shipment

A form to create a new shipment:
- Select buyer (dropdown — populated from `/api` or seeded data)
- Select product (dropdown)
- Enter: quantity, unit_price (auto-calculate `total_fob = quantity × unit_price` live as they type)
- Select: incoterm, port of loading, port of discharge, shipping line, container type, freight forwarder, CHA
- Enter: freight cost, insurance, payment terms, estimated departure date
- Submit → calls `POST /api/shipments` → redirects to the new shipment's detail page

**Validation:**
- Required fields must be enforced
- `total_fob` auto-calculates — user doesn't type it
- Sensible defaults where possible (e.g., port_of_discharge pre-fills based on buyer's country)

### Frontend Expectations:

1. **Loading states** — show spinners or skeletons while data loads. No blank screens.
2. **Error handling** — if an API fails, show a message. Not a white screen or console error.
3. **Empty states** — if filters return 0 results, show "No shipments found" with a suggestion to clear filters.
4. **Filter persistence** — if user filters the shipment list, clicks into a detail page, and presses browser back, the filters should still be there. (Use URL params, state management, or any approach — just make it work.)
5. **Clean, professional UI** — it doesn't need to be designer-level beautiful. But it should look like a real product, not a homework project. Consistent spacing, proper alignment, readable typography.

**Tip:** For rapid frontend development, consider using AI-powered tools like [Lovable](https://lovable.dev), [Bolt](https://bolt.new), or [v0](https://v0.dev) to accelerate your UI development.

---

## Deployment

Deploy the app (backend + frontend) so we can access it via a **single URL**.

Free options:
- **Render** — free tier supports Python backend + static frontend
- **Railway** — free tier, supports full stack
- **Vercel** (frontend) + **Render** (backend) — common combo
- **Any other free platform**

Local Development:
- **ScreenRecording** — Showcase your development in local environment.

Database: SQLite is fine for deployment (it's a demo). If you use Postgres, Railway/Render offer free tiers.

**Submit the live URL in your README.**

---

## What You Must Submit

A **GitHub repository** (public or shared with us) containing:

```
tradeboard/
├── backend/
│   ├── main.py                    # App entry point
│   ├── models.py                  # Database models
│   ├── schemas.py                 # Request/response schemas
│   ├── routes/                    # API route files
│   ├── services/                  # Business logic (status machine, alerts)
│   ├── seed.py                    # Seed data script
│   ├── tests/                     # At least 5 tests
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── pages/                 # Dashboard, ShipmentsList, ShipmentDetail, CreateShipment
│   │   ├── components/            # Reusable components
│   │   └── App.tsx
│   └── package.json
├── DESIGN_DECISIONS.md
└── README.md                      # Setup instructions + deployed URL
```

This structure is a suggestion. Organize however makes sense — but it should be clear and navigable.

---

## `DESIGN_DECISIONS.md`

Answer these questions. Be honest and concise.

1. **What database did you use and why?** What trade-offs did you consider?

2. **How did you implement the status machine?** Describe your approach. Why this way vs. other options?

3. **How does the alert system work?** Does it compute alerts on every request? Cache them? Something else? Why?

4. **What was the hardest bug you encountered?** Describe the bug, how you found it, how you fixed it.

5. **How did you handle filter persistence** on the shipments list page? What approach did you use and why?

6. **What would break first if this had 10,000 shipments instead of 100?** How would you fix it?

7. **What are you most proud of? What would you improve with 2 more days?**

---

## Evaluation Criteria

> **Full criteria table:** [`schemas/evaluation_criteria.csv`](schemas/evaluation_criteria.csv)

| Criteria | Weight | What We're Looking For |
|----------|--------|----------------------|
| **Does it work?** | 15% | Deployed URL loads. We can navigate all 4 pages. Creating a shipment works. Status changes work. |
| **Backend quality** | 25% | State machine enforced. Alerts are correct. Filters work. Proper validation and error messages. Tests pass. |
| **Frontend quality** | 30% | Dashboard is informative. Shipment list filters work. Detail page is complete and useful. Create form validates and auto-calculates. Loading/error/empty states handled. Looks professional. |
| **Data model & seed data** | 10% | Schema makes sense. Seed data is realistic. The problems planted are visible on the dashboard. |
| **Design Decisions doc** | 20% | Shows real understanding. Honest about challenges. Can explain why, not just what. |

---

## A Note on Using AI

**Use AI tools freely.** ChatGPT, Copilot, Claude, v0, Bolt, Cursor — whatever makes you productive. We encourage it. It's how real engineers work today.

**But after submission, we will do a 30-minute call where we ask:**

- *"Show me what happens when I try to skip a status. Where's the code that prevents it?"*
- *"How does this alert know the payment is overdue? Walk me through the logic."*
- *"Why did you choose this charting library?"*
- *"This filter isn't working for X — debug it live with me."*
- *"If I add a new document type tomorrow, what do you need to change?"*

If you can answer confidently, you pass. If you can't explain your own code, it won't matter how polished it looks.

**Use AI as your pair programmer. Understand everything it writes for you. That's the skill we're testing.**

Good luck. Build something you're proud of. 🚀
