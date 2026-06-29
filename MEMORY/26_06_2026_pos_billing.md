# OCSession1 — POS Billing Feature Implementation

## Overview
Built a POS billing feature with two endpoints (Create Bill & Get Bills) on the InvoHydra backend. Work involved adding new tables, modifying the Prisma schema to match the live DB, and implementing the controller/routes.

---

## 1. Branch Setup
- **Starting branch:** `pos`
- **New branch created:** `posOc` (`git checkout -b posOc`)

---

## 2. Schema Changes (`prisma/schema.prisma`)

### 2.1 User Model — Added missing columns (to match live DB)
The schema file was missing 3 columns that existed in the live PostgreSQL database. Added:
```
emailVerificationCode  String?
emailCodeExpiry        DateTime?
passwordHash          String?
```
Also added the relation for the new POS feature:
```
posBills  PosBill[]
```

### 2.2 PosBill Model — New table
```prisma
model PosBill {
  id                Int       @id @default(autoincrement())
  userId            String
  user              User      @relation(fields: [userId], references: [id])
  label             String
  invoiceDate       String
  customer          String?
  paymentMode       String    @default("Cash")
  amountReceived    Float     @default(0)
  billDiscount      Float     @default(0)
  additionalCharges Float     @default(0)
  remarks           String?
  items             PosItem[]
  createdAt         DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  @@map("pos_bills")
}
```

### 2.3 PosItem Model — New table
```prisma
model PosItem {
  id          Int      @id @default(autoincrement())
  billId      Int
  bill        PosBill  @relation(fields: [billId], references: [id], onDelete: Cascade)
  code        String
  name        String
  qty         Float
  unit        String
  price       Float
  discount    Float    @default(0)
  tax         Float    @default(0)
  createdAt   DateTime @default(now())
  @@map("pos_items")
}
```

### 2.4 Why `prisma db push` was NOT used
Running `npx prisma db push` would have tried to sync the schema file to the DB. Since the schema was missing `emailVerificationCode`, `emailCodeExpiry`, and `passwordHash`, Prisma warned it would **drop those columns** (which had live data). The command **failed with an error** — no changes were made.

Instead, tables were created via **raw SQL** (`CREATE TABLE IF NOT EXISTS`) to only add new tables without touching existing ones. Later, the 3 missing columns were added to `schema.prisma` so the file matches the DB exactly.

---

## 3. Route File (`src/routes/posRouter.js`)
Cleaned down to a single route:
```js
router.post('/', authMiddleware, posController.createBill);
router.get('/', authMiddleware, posController.getBills);
```
Mounted at `/api/pos` in `server.js:121`.

---

## 4. Controller (`src/controllers/posController.js`)

### 4.1 POST /api/pos — `createBill`

**Request body:**
```json
{
  "invoiceDate": "23/06/2026",
  "customer": "John Doe",
  "paymentMode": "Cash",
  "amountReceived": 500,
  "billDiscount": 20,
  "additionalCharges": 10,
  "remarks": "Thanks",
  "items": [
    {
      "code": "PROD001",
      "name": "Product Name",
      "qty": 2,
      "unit": "PCS",
      "price": 250,
      "discount": 0,
      "tax": 18
    }
  ]
}
```

**Flow:**
1. Validates `invoiceDate` (required) and `items` array (required, at least 1)
2. Validates each item has `code`, `name`, `qty`, `unit`, `price`
3. Generates auto-incrementing label (`#1`, `#2`, etc.) by querying the last bill
4. Creates `PosBill` + nested `PosItem` records in a single Prisma call
5. Returns `201` with the created bill including items

**Error handling:** Missing fields → 400, DB errors → 500

### 4.2 GET /api/pos — `getBills`

**Query params:** `?invoiceDate=23/06/2026` (optional, exact match)

**Flow:**
1. Always filters by `userId` from auth token
2. Optionally filters by `invoiceDate` string
3. Includes items, ordered by `createdAt` desc
4. Returns `200` with bills array

---

## 5. Database Table Creation
Tables created via raw SQL (no existing data touched):
```sql
CREATE TABLE IF NOT EXISTS pos_bills (
  id SERIAL PRIMARY KEY,
  "userId" TEXT NOT NULL REFERENCES users(id),
  label TEXT NOT NULL,
  "invoiceDate" TEXT NOT NULL,
  customer TEXT,
  "paymentMode" TEXT NOT NULL DEFAULT 'Cash',
  "amountReceived" DOUBLE PRECISION NOT NULL DEFAULT 0,
  "billDiscount" DOUBLE PRECISION NOT NULL DEFAULT 0,
  "additionalCharges" DOUBLE PRECISION NOT NULL DEFAULT 0,
  remarks TEXT,
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW(),
  "updatedAt" TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS pos_items (
  id SERIAL PRIMARY KEY,
  "billId" INTEGER NOT NULL REFERENCES pos_bills(id) ON DELETE CASCADE,
  code TEXT NOT NULL,
  name TEXT NOT NULL,
  qty DOUBLE PRECISION NOT NULL,
  unit TEXT NOT NULL,
  price DOUBLE PRECISION NOT NULL,
  discount DOUBLE PRECISION NOT NULL DEFAULT 0,
  tax DOUBLE PRECISION NOT NULL DEFAULT 0,
  "createdAt" TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## 6. Testing

Bearer token used: `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...` (user: `cmp0ssfmn000t01rznqeuvwyr`)

### 6.1 Initial Smoke Test
- **POST /api/pos** ✅ — Created bill `#1` (Notebook x3 @ ₹50) and bill `#2` (Pen x10 @ ₹10)
- **GET /api/pos** ✅ — Returned both bills with items, newest first

### 6.2 Multi-Date & Multi-Item Test

**POST 1 — 26/06/2026 (3 items)**
```json
{
  "invoiceDate": "26/06/2026",
  "items": [
    {"code":"P001","name":"Notebook","qty":5,"unit":"PCS","price":50,"tax":9},
    {"code":"P002","name":"Pen","qty":20,"unit":"PCS","price":10,"tax":5},
    {"code":"P003","name":"Eraser","qty":10,"unit":"PCS","price":5,"tax":3}
  ]
}
```
✅ Created bill `#3` (id=3) with 3 items linked correctly.

**POST 2 — 27/06/2026 (2 items)**
```json
{
  "invoiceDate": "27/06/2026",
  "items": [
    {"code":"P004","name":"Marker","qty":4,"unit":"PCS","price":30,"tax":6},
    {"code":"P005","name":"Ruler","qty":15,"unit":"PCS","price":8,"tax":4}
  ]
}
```
✅ Created bill `#4` (id=4) with 2 items linked correctly.

**GET ?invoiceDate=26/06/2026** — 3 bills returned
| Bill | Items | |
|------|-------|-|
| #1 | Notebook x3 = ₹150 | ← earlier |
| #2 | Pen x10 = ₹100 | ← earlier |
| #3 | Notebook x5 + Pen x20 + Eraser x10 = ₹500 | ← new |

**GET ?invoiceDate=27/06/2026** — 1 bill returned
| Bill | Items | |
|------|-------|-|
| #4 | Marker x4 + Ruler x15 = ₹240 | ← new |

✅ All items correctly associated with their respective bills and dates. No data leakage across dates.

### 6.3 Data Integrity
- All existing DB tables & data untouched
- Auto-increment labels (`#1` → `#4`) working sequentially
- `userId` correctly set to authenticated user on all records

---

## 7. Key decisions
- **Customer field:** Stored as `String?` (plain text), not a relation to `Customer` model — suitable for walk-in POS
- **Label system:** Global counter (`#1`, `#2`) across all users — can be scoped per-user later if needed
- **No DB migrations ran:** Raw SQL used to avoid data loss from schema mismatch
- **Schema updated to match DB:** Missing columns added to `schema.prisma` to keep Prisma in sync
