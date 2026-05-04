# CartAlchemy — Scheduled Orders Architecture

I led end-to-end delivery of the Scheduled Orders feature for CartAlchemy v1 — a B2B e-commerce platform serving the education sector. The feature lets customers create recurring product orders with configurable frequency, per-order budget limits, a budget increase request and approval workflow, and a full email notification lifecycle. Orders are fulfilled automatically by the ERP on schedule; CartAlchemy provides the UI layer on top.

The key technical challenge was that **all schedule data lives in IBM i (AS400) DB2/400 tables** — CartAlchemy never owns that data. The Node.js application communicates with IBM i exclusively through a custom JT400 Java bridge (`@intygrate/nodejs-jt400-javabridge`), calling stored procedures and executing SQL directly against DB2/400. MongoDB's role is narrow: it mirrors a subset of held order data (batch numbers) via a cron sync job to support UI display.

---

## 1. System Architecture

```mermaid
flowchart LR
  subgraph Browser["Customer Browser"]
    UI["Web UI"]
  end

  subgraph CartAlchemy["CartAlchemy — Node.js / Express"]
    WR["website/packages/erp-iptor/routes.js"]
    EP["admin/packages/erp-iptor/index.js"]
    CRON["admin/lib/migrateHeldOrders.js\n(cron sync)"]
  end

  subgraph JT400["JT400 Bridge"]
    BRIDGE["@intygrate/nodejs-jt400-javabridge\n(Java ↔ Node.js)"]
  end

  subgraph ERP["IBM i ERP"]
    SP["Stored Procedures\n(Z1NR460, Z1NR003B, CSBR410…)"]
    DB2["DB2/400 Tables\n(Z1OSCH, Z1OSCD, Z1OSCR, ANOACSOH)"]
    SP --> DB2
  end

  subgraph Mongo["MongoDB"]
    ORDERS["orders collection\n(batchNumber mirror)"]
  end

  UI -->|"HTTP requests"| WR
  WR --> EP
  EP -->|"program calls"| BRIDGE
  BRIDGE -->|"JT400 / TCP"| SP
  DB2 -->|"held order query"| CRON
  CRON -->|"sync batchNumber"| ORDERS
  ORDERS -->|"UI display"| EP
```

---

## 2. ERP Data Model

All ERP tables reside on IBM i / DB2/400. MongoDB holds only a projection of held order data for UI purposes.

```mermaid
classDiagram
  direction TB

  class Z1OSCH {
    +Number scheduleCode PK
    +String customerCode
    +String scheduleName
    +String status active / deleted
    +String deliveryFrequencyType
    +String intervalDays
    +Number lastDeliveryDate
    +Number annualBudgetLimit
    +Number ytdAmountSpent
    +Number temporaryBudgetAdjustment
    +String skipNext Y / N
    +String warehouseCode
    +String orderTypeCode
    +String userId
  }

  class Z1OSCD {
    +Number scheduleCode FK
    +String productCode
    +Number orderQuantity
    +String skipLine Y / N
  }

  class Z1OSCR {
    +Number scheduleCode FK
    +Number requestDate
    +String requestTime
    +Number requestedAmount
    +Number temporaryRequestAmount
    +Number approvalActionDate
    +String approvalActionTime
    +String approvalCode empty=pending
    +String notes
  }

  class ANOACSOH {
    +String heldFlag Y=held
    +String orderNumber
    +String batchNumber
  }

  class MongoDB_orders {
    +ObjectId _id
    +String orderNumber
    +String batchNumber ERP link
    +String orderStatus
    +Object payment type subStatus
    +String orderCustomerCode
    +Array orderProducts
    +Number orderTotal
    +Date orderDate
  }

  Z1OSCH "1" --> "many" Z1OSCD : has lines
  Z1OSCH "1" --> "many" Z1OSCR : has requests
  ANOACSOH ..> MongoDB_orders : synced via cron
```

> **Note:** Z1OSCH, Z1OSCD, Z1OSCR, and ANOACSOH all live on IBM i / DB2/400. MongoDB_orders is a CartAlchemy-side mirror — batch numbers are the only ERP field synced into it.

---

## 3. Schedule Creation & Line Management

```mermaid
flowchart TD
  A["Customer clicks 'Create Schedule'"] --> B["POST /customer/scheduled-orders/new\n(routes.js)"]
  B --> C["addScheduledOrder()\nerp-iptor/index.js"]
  C --> D["JT400 Bridge\n@intygrate/nodejs-jt400-javabridge"]
  D --> E["Z1NR460 stored proc\ngenerate next schedule number"]
  E --> F["INSERT INTO Z1OSCH\n(schedule header record)"]
  F --> G["Schedule created — redirect to detail page"]

  G --> H["Customer adds a product line"]
  H --> I["POST /customer/scheduled-orders/:id/line/add"]
  I --> J["addScheduledOrderLine()\nerp-iptor/index.js"]
  J --> D
  D --> K["Z1NR003B stored proc\nproduct pricing + customer discounts"]
  K --> L["INSERT INTO Z1OSCD\n(schedule line record)"]
  L --> M["Line saved — page refreshes"]
```

---

## 4. Order Request & Approval Workflow

```mermaid
sequenceDiagram
  participant C as Customer
  participant CA as CartAlchemy
  participant JT as JT400 Bridge
  participant ERP as IBM i ERP
  participant A as School Admin

  C->>CA: POST /customer/scheduled-orders/:id/request
  CA->>JT: addRequestInfo(req, scheduleOrder)
  JT->>ERP: INSERT INTO Z1OSCR (status = pending)
  ERP-->>JT: request record created
  JT-->>CA: success
  CA-->>C: redirect — request submitted

  Note over ERP: On schedule date, ERP processes<br/>schedule and generates held order<br/>in ANOACSOH

  A->>CA: GET /customer/held-orders
  CA->>JT: getHeldOrders({ customerCodes })
  JT->>ERP: SELECT FROM ANOACSOH (held orders)
  ERP-->>JT: held order list + batch totals
  JT-->>CA: held orders
  CA-->>A: render held orders page

  A->>CA: POST /customer/scheduled-orders/:id/request/:requestId/approve
  CA->>JT: updateRequestInfo(requestId, action, actionDate, actionTime)
  JT->>ERP: UPDATE Z1OSCR (status = approved)
  ERP-->>JT: updated
  JT-->>CA: success

  ERP->>ERP: CSBR410 / CSBR450 — generate sales order from held order
  CA-->>A: redirect — approval confirmed
```

---

## 5. Held Orders Cron Sync

`migrateHeldOrders.js` runs on a schedule to keep MongoDB batch numbers in sync with IBM i. This is a one-way sync — ERP is always the source of truth.

```mermaid
flowchart TD
  A["Cron fires\nmigrateHeldOrders.js"] --> B["getHeldOrders()\nerp-iptor/index.js"]
  B --> C["JT400 Bridge"]
  C --> D["SELECT FROM ANOACSOH\n(held orders)"]
  D --> E{"Has batch number?"}
  E -- No --> F["Skip — no batch to sync"]
  E -- Yes --> G["Find matching order\nin MongoDB"]
  G --> H{"MongoDB order\nhas batchNumber?"}
  H -- Yes --> I["Skip — already in sync"]
  H -- No --> J["UPDATE orders\nSET batchNumber from ERP"]
  J --> K["Log sync complete"]
  F --> K
  I --> K
  K --> L["Cron cycle complete"]
```

---

## 6. Email Notifications

Email is sent via a shared `sendEmail` / `getEmailTemplate` utility from `@cartalchemy/common/email`. Scheduled orders have two distinct trigger sources: user actions in the UI (transactional), and callbacks from the IBM i ERP system (automated reminders).

### Trigger Sources

```mermaid
flowchart LR
    subgraph UI["User Actions (CartAlchemy)"]
        U1["Create schedule\nPOST /scheduled-orders/add-order"]
        U2["Delete schedule\nDELETE /scheduled-orders/delete-order"]
        U3["Update schedule\nPOST /scheduled-orders/change-order"]
        U4["Skip schedule or line\nPOST change-order or change-line"]
        U5["Request budget increase\nPOST /scheduled-orders/budget-request"]
        U6["Approve or decline budget\nPOST /scheduled-orders/process-request"]
    end

    subgraph ERP["ERP-triggered (IBM i calls CartAlchemy)"]
        E1["POST /scheduledemail?p3=Y\nskip reminder"]
        E2["POST /scheduledemail?p2=Y\nover-budget reminder"]
        E3["POST /scheduledemail?p5=Y\n3-day reminder"]
        E4["POST /scheduledemail?p5=N\n24-hour reminder"]
    end

    U1 -->|"to: customer"| T1["scheduled-order-created"]
    U2 -->|"to: customer"| T2["scheduled-order-deleted"]
    U3 -->|"to: customer\nskip=N"| T3["scheduled-order-updated"]
    U4 -->|"to: customer\nskip=Y"| T4["scheduled-order-update-skip"]
    U5 -->|"to: approver"| T5["scheduled-order-budget-request"]
    U6 -->|"to: customer\naction=Y"| T6["scheduled-order-budget-approve"]
    U6 -->|"to: customer\naction=N"| T7["scheduled-order-budget-decline"]

    E1 -->|"to: customer"| T8["scheduled-order-skipped-reminder"]
    E2 -->|"to: customer"| T9["scheduled-order-overbudget-reminder"]
    E3 -->|"to: customer"| T10["scheduled-order-3d-reminder"]
    E4 -->|"to: customer"| T11["scheduled-order-24-reminder"]
```

### Email Template Reference

| Template | Subject | Recipient | Trigger |
|----------|---------|-----------|---------|
| `scheduled-order-created` | Scheduled Order Created | Customer | Schedule created in UI |
| `scheduled-order-deleted` | Scheduled Order Deleted | Customer | Schedule deleted in UI |
| `scheduled-order-updated` | Scheduled Order Updated | Customer | Schedule settings changed |
| `scheduled-order-update-skip` | Scheduled Order Updated | Customer | Skip flag set on schedule or line |
| `scheduled-order-budget-request` | Scheduled Order Budget Request | **Approver** | Customer requests budget increase |
| `scheduled-order-budget-approve` | Scheduled Order Budget Approved | Customer | Approver approves budget request |
| `scheduled-order-budget-decline` | Scheduled Order Budget Declined | Customer | Approver declines budget request |
| `scheduled-order-skipped-reminder` | MTA Scheduled Order - {code} | Customer | ERP: order set to skip |
| `scheduled-order-overbudget-reminder` | MTA Scheduled Order - {code} | Customer | ERP: order exceeds budget |
| `scheduled-order-3d-reminder` | MTA Scheduled Order - {code} | Customer | ERP: 3 days before order processes |
| `scheduled-order-24-reminder` | MTA Scheduled Order - {code} | Customer | ERP: 24 hours before order processes |

**ERP-triggered emails:** The IBM i system calls `POST /scheduledemail` directly with query parameters (`p1`–`p5`) to select the template. CartAlchemy resolves the customer email by looking up the customerCode from the schedule record in DB2, then dispatches the appropriate template — the ERP drives the notification timing, CartAlchemy handles delivery.

**Approver routing:** Budget request emails go to the approver email fetched via `getApproverEmailForScheduledOrder()`, not to the customer. Approval/decline responses then go back to the customer.

---

## Key File Reference

| File | Purpose |
|------|---------|
| `/admin/packages/erp-iptor/index.js` | Core ERP integration — all schedule CRUD and ERP queries (~6000 lines) |
| `/website/packages/erp-iptor/routes.js` | Express route definitions for scheduled orders UI |
| `/website/packages/erp-iptor/index.js` | Website package init and nav config |
| `/admin/lib/migrateHeldOrders.js` | Cron job — syncs ERP batch numbers into MongoDB |
| `/admin/packages/erp-iptor/programs/z1nr460.js` | Stored proc wrapper — generate schedule number |
| `/admin/packages/erp-iptor/programs/z1nr003b.js` | Stored proc wrapper — product pricing |
| `/admin/packages/erp-iptor/programs/csbr410.js` | Stored proc wrapper — order operations |
| `/admin/packages/erp-iptor/programs/z1nr205.js` | Stored proc wrapper — order line operations |
