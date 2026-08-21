# Himalaya Traders — Phase-wise Architecture

**Pair with:** [DEVELOPMENT_PLAN.md](./DEVELOPMENT_PLAN.md)  
**Stack:** React SPA ↔ Express REST (`/api/v1`) ↔ Prisma ↔ MySQL  
**Pattern:** 3-tier, feature modules, JWT + RBAC, transactional billing

Use this file to **see** what exists after each phase. Grey / missing boxes in a phase diagram are **not built yet**.

---

## How to read

| Symbol | Meaning |
|---|---|
| Solid box | Lives after that phase |
| Dashed box | Planned, not built yet |
| `FE` | React page / client |
| `BE` | Express module |
| `DB` | MySQL table |

---

## 0. Target architecture (end of Phase 13)

This is the **finished** system. Phases 0–12 grow toward this picture.

```mermaid
flowchart TB
  subgraph POS["POS counter machine"]
    Browser["React SPA + Tailwind<br/>Zustand cart · JWT in memory"]
    PrintUI["80mm print CSS / PDF"]
    Bridge["Optional: Node print-bridge<br/>localhost"]
    Printer["80mm ESC/POS thermal<br/>USB / Bluetooth"]
  end

  subgraph VPS["Cloud / VPS"]
    API["Express REST /api/v1"]
    MW["helmet · CORS · rate-limit<br/>JWT · RBAC · validate"]
    Mods["Modules: auth users products<br/>categories customers orders<br/>payments expenses reports settings"]
    ORM["Prisma"]
    DB[("MySQL")]
    Cron["Daily DB backup · 30 days"]
  end

  Cashier["Cashier / Admin"] --> Browser
  Browser -->|"HTTPS JSON + Bearer"| API
  API --> MW --> Mods --> ORM --> DB
  Browser --> PrintUI
  PrintUI -.->|"if USB unreliable"| Bridge --> Printer
  PrintUI -->|"fallback"| PDF["PDF download"]
  Cron --> DB
```

### Request path (every authenticated call)

```mermaid
sequenceDiagram
  participant UI as React SPA
  participant API as Express
  participant Auth as JWT + RBAC
  participant Svc as Module service
  participant DB as MySQL

  UI->>API: HTTPS /api/v1/...
  API->>Auth: verify access token
  alt token expired
    UI->>API: POST /auth/refresh (httpOnly cookie)
    API-->>UI: new access token
    UI->>API: retry original request
  end
  Auth->>Auth: role check (admin vs staff)
  Auth->>Svc: controller
  Svc->>DB: Prisma (parameterized)
  DB-->>UI: JSON
```

### Physical / deploy view

```mermaid
flowchart LR
  subgraph Counter["Shop POS laptop"]
    SPA[Chrome / Edge]
    T80[Thermal printer]
  end
  subgraph Host["VPS e.g. Hostinger / DigitalOcean"]
    Node[Node.js API]
    MySQL[(MySQL)]
  end
  SPA -->|"HTTPS"| Node
  Node --> MySQL
  SPA -->|"localhost print or browser print"| T80
```

---

## 1. Final folder architecture

```mermaid
flowchart TB
  subgraph FE["frontend/src"]
    P["pages/<br/>Login Dashboard POS Invoices<br/>Inventory Customers Reports<br/>Expenses Settings"]
    C["components/ui · layout · pos"]
    Z["store/cartStore Zustand"]
    A["api/client + interceptors"]
    PR["services/printer.js"]
    R["routes ProtectedRoute RoleRoute"]
  end

  subgraph BE["backend/src"]
    APP[app.js]
    MD["middleware/<br/>auth rbac validate errorHandler"]
    M["modules/*  controller + service + routes"]
    U["utils/<br/>jwt billing invoiceNumber"]
  end

  P --> A --> APP
  APP --> MD --> M --> Prisma["prisma/schema.prisma"]
  PR -.-> Print["printer / PDF"]
```

---

## 2. Data model (complete ER)

Built table-by-table across phases. This is the **final** schema.

```mermaid
erDiagram
  users ||--o{ orders : creates
  users ||--o{ stock_movements : logs
  users ||--o{ expenses : records
  categories ||--o{ products : contains
  products ||--o{ stock_movements : history
  products ||--o{ order_items : sold_as
  customers ||--o{ orders : billed_to
  customers ||--o{ payments : dues
  orders ||--|{ order_items : lines
  orders ||--o{ payments : split_or_full
  orders ||--o{ stock_movements : sale_or_void
  settings ||--|| settings : key_value

  users {
    int id PK
    string email UK
    string role
    string pin_hash
    boolean is_active
  }
  categories {
    int id PK
    string name UK
  }
  products {
    int id PK
    string sku UK
    int category_id FK
    decimal selling_price
    int stock_qty
    boolean is_active
  }
  stock_movements {
    int id PK
    int product_id FK
    int change_qty
    string reason
    int reference_order_id FK
  }
  customers {
    int id PK
    string phone UK
    decimal due_balance
  }
  orders {
    int id PK
    string invoice_no UK
    int customer_id FK
    decimal grand_total
    string payment_mode
    string status
  }
  order_items {
    int id PK
    int order_id FK
    int product_id FK
    string item_type
    decimal line_total
  }
  payments {
    int id PK
    int order_id FK
    int customer_id FK
    decimal amount
    string mode
  }
  expenses {
    int id PK
    string category
    decimal amount
    date spent_at
  }
  settings {
    string key PK
    text value
  }
```

---

## 3. Module dependency graph (build order)

Arrows = “must exist first”.

```mermaid
flowchart TB
  P0[Phase 0 Foundation]
  P1[Phase 1 Auth]
  P2[Phase 2 Users]
  P3[Phase 3 Settings]
  P4[Phase 4 Inventory]
  P5[Phase 5 Customers]
  P6[Phase 6 POS / Orders]
  P7[Phase 7 Invoices]
  P8[Phase 8 Due payments]
  P9[Phase 9 Dashboard]
  P10[Phase 10 Expenses]
  P11[Phase 11 Reports]
  P12[Phase 12 Printer]
  P13[Phase 13 Go-live]

  P0 --> P1
  P1 --> P2
  P1 --> P3
  P1 --> P5
  P1 --> P10
  P3 --> P4
  P3 --> P6
  P4 --> P6
  P5 --> P6
  P6 --> P7
  P6 --> P8
  P5 --> P8
  P6 --> P9
  P4 --> P9
  P6 --> P11
  P10 --> P11
  P7 --> P12
  P9 --> P13
  P11 --> P13
  P12 --> P13
```

---

## 4. Phase-wise architecture (what you can see after each phase)

Each diagram is **cumulative**: earlier boxes stay; new boxes are marked `NEW`.

---

### PHASE 0 — Foundation

Empty runnable skeleton. No login, no business data.

```mermaid
flowchart TB
  subgraph FE["Frontend — NEW"]
    Vite["Vite + React + Tailwind"]
    UIKit["Button Input Modal Table Toast"]
    Client["api/client.js  base /api/v1"]
  end
  subgraph BE["Backend — NEW"]
    Express["Express app.js"]
    Health["GET /health"]
    MW0["CORS helmet JSON parser"]
    Prisma0["Prisma connected"]
  end
  subgraph DB["MySQL"]
    Empty[("no business tables yet")]
  end
  Vite --> Client --> Express --> Health
  Express --> MW0
  Prisma0 --> Empty
```

| Layer | Exists | Does not exist |
|---|---|---|
| FE | App shell, UI kit, API client stub | Pages, cart, routes |
| BE | Health, env, error handler | Auth, modules |
| DB | Connection only | All tables |

---

### PHASE 1 — Authentication

First real feature. JWT session. Role guards (empty pages still 404 except Login).

```mermaid
flowchart TB
  subgraph FE["Frontend"]
    Login["Login page — NEW"]
    Guard["ProtectedRoute — NEW"]
    Mem["Access token in memory — NEW"]
    Idle["POS idle timer stub — NEW"]
  end
  subgraph BE["Backend"]
    AuthM["modules/auth — NEW"]
    JWT["JWT access 1h + refresh 7d — NEW"]
    RBAC["rbac middleware — NEW"]
    RL["Login rate-limit — NEW"]
  end
  subgraph DB["MySQL"]
    Users[("users — NEW")]
  end
  Login -->|"POST /auth/login"| AuthM --> JWT
  Guard -->|"Bearer"| RBAC
  AuthM --> Users
```

```mermaid
sequenceDiagram
  participant C as Cashier
  participant L as Login.jsx
  participant A as POST /auth/login
  participant U as users table

  C->>L: email + password
  L->>A: credentials
  A->>U: bcrypt compare
  U-->>A: role admin/staff
  A-->>L: access JWT + refresh cookie
  L->>L: redirect /dashboard (placeholder until P9)
```

| New APIs | New tables | New screens |
|---|---|---|
| `POST /auth/login` `refresh` `logout` `verify-pin` | `users` | Login |

---

### PHASE 2 — Users & Admin PIN

Staff accounts live inside Settings (Users tab). PIN hash on `users`.

```mermaid
flowchart LR
  subgraph FE
    UsersTab["Settings → Users tab — NEW"]
  end
  subgraph BE
    UsersM["modules/users — NEW"]
    AuthM["modules/auth"]
  end
  subgraph DB
    Users[("users + pin_hash")]
  end
  UsersTab --> UsersM --> Users
  AuthM -->|"verify-pin"| Users
```

| New APIs | Screens |
|---|---|
| `GET/POST /users` `PUT /users/:id` PIN + password reset | Settings Users |

---

### PHASE 3 — Settings (config store)

Billing **must not hardcode** GST / rates. Key-value `settings` table.

```mermaid
flowchart TB
  subgraph FE
    S1["Tab GST & Billing — NEW"]
    S2["Tab Shop Info — NEW"]
    S3["Tab Printer — NEW"]
    S4["Tab Users"]
  end
  subgraph BE
    SetM["modules/settings — NEW"]
  end
  subgraph DB
    Set[("settings key/value JSON — NEW")]
  end
  S1 --> SetM
  S2 --> SetM
  S3 --> SetM
  SetM --> Set
```

POS later **reads** these keys: `gst_percent`, `gst_split`, `staff_discount_cap_percent`, flex/card rates, invoice prefix, auto-print.

---

### PHASE 4 — Inventory

Product master + stock ledger. Staff = view only (RBAC on BE + hidden buttons on FE).

```mermaid
flowchart TB
  subgraph FE
    Inv["Inventory list — NEW"]
    ProdModal["Add/Edit Product modal"]
    Adj["Adjust Stock modal"]
    Hist["Stock History drawer"]
  end
  subgraph BE
    Cat["modules/categories — NEW"]
    Prod["modules/products — NEW"]
  end
  subgraph DB
    C[("categories — NEW")]
    P[("products — NEW")]
    SM[("stock_movements — NEW")]
  end
  Inv --> Prod --> P
  Inv --> Cat --> C
  Adj --> Prod
  Prod --> SM
  P --> C
```

**Stock write rules (from this phase on):** never change `products.stock_qty` without a `stock_movements` row.

| Reason enum | Who writes it | Phase |
|---|---|---|
| `purchase` / `damage` / `adjustment` | Admin adjust modal | 4 |
| `sale` | POS generate invoice | 6 |
| `void` | Admin void invoice | 7 |

---

### PHASE 5 — Customers

Master data for POS auto-fill (phone). Dues column exists; **payments in Phase 8**.

```mermaid
flowchart LR
  subgraph FE
    CL["Customers list — NEW"]
    CD["Customer detail — NEW"]
  end
  subgraph BE
    CM["modules/customers — NEW"]
  end
  subgraph DB
    Cust[("customers — NEW<br/>due_balance = 0")]
  end
  CL --> CM --> Cust
  CD --> CM
```

POS (next phase) will `GET /customers?search=` on phone debounce.

---

### PHASE 6 — POS / Billing  ★ critical path

First money flow. **One DB transaction:** order + items + stock + payment + due.

```mermaid
flowchart TB
  subgraph FE["POS screen"]
    Search["Product search 300ms"]
    Cart["Zustand cart"]
    Flex["Flex / Card modals"]
    Pay["Cash UPI Card Split Credit"]
    Gen["Generate Invoice"]
  end
  subgraph BE
    Ord["modules/orders — NEW"]
    Bill["utils/billing.js — NEW"]
    InvNo["utils/invoiceNumber.js — NEW"]
    Pin["POST /auth/verify-pin"]
  end
  subgraph TX["MySQL TRANSACTION"]
    O[("orders — NEW")]
    OI[("order_items — NEW")]
    PayT[("payments — NEW")]
    SM[("stock_movements sale")]
    P[("products.stock_qty --")]
    C[("customers.due_balance ++ if credit")]
  end
  Search --> Cart --> Gen
  Flex --> Cart
  Pay --> Gen
  Gen --> Ord
  Ord --> Bill
  Ord --> InvNo
  Ord --> TX
  Gen -.->|"discount over cap"| Pin
```

```mermaid
sequenceDiagram
  participant POS as POS UI
  participant API as POST /orders
  participant DB as MySQL txn

  POS->>API: cart + customer + pay mode
  API->>API: calc subtotal discount GST round
  API->>DB: BEGIN
  API->>DB: insert orders (invoice_no)
  API->>DB: insert order_items snapshots
  API->>DB: stock_movements sale + decrement stock
  alt credit / partial
    API->>DB: increment customer.due_balance
  end
  API->>DB: insert payments
  alt any step fails
    DB-->>API: ROLLBACK
    API-->>POS: 4xx/5xx no partial bill
  else
    DB-->>API: COMMIT
    API-->>POS: invoice_no
  end
```

**Line types inside `order_items`**

| `item_type` | `product_id` | Stock |
|---|---|---|
| `product` | set | Deduct |
| `flex_print` | null | No |
| `card_print` | null | No |

---

### PHASE 7 — Invoice management

Read model + admin void (reverse of Phase 6 txn).

```mermaid
flowchart TB
  subgraph FE
    List["Invoices list + filters — NEW"]
    Det["Invoice detail / preview — NEW"]
    VoidBtn["Void modal Admin only"]
    Exp["Export PDF / Excel"]
  end
  subgraph BE
    Ord["modules/orders"]
    Void["POST /orders/:id/void — NEW"]
  end
  subgraph DB
    O[("orders status voided")]
    SM[("stock_movements reason=void")]
    C[("due_balance reversed")]
  end
  List --> Ord
  Det --> Ord
  VoidBtn --> Void --> DB
  Exp --> Ord
```

Reprint **does not** re-price. It prints `order_items` snapshot.

---

### PHASE 8 — Due payments

Standalone payment (no new invoice).

```mermaid
flowchart LR
  subgraph FE
    Modal["Collect Due modal — NEW"]
  end
  subgraph BE
    PayM["modules/payments — NEW"]
  end
  subgraph DB
    Pay[("payments order_id NULL")]
    Cust[("customers.due_balance --")]
  end
  Modal -->|"POST /customers/:id/payments"| PayM
  PayM --> Pay
  PayM --> Cust
```

---

### PHASE 9 — Dashboard

Landing page. Poll every **60s**.

```mermaid
flowchart TB
  subgraph FE
    Dash["Dashboard — NEW"]
    Cards["Today / Month sales · orders"]
    Low["Low-stock list + bell"]
    Top["Top 5 products"]
    Dues["Pending dues ₹"]
    CTA["New Bill → /pos"]
  end
  subgraph BE
    Rpt["GET /reports/dashboard — NEW"]
  end
  Dash -->|"load + 60s poll"| Rpt
  Rpt --> O[("orders")]
  Rpt --> P[("products")]
  Rpt --> C[("customers")]
```

---

### PHASE 10 — Expenses

Admin-only. Feeds P&L in Phase 11.

```mermaid
flowchart LR
  FE["Expenses page — NEW"] --> BE["modules/expenses — NEW"] --> DB[("expenses — NEW")]
```

Staff: route hidden + API 403.

---

### PHASE 11 — Reports & Analytics

```mermaid
flowchart TB
  subgraph FE
    Tabs["Reports tabs — NEW"]
    Sales["Sales daily/monthly"]
    ProdR["Top / least products"]
    CustR["Customer-wise"]
    PL["Profit / Loss"]
  end
  subgraph BE
    RM["modules/reports — NEW"]
  end
  Tabs --> RM
  RM --> O[("orders + items")]
  RM --> E[("expenses")]
  RM --> P[("products cost_price")]
```

**P&L formula:** `sales − expenses − COGS`  
**COGS:** `sum(cost_price × qty sold)` — store cost on the line at sale time (Phase 6 snapshot).

Staff: **Sales tab, today only**. Other tabs hidden; APIs 403.

---

### PHASE 12 — Thermal print

Print is a **client-side** concern. API only stores the invoice snapshot.

```mermaid
flowchart TB
  subgraph POS_PC["POS machine"]
    UI["Invoice detail / POS success"]
    Svc["services/printer.js — NEW"]
    CSS["80mm @page CSS"]
    PDF["PDF fallback"]
    Bridge["node-thermal-printer bridge — optional"]
    HW["USB / Bluetooth printer"]
  end
  UI --> Svc
  Svc --> CSS
  CSS -->|"window.print"| HW
  Svc --> PDF
  Svc -.-> Bridge --> HW
```

| Path | When |
|---|---|
| Browser print 80mm | Default v1 |
| Localhost print-bridge | If USB/Bluetooth from browser is unreliable |
| PDF download | No printer |

---

### PHASE 13 — Production topology

```mermaid
flowchart TB
  subgraph Staging
    SAPI[API staging]
    SDB[(MySQL staging)]
  end
  subgraph Prod
    PAPI[API production HTTPS]
    PDB[(MySQL)]
    BAK["Daily dump 30-day retain — NEW"]
    MON["Uptime monitor 99.5% — NEW"]
  end
  ClientReview[Client sign-off] --> Staging
  Staging --> Prod
  PAPI --> PDB
  BAK --> PDB
  MON --> PAPI
```

---

## 5. Layer view by phase (cheat sheet)

Read left → right. `●` = exists. Blank = not yet.

| Layer / piece | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| React shell + UI kit | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| Login + JWT + RBAC | | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| Users / PIN | | | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| Settings | | | | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| Inventory | | | | | ● | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| Customers | | | | | | ● | ● | ● | ● | ● | ● | ● | ● | ● |
| POS + orders txn | | | | | | | ● | ● | ● | ● | ● | ● | ● | ● |
| Invoice list / void | | | | | | | | ● | ● | ● | ● | ● | ● | ● |
| Due collect | | | | | | | | | ● | ● | ● | ● | ● | ● |
| Dashboard | | | | | | | | | | ● | ● | ● | ● | ● |
| Expenses | | | | | | | | | | | ● | ● | ● | ● |
| Reports / P&L | | | | | | | | | | | | ● | ● | ● |
| Thermal / PDF | | | | | | | | | | | | | ● | ● |
| Staging / backups | | | | | | | | | | | | | | ● |

---

## 6. Frontend architecture (final)

```mermaid
flowchart TB
  subgraph Routes
    Pub["/login"]
    AuthG["ProtectedRoute"]
    AdminG["RoleRoute admin"]
  end

  Pub --> Login
  AuthG --> Dash["/dashboard"]
  AuthG --> POS["/pos"]
  AuthG --> Inv["/invoices"]
  AuthG --> Stock["/inventory"]
  AuthG --> Cust["/customers"]
  AuthG --> Rep["/reports"]
  AdminG --> Exp["/expenses"]
  AdminG --> Set["/settings"]

  POS --> Zustand["cartStore"]
  Zustand --> API["api/client interceptor"]
  POS --> Printer["printer.js"]
```

**State split**

| State | Where | Why |
|---|---|---|
| Access JWT | Memory | XSS: not localStorage |
| Refresh token | httpOnly cookie | Auto refresh |
| Cart | Zustand | POS-only, discard after bill |
| Settings | React query / load on login | GST, rates, shop |
| Role | JWT payload | UI hide only; BE enforces |

---

## 7. Backend architecture (final)

Every module: `routes → controller → service → Prisma`.

```mermaid
flowchart LR
  R[Route] --> C[Controller]
  C --> V[validate]
  C --> A[auth + rbac]
  C --> S[Service]
  S --> P[Prisma]
  S --> U[utils billing / invoiceNo]
```

| Module | Owns tables | Critical rule |
|---|---|---|
| `auth` | — (uses users) | Rate-limit login |
| `users` | users | PIN hashed |
| `settings` | settings | Admin only |
| `categories` | categories | Unique name |
| `products` | products, stock_movements | Soft delete |
| `customers` | customers | Phone unique |
| `orders` | orders, order_items | **Single transaction** |
| `payments` | payments | Due never &lt; 0 |
| `expenses` | expenses | Admin only |
| `reports` | read-only | Staff today-only |

---

## 8. Security architecture

```mermaid
flowchart TB
  HTTPS[HTTPS only]
  Login[Login rate-limit]
  Hash[bcrypt ≥ 10]
  JWT[Short access + refresh]
  FE[FE route guards — UX only]
  BE[BE middleware — real enforcement]
  SQL[Parameterized Prisma]
  Audit[stock / void / PIN override logs]

  HTTPS --> Login --> Hash --> JWT
  JWT --> FE
  JWT --> BE
  BE --> SQL
  BE --> Audit
```

Staff **must** get 403 on: product write, stock adjust, void, expenses, settings, full reports — even if they call the API from Postman.

---

## 9. POS billing pipeline (architecture of the calculation)

```mermaid
flowchart LR
  Lines[Cart lines] --> Sub[Subtotal]
  Sub --> Disc[Discount ₹ or %]
  Disc --> GST[GST or CGST+SGST]
  GST --> Round[Nearest rupee]
  Round --> Pay{Payment}
  Pay -->|cash upi card| Paid[payments row]
  Pay -->|split| Split[N payment rows sum = total]
  Pay -->|credit| Due[due_balance += unpaid]
```

Default until client sign-off: **discount before GST**, split CGST+SGST on.

---

## 10. After each phase — what a developer can demo

| Phase | Visible architecture | Demo |
|---|---|---|
| 0 | Empty 3-tier | `GET /health` + blank React |
| 1 | Auth wall | Login as seeded admin |
| 2 | Users module | Create a staff user |
| 3 | Config plane | Save GST 18%, shop name |
| 4 | Inventory plane | Add product, adjust stock, see history |
| 5 | CRM plane | Add customer by phone |
| 6 | Commerce plane | Walk-in cash bill, stock drops |
| 7 | Audit plane | List invoice, reprint, admin void |
| 8 | Ledger plane | Collect due, balance drops |
| 9 | Ops plane | Dashboard cards + New Bill |
| 10 | Cost plane | Add rent expense |
| 11 | Analytics plane | P&L for a date range |
| 12 | Device plane | 80mm print or PDF |
| 13 | Production plane | Staging + backup + HTTPS |

---

*Architecture grows only in this order. Do not add Reports or Printer before POS — those layers have nothing to read.*
