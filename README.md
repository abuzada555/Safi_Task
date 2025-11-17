# Invoicing & Expense Claims Platform

Spring Boot (backend) + Angular (frontend) application for:

- Creating and managing **expense claims**
- Driving them through a **workflow** (draft → submitted → approved → invoiced)
- Generating **invoices**
- Updating **stock** based on approved invoices
- Downloading **PDF invoices** with Ninova Technology branding

---

## 1. Prerequisites

Make sure these are installed:

- **Node.js**: 20+
- **npm**: 10+
- **JDK**: 17+
- **Maven Wrapper**: already included in `/backend` (`mvnw` / `mvnw.cmd`)

Verify your toolchain:

```bash
node -v
npm -v
java -version
./backend/mvn -v     # or backend\mvnw.cmd on Windows
```

---

## 2. Project structure

Clone and inspect the repository:

```bash
git clone https://github.com/abuzada555/Safi_Task.git
cd safi_task
ls
```

You should see at least:

- `backend/` – Spring Boot application (REST API + workflow)
- `frontend/` – Angular 17 standalone SPA

---

## 3. Configuration (optional)

### 3.1 Backend config

Default backend:

- Runs on `http://localhost:8080`
- Uses in-memory **H2** database

To override DB, ports, etc.:

1. Copy the default config:

   ```bash
   cp backend/src/main/resources/application.yml       backend/src/main/resources/application-local.yml
   ```

2. Edit `application-local.yml` (DB URL, username, password, `server.port`, etc.)

3. Run with `spring.profiles.active=local` if needed.

### 3.2 Frontend config

Default API base URL:

```ts
// frontend/src/app/api.service.ts
baseUrl = "http://localhost:8080/api";
```

If your backend runs on a different host/port, update this value.

---

## 4. Run the app locally (step-by-step)

### 4.1 Start the backend (Spring Boot)

In **terminal 1**:

```bash
cd backend
./mvn spring-boot:run
# or: mvn spring-boot:run
```

Wait until you see:

```text
Started Application in ... seconds
```

The API will be available at:

- `http://localhost:8080`
- Base path: `/api` (e.g., `http://localhost:8080/api/claims`)

By default it uses **H2 in-memory** storage.

---

### 4.2 Start the frontend (Angular)

In **terminal 2**:

```bash
cd frontend
npm install
npm run start
```

Then open:

- `http://localhost:4200/` in your browser

The Angular dev server proxies API calls to the backend automatically.

---

### 4.3 Optional: run backend tests

```bash
cd backend
./mvn test
```

This validates:

- Workflow rules
- Service logic
- Repository mappings

---

### 4.4 Optional: run frontend tests

```bash
cd frontend
npm run test
```

Uses **Karma/Jasmine** to validate components and the API service.

---

## 5. End-to-end usage (click-through flow)

Once both servers are running:

### Step 1 – Create a draft claim

- Go to the **“Create an Expense Claim”** form.
- Fill in claimant details.
- Add one or more line items (`description`, `qty`, `unitPrice`, `taxRate`).
- Click **Save Draft** → calls `POST /api/claims`.

---

### Step 2 – Submit and review

- Find your new claim in the **claims table**.
- Click it to see details and allowed transitions.
- Choose `SUBMITTED` from the workflow dropdown → **Apply Transition**.
- Then move to `UNDER_REVIEW`.

---

### Step 3 – Approve and generate invoice

- Transition claim to `APPROVED`.
- Then transition to `INVOICED`.
- At `INVOICED`, the backend:

  - Creates/updates the **Invoice**
  - Links it to the claim

- You’ll see the invoice appear in the **invoices table**.

---

### Step 4 – Approve invoice and update stock

- In the **Invoices** panel, click **Approve** on an invoice.

  - Calls: `POST /api/invoices/{id}/approve`

- Go to the **Current Stock** panel.

  - Stock quantities will now reflect the invoice items.

---

### Step 5 – Regress workflow (optional)

- From the claim detail pane, transition **back** from `INVOICED` → `APPROVED`.
- The backend will:

  - Delete/revert the invoice
  - Adjust stock **downwards** automatically

---

### Step 6 – Download PDF invoice (optional)

- In the invoices table, click **PDF**.

  - Client calls: `GET /api/invoices/{id}/pdf-data`

- Angular uses **pdfmake** to render the PDF in-browser.

---

## 6. Architecture overview (short)

### 6.1 Tech stack

| Layer       | Tech                                             | Responsibility                                                |
| ----------- | ------------------------------------------------ | ------------------------------------------------------------- |
| Frontend    | Angular 17 standalone + Signals + Reactive Forms | SPA UI, forms, tables, transitions, PDF rendering via pdfmake |
| Backend     | Spring Boot 3 + JPA/Hibernate + H2/MySQL         | REST API, workflow enforcement, invoice & stock logic         |
| Persistence | Relational DB (H2 for dev, MySQL for real DB)    | Stores claims, invoices, history, and aggregated stock        |

### 6.2 Workflow goals

- **Single workflow engine** (`ClaimWorkflow`) controls all status transitions.
- **Reversible**:

  - Invoices and stock are **derived** from claims.
  - Moving backward removes/reverts these artifacts.

- **Task-friendly UI**:

  - Claims, invoices, history, and stock visible from one screen.

---

## 7. Backend flow (conceptual)

### 7.1 Main entities

| Entity / Table            | Purpose                               |
| ------------------------- | ------------------------------------- |
| `ExpenseClaim`            | Root claim (status, claimant, totals) |
| `ClaimItem`               | Line items under a claim              |
| `StatusHistory`           | Audit trail of status changes         |
| `Invoice` / `InvoiceItem` | Invoice derived from approved claims  |
| `StockSummary`            | Aggregated quantities per item code   |

Relationships:

- `ExpenseClaim` 1 → N `ClaimItem`
- `ExpenseClaim` 1 → N `StatusHistory`
- `ExpenseClaim` 1 → 1 `Invoice` (optional)
- `Invoice` 1 → N `InvoiceItem`
- `StockSummary` aggregates by `itemCode`

### 7.2 Claim workflow (state machine)

Allowed transitions:

- `DRAFT → SUBMITTED`
- `SUBMITTED → UNDER_REVIEW` or `SUBMITTED → DRAFT`
- `UNDER_REVIEW → APPROVED` or `UNDER_REVIEW → SUBMITTED`
- `APPROVED → INVOICED` or `APPROVED → UNDER_REVIEW`
- `INVOICED → APPROVED` (backwards only)

Backend enforces these rules; frontend just shows what’s allowed.

---

## 8. REST API summary

Base URL: `http://localhost:8080/api`

### 8.1 Claims

- `POST /claims`  
  Create draft claim.

- `PUT /claims/{id}`  
  Update claim (only while `DRAFT`).

- `GET /claims?page=0&size=10`  
  Paginated claims list.

- `GET /claims/{id}`  
  Single claim + allowed transitions.

- `GET /claims/{id}/history`  
  Status history list.

- `POST /claims/{id}/transition`  
  Body: `{ "targetStatus": "APPROVED", "comment": "optional" }`  
  Moves claim in workflow.

---

### 8.2 Invoices

- `GET /invoices?page=0&size=10`  
  Paginated invoice list.

- `GET /invoices/{id}`  
  Invoice details + items.

- `POST /invoices/{id}/approve`  
  Approve invoice and update stock.

- `GET /invoices/{id}/pdf-data`  
  Returns JSON payload for pdfmake.

---

### 8.3 Stock

- `GET /stock`  
  Snapshot of `StockSummary` (item, quantity, provenance).

---

## 9. PDF branding (header & footer images)

PDFs use static URLs pointing to Ninova Technology branding:

```ts
const headerUrl =
  "https://placehold.co/800x120/1e3a8a/ffffff?text=Ninova+Technology";

const footerUrl =
  "https://placehold.co/800x80/111827/f9fafb?text=Ninova+Technology+%7C+Borooq+Tower,+Bldg.+190,+St.+836,+Doha";
```

Located in: `frontend/src/app/app.ts`

To rebrand:

1. Replace the URLs above with your own image URLs.
2. Rebuild/restart the frontend.

---

## 10. Step-by-step API test with `curl` (optional)

Assuming backend at `http://localhost:8080`:

1. **Create draft claim**

   ```bash
   curl -X POST http://localhost:8080/api/claims      -H 'Content-Type: application/json'      -d '{
           "claimant": "Alex Ops",
           "items": [
             {"description": "Travel", "qty": 2, "unitPrice": 120, "taxRate": 0.1},
             {"description": "Meals", "qty": 5, "unitPrice": 25, "taxRate": 0.05}
           ]
         }'
   ```

2. **List claims**

   ```bash
   curl 'http://localhost:8080/api/claims?page=0&size=10'
   ```

3. **Transition claim to SUBMITTED**

   ```bash
   curl -X POST http://localhost:8080/api/claims/{claimId}/transition      -H 'Content-Type: application/json'      -d '{ "targetStatus": "SUBMITTED", "comment": "Ready for review" }'
   ```

4. **Transition to UNDER_REVIEW, APPROVED, then INVOICED** (repeat with new `targetStatus`).

5. **List invoices**

   ```bash
   curl 'http://localhost:8080/api/invoices?page=0&size=10'
   ```

6. **Approve an invoice**

   ```bash
   curl -X POST http://localhost:8080/api/invoices/{invoiceId}/approve
   ```

7. **Get PDF data**

   ```bash
   curl http://localhost:8080/api/invoices/{invoiceId}/pdf-data
   ```

8. **Check stock**

   ```bash
   curl http://localhost:8080/api/stock
   ```

9. **View claim history**

   ```bash
   curl http://localhost:8080/api/claims/{claimId}/history
   ```

10. **Regress from INVOICED → APPROVED**

```bash
curl -X POST http://localhost:8080/api/claims/{claimId}/transition      -H 'Content-Type: application/json'      -d '{ "targetStatus": "APPROVED", "comment": "Need to adjust stock" }'
```

---

## 11. Troubleshooting (quick table)

| Issue                                 | Check / Fix                                                                                       |
| ------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Browser shows `ECONNREFUSED`          | Is backend on `http://localhost:8080`? Does `ApiService.baseUrl` match?                           |
| Spring Boot: `Address already in use` | Another process uses port 8080. Change `server.port` in `application.yml` and update frontend.    |
| H2 data disappears after restart      | Switch to MySQL/PostgreSQL and configure JDBC URL in `application.yml` / `application-local.yml`. |
| `npm install` fails                   | Ensure Node.js 20+, remove `frontend/node_modules`, try again.                                    |
