# Himalaya Traders — Phase-wise Development Plan

**Project:** Billing, Inventory & POS Management System  
**Stack:** React.js + Tailwind CSS · Node.js + Express · Prisma · MySQL · JWT  
**Audience:** Frontend, Backend, QA — sequential build order  
**Source:** Himalaya Traders SRS v2.0 (Developer Reference Edition)

---

## How to use this document

Developers **must complete a phase before starting the next**, unless a phase is marked *can run in parallel*. Each phase lists:

- What to build
- Database tables
- API endpoints
- Screens, fields, and **every button**
- Role visibility (Admin / Staff)
- Done-when (acceptance)

Out of current scope (do **not** build now): barcode scanner, native mobile app, WhatsApp invoice sharing, online ordering portal.

---

## 1. Project snapshot

| Item | Detail |
|---|---|
| App type | Web POS, responsive, min. 1280×720 |
| Roles | `admin`, `staff` |
| Core job | GST invoice + live inventory + customer dues + expenses + reports |
| Printer | 80mm ESC/POS thermal (USB/Bluetooth) + PDF fallback |
| Auth | JWT access 1 hr + refresh 7 days; auto-logout 30 min idle on POS |
| Hosting | VPS (API + MySQL) |

### 1.1 Module map (build order)

| # | Module | Screens | Depends on |
|---|---|---|---|
| 1 | Foundation | — | — |
| 2 | Authentication | Login | Foundation |
| 3 | Users & RBAC | Settings → Users tab | Auth |
| 4 | Settings (core) | Settings | Auth |
| 5 | Categories & Inventory | Inventory | Settings |
| 6 | Customers | Customers | Auth |
| 7 | POS / Billing | POS | Inventory + Customers + Settings |
| 8 | Invoice Management | Invoices | POS |
| 9 | Due Payments | Customer detail | Customers + POS |
| 10 | Dashboard | Dashboard | POS + Inventory + Customers |
| 11 | Expenses | Expenses | Auth |
| 12 | Reports & Analytics | Reports | POS + Expenses + Inventory |
| 13 | Thermal Print | Print preview / auto-print | Invoices |
| 14 | QA, Staging, Go-live | — | All modules |

### 1.2 Sidebar / navigation (final app)

| Menu item | Route | Admin | Staff | Icon suggestion |
|---|---|---|---|---|
| Dashboard | `/dashboard` | Yes | Yes | Home |
| New Bill (POS) | `/pos` | Yes | Yes | Plus / Cart |
| Invoices | `/invoices` | Yes | Yes | File |
| Inventory | `/inventory` | Yes | View only | Box |
| Customers | `/customers` | Yes | Yes | Users |
| Reports | `/reports` | Full | Daily sales only | Chart |
| Expenses | `/expenses` | Yes | Hidden | Wallet |
| Settings | `/settings` | Yes | Hidden | Gear |
| Logout | — | Yes | Yes | Log-out |

**Header (all authenticated pages)**

| Element | Behaviour |
|---|---|
| Shop name / logo | From Settings |
| Logged-in user name + role badge | `Admin` / `Staff` |
| Low-stock bell | Count badge; click opens alert list (dismissible) |
| Logout button | Clears tokens, redirect to Login |

---

## 2. Permission matrix (buttons & actions)

| Action | Admin | Staff | Notes |
|---|---|---|---|
| Login & Dashboard | Full | Full | |
| New Bill / Generate Invoice | Full | Full | |
| Apply discount within cap | Full | Full | Cap from Settings |
| Apply discount beyond cap | Full | PIN override | `POST /auth/verify-pin` |
| Void / delete invoice | Full | Hidden | |
| Product create / edit / delete | Full | View only | No Add/Edit/Delete buttons for Staff |
| Stock adjust | Full | Hidden | |
| Expenses | Full | Hidden | |
| Settings | Full | Hidden | |
| Staff user management | Full | Hidden | |
| Reports | Full | Today’s sales only | Hide other report tabs |
| Record customer due payment | Full | Full | |

---

## 3. Phase-wise plan

---

# PHASE 0 — Foundation & project setup

**Goal:** Empty but runnable app. Both repos (or monorepo) boot, DB connects, env is clean.  
**Duration (guide):** 2–3 days  
**Parallel:** No — everything depends on this.

### Tasks

| # | Task | Owner | Done when |
|---|---|---|---|
| 0.1 | Create repo: `frontend/` (Vite + React + Tailwind) and `backend/` (Express + Prisma) | Both | `npm run dev` works on both |
| 0.2 | Feature-based folder structure (see below) | Both | Matches SRS §5 |
| 0.3 | `.env` files: `DATABASE_URL`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `PORT`, `CORS_ORIGIN` | Backend | No secrets in git |
| 0.4 | Prisma init + MySQL connection | Backend | `prisma migrate` succeeds |
| 0.5 | API client on frontend (Axios/fetch) with base `/api/v1` | Frontend | 401 interceptor stub ready |
| 0.6 | CORS, helmet, rate-limit (login later), JSON body parser | Backend | Health check `GET /api/v1/health` returns 200 |
| 0.7 | Shared UI kit: Button, Input, Select, Modal, Table, Badge, Toast | Frontend | Story-less but reusable |

### Suggested folders

**Backend**

```
backend/
  prisma/schema.prisma
  src/
    app.js
    middleware/auth.js, rbac.js, errorHandler.js, validate.js
    modules/auth|users|products|categories|customers|orders|payments|expenses|reports|settings
    utils/jwt.js, billing.js, invoiceNumber.js
```

**Frontend**

```
frontend/src/
  pages/Login|Dashboard|POS|Invoices|Inventory|Customers|Reports|Expenses|Settings
  components/ui|layout|pos|inventory|...
  store/cartStore.js (Zustand)
  api/client.js, auth.js, ...
  services/printer.js
  routes/ProtectedRoute.jsx, RoleRoute.jsx
```

### Buttons in this phase

| Screen | Button | Action |
|---|---|---|
| (none yet) | — | Health-check only |

---

# PHASE 1 — Authentication

**Goal:** Login, JWT session, role guards, idle logout.  
**Duration:** 3–4 days  
**Depends on:** Phase 0

### Database

| Table | Columns to create now |
|---|---|
| `users` | id, name, email, password_hash, role (`admin`/`staff`), pin_hash, is_active, created_at, updated_at |

Seed **one Admin** user via Prisma seed (email + bcrypt password, min 10 rounds).

### APIs

| Method | Endpoint | Access | Body / notes |
|---|---|---|---|
| POST | `/auth/login` | Public | `{ email, password }` → access + refresh; rate-limit |
| POST | `/auth/refresh` | Auth | Refresh cookie / token → new access token |
| POST | `/auth/logout` | Auth | Invalidate refresh token |
| POST | `/auth/verify-pin` | Staff | `{ pin }` — used later in POS/void |

### Screen: Login (`/login`)

| Field / control | Type | Required | Validation |
|---|---|---|---|
| Email / Username | Text | Yes | Email format |
| Password | Password (show/hide eye) | Yes | Min length 6 (UI); server checks hash |
| Remember me (optional) | Checkbox | No | Keep refresh cookie |

| Button | Label | Role | Action | After click |
|---|---|---|---|---|
| Primary | **Login** | Public | `POST /auth/login` | Redirect `/dashboard`; store access in memory; refresh in httpOnly cookie |
| Icon | **Show / Hide password** | Public | Toggle input type | — |
| Link | **Forgot password?** | — | Out of scope v1 | Hide or disable |

**Error states:** invalid credentials toast; inactive user blocked; network error toast.

### Cross-cutting (this phase)

| Feature | Behaviour |
|---|---|
| Protected routes | Unauthenticated → `/login` |
| Role routes | Staff hitting `/settings`, `/expenses` → 403 page |
| Token refresh | API interceptor retries once on 401 |
| POS idle logout | 30 min no activity → logout + toast “Session expired” |
| Frontend role check | UI only; **backend middleware is source of truth** |

---

# PHASE 2 — Users (staff accounts) + Admin PIN

**Goal:** Admin can create/disable staff. Admin PIN stored for override.  
**Duration:** 2 days  
**Depends on:** Phase 1  
**UI lives in:** Settings → Users tab (Settings shell can be a placeholder until Phase 4)

### APIs (add under users module)

| Method | Endpoint | Access | Description |
|---|---|---|---|
| GET | `/users` | Admin | List staff + admin |
| POST | `/users` | Admin | Create staff (name, email, password, role) |
| PUT | `/users/:id` | Admin | Update name, role, is_active |
| PUT | `/users/:id/pin` | Admin | Set/update 4-digit PIN (hashed) |
| PUT | `/users/:id/password` | Admin | Reset password |

### Screen: Settings → Users

| Field (Add / Edit User modal) | Type | Required |
|---|---|---|
| Name | Text | Yes |
| Email | Email | Yes, unique |
| Password | Password | Yes on create |
| Role | Select: Admin / Staff | Yes |
| Active | Toggle | Default on |
| Admin PIN (4 digit) | Password/number | Admin users only |

| Button | Label | Role | Action |
|---|---|---|---|
| Primary | **Add User** | Admin | Opens modal |
| Table row | **Edit** | Admin | Opens modal with values |
| Table row | **Reset Password** | Admin | Opens password modal |
| Table row | **Set PIN** | Admin | Opens PIN modal |
| Table row | **Deactivate / Activate** | Admin | Toggle `is_active` (confirm) |
| Modal | **Save** | Admin | POST/PUT user |
| Modal | **Cancel** | Admin | Close modal |

Staff: this entire tab is hidden.

---

# PHASE 3 — Settings (core config)

**Goal:** Key-value settings used by billing, invoices, and print. Build this **before** POS so rates/GST are not hardcoded.  
**Duration:** 3 days  
**Depends on:** Phase 1

### Database

| Table | Columns |
|---|---|
| `settings` | `key` VARCHAR(50) PK, `value` TEXT (JSON) |

### Seed keys

| Key | Example value | Used by |
|---|---|---|
| `gst_percent` | `18` | POS, invoice |
| `gst_split` | `true` (CGST+SGST half each) | POS, invoice |
| `discount_before_gst` | `true` | POS calc |
| `staff_discount_cap_percent` | `10` | POS discount |
| `invoice_prefix` | `HT-2026-` | Invoice number |
| `invoice_next_number` | `1` | Auto increment |
| `rounding_rule` | `nearest_rupee` | Grand total |
| `shop_name` | Himalaya Traders | Header, print |
| `shop_address` | … | Print |
| `shop_phone` | … | Print |
| `shop_gstin` | … | Print |
| `shop_logo_url` | … | Print |
| `flex_rate_normal` | number | POS flex line |
| `flex_rate_star` | number | POS flex line |
| `flex_rate_backlit` | number | POS flex line |
| `card_rates` | `{ "matte": x, "glossy": y, "premium": z }` | POS card line |
| `printer_width_mm` | `80` | Print |
| `auto_print` | `true` | After generate invoice |
| `low_stock_default` | `5` | New products |
| `invoice_edit_window` | `same_day` | Void/edit |
| `invoice_footer_note` | Thank you / return policy | Print |

### APIs

| Method | Endpoint | Access |
|---|---|---|
| GET | `/settings` | Admin |
| PUT | `/settings` | Admin — partial update by keys |

### Screen: Settings (tabs)

| Tab | Who sees it |
|---|---|
| GST & Billing | Admin |
| Shop Info | Admin |
| Printer | Admin |
| Users | Admin (from Phase 2) |

#### Tab: GST & Billing

| Field | Control |
|---|---|
| GST % | Number |
| Split CGST + SGST | Toggle |
| Discount before GST | Toggle (default ON) |
| Staff discount cap % | Number |
| Invoice prefix | Text |
| Next invoice number | Number |
| Rounding | Select: nearest rupee / none |
| Default low-stock threshold | Number |

| Button | Action |
|---|---|
| **Save GST & Billing** | PUT settings; toast success |
| **Reset to defaults** | Confirm, then restore seed values |

#### Tab: Shop Info

| Field | Control |
|---|---|
| Shop name | Text |
| Address | Textarea |
| Phone | Text |
| GSTIN | Text |
| Logo | File upload → store URL/path |

| Button | Action |
|---|---|
| **Upload Logo** | File picker |
| **Remove Logo** | Clear logo |
| **Save Shop Info** | PUT settings |

#### Tab: Printer

| Field | Control |
|---|---|
| Paper width | Select 80mm (default) |
| Auto-print on invoice | Toggle |
| Footer / thank-you note | Textarea |
| Test print | Button (enabled after Phase 13) |

| Button | Action |
|---|---|
| **Save Printer Settings** | PUT |
| **Test Print** | Phase 13 — print sample receipt |

---

# PHASE 4 — Categories & Inventory

**Goal:** Product master, stock, adjustments, history, soft-delete.  
**Duration:** 5–6 days  
**Depends on:** Phase 3 (default threshold)

### Database

| Table | Purpose |
|---|---|
| `categories` | name unique |
| `products` | sku, name, category_id, cost_price, selling_price, stock_qty, low_stock_threshold, unit, is_active |
| `stock_movements` | product_id, change_qty, reason (`sale`/`void`/`adjustment`/`purchase`/`damage`), reference_order_id, created_by |

### APIs

| Method | Endpoint | Access | Description |
|---|---|---|---|
| GET | `/categories` | All | List |
| POST | `/categories` | Admin | Create |
| PUT | `/categories/:id` | Admin | Rename |
| DELETE | `/categories/:id` | Admin | Only if unused, or block |
| GET | `/products?search=&category=` | All | Search name/SKU/category |
| POST | `/products` | Admin | Create (SKU auto or manual) |
| PUT | `/products/:id` | Admin | Update |
| DELETE | `/products/:id` | Admin | Soft-delete `is_active=false` |
| POST | `/products/:id/adjust-stock` | Admin | `{ change_qty, reason, note }` |
| GET | `/products/:id/stock-history` | Admin | Movement log |

SKU: auto-generate if blank (e.g. `HT-P-00041`). Search must return in **< 500ms** (index sku, name).

### Screen: Inventory list (`/inventory`)

**Toolbar**

| Control | Type | Role |
|---|---|---|
| Search | Input (name / SKU) | All |
| Category filter | Select | All |
| Stock filter | Select: All / Low stock / Out of stock | All |
| **Add Product** | Button | Admin only |
| **Add Category** | Button | Admin only |
| **Export** (optional v1) | Button | Admin |

**Table columns:** SKU · Name · Category · Cost · Selling price · Stock · Unit · Low-stock badge · Status · Actions

| Button / icon | Label | Role | Action |
|---|---|---|---|
| Primary | **Add Product** | Admin | Open product modal |
| Secondary | **Add Category** | Admin | Open category modal |
| Row | **Edit** | Admin | Edit product modal |
| Row | **Adjust Stock** | Admin | Stock adjust modal |
| Row | **History** | Admin | Stock history drawer |
| Row | **Deactivate** | Admin | Soft-delete confirm |
| Badge | **Low** | All | Visual only if stock ≤ threshold |

Staff: table is **view-only** — no Add / Edit / Adjust / Deactivate.

#### Modal: Add / Edit Product

| Field | Type | Required | Notes |
|---|---|---|---|
| Name | Text | Yes | |
| Category | Select (+ quick add) | Yes | |
| SKU | Text | No | Auto if empty |
| Cost price | Number | Yes | |
| Selling price | Number | Yes | Must be ≥ 0 |
| Opening stock | Number | Yes on create | Writes `purchase` movement |
| Low-stock threshold | Number | No | Default from settings |
| Unit | Select: pcs / box / kg / roll / other | Yes | default pcs |
| Active | Toggle | — | |

| Button | Action |
|---|---|
| **Save Product** | POST/PUT; close; refresh list |
| **Cancel** | Close without save |

#### Modal: Add Category

| Field | Type |
|---|---|
| Category name | Text unique |

| Button | Action |
|---|---|
| **Save** | POST category |
| **Cancel** | Close |

#### Modal: Adjust Stock

| Field | Type | Notes |
|---|---|---|
| Type | Radio: Add / Remove | Maps to +/− qty |
| Quantity | Number > 0 | |
| Reason | Select: Purchase / Damage / Adjustment | |
| Note | Text | Optional but recommended |

| Button | Action |
|---|---|
| **Confirm Adjustment** | POST adjust-stock; must log user + timestamp |
| **Cancel** | Close |

#### Drawer: Stock History

Columns: Date · Change (+/−) · Reason · Resulting balance · Linked order/adjustment id · User

| Button | Action |
|---|---|
| **Close** | Close drawer |

---

# PHASE 5 — Customer Management (master)

**Goal:** Customer CRUD + search by phone (POS will reuse this API). Dues display; **payment recording in Phase 8**.  
**Duration:** 3 days  
**Depends on:** Phase 1

### Database

| Table | Columns |
|---|---|
| `customers` | name, phone UNIQUE, address, notes, due_balance default 0 |

### APIs

| Method | Endpoint | Access |
|---|---|---|
| GET | `/customers?search=` | All — name or phone |
| POST | `/customers` | All |
| PUT | `/customers/:id` | All (or Admin-only if you prefer; SRS says CRUD) |
| GET | `/customers/:id` | All — detail + due |
| GET | `/customers/:id/orders` | All — stub until Phase 7, then wire |

### Screen: Customers list (`/customers`)

**Toolbar**

| Control | Type |
|---|---|
| Search | Name / phone |
| Due filter | All / Has due |
| **Add Customer** | Button |

**Table:** Name · Phone · Address · Due balance (highlight if > 0) · Actions

| Button | Role | Action |
|---|---|---|
| **Add Customer** | All | Open modal |
| Row **View** | All | Customer detail page |
| Row **Edit** | All | Edit modal |
| Row **Collect Due** | All | Disabled until Phase 8; then open payment modal |

#### Modal: Add / Edit Customer

| Field | Required |
|---|---|
| Name | Yes |
| Phone | Yes, unique |
| Address | No |
| Notes | No |

| Button | Action |
|---|---|
| **Save Customer** | POST/PUT |
| **Cancel** | Close |

### Screen: Customer detail (`/customers/:id`)

| Section | Content |
|---|---|
| Header | Name, phone, address, **Due: ₹X** |
| Purchase history | Invoice table (empty until Phase 7) |
| Due payments | List (empty until Phase 8) |

| Button | Role | Action |
|---|---|---|
| **Edit** | All | Edit modal |
| **New Bill** | All | `/pos?customerId=` prefill |
| **Record Payment** | All | Phase 8 |
| **Back** | All | List |

---

# PHASE 6 — POS / Billing (core)

**Goal:** Fast checkout. Walk-in cash sale in **≤ 5 clicks** after product search. Invoice + stock + due in **one DB transaction**. Calc in **< 2s**.  
**Duration:** 8–10 days  
**Depends on:** Phase 3 + 4 + 5  
**This is the most important phase.**

### Database (create now)

| Table | Notes |
|---|---|
| `orders` | invoice_no unique, customer_id nullable, subtotal, discount_amount, gst_amount, grand_total, payment_mode (`cash`/`upi`/`card`/`split`/`credit`), status (`completed`/`voided`), created_by |
| `order_items` | product_id nullable, item_type (`product`/`flex_print`/`card_print`), description snapshot, quantity, unit_rate, line_total |
| `payments` | order_id, customer_id, amount, mode (`cash`/`upi`/`card`), paid_at |

Also write `stock_movements` with reason `sale` inside the same transaction.

### Billing formulas (unit-test these)

| Step | Rule |
|---|---|
| Product line | `line_total = qty × unit_rate` |
| Flex print | `line_total = height_ft × width_ft × rate_per_sqft` (2 decimals) |
| Card print | `line_total = qty × rate_per_card` |
| Subtotal | Sum of line totals |
| Discount | % or fixed; **default before GST**; staff capped; over cap → admin PIN |
| GST | One rate on (subtotal − discount) **or** CGST+SGST each half |
| Grand total | Round to nearest rupee if setting on |
| Split pay | Sum of mode amounts **must equal** grand total |
| Credit | Add unpaid amount to `customers.due_balance` |

### APIs

| Method | Endpoint | Access | Notes |
|---|---|---|---|
| GET | `/products?search=` | All | Debounce 300ms from POS |
| GET | `/customers?search=` | All | Phone match auto-fill |
| POST | `/orders` | All | Transactional create |
| POST | `/auth/verify-pin` | Staff | Discount over cap |

### Screen: POS (`/pos`)

Layout (desktop): **Left = search + results**, **Right = cart + totals + pay**.

#### A. Customer bar (top)

| Field | Type | Behaviour |
|---|---|---|
| Customer name | Text | Optional; default **Walk-in Customer** |
| Phone | Text | On match: fill name, show last purchases + outstanding due chip |
| Due chip | Badge | If due > 0, red “Due ₹X” |

| Button | Action |
|---|---|
| **Clear Customer** | Reset to walk-in |
| **+ New Customer** | Mini modal (name + phone) then select |

#### B. Product search

| Control | Behaviour |
|---|---|
| Search box | Debounce 300ms; name / SKU / category |
| Result row | Click/tap → add qty 1; same product again → qty++ |
| Empty state | “No product found” + shortcut **Add Flex** / **Add Card** |

#### C. Cart line (product)

| Control | Behaviour |
|---|---|
| Name | Read-only snapshot |
| Unit price | Shown; Admin may edit? SRS: unit price from product; keep read-only unless you allow override later |
| Qty | Editable + **−** / **+** stepper |
| Line total | Live |
| Stock check | If qty > stock → block + inline warning |
| Delete icon | Remove line |

#### D. Custom lines

| Button | Opens |
|---|---|
| **Add Flex Printing** | Flex modal |
| **Add Card Printing** | Card modal |

**Flex modal fields**

| Field | Notes |
|---|---|
| Height (ft) | Number |
| Width (ft) | Number |
| Type | Normal / Star / Backlit |
| Rate / sq.ft | Auto from settings, **editable** |
| Preview total | Live `H × W × rate` |

| Button | Action |
|---|---|
| **Add to Cart** | Non-inventory line; no stock |
| **Cancel** | Close |

**Card modal fields**

| Field | Notes |
|---|---|
| Card type | Text or select |
| Quality | Matte / Glossy / Premium |
| Quantity | Number |
| Rate per card | Auto from settings, editable |
| Preview total | `qty × rate` |

| Button | Action |
|---|---|
| **Add to Cart** | Non-inventory line |
| **Cancel** | Close |

#### E. Cart footer / totals

Display: Subtotal · Discount · GST breakdown (CGST/SGST or single) · **Grand Total**

| Control | Type | Rules |
|---|---|---|
| Discount type | Radio: % / ₹ | |
| Discount value | Number | Staff: cannot exceed cap without PIN |
| Payment mode | Cash / UPI / Card / Split / Credit | Credit needs a real customer (not walk-in) |
| Split amounts | Cash + UPI + Card fields | Must sum to grand total |
| Amount received (cash) | Number | Optional change due display |

#### F. POS action buttons (critical)

| Button | Label | Role | Enabled when | Action |
|---|---|---|---|---|
| Danger outline | **Clear Cart** | All | Cart not empty | Confirm dialog → empty cart |
| Icon per line | **Remove item** | All | Always | Remove that line |
| Stepper | **−** / **+** | All | Qty ≥ 1; + blocked at stock | Change qty |
| Secondary | **Add Flex Printing** | All | Always | Modal |
| Secondary | **Add Card Printing** | All | Always | Modal |
| Ghost | **Hold Bill** (optional v1) | — | Out of SRS | Skip unless requested |
| Primary | **Generate Invoice** | All | Cart ≥ 1 line, payment valid | `POST /orders`; print if auto-print |
| Secondary | **Print** | All | After invoice created | Manual print |
| Link | **New Bill** | All | After success | Reset screen for next customer |

**Confirm dialogs**

| Trigger | Message | Buttons |
|---|---|---|
| Clear cart | “Remove all items from cart?” | **Cancel** · **Clear** |
| Generate (credit) | “Add ₹X to customer due?” | **Cancel** · **Confirm** |
| Discount over cap (staff) | Enter 4-digit Admin PIN | **Cancel** · **Verify PIN** |
| Qty > stock | Inline warning, no dialog | Fix qty |

**After successful invoice**

| Element | Action |
|---|---|
| Success toast | Invoice no. `HT-2026-000123` |
| Auto print | If setting on |
| Buttons | **Print Again** · **New Bill** · **View Invoice** |

Do **not** recompute prices on reprint later — store snapshots on `order_items.description` + rates.

---

# PHASE 7 — Invoice Management

**Goal:** List, filter, detail, reprint, export, void.  
**Duration:** 4–5 days  
**Depends on:** Phase 6

### APIs

| Method | Endpoint | Access |
|---|---|---|
| GET | `/orders?date_from=&date_to=&customer=&payment_mode=&staff=` | All |
| GET | `/orders/:id` | All |
| POST | `/orders/:id/void` | Admin — reverse stock + due; status `voided` |
| GET | `/orders/export?format=xlsx\|pdf` | All (filtered set) |

Pagination newest-first. Void only inside configurable window (default same day). Audit log: user, timestamp, reason.

### Screen: Invoices list (`/invoices`)

**Filters**

| Control | Type |
|---|---|
| Date from / to | Date picker |
| Customer | Search name/phone |
| Payment mode | Select |
| Staff | Select (who created) |
| Status | All / Completed / Voided |

| Button | Role | Action |
|---|---|---|
| **Apply Filters** | All | Reload list |
| **Clear Filters** | All | Reset |
| **Export Excel** | All | Download xlsx |
| **Export PDF** | All | Download pdf |
| Row **View** | All | Detail page |
| Row **Reprint** | All | Send stored snapshot to printer / PDF |
| Row **Void** | Admin only | Confirm + reason → void |

**Table columns:** Invoice no. · Date/time · Customer · Items count · Subtotal · GST · Discount · Total · Payment mode · Staff · Status · Actions

### Screen: Invoice detail (`/invoices/:id`)

| Section | Content |
|---|---|
| Header | Invoice no., date, status badge, staff |
| Shop block | Name, address, GSTIN, logo |
| Customer | Name, phone |
| Line items | Snapshot name, qty, rate, line total |
| Totals | Subtotal, discount, GST split, grand total |
| Payment | Mode(s) and amounts |

| Button | Role | Action |
|---|---|---|
| **Reprint** | All | Thermal or browser print |
| **Download PDF** | All | Fallback |
| **Void Invoice** | Admin | Modal: reason required |
| **Back to list** | All | `/invoices` |

**Void modal**

| Field | Required |
|---|---|
| Reason | Yes |

| Button | Action |
|---|---|
| **Confirm Void** | Reverse stock (`reason=void`), reverse due/payments, keep row for audit |
| **Cancel** | Close |

Staff: Void button **not rendered**. Backend still rejects.

---

# PHASE 8 — Due payments (customer)

**Goal:** Collect outstanding without a new invoice.  
**Duration:** 2 days  
**Depends on:** Phase 5 + 6

### APIs

| Method | Endpoint | Access |
|---|---|---|
| POST | `/customers/:id/payments` | All — `{ amount, mode, paid_at }` |
| GET | `/customers/:id/orders` | All — history |

`payments.order_id` null; `customer_id` set; decrement `due_balance` (not below 0).

### UI: Collect Due modal (from list or detail)

| Field | Type |
|---|---|
| Outstanding | Read-only ₹ |
| Amount | Number ≤ due |
| Mode | Cash / UPI / Card |
| Date | Date, default today |
| Note | Optional |

| Button | Action |
|---|---|
| **Record Payment** | POST payment; refresh due |
| **Cancel** | Close |

Customer detail: payment history table (date, amount, mode, user).

---

# PHASE 9 — Dashboard

**Goal:** Landing after login. Cards + charts. Refresh on load + **60s poll**.  
**Duration:** 3 days  
**Depends on:** Phase 6 + 4 + 8

### Data widgets

| Card / widget | Source | Click-through |
|---|---|---|
| Today’s sales ₹ | Orders today, not voided | Invoices filtered today |
| This month’s sales ₹ | Month | Reports |
| Orders today / this month | Count | Invoices |
| Pending dues ₹ | Sum customer.due_balance | Customers?due=1 |
| Low stock list | Products below threshold | Inventory low filter |
| Top 5 products (qty, this month) | order_items | Reports → Products |

Suggested extra APIs (or one `GET /reports/dashboard`):

| Method | Endpoint | Access |
|---|---|---|
| GET | `/reports/dashboard` | All — staff still sees own-day figures; dues/low-stock OK |

### Screen: Dashboard (`/dashboard`)

| Button | Action |
|---|---|
| **New Bill** | Navigate `/pos` (primary CTA) |
| Low-stock row **View** | `/inventory?low=1` |
| **View all invoices** | `/invoices` |
| Bell (header) | Dismissible low-stock alerts |

Staff: same dashboard; no expenses/settings links.

---

# PHASE 10 — Expense Tracking

**Goal:** Log costs; feed P&L.  
**Duration:** 2–3 days  
**Depends on:** Phase 1  
**Access:** Admin only

### Database

| Table | Columns |
|---|---|
| `expenses` | category, amount, note, spent_at, created_by |

Categories: **Rent · Utilities · Raw material · Salary · Misc** (select, not free-text only).

### APIs

| Method | Endpoint | Access |
|---|---|---|
| GET | `/expenses?from=&to=&category=` | Admin |
| POST | `/expenses` | Admin |
| PUT | `/expenses/:id` | Admin (optional, same-day) |
| DELETE | `/expenses/:id` | Admin (optional) |

### Screen: Expenses (`/expenses`)

**Toolbar:** date range · category filter · **Add Expense**

**Table:** Date · Category · Amount · Note · Added by · Actions

#### Modal: Add Expense

| Field | Required |
|---|---|
| Category | Yes |
| Amount | Yes |
| Date | Yes |
| Note | No |

| Button | Role | Action |
|---|---|---|
| **Add Expense** | Admin | Open modal |
| **Save** | Admin | POST |
| **Cancel** | Admin | Close |
| Row **Edit** | Admin | Optional |
| Row **Delete** | Admin | Confirm |

Staff: route hidden + API 403.

---

# PHASE 11 — Reports & Analytics

**Goal:** Sales / products / customers / P&L + Excel/CSV export.  
**Duration:** 4–5 days  
**Depends on:** Phase 6 + 10 + 4

### APIs

| Method | Endpoint | Access |
|---|---|---|
| GET | `/reports/sales?range=` | Admin; Staff = **today only** |
| GET | `/reports/products?range=` | Admin |
| GET | `/reports/customers?range=` | Admin |
| GET | `/reports/profit-loss?range=` | Admin |
| GET | `/reports/*/export` | Same as parent |

**P&L:** `total sales − total expenses − COGS` where COGS = `cost_price × qty sold` (use snapshot if you store cost on line; otherwise current cost — **prefer storing cost_price on order_item at sale time** in Phase 6).

### Screen: Reports (`/reports`) — tabs

| Tab | Staff |
|---|---|
| Sales | Today only |
| Products | Hidden |
| Customers | Hidden |
| Profit-Loss | Hidden |

**Every tab:** Date range picker (Today / This week / This month / Custom).

| Button | Tab | Action |
|---|---|---|
| **Apply** | All | Reload |
| **Export Excel / CSV** | All | Download |
| Toggle | Products | Sort by quantity **or** revenue; Top / Least |

**Sales report columns:** Date · Orders · Total sales · Avg order value · GST · Discount  

**Products columns:** Product · Qty sold · Revenue · (optional COGS)  

**Customers columns:** Customer · Phone · Invoice count · Sales total · Due  

**P&L cards:** Sales · Expenses · COGS · **Net profit/loss**

---

# PHASE 12 — Thermal printer + PDF fallback

**Goal:** 80mm receipt. Auto-print after invoice. Reprint uses **stored snapshot**.  
**Duration:** 4 days  
**Depends on:** Phase 7  
**Can start a spike in parallel with Phase 6** (print layout only).

### Approach

| Path | When |
|---|---|
| Browser print CSS `@page` 80mm | Default v1 |
| Local bridge `node-thermal-printer` on localhost | If USB/Bluetooth reliability needs it |
| PDF download | No printer |

### Receipt content (fixed order)

1. Logo + shop name, address, phone, GSTIN  
2. Invoice no. + date/time  
3. Customer name / phone (or Walk-in)  
4. Item table: name, qty, rate, amount  
5. Subtotal, discount, GST (CGST/SGST or single), grand total  
6. Payment mode  
7. Footer thank-you + return policy  

### Buttons (already on POS + Invoice detail)

| Button | Behaviour |
|---|---|
| **Print** / **Reprint** | Send snapshot; do not recalc prices |
| **Download PDF** | Same layout as PDF |
| Settings **Test Print** | Sample with dummy lines |

---

# PHASE 13 — Hardening, QA, staging, go-live

**Goal:** Safe production.  
**Duration:** 5–7 days  
**Depends on:** All functional phases

### Testing checklist

| # | Test | Priority |
|---|---|---|
| 1 | Unit: GST, discount before/after, flex/card formulas, rounding | P0 |
| 2 | Integration: create order — stock + due; **rollback on failure** | P0 |
| 3 | Void reverses stock and due; invoice stays VOIDED | P0 |
| 4 | Staff cannot hit admin APIs (crafted request) | P0 |
| 5 | Discount over cap blocked without valid PIN | P0 |
| 6 | Credit sale rejected for walk-in | P0 |
| 7 | Split payment must equal total | P0 |
| 8 | Soft-deleted product still shows on old invoices | P1 |
| 9 | Idle 30 min logout on POS | P1 |
| 10 | Real 80mm print formatting | P0 before go-live |
| 11 | Login rate-limit | P1 |
| 12 | Indexes: sku, phone, invoice_no, order date | P1 |

### Deployment

| Item | Detail |
|---|---|
| Staging URL | Client review before go-live |
| HTTPS | Required |
| Env vars | DB, JWT, GST defaults — never in git |
| Backups | Daily MySQL, 30-day retain |
| Monitoring | API uptime alert (target 99.5%) |
| Seed | Admin user + settings + sample category |

### Go-live sequence

| Step | Action |
|---|---|
| 1 | Staging sign-off (client) |
| 2 | Confirm open questions (GST split, discount order, printer model, invoice format) |
| 3 | Production migrate + seed admin |
| 4 | Enter shop GSTIN, rates, logo, printer |
| 5 | Load product master (CSV import if time; else manual) |
| 6 | Train cashier: search → qty → pay → print |
| 7 | Shadow day (old + new billing) if possible |
| 8 | Cutover |

---

## 4. Full button catalogue (quick index)

Use this when building UI so no action is missed.

| Screen | Buttons / actions |
|---|---|
| Login | Login, Show/Hide password |
| Header | Bell alerts, Logout |
| Sidebar | Dashboard, New Bill, Invoices, Inventory, Customers, Reports, Expenses, Settings |
| Dashboard | New Bill, View invoice/low-stock links |
| POS | Clear Customer, + New Customer, Search add, −/+, Remove line, Clear Cart, Add Flex, Add Card, Add to Cart (modals), Cancel (modals), Generate Invoice, Print, New Bill, Verify PIN, Confirm credit |
| Invoices | Apply/Clear filters, Export Excel, Export PDF, View, Reprint, Void, Confirm Void, Download PDF, Back |
| Inventory | Add Product, Add Category, Edit, Adjust Stock, History, Deactivate, Save/Cancel on all modals, Confirm Adjustment, Close drawer |
| Customers | Add Customer, View, Edit, New Bill, Record Payment, Save/Cancel |
| Expenses | Add Expense, Save, Edit, Delete |
| Reports | Tab switch, Apply range, Export Excel/CSV, Top/Least toggle |
| Settings GST | Save, Reset defaults |
| Settings Shop | Upload Logo, Remove Logo, Save |
| Settings Printer | Save, Test Print |
| Settings Users | Add User, Edit, Reset Password, Set PIN, Deactivate, Save, Cancel |

---

## 5. Suggested sprint calendar (approx. 8–10 weeks)

| Week | Phase | Deliverable for demo |
|---|---|---|
| 1 | 0 + 1 | Login works, empty shell |
| 2 | 2 + 3 | Admin settings + staff user |
| 3 | 4 | Inventory CRUD + stock adjust |
| 4 | 5 | Customers |
| 5–6 | 6 | **POS generates real invoices** |
| 7 | 7 + 8 | Invoice list, void, dues |
| 8 | 9 + 10 | Dashboard + expenses |
| 9 | 11 + 12 | Reports + 80mm print |
| 10 | 13 | QA, staging, go-live |

Two developers (1 FE + 1 BE) can overlap **within** a phase (API first, then UI), but **do not skip POS dependencies**.

---

## 6. Developer sequence (who does what)

| Order | Backend | Frontend | Blocked until |
|---|---|---|---|
| 1 | Prisma schema users, auth routes | Login + guards + API client | — |
| 2 | Users CRUD + PIN | Settings Users tab | Auth |
| 3 | Settings GET/PUT + seed | Settings 3 tabs | Auth |
| 4 | Categories + products + stock | Inventory screens | Settings seed |
| 5 | Customers CRUD | Customers screens | Auth |
| 6 | `billing.js` unit tests, then POST /orders | POS cart Zustand + UI | Products + customers + settings |
| 7 | List/void/export orders | Invoices pages | POST /orders |
| 8 | Due payments | Collect due modal | Customers + orders |
| 9 | Dashboard aggregate endpoint | Dashboard cards/poll | Orders |
| 10 | Expenses + P&L query | Expenses + Reports | Orders + expenses |
| 11 | — | Print CSS + PDF + optional bridge | Invoice snapshot |
| 12 | Tests, backups, deploy | QA on 80mm hardware | All |

---

## 7. Open questions (confirm before Phase 6 freeze)

| # | Question | Default in SRS (use until client answers) |
|---|---|---|
| 1 | Discount before or after GST? | **Before GST** |
| 2 | Split CGST+SGST or single GST? | **Split** (toggle in settings) |
| 3 | Printer model — USB or Bluetooth? | Browser print first; bridge if needed |
| 4 | Invoice number format + starting no.? | `HT-2026-000001` |

---

## 8. Definition of done (whole project)

- [ ] Admin and Staff can log in; Staff cannot void, edit products, open Settings/Expenses, or see full reports  
- [ ] Walk-in cash bill: search → add → pay cash → invoice + stock down + optional print  
- [ ] Flex and card lines calculate correctly and do not touch stock  
- [ ] Credit sale increases customer due; due payment decreases it  
- [ ] Void restores stock and due; invoice remains in list as VOIDED  
- [ ] GST, discount cap + PIN, rounding match settings  
- [ ] Low stock on dashboard; 60s refresh  
- [ ] P&L = sales − expenses − COGS  
- [ ] 80mm print or PDF fallback  
- [ ] Staging signed off; backups running  

---

*End of plan. Build Phase 0 → 13 in order. POS (Phase 6) is the critical path.*
