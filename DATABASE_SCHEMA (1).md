# Database Schema — Qsquare General ERP (Sales Flow)

**Database:** PostgreSQL  
**Scope:** Complete Sales & Purchase Flow — Item Master → Customer → Quotation → Sales Order → Delivery Note → Sales Invoice → Sales Return → Credit Note → Receipt / Payment · Supplier → Purchase Request → Purchase Order → GRN → Purchase Invoice → Supplier Payment → Debit Note  
**Compliance:** ZATCA Phase-2 (Saudi e-Invoicing)

---

## Design Principles

| # | Rule |
|---|---|
| 1 | All PKs use `BIGSERIAL`. A secondary `uuid UUID DEFAULT gen_random_uuid()` is used for external-facing API IDs (prevents enumeration). |
| 2 | All monetary amounts use `NUMERIC(18,4)` — Saudi VAT arithmetic requires 4 decimal places. |
| 3 | Computed totals (`taxable_value`, `vat_amount`, `line_total`, `grand_total`) are **STORED by the application at document creation time**, never silently recomputed from master data that may change later. |
| 4 | Soft deletes on all transactional tables via `deleted_at TIMESTAMPTZ NULL`. Masters use a `status` column. |
| 5 | Every table has: `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`, `created_by BIGINT FK→app_user`. |
| 6 | Every transactional document carries `branch_id FK→branch`. |
| 7 | All customer-facing master tables include `name_ar VARCHAR(255)` for Arabic. |
| 8 | ZATCA Phase-2 fields are first-class columns on `sales_invoice` and `sales_return`. |
| 9 | Stock movements are recorded in an insert-only `inventory_transaction` ledger. A materialized view `item_stock_balance` provides fast current-stock reads. |
| 10 | Document numbering (QT-2026-0001, etc.) is managed by the `document_sequence` table using `SELECT … FOR UPDATE` to prevent race conditions. |

---

## Table of Contents

1. [Organization](#1-organization)
2. [Masters](#2-masters)
3. [Inventory](#3-inventory)
4. [Party (Customer / Supplier)](#4-party)
5. [Sales Masters](#5-sales-masters)
6. [Sales Documents](#6-sales-documents)
7. [Purchase Masters](#7-purchase-masters)
8. [Purchase Documents](#8-purchase-documents)
9. [Trigger Architecture](#9-trigger-architecture)
10. [ER Diagram](#10-er-diagram)
11. [Migration Order](#11-migration-order)
12. [ZATCA Notes](#12-zatca-notes)
13. [Table Purpose Reference](#13-table-purpose-reference)
14. [What to Build Now vs Later](#14-what-to-build-now-vs-later)

---

## 1. Organization

### `company`
Singleton table. One row per installation.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `name` | VARCHAR(255) | NOT NULL | |
| `name_ar` | VARCHAR(255) | | |
| `vat_number` | VARCHAR(20) | NOT NULL | 15-digit Saudi VAT |
| `cr_number` | VARCHAR(30) | | Commercial Registration |
| `address` | TEXT | | |
| `city` | VARCHAR(100) | | |
| `country` | VARCHAR(100) | NOT NULL DEFAULT 'Saudi Arabia' | |
| `phone` | VARCHAR(30) | | |
| `email` | VARCHAR(255) | | |
| `logo_path` | VARCHAR(500) | | |
| `currency` | CHAR(3) | NOT NULL DEFAULT 'SAR' | |
| `fiscal_year_start` | DATE | NOT NULL | |
| `zatca_env` | VARCHAR(20) | NOT NULL DEFAULT 'sandbox' | sandbox \| production |
| `zatca_device_id` | VARCHAR(100) | | EGS serial number |
| `zatca_cert` | TEXT | | PEM certificate from ZATCA |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `branch`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `company_id` | BIGINT | NOT NULL FK → company(id) | |
| `code` | VARCHAR(20) | NOT NULL UNIQUE | e.g. RUH-001 |
| `name` | VARCHAR(255) | NOT NULL | e.g. Riyadh Main HQ |
| `name_ar` | VARCHAR(255) | | |
| `city` | VARCHAR(100) | | |
| `address` | TEXT | | |
| `manager_user_id` | BIGINT | FK → app_user(id) DEFERRABLE | |
| `is_main` | BOOLEAN | NOT NULL DEFAULT false | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| inactive |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraint:** `UNIQUE INDEX branch_main ON branch(company_id) WHERE is_main = true` — only one main branch.

---

### `role`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `name` | VARCHAR(100) | NOT NULL UNIQUE | |
| `description` | TEXT | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| inactive |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `role_permission`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `role_id` | BIGINT | NOT NULL FK → role(id) ON DELETE CASCADE | |
| `module` | VARCHAR(100) | NOT NULL | e.g. Sales, Inventory, Finance |
| `can_create` | BOOLEAN | NOT NULL DEFAULT false | |
| `can_view` | BOOLEAN | NOT NULL DEFAULT false | |
| `can_update` | BOOLEAN | NOT NULL DEFAULT false | |
| `can_delete` | BOOLEAN | NOT NULL DEFAULT false | |
| `can_approve` | BOOLEAN | NOT NULL DEFAULT false | |
| `can_export` | BOOLEAN | NOT NULL DEFAULT false | |

**Constraint:** UNIQUE (role_id, module)

---

### `app_user`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `name` | VARCHAR(255) | NOT NULL | |
| `email` | VARCHAR(255) | NOT NULL UNIQUE | |
| `password_hash` | VARCHAR(500) | NOT NULL | |
| `role_id` | BIGINT | NOT NULL FK → role(id) | |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `employee_id` | BIGINT | FK → employee(id) | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| inactive |
| `last_login_at` | TIMESTAMPTZ | | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Indexes:** `(email)`, `(branch_id)`, `(role_id)`

---

### `document_sequence`
Centralized sequential numbering. Use `SELECT … FOR UPDATE` before generating a new document number.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `doc_type` | VARCHAR(50) | NOT NULL | QT, SO, DN, INV, RET, CN, RCP · REQ, PO, GRN, PINV, SPAY, DBN |
| `fiscal_year` | SMALLINT | NOT NULL | e.g. 2026 |
| `last_number` | INT | NOT NULL DEFAULT 0 | |
| `prefix` | VARCHAR(20) | NOT NULL | e.g. INV-2026 |
| `pad_digits` | SMALLINT | NOT NULL DEFAULT 4 | |

**Constraint:** UNIQUE (branch_id, doc_type, fiscal_year)

---

## 2. Masters

### `area`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(20) | NOT NULL UNIQUE | e.g. AR001 |
| `name` | VARCHAR(255) | NOT NULL | e.g. Riyadh Central |
| `name_ar` | VARCHAR(255) | | |
| `region` | VARCHAR(100) | | Central, Western, Eastern |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| inactive |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `route`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(20) | NOT NULL UNIQUE | e.g. RT001 |
| `name` | VARCHAR(255) | NOT NULL | |
| `name_ar` | VARCHAR(255) | | |
| `area_id` | BIGINT | NOT NULL FK → area(id) | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Index:** `(area_id)`

---

### `brand`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(20) | NOT NULL UNIQUE | |
| `name` | VARCHAR(255) | NOT NULL | |
| `name_ar` | VARCHAR(255) | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `unit_of_measure`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `name` | VARCHAR(100) | NOT NULL | e.g. Kilogram |
| `symbol` | VARCHAR(20) | NOT NULL UNIQUE | e.g. KG, PCS, BOX |
| `uom_type` | VARCHAR(20) | NOT NULL | count \| weight \| length \| volume |
| `base_unit_id` | BIGINT | FK → unit_of_measure(id) | NULL = this IS the base unit |
| `conversion_factor` | NUMERIC(18,8) | | qty_in_base = qty × factor |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Check:** `(base_unit_id IS NULL AND conversion_factor IS NULL) OR (base_unit_id IS NOT NULL AND conversion_factor IS NOT NULL)`

---

### `warehouse`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(20) | NOT NULL UNIQUE | e.g. WH-001 |
| `name` | VARCHAR(255) | NOT NULL | |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `manager_user_id` | BIGINT | FK → app_user(id) | |
| `address` | TEXT | | |
| `location_lat` | NUMERIC(10,7) | | GPS |
| `location_lng` | NUMERIC(10,7) | | GPS |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Index:** `(branch_id)`

---

### `bin`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `name` | VARCHAR(100) | NOT NULL | e.g. A-01-S1 |
| `warehouse_id` | BIGINT | NOT NULL FK → warehouse(id) | |
| `bin_type` | VARCHAR(20) | NOT NULL DEFAULT 'rack' | rack \| shelf \| floor |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraint:** UNIQUE (warehouse_id, name)

---

### `credit_term`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(20) | NOT NULL UNIQUE | e.g. CT001 |
| `name` | VARCHAR(100) | NOT NULL | e.g. Net 30 |
| `name_ar` | VARCHAR(100) | | |
| `days` | SMALLINT | NOT NULL CHECK (days >= 0) | 0 = Immediate |
| `credit_limit` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Default limit for this term |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `payment_method`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(20) | NOT NULL UNIQUE | e.g. PM001 |
| `name` | VARCHAR(100) | NOT NULL | e.g. Main Cash |
| `name_ar` | VARCHAR(100) | | |
| `method_type` | VARCHAR(20) | NOT NULL | cash \| bank \| card \| cheque \| other |
| `linked_account_id` | BIGINT | FK → coa_account(id) | GL account to debit on receipt |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `tax`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `name` | VARCHAR(100) | NOT NULL | e.g. Standard VAT 15% |
| `name_ar` | VARCHAR(100) | | |
| `rate` | NUMERIC(8,4) | NOT NULL CHECK (rate >= 0 AND rate <= 100) | |
| `tax_type` | VARCHAR(20) | NOT NULL DEFAULT 'both' | sales \| purchase \| both |
| `calculation` | VARCHAR(20) | NOT NULL DEFAULT 'exclusive' | inclusive \| exclusive |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `coa_account`
Hierarchical Chart of Accounts with unlimited depth via adjacency list + materialized path.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(30) | NOT NULL UNIQUE | |
| `name` | VARCHAR(255) | NOT NULL | |
| `name_ar` | VARCHAR(255) | | |
| `account_type` | VARCHAR(20) | NOT NULL | Asset \| Liability \| Equity \| Income \| Expense |
| `account_category` | VARCHAR(20) | NOT NULL | GROUP \| LEDGER \| SUB_LEDGER |
| `parent_id` | BIGINT | FK → coa_account(id) | NULL = root group |
| `level` | SMALLINT | NOT NULL DEFAULT 0 | 0=root, 1=group, 2=ledger, 3=sub-ledger |
| `materialized_path` | VARCHAR(1000) | | e.g. '1000/1100/1110' — for fast subtree queries |
| `branch_id` | BIGINT | FK → branch(id) | NULL = shared across all branches |
| `opening_balance` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| frozen \| inactive |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Business rule:** GROUP accounts cannot be posted to (app layer).

---

## 3. Inventory

### `item_category`
Self-referencing tree, unlimited depth.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `code` | VARCHAR(30) | NOT NULL UNIQUE | |
| `name` | VARCHAR(255) | NOT NULL | |
| `name_ar` | VARCHAR(255) | | |
| `parent_id` | BIGINT | FK → item_category(id) | NULL = root category |
| `level` | SMALLINT | NOT NULL DEFAULT 0 | |
| `materialized_path` | VARCHAR(2000) | | e.g. 'Electronics/Laptops/Gaming' |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Index:** `(parent_id)`

---

### `item`
The most-joined table in the system.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `item_code` | VARCHAR(50) | NOT NULL UNIQUE | |
| `barcode` | VARCHAR(100) | UNIQUE | |
| `name` | VARCHAR(500) | NOT NULL | |
| `name_ar` | VARCHAR(500) | | |
| `description` | TEXT | | |
| `item_image_path` | VARCHAR(500) | | |
| `category_id` | BIGINT | NOT NULL FK → item_category(id) | |
| `brand_id` | BIGINT | FK → brand(id) | |
| `base_uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | Primary stock-keeping UOM |
| `cost_price` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | LPR — Last Purchase Rate |
| `selling_price` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | LSR — Last Selling Rate |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | Saudi standard VAT; 0 for zero-rated |
| `reorder_level` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `tracking_type` | VARCHAR(20) | NOT NULL DEFAULT 'none' | none \| batch \| serial |
| `has_expiry` | BOOLEAN | NOT NULL DEFAULT false | |
| `is_hotkey` | BOOLEAN | NOT NULL DEFAULT false | POS hotkey shortcut |
| `hotkey_order` | SMALLINT | | Sort order for POS hotkeys |
| `is_service` | BOOLEAN | NOT NULL DEFAULT false | Service items have no stock movement |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| inactive |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | Soft delete |

**Indexes:** `(item_code)`, `(barcode)`, `(category_id)`, `(status) WHERE deleted_at IS NULL`

---

### `item_supplier`
Many-to-many: an item can have multiple supplier sources.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `supplier_id` | BIGINT | NOT NULL FK → party(id) | Must have is_supplier = true |
| `supplier_sku` | VARCHAR(100) | | Supplier's own part number |
| `purchase_price` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `is_primary` | BOOLEAN | NOT NULL DEFAULT false | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraints:** UNIQUE (item_id, supplier_id) · `UNIQUE INDEX isup_primary ON item_supplier(item_id) WHERE is_primary = true`

---

### `item_uom`
Multi-UOM per item: BOX (12 PCS), CARTON (24 PCS), each with its own barcode and price tiers.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `conversion_factor` | NUMERIC(18,8) | NOT NULL | qty_in_base = qty × factor |
| `barcode` | VARCHAR(100) | | Per-UOM barcode |
| `cost_price` | NUMERIC(18,4) | | |
| `rate1` | NUMERIC(18,4) | | Retail price |
| `rate2` | NUMERIC(18,4) | | Wholesale price |
| `rate3` | NUMERIC(18,4) | | Special price |
| `is_default` | BOOLEAN | NOT NULL DEFAULT false | Default UOM on sales forms |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraints:** UNIQUE (item_id, uom_id) · `UNIQUE INDEX iuom_default ON item_uom(item_id) WHERE is_default = true`

---

### `batch`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | item.tracking_type must be 'batch' |
| `batch_number` | VARCHAR(100) | NOT NULL | |
| `expiry_date` | DATE | | |
| `warehouse_id` | BIGINT | NOT NULL FK → warehouse(id) | |
| `bin_id` | BIGINT | FK → bin(id) | |
| `quantity` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED; updated by inventory_transaction trigger |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| expired |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraint:** UNIQUE (item_id, batch_number, warehouse_id)  
**Indexes:** `(item_id)`, `(expiry_date) WHERE status = 'active'`

---

### `serial`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | item.tracking_type must be 'serial' |
| `serial_number` | VARCHAR(200) | NOT NULL | |
| `warehouse_id` | BIGINT | NOT NULL FK → warehouse(id) | |
| `bin_id` | BIGINT | FK → bin(id) | |
| `batch_id` | BIGINT | FK → batch(id) | Optional link to batch |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'available' | available \| sold \| returned |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraint:** UNIQUE (item_id, serial_number)  
**Index:** `(status) WHERE status = 'available'`

---

### `inventory_transaction`
Insert-only stock ledger. Every stock movement creates a row here. Current stock = `SUM(qty)` per item + warehouse.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `warehouse_id` | BIGINT | NOT NULL FK → warehouse(id) | |
| `bin_id` | BIGINT | FK → bin(id) | |
| `batch_id` | BIGINT | FK → batch(id) | |
| `serial_id` | BIGINT | FK → serial(id) | |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `qty` | NUMERIC(18,4) | NOT NULL | Negative = out (sale); positive = in (return/receipt) |
| `unit_cost` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `transaction_type` | VARCHAR(50) | NOT NULL | See values below |
| `source_doc_type` | VARCHAR(50) | | sales_invoice, delivery_note, etc. |
| `source_doc_id` | BIGINT | | FK to the source document |
| `source_doc_line_id` | BIGINT | | FK to the source document line |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `transaction_date` | DATE | NOT NULL | |
| `transaction_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `remarks` | TEXT | | |

**`transaction_type` values:** `sale_invoice` · `sale_return` · `purchase_receipt` · `purchase_return` · `stock_transfer_out` · `stock_transfer_in` · `inventory_adjustment` · `opening_stock`

**Indexes:** `(item_id, warehouse_id)` · `(transaction_date)` · `(source_doc_type, source_doc_id)` · `(batch_id)` · `(serial_id)`

---

### `item_stock_balance` (Materialized View)
Refreshed by trigger after every `inventory_transaction` insert.

```sql
SELECT
  item_id,
  warehouse_id,
  bin_id,
  uom_id,
  SUM(qty) AS qty_on_hand
FROM inventory_transaction
GROUP BY item_id, warehouse_id, bin_id, uom_id;
```

---

### `inventory_closing`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `financial_year` | VARCHAR(10) | NOT NULL | e.g. 2025-2026 |
| `closing_date` | DATE | NOT NULL | |
| `warehouse_id` | BIGINT | FK → warehouse(id) | NULL = all warehouses |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'Draft' | Draft \| Closed |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `closed_by` | BIGINT | FK → app_user(id) | |
| `closed_at` | TIMESTAMPTZ | | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `inventory_closing_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `closing_id` | BIGINT | NOT NULL FK → inventory_closing(id) ON DELETE CASCADE | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `warehouse_id` | BIGINT | NOT NULL FK → warehouse(id) | |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `system_qty` | NUMERIC(18,4) | NOT NULL | Snapshot from inventory_transaction at closing_date |
| `physical_qty` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Entered by warehouse staff |
| `variance` | NUMERIC(18,4) | GENERATED ALWAYS AS (physical_qty - system_qty) STORED | |
| `cost_price` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `variance_value` | NUMERIC(18,4) | GENERATED ALWAYS AS ((physical_qty - system_qty) * cost_price) STORED | |

---

## 4. Party

### `party`
Unified Customer + Supplier entity. A party can be both simultaneously (`is_customer = true AND is_supplier = true`).

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `party_code` | VARCHAR(50) | NOT NULL UNIQUE | Auto-generated: PTY-0001 |
| `name` | VARCHAR(500) | NOT NULL | |
| `name_ar` | VARCHAR(500) | | |
| `mobile` | VARCHAR(30) | | |
| `phone` | VARCHAR(30) | | |
| `whatsapp` | VARCHAR(30) | | |
| `email` | VARCHAR(255) | | |
| `vat_number` | VARCHAR(20) | UNIQUE | 15-digit Saudi VAT |
| `is_customer` | BOOLEAN | NOT NULL DEFAULT false | |
| `is_supplier` | BOOLEAN | NOT NULL DEFAULT false | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| inactive \| blocked |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | Soft delete |

**Constraint:** `CHECK (is_customer = true OR is_supplier = true)` — must have at least one role.  
**Check:** `CHECK (vat_number ~ '^3[0-9]{13}3$')` when not null.  
**Indexes:** `(vat_number)` · `(mobile)` · `(is_customer) WHERE is_customer = true` · GIN full-text on `name || name_ar`

---

### `customer_category`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `code` | VARCHAR(30) | | e.g. VIP, RTL |
| `name` | VARCHAR(255) | NOT NULL UNIQUE | |
| `description` | TEXT | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `customer_data`
One-to-one extension of `party` for customer-specific fields.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `party_id` | BIGINT | PK FK → party(id) ON DELETE CASCADE | |
| `company_name` | VARCHAR(500) | | |
| `name_ar` | VARCHAR(500) | | |
| `logo_file_path` | VARCHAR(500) | | |
| `category_id` | BIGINT | FK → customer_category(id) | |
| `credit_limit` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `credit_days` | SMALLINT | NOT NULL DEFAULT 0 | |
| `payment_term` | VARCHAR(20) | NOT NULL DEFAULT 'Immediate' | Immediate \| Net7 \| Net15 \| Net30 \| Net45 \| Net60 |
| `customer_type` | VARCHAR(10) | NOT NULL DEFAULT 'B2C' | B2B \| B2C \| B2G |
| `price_list_id` | BIGINT | FK → price_list(id) | |
| `outstanding` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED; updated by payment trigger |
| `credit_status` | VARCHAR(20) | NOT NULL DEFAULT 'Good' | Good \| Warning \| Overdue |
| `city` | VARCHAR(100) | | |
| `country` | VARCHAR(100) | DEFAULT 'Saudi Arabia' | |
| `region` | VARCHAR(100) | | |
| `address` | TEXT | | |
| `national_address` | VARCHAR(500) | | Saudi National Address for ZATCA |
| `state` | VARCHAR(100) | | For ZATCA address block |
| `state_code` | VARCHAR(20) | | Two-letter ISO |
| `cr_number` | VARCHAR(30) | | Commercial Registration |
| `area_id` | BIGINT | FK → area(id) | |
| `route_id` | BIGINT | FK → route(id) | |
| `salesperson_id` | BIGINT | FK → salesperson(id) | |
| `credit_term_id` | BIGINT | FK → credit_term(id) | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `customer_branch`
A customer can have multiple branch addresses.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `party_id` | BIGINT | NOT NULL FK → party(id) | |
| `branch_name` | VARCHAR(255) | NOT NULL | |
| `city` | VARCHAR(100) | | |
| `address` | TEXT | | |
| `phone` | VARCHAR(30) | | |
| `contact_person` | VARCHAR(255) | | |
| `is_primary` | BOOLEAN | NOT NULL DEFAULT false | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Index:** `(party_id)`

---

## 5. Sales Masters

### `salesperson`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `employee_id` | BIGINT | FK → employee(id) | Optional HR link |
| `name` | VARCHAR(255) | NOT NULL | |
| `email` | VARCHAR(255) | | |
| `phone` | VARCHAR(30) | | |
| `commission_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 0 | Percentage (e.g. 2.5 = 2.5%) |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `price_list`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `name` | VARCHAR(255) | NOT NULL UNIQUE | |
| `currency` | CHAR(3) | NOT NULL DEFAULT 'SAR' | |
| `branch_id` | BIGINT | FK → branch(id) | NULL = all branches |
| `valid_from` | DATE | | |
| `valid_to` | DATE | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `price_list_item`
Per-item overrides with tier pricing support.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `price_list_id` | BIGINT | NOT NULL FK → price_list(id) | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `unit_price` | NUMERIC(18,4) | NOT NULL | |
| `min_qty` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Apply when ordered qty >= min_qty |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraint:** UNIQUE (price_list_id, item_id, uom_id, min_qty)

---

### `promotion`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `name` | VARCHAR(255) | NOT NULL | |
| `promo_type` | VARCHAR(30) | NOT NULL | ItemDiscount \| BillDiscount \| BuyOneGetOne \| MixCombo |
| `scope_description` | TEXT | | Human-readable: 'All Accessories', 'Bills above SAR 1,000' |
| `discount_value` | NUMERIC(18,4) | | For ItemDiscount / BillDiscount |
| `discount_type` | VARCHAR(10) | NOT NULL DEFAULT 'percent' | percent \| flat |
| `min_bill_amount` | NUMERIC(18,4) | | BillDiscount trigger threshold |
| `combo_price` | NUMERIC(18,4) | | MixCombo bundle price |
| `branch_id` | BIGINT | FK → branch(id) | NULL = all branches |
| `valid_from` | DATE | | |
| `valid_to` | DATE | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `promotion_item`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `promotion_id` | BIGINT | NOT NULL FK → promotion(id) | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `role` | VARCHAR(20) | NOT NULL DEFAULT 'target' | trigger (buy this) \| target (get this free) |
| `qty_required` | NUMERIC(18,4) | NOT NULL DEFAULT 1 | |
| `discount_value` | NUMERIC(18,4) | | Per-item override |
| `discount_type` | VARCHAR(10) | | percent \| flat |

**Constraint:** UNIQUE (promotion_id, item_id, role)

---

## 6. Sales Documents

### Document Number Format
`{DOC_TYPE}-{YYYY}-{NNNN}` — e.g. `INV-2026-0001`  
Generated via `document_sequence` table with `SELECT … FOR UPDATE`.

---

### `quotation`
**Status flow:** `pending` → `approved` / `rejected` → `converted` (locked)

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `quote_number` | VARCHAR(30) | NOT NULL UNIQUE | QT-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `customer_id` | BIGINT | NOT NULL FK → party(id) | is_customer must be true |
| `salesperson_id` | BIGINT | FK → salesperson(id) | |
| `created_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `expiry_date` | DATE | | |
| `validity_days` | SMALLINT | NOT NULL DEFAULT 30 | |
| `ref_no` | VARCHAR(100) | | Customer's RFQ number |
| `tax_option` | VARCHAR(20) | NOT NULL DEFAULT 'exclusive' | inclusive \| exclusive |
| `remarks` | TEXT | | |
| `terms_and_conditions` | TEXT | | |
| `sub_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `total_discount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `vat_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `grand_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'pending' | pending \| approved \| rejected \| converted |
| `converted_to_order_id` | BIGINT | FK → sales_order(id) | Set when converted |
| `approved_by` | BIGINT | FK → app_user(id) | |
| `approved_at` | TIMESTAMPTZ | | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |

**Indexes:** `(customer_id)` · `(branch_id)` · `(status)` · `(quote_number)`

---

### `quotation_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `quotation_id` | BIGINT | NOT NULL FK → quotation(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | Display order |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | Denormalized snapshot |
| `item_name` | VARCHAR(500) | NOT NULL | Denormalized snapshot |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | Denormalized |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | |
| `rate` | NUMERIC(18,4) | NOT NULL CHECK (rate >= 0) | |
| `discount_pct` | NUMERIC(8,4) | NOT NULL DEFAULT 0 CHECK (0–100) | |
| `taxable_value` | NUMERIC(18,4) | NOT NULL | STORED: qty × rate × (1 − disc/100) |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | Snapshot at quote time |
| `vat_amount` | NUMERIC(18,4) | NOT NULL | STORED: taxable × vat_rate / 100 |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED: taxable + vat_amount |
| `note` | TEXT | | |

**Index:** `(quotation_id)`

---

### `sales_order`
**Status flow:** `confirmed` → `processing` → `delivered` | `cancelled`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `order_number` | VARCHAR(30) | NOT NULL UNIQUE | SO-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `customer_id` | BIGINT | NOT NULL FK → party(id) | |
| `quotation_id` | BIGINT | FK → quotation(id) | Source quotation if converted |
| `salesperson_id` | BIGINT | FK → salesperson(id) | |
| `sales_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `delivery_date` | DATE | | Expected delivery date |
| `ref_no` | VARCHAR(100) | | Links back to QT number (display) |
| `tax_option` | VARCHAR(20) | NOT NULL DEFAULT 'exclusive' | |
| `remarks` | TEXT | | |
| `terms_and_conditions` | TEXT | | |
| `sub_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `total_discount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `vat_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `grand_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'confirmed' | confirmed \| processing \| delivered \| cancelled |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |

**Indexes:** `(customer_id)` · `(quotation_id)` · `(branch_id)` · `(status)`

---

### `sales_order_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `order_id` | BIGINT | NOT NULL FK → sales_order(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | |
| `item_name` | VARCHAR(500) | NOT NULL | |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | |
| `qty_delivered` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Updated by delivery_note trigger |
| `qty_invoiced` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Updated by invoice trigger |
| `rate` | NUMERIC(18,4) | NOT NULL CHECK (rate >= 0) | |
| `discount_pct` | NUMERIC(8,4) | NOT NULL DEFAULT 0 | |
| `taxable_value` | NUMERIC(18,4) | NOT NULL | STORED |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | |
| `vat_amount` | NUMERIC(18,4) | NOT NULL | STORED |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED |
| `note` | TEXT | | |

**Index:** `(order_id)`

---

### `delivery_note`
**Status flow:** `pending` → `dispatched` → `delivered` | `cancelled`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `delivery_number` | VARCHAR(30) | NOT NULL UNIQUE | DN-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `customer_id` | BIGINT | NOT NULL FK → party(id) | |
| `warehouse_id` | BIGINT | NOT NULL FK → warehouse(id) | Stock taken from this warehouse |
| `source_type` | VARCHAR(20) | NOT NULL | sales_order \| quotation |
| `source_id` | BIGINT | NOT NULL | FK to sales_order.id OR quotation.id (polymorphic) |
| `delivery_date` | DATE | NOT NULL | |
| `driver_name` | VARCHAR(255) | | |
| `vehicle_plate` | VARCHAR(50) | | |
| `remarks` | TEXT | | |
| `converted_to_invoice_id` | BIGINT | FK → sales_invoice(id) | Set when converted |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'pending' | pending \| dispatched \| delivered \| cancelled |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |

**Indexes:** `(customer_id)` · `(source_type, source_id)` · `(warehouse_id)` · `(status)`

---

### `delivery_note_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `delivery_note_id` | BIGINT | NOT NULL FK → delivery_note(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | |
| `item_name` | VARCHAR(500) | NOT NULL | |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `source_line_id` | BIGINT | | FK to sales_order_line or quotation_line |
| `ordered_qty` | NUMERIC(18,4) | NOT NULL | From source document |
| `delivered_qty` | NUMERIC(18,4) | NOT NULL CHECK (delivered_qty >= 0) | Partial delivery supported |
| `batch_id` | BIGINT | FK → batch(id) | |
| `serial_id` | BIGINT | FK → serial(id) | |
| `bin_id` | BIGINT | FK → bin(id) | Picked from this bin |
| `note` | TEXT | | |

**Check:** `delivered_qty <= ordered_qty`  
**Index:** `(delivery_note_id)`

---

### `sales_invoice`
The most complex table. Contains ZATCA compliance fields, payment split, and AR tracking.  
**Status flow:** `unpaid` → `partial` → `paid` | `cancelled` | `void`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `invoice_number` | VARCHAR(30) | NOT NULL UNIQUE | INV-2026-0001 (per-branch sequence) |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `customer_id` | BIGINT | NOT NULL FK → party(id) | |
| **Source links** | | | All nullable — invoice can be standalone |
| `delivery_note_id` | BIGINT | FK → delivery_note(id) | |
| `sales_order_id` | BIGINT | FK → sales_order(id) | |
| `quotation_id` | BIGINT | FK → quotation(id) | |
| **Classification** | | | |
| `invoice_type` | VARCHAR(10) | NOT NULL DEFAULT 'B2C' | B2B \| B2C |
| `invoice_mode` | VARCHAR(20) | NOT NULL DEFAULT 'credit' | cash \| credit |
| `tax_option` | VARCHAR(20) | NOT NULL DEFAULT 'exclusive' | inclusive \| exclusive |
| **Customer snapshot** | | Denormalized — must not change with master data | |
| `customer_name` | VARCHAR(500) | NOT NULL | |
| `customer_name_ar` | VARCHAR(500) | | |
| `customer_vat_number` | VARCHAR(20) | | |
| `customer_cr_number` | VARCHAR(30) | | |
| `customer_address` | TEXT | | |
| `customer_national_address` | VARCHAR(500) | | Saudi National Address |
| `customer_city` | VARCHAR(100) | | |
| `customer_state` | VARCHAR(100) | | |
| `customer_state_code` | VARCHAR(20) | | Two-letter ISO |
| `customer_mobile` | VARCHAR(30) | | |
| **Dates** | | | |
| `invoice_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `due_date` | DATE | | invoice_date + credit_days |
| **Sales team** | | | |
| `salesperson_id` | BIGINT | FK → salesperson(id) | |
| **Reference fields** | | | |
| `ref_no` | VARCHAR(100) | | |
| `additional_field1` | VARCHAR(255) | | |
| `additional_field2` | VARCHAR(255) | | |
| `remarks` | TEXT | | |
| `terms_and_conditions` | TEXT | | |
| **Line totals** | | STORED, updated on line save | |
| `sub_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Sum of taxable_value across lines |
| `total_discount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Sum of line discounts |
| `freight_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Shipping charge |
| `footer_discount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Document-level discount |
| `agent_commission` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `adjustment_sign` | CHAR(1) | NOT NULL DEFAULT '+' | + \| − |
| `adjustment_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `adjustment_account_id` | BIGINT | FK → coa_account(id) | GL for the adjustment |
| `vat_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `round_off` | BOOLEAN | NOT NULL DEFAULT false | |
| `round_off_sign` | CHAR(1) | NOT NULL DEFAULT '+' | |
| `round_off_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `grand_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| **Payment split** | | For cash invoices | |
| `cash_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `card_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `upi_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `bank_transfer_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `credit_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `advance_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Applied advance/credit note |
| `card_machine_id` | BIGINT | FK → pos_register(id) | Terminal used for card payment |
| `bank_account_id` | BIGINT | FK → coa_account(id) | Bank account for transfer |
| **Credit terms** | | | |
| `credit_days` | SMALLINT | NOT NULL DEFAULT 0 | |
| `ledger_account_id` | BIGINT | FK → coa_account(id) | AR GL account |
| **AR tracking** | | STORED; updated by payment trigger | |
| `paid_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `outstanding_balance` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | grand_total − paid_amount |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'unpaid' | unpaid \| partial \| paid \| cancelled \| void |
| **ZATCA Phase-2** | | | |
| `zatca_invoice_uuid` | UUID | UNIQUE DEFAULT gen_random_uuid() | Required in TLV QR |
| `zatca_hash` | VARCHAR(500) | | Hash of this invoice's XML (for chaining) |
| `zatca_qr` | TEXT | | Base64-encoded TLV QR code |
| `zatca_xml` | TEXT | | Full UBL 2.1 XML (stored for resubmission) |
| `zatca_status` | VARCHAR(30) | NOT NULL DEFAULT 'pending' | pending \| reported \| cleared \| rejected \| not_applicable |
| `zatca_reported_at` | TIMESTAMPTZ | | |
| `zatca_clearance_at` | TIMESTAMPTZ | | |
| `zatca_rejection_reason` | TEXT | | |
| **Audit** | | | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |
| `voided_by` | BIGINT | FK → app_user(id) | |
| `voided_at` | TIMESTAMPTZ | | |
| `void_reason` | TEXT | | |

**Indexes:** `(customer_id)` · `(branch_id)` · `(status)` · `(invoice_date)` · `(invoice_number)` · `(delivery_note_id)` · `(sales_order_id)` · `(zatca_status) WHERE zatca_status != 'not_applicable'` · `(customer_id, outstanding_balance) WHERE outstanding_balance > 0`

---

### `sales_invoice_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `invoice_id` | BIGINT | NOT NULL FK → sales_invoice(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | Snapshot |
| `item_name` | VARCHAR(500) | NOT NULL | Snapshot |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | Snapshot |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | |
| `rate` | NUMERIC(18,4) | NOT NULL CHECK (rate >= 0) | |
| `discount_pct` | NUMERIC(8,4) | NOT NULL DEFAULT 0 CHECK (0–100) | |
| `taxable_value` | NUMERIC(18,4) | NOT NULL | STORED: qty × rate × (1 − disc/100) |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | Snapshot from item at invoice time |
| `vat_amount` | NUMERIC(18,4) | NOT NULL | STORED: taxable × vat_rate / 100 |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED: taxable + vat_amount |
| `note` | TEXT | | Per-line note |
| `batch_id` | BIGINT | FK → batch(id) | Required if item.tracking_type = 'batch' |
| `serial_id` | BIGINT | FK → serial(id) | Required if item.tracking_type = 'serial' |
| `source_line_id` | BIGINT | | Traces back to sales_order_line or delivery_note_line |

**Indexes:** `(invoice_id)` · `(item_id)` · `(batch_id)` · `(serial_id)`

---

### `sales_return`
**Status flow:** `pending` → `approved` → `refunded` | `cancelled`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `return_number` | VARCHAR(30) | NOT NULL UNIQUE | RET-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `customer_id` | BIGINT | NOT NULL FK → party(id) | |
| `invoice_id` | BIGINT | FK → sales_invoice(id) | NULL if return_type = 'without_invoice' |
| `return_type` | VARCHAR(25) | NOT NULL DEFAULT 'with_invoice' | with_invoice \| without_invoice |
| `invoice_type` | VARCHAR(10) | NOT NULL DEFAULT 'B2C' | B2B \| B2C |
| `return_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `return_reason` | TEXT | | |
| `sub_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `vat_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `refund_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED: sub_total + vat_amount |
| `zatca_invoice_uuid` | UUID | UNIQUE DEFAULT gen_random_uuid() | ZATCA: this is a "Credit Invoice" |
| `zatca_qr` | TEXT | | |
| `zatca_xml` | TEXT | | |
| `zatca_status` | VARCHAR(30) | NOT NULL DEFAULT 'pending' | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'pending' | pending \| approved \| refunded \| cancelled |
| `approved_by` | BIGINT | FK → app_user(id) | |
| `approved_at` | TIMESTAMPTZ | | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |

**Indexes:** `(customer_id)` · `(invoice_id)` · `(status)` · `(return_date)`

---

### `sales_return_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `return_id` | BIGINT | NOT NULL FK → sales_return(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | |
| `item_name` | VARCHAR(500) | NOT NULL | |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | |
| `rate` | NUMERIC(18,4) | NOT NULL CHECK (rate >= 0) | |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | |
| `taxable_value` | NUMERIC(18,4) | NOT NULL | STORED |
| `vat_amount` | NUMERIC(18,4) | NOT NULL | STORED |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED |
| `batch_id` | BIGINT | FK → batch(id) | |
| `serial_id` | BIGINT | FK → serial(id) | |
| `source_invoice_line_id` | BIGINT | FK → sales_invoice_line(id) | Original invoice line |

**Index:** `(return_id)`

---

### `credit_note`
**Status flow:** `Draft` → `Approved` | `Cancelled`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `credit_note_number` | VARCHAR(30) | NOT NULL UNIQUE | CN-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `customer_id` | BIGINT | NOT NULL FK → party(id) | |
| `invoice_id` | BIGINT | FK → sales_invoice(id) | The invoice being credited |
| `return_id` | BIGINT | FK → sales_return(id) | If created from a return |
| `amount` | NUMERIC(18,4) | NOT NULL CHECK (amount > 0) | |
| `reason` | TEXT | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'Draft' | Draft \| Approved \| Cancelled |
| `approved_by` | BIGINT | FK → app_user(id) | |
| `approved_at` | TIMESTAMPTZ | | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Indexes:** `(customer_id)` · `(invoice_id)` · `(status)`

---

### `credit_note_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `credit_note_id` | BIGINT | NOT NULL FK → credit_note(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | |
| `item_name` | VARCHAR(500) | NOT NULL | |
| `qty` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `rate` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED: qty × rate |

---

### `sales_payment`
A receipt from a customer. One payment can settle multiple invoices via allocations.  
**Numbering:** `RCP-2026-0001`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `receipt_number` | VARCHAR(30) | NOT NULL UNIQUE | RCP-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `party_id` | BIGINT | NOT NULL FK → party(id) | |
| `payment_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `payment_method` | VARCHAR(20) | NOT NULL | cash \| bank \| pos \| cheque |
| `bank_account_id` | BIGINT | FK → coa_account(id) | If method = bank |
| `pos_machine_id` | BIGINT | FK → pos_register(id) | If method = pos |
| `cheque_number` | VARCHAR(50) | | If method = cheque |
| `cheque_date` | DATE | | |
| `cheque_bank` | VARCHAR(100) | | |
| `payment_amount` | NUMERIC(18,4) | NOT NULL CHECK (payment_amount > 0) | Total received |
| `advance_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Portion not allocated to any invoice |
| `narration` | TEXT | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'completed' | completed \| voided |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Business rule:** `SUM(sales_payment_allocation.allocated_amount) + advance_amount = payment_amount` (app enforced)

**Indexes:** `(party_id)` · `(payment_date)` · `(branch_id)`

---

### `sales_payment_allocation`
Links a receipt to specific invoices. Trigger on this table updates invoice AR status.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `payment_id` | BIGINT | NOT NULL FK → sales_payment(id) ON DELETE CASCADE | |
| `invoice_id` | BIGINT | NOT NULL FK → sales_invoice(id) | |
| `allocated_amount` | NUMERIC(18,4) | NOT NULL CHECK (allocated_amount > 0) | |
| `allocated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraint:** UNIQUE (payment_id, invoice_id)  
**Indexes:** `(payment_id)` · `(invoice_id)`

> **Trigger on INSERT/UPDATE/DELETE:**
> 1. `UPDATE sales_invoice SET paid_amount = SUM(allocated_amount), outstanding_balance = grand_total − paid_amount, status = CASE WHEN outstanding_balance = 0 THEN 'paid' WHEN paid_amount > 0 THEN 'partial' ELSE 'unpaid' END`
> 2. `UPDATE customer_data SET outstanding = SUM(sales_invoice.outstanding_balance) WHERE status != 'paid'`

---

### Supporting: `employee`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `employee_code` | VARCHAR(50) | NOT NULL UNIQUE | |
| `name` | VARCHAR(255) | NOT NULL | |
| `department` | VARCHAR(100) | | |
| `position` | VARCHAR(100) | | |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `phone` | VARCHAR(30) | | |
| `email` | VARCHAR(255) | | |
| `hire_date` | DATE | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### Supporting: `audit_log`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `user_id` | BIGINT | FK → app_user(id) | |
| `branch_id` | BIGINT | FK → branch(id) | |
| `action` | VARCHAR(50) | NOT NULL | CREATE \| UPDATE \| DELETE \| APPROVE \| VOID \| LOGIN |
| `table_name` | VARCHAR(100) | NOT NULL | |
| `record_id` | BIGINT | | |
| `old_data` | JSONB | | Previous state |
| `new_data` | JSONB | | New state |
| `ip_address` | INET | | |
| `occurred_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

## 7. Purchase Masters

### `supplier_category`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `code` | VARCHAR(30) | | e.g. SUP-VIP |
| `name` | VARCHAR(255) | NOT NULL UNIQUE | |
| `description` | TEXT | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `supplier_data`
One-to-one extension of `party` for supplier-specific fields. Mirrors `customer_data` on the sales side.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `party_id` | BIGINT | PK FK → party(id) ON DELETE CASCADE | |
| `company_name` | VARCHAR(500) | | |
| `company_name_ar` | VARCHAR(500) | | |
| `purchaser_id` | BIGINT | FK → purchaser(id) | Assigned internal buyer |
| `category_id` | BIGINT | FK → supplier_category(id) | |
| `supplier_type` | VARCHAR(10) | NOT NULL DEFAULT 'B2B' | B2B \| B2C \| B2G |
| `credit_term_id` | BIGINT | FK → credit_term(id) | |
| `credit_limit` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `credit_days` | SMALLINT | NOT NULL DEFAULT 0 | |
| `payment_term` | VARCHAR(20) | NOT NULL DEFAULT 'Immediate' | Immediate \| Net7 \| Net15 \| Net30 \| Net45 \| Net60 |
| `outstanding` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED; updated by supplier payment trigger |
| `credit_status` | VARCHAR(20) | NOT NULL DEFAULT 'Good' | Good \| Warning \| Overdue |
| `opening_balance` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | |
| `area_id` | BIGINT | FK → area(id) | |
| `bank_name` | VARCHAR(255) | | |
| `account_name` | VARCHAR(255) | | |
| `account_number` | VARCHAR(100) | | |
| `iban` | VARCHAR(50) | | |
| `bank_account_copy_path` | VARCHAR(500) | | Uploaded file path |
| `left_doc_path` | VARCHAR(500) | | e.g. VAT certificate |
| `right_doc_path` | VARCHAR(500) | | e.g. CR certificate |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

### `purchaser`
Internal buyer / procurement staff — mirrors `salesperson` on the sales side.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `employee_id` | BIGINT | FK → employee(id) | Optional HR link |
| `name` | VARCHAR(255) | NOT NULL | |
| `email` | VARCHAR(255) | | |
| `phone` | VARCHAR(30) | | |
| `commission_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 0 | Percentage (e.g. 2.5 = 2.5%) |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'active' | active \| inactive |
| `created_by` | BIGINT | FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

---

## 8. Purchase Documents

### Document Number Format
`{DOC_TYPE}-{YYYY}-{NNNN}` — e.g. `PO-2026-0001`, `GRN-2026-0001`, `PINV-2026-0001`  
Generated via `document_sequence` table with `SELECT … FOR UPDATE`.

---

### `purchase_request`
Internal requisition raised by a department before a Purchase Order is issued.  
**Status flow:** `pending` → `approved` / `rejected`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `request_number` | VARCHAR(30) | NOT NULL UNIQUE | REQ-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `requested_by` | BIGINT | NOT NULL FK → app_user(id) | User raising the request |
| `department` | VARCHAR(100) | NOT NULL | Production \| Maintenance \| IT \| HR \| Sales \| Accounting |
| `request_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `notes` | TEXT | | Purpose / justification |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'pending' | pending \| approved \| rejected |
| `approved_by` | BIGINT | FK → app_user(id) | |
| `approved_at` | TIMESTAMPTZ | | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |

**Indexes:** `(branch_id)` · `(status)` · `(request_date)`

---

### `purchase_request_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `request_id` | BIGINT | NOT NULL FK → purchase_request(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | FK → item(id) | NULL if requesting a new item not yet in master |
| `item_name` | VARCHAR(500) | NOT NULL | From item master or entered manually |
| `description` | TEXT | | |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | |
| `required_date` | DATE | | When the item is needed |
| `is_perishable` | BOOLEAN | NOT NULL DEFAULT false | |
| `expiry_date` | DATE | | Required if is_perishable = true |

**Index:** `(request_id)`

---

### `purchase_order`
Formal purchase order sent to the supplier.  
**Status flow:** `draft` → `sent` → `partially_received` → `received` → `closed`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `po_number` | VARCHAR(30) | NOT NULL UNIQUE | PO-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `supplier_id` | BIGINT | NOT NULL FK → party(id) | is_supplier must be true |
| `purchaser_id` | BIGINT | FK → purchaser(id) | |
| `source_request_id` | BIGINT | FK → purchase_request(id) | Source PR if converted |
| `order_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `expected_delivery` | DATE | | Auto-filled from supplier credit term |
| `ref_no` | VARCHAR(100) | | Internal / external reference |
| `quotation_ref` | VARCHAR(100) | | Supplier's quotation number |
| `quotation_attachment_path` | VARCHAR(500) | | |
| `currency` | CHAR(3) | NOT NULL DEFAULT 'SAR' | |
| `sub_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `vat_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `grand_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'draft' | draft \| sent \| partially_received \| received \| closed |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |

**Indexes:** `(supplier_id)` · `(branch_id)` · `(status)` · `(po_number)`

---

### `purchase_order_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `po_id` | BIGINT | NOT NULL FK → purchase_order(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | Denormalized snapshot |
| `item_name` | VARCHAR(500) | NOT NULL | Denormalized snapshot |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | Snapshot |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | Ordered quantity |
| `qty_received` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Updated by GRN trigger |
| `qty_invoiced` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Updated by purchase invoice trigger |
| `unit_price` | NUMERIC(18,4) | NOT NULL CHECK (unit_price >= 0) | |
| `discount_pct` | NUMERIC(8,4) | NOT NULL DEFAULT 0 CHECK (0–100) | |
| `taxable_value` | NUMERIC(18,4) | NOT NULL | STORED: qty × unit_price × (1 − disc/100) |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | Snapshot at PO time |
| `vat_amount` | NUMERIC(18,4) | NOT NULL | STORED |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED |

**Index:** `(po_id)`

---

### `grn`
Goods Received Note — records physical receipt of goods against a Purchase Order. Supports partial receipt.  
**Status flow:** `draft` → `confirmed` | `cancelled`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `grn_number` | VARCHAR(30) | NOT NULL UNIQUE | GRN-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `supplier_id` | BIGINT | NOT NULL FK → party(id) | |
| `po_id` | BIGINT | FK → purchase_order(id) | NULL if GRN without a PO |
| `warehouse_id` | BIGINT | NOT NULL FK → warehouse(id) | Receiving warehouse |
| `received_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `remarks` | TEXT | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'draft' | draft \| confirmed \| cancelled |
| `confirmed_by` | BIGINT | FK → app_user(id) | |
| `confirmed_at` | TIMESTAMPTZ | | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Indexes:** `(supplier_id)` · `(po_id)` · `(warehouse_id)` · `(status)` · `(received_date)`

---

### `grn_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `grn_id` | BIGINT | NOT NULL FK → grn(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | Snapshot |
| `item_name` | VARCHAR(500) | NOT NULL | Snapshot |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | Snapshot |
| `po_line_id` | BIGINT | FK → purchase_order_line(id) | NULL if GRN without PO |
| `ordered_qty` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | From PO line |
| `received_qty` | NUMERIC(18,4) | NOT NULL CHECK (received_qty >= 0) | Actual qty received |
| `damaged_qty` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Damaged units |
| `accepted_qty` | NUMERIC(18,4) | GENERATED ALWAYS AS (received_qty - damaged_qty) STORED | |
| `unit_cost` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Purchase cost per unit |
| `batch_id` | BIGINT | FK → batch(id) | For batch-tracked items |
| `batch_number` | VARCHAR(100) | | New batch created inline on GRN |
| `manufacturing_date` | DATE | | |
| `expiry_date` | DATE | | Auto-alert triggered 30 days before expiry |
| `bin_id` | BIGINT | FK → bin(id) | Putaway location (nullable) |
| `note` | TEXT | | |

**Check:** `received_qty >= damaged_qty`  
**Indexes:** `(grn_id)` · `(item_id)` · `(batch_id)` · `(expiry_date) WHERE expiry_date IS NOT NULL`

---

### `purchase_invoice`
Supplier invoice with 3-way match validation (PO / GRN / Invoice).  
**Status flow:** `pending` → `partially_paid` → `paid` | `overdue`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `invoice_number` | VARCHAR(30) | NOT NULL UNIQUE | PINV-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `supplier_id` | BIGINT | NOT NULL FK → party(id) | |
| `purchaser_id` | BIGINT | FK → purchaser(id) | |
| `po_id` | BIGINT | FK → purchase_order(id) | Linked PO (nullable — standalone invoice allowed) |
| `grn_id` | BIGINT | FK → grn(id) | Linked GRN (nullable) |
| `invoice_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `due_date` | DATE | | invoice_date + payment_term days |
| `payment_term` | VARCHAR(20) | NOT NULL DEFAULT 'Immediate' | |
| `payment_method` | VARCHAR(30) | | bank_transfer \| cash \| corporate_card \| cheque |
| `bank_account_id` | BIGINT | FK → coa_account(id) | Bank account for payment |
| `tax_option` | VARCHAR(20) | NOT NULL DEFAULT 'exclusive' | inclusive \| exclusive |
| `ref_no` | VARCHAR(100) | | Supplier's own invoice number |
| `sub_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `total_discount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `vat_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `grand_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `paid_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED; updated by supplier payment trigger |
| `outstanding_balance` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | grand_total − paid_amount |
| `three_way_match_status` | VARCHAR(20) | NOT NULL DEFAULT 'pending' | pending \| matched \| mismatched |
| `additional_note1` | VARCHAR(255) | | |
| `additional_note2` | VARCHAR(255) | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'pending' | pending \| partially_paid \| paid \| overdue |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | | |

**Indexes:** `(supplier_id)` · `(branch_id)` · `(status)` · `(invoice_date)` · `(po_id)` · `(grn_id)` · `(supplier_id, outstanding_balance) WHERE outstanding_balance > 0`

---

### `purchase_invoice_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `invoice_id` | BIGINT | NOT NULL FK → purchase_invoice(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | Snapshot |
| `item_name` | VARCHAR(500) | NOT NULL | Snapshot |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | Snapshot |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | |
| `unit_price` | NUMERIC(18,4) | NOT NULL CHECK (unit_price >= 0) | |
| `discount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Flat discount amount |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | Snapshot |
| `taxable_value` | NUMERIC(18,4) | NOT NULL | STORED: (qty × unit_price) − discount |
| `vat_amount` | NUMERIC(18,4) | NOT NULL | STORED |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED |
| `po_line_id` | BIGINT | FK → purchase_order_line(id) | Traces to PO line |
| `grn_line_id` | BIGINT | FK → grn_line(id) | Traces to GRN line |

**Indexes:** `(invoice_id)` · `(item_id)`

---

### `supplier_payment`
An outgoing payment to a supplier. One payment can settle multiple invoices via allocations.  
**Numbering:** `SPAY-2026-0001`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `payment_number` | VARCHAR(30) | NOT NULL UNIQUE | SPAY-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `supplier_id` | BIGINT | NOT NULL FK → party(id) | |
| `payment_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `payment_method` | VARCHAR(30) | NOT NULL | bank_transfer \| cash \| corporate_card \| cheque |
| `bank_account_id` | BIGINT | FK → coa_account(id) | If method = bank_transfer |
| `cheque_number` | VARCHAR(50) | | |
| `cheque_date` | DATE | | |
| `cheque_bank` | VARCHAR(100) | | |
| `payment_amount` | NUMERIC(18,4) | NOT NULL CHECK (payment_amount > 0) | Total paid out |
| `advance_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | Portion not linked to any invoice |
| `transaction_ref` | VARCHAR(200) | | Bank transaction reference |
| `narration` | TEXT | | |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'completed' | completed \| voided |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Business rule:** `SUM(supplier_payment_allocation.allocated_amount) + advance_amount = payment_amount` (app enforced)  
**Indexes:** `(supplier_id)` · `(payment_date)` · `(branch_id)`

---

### `supplier_payment_allocation`
Links an outgoing payment to specific purchase invoices. Trigger on this table updates invoice AP status.

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `payment_id` | BIGINT | NOT NULL FK → supplier_payment(id) ON DELETE CASCADE | |
| `invoice_id` | BIGINT | NOT NULL FK → purchase_invoice(id) | |
| `allocated_amount` | NUMERIC(18,4) | NOT NULL CHECK (allocated_amount > 0) | |
| `allocated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Constraint:** UNIQUE (payment_id, invoice_id)  
**Indexes:** `(payment_id)` · `(invoice_id)`

> **Trigger on INSERT/UPDATE/DELETE:**
> 1. `UPDATE purchase_invoice SET paid_amount = SUM(allocated_amount), outstanding_balance = grand_total − paid_amount, status = CASE WHEN outstanding_balance = 0 THEN 'paid' WHEN paid_amount > 0 THEN 'partially_paid' ELSE 'pending' END`
> 2. `UPDATE supplier_data SET outstanding = SUM(purchase_invoice.outstanding_balance) WHERE status != 'paid'`

---

### `debit_note`
Issued to a supplier for returned goods, shortages, or billing adjustments.  
**Status flow:** `draft` → `approved` | `cancelled`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `uuid` | UUID | NOT NULL UNIQUE DEFAULT gen_random_uuid() | |
| `debit_note_number` | VARCHAR(30) | NOT NULL UNIQUE | DBN-2026-0001 |
| `branch_id` | BIGINT | NOT NULL FK → branch(id) | |
| `supplier_id` | BIGINT | NOT NULL FK → party(id) | |
| `invoice_id` | BIGINT | FK → purchase_invoice(id) | NULL if return_type = 'without_invoice' |
| `return_type` | VARCHAR(25) | NOT NULL DEFAULT 'with_invoice' | with_invoice \| without_invoice |
| `debit_date` | DATE | NOT NULL DEFAULT CURRENT_DATE | |
| `reason` | TEXT | | Return / shortage / adjustment reason |
| `sub_total` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `vat_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED |
| `total_amount` | NUMERIC(18,4) | NOT NULL DEFAULT 0 | STORED: sub_total + vat_amount |
| `status` | VARCHAR(20) | NOT NULL DEFAULT 'draft' | draft \| approved \| cancelled |
| `approved_by` | BIGINT | FK → app_user(id) | |
| `approved_at` | TIMESTAMPTZ | | |
| `created_by` | BIGINT | NOT NULL FK → app_user(id) | |
| `created_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | NOT NULL DEFAULT now() | |

**Indexes:** `(supplier_id)` · `(invoice_id)` · `(status)` · `(debit_date)`

---

### `debit_note_line`

| Column | Type | Constraints | Notes |
|---|---|---|---|
| `id` | BIGSERIAL | PK | |
| `debit_note_id` | BIGINT | NOT NULL FK → debit_note(id) ON DELETE CASCADE | |
| `line_number` | SMALLINT | NOT NULL | |
| `item_id` | BIGINT | NOT NULL FK → item(id) | |
| `item_code` | VARCHAR(50) | NOT NULL | Snapshot |
| `item_name` | VARCHAR(500) | NOT NULL | Snapshot |
| `uom_id` | BIGINT | NOT NULL FK → unit_of_measure(id) | |
| `uom_symbol` | VARCHAR(20) | NOT NULL | Snapshot |
| `qty` | NUMERIC(18,4) | NOT NULL CHECK (qty > 0) | |
| `rate` | NUMERIC(18,4) | NOT NULL CHECK (rate >= 0) | |
| `vat_rate` | NUMERIC(8,4) | NOT NULL DEFAULT 15 | |
| `taxable_value` | NUMERIC(18,4) | NOT NULL | STORED: qty × rate |
| `vat_amount` | NUMERIC(18,4) | NOT NULL | STORED |
| `line_total` | NUMERIC(18,4) | NOT NULL | STORED |
| `source_grn_line_id` | BIGINT | FK → grn_line(id) | Traces back to original GRN receipt line |

**Index:** `(debit_note_id)`

---

## 9. Trigger Architecture

| Trigger | Table | Event | Effect |
|---|---|---|---|
| `trg_invoice_payment_status` | `sales_payment_allocation` | INSERT / UPDATE / DELETE | Updates `sales_invoice.paid_amount`, `outstanding_balance`, `status` |
| `trg_customer_outstanding` | `sales_invoice` | INSERT / UPDATE | Updates `customer_data.outstanding` |
| `trg_inventory_out` | `sales_invoice_line` | INSERT | Inserts row in `inventory_transaction` (negative qty — stock leaves) |
| `trg_inventory_return` | `sales_return_line` | INSERT | Inserts row in `inventory_transaction` (positive qty — stock returns) |
| `trg_batch_qty_update` | `inventory_transaction` | INSERT | Updates `batch.quantity` |
| `trg_serial_status_sale` | `inventory_transaction` | INSERT (type = sale_invoice) | Sets `serial.status = 'sold'` |
| `trg_serial_status_return` | `inventory_transaction` | INSERT (type = sale_return) | Sets `serial.status = 'returned'` |
| `trg_so_line_qty_delivered` | `delivery_note_line` | INSERT / UPDATE | Updates `sales_order_line.qty_delivered` |
| `trg_so_line_qty_invoiced` | `sales_invoice_line` | INSERT | Updates `sales_order_line.qty_invoiced` |
| `trg_stock_balance_refresh` | `inventory_transaction` | INSERT | Refreshes `item_stock_balance` materialized view |
| `trg_grn_stock_in` | `grn_line` | INSERT (when GRN status = confirmed) | Inserts row in `inventory_transaction` (+qty, type = purchase_receipt) |
| `trg_debit_note_stock_out` | `debit_note_line` | INSERT (when debit_note status = approved) | Inserts row in `inventory_transaction` (−qty, type = purchase_return) |
| `trg_supplier_payment_status` | `supplier_payment_allocation` | INSERT / UPDATE / DELETE | Updates `purchase_invoice.paid_amount`, `outstanding_balance`, `status` |
| `trg_supplier_outstanding` | `purchase_invoice` | INSERT / UPDATE | Updates `supplier_data.outstanding` |
| `trg_po_line_qty_received` | `grn_line` | INSERT / UPDATE | Updates `purchase_order_line.qty_received` |
| `trg_po_line_qty_invoiced` | `purchase_invoice_line` | INSERT | Updates `purchase_order_line.qty_invoiced` |
| `trg_batch_create_on_grn` | `grn_line` | INSERT (batch-tracked items, GRN confirmed) | INSERTs or UPDATEs `batch` with `batch_number`, `expiry_date`, `manufacturing_date` from GRN |

---

## 10. ER Diagram

```
ORGANIZATION
  company (1)──(N) branch
  branch  (1)──(N) app_user ──(N:1) role ──(1)──(N) role_permission

MASTERS
  area (1)──(N) route
  branch (1)──(N) warehouse (1)──(N) bin
  unit_of_measure ─── (self-ref: base_unit_id)
  coa_account ──────── (self-ref: parent_id)
  item_category ─────── (self-ref: parent_id)

INVENTORY
  item_category (1)──(N) item
  item (1)──(N) item_supplier ──(N:1) party [supplier]
  item (1)──(N) item_uom ──(N:1) unit_of_measure
  item (1)──(N) batch ──(N:1) warehouse, bin
  item (1)──(N) serial ──(N:1) warehouse, bin, batch
  item + warehouse ──► inventory_transaction (insert-only ledger)
  inventory_transaction ──► item_stock_balance (materialized view)

PARTY
  party ──(1:0..1)── customer_data ──(N:1) area, route, price_list, salesperson
  party (1)──(N) customer_branch

SALES MASTERS
  price_list (1)──(N) price_list_item ──(N:1) item
  promotion (1)──(N) promotion_item ──(N:1) item

COMPLETE SALES DOCUMENT CHAIN
─────────────────────────────────────────────────────────────────────

       party (customer)
            │
            ▼
    [1] quotation
         │  └──(N) quotation_line ──(N:1) item
         │   status: pending → approved → converted
         │ converted_to_order_id
         ▼
    [2] sales_order
         │  └──(N) sales_order_line ──(N:1) item
         │   tracks: qty_delivered, qty_invoiced per line
         │   status: confirmed → processing → delivered
         │
    ┌────┴────────────────────────────────┐
    │ source_id (sales_order or quotation) │ (direct conversion)
    ▼                                      ▼
[3] delivery_note                   [4] sales_invoice
     │  └──(N) delivery_note_line         │  └──(N) sales_invoice_line
     │   ordered_qty vs delivered_qty     │   ─► inventory_transaction (−qty)
     │   status: pending → dispatched     │   ─► ZATCA QR + XML
     │           → delivered              │   status: unpaid → partial → paid
     │ converted_to_invoice_id            │
     └──────────────────────────────────►─┘
                                          │
               ┌──────────────────────────┤
               │ invoice_id               │
               ▼                          │
    [5] sales_return                      │
         │  └──(N) sales_return_line      │
         │   ─► inventory_transaction (+qty)
         │   status: pending → approved → refunded
         │ return_id                      │
         ▼                               ▼
    [6] credit_note            [7] sales_payment
         └──(N) credit_note_line    └──(N) sales_payment_allocation
                                          └──(N:1) sales_invoice
                                    TRIGGER: updates invoice.paid_amount
                                             and invoice.status
                                    TRIGGER: updates customer_data.outstanding

PURCHASE MASTERS
  party [supplier] ──(1:0..1)── supplier_data ──(N:1) purchaser, credit_term, area
  supplier_data ──(N:1) supplier_category

COMPLETE PURCHASE DOCUMENT CHAIN
─────────────────────────────────────────────────────────────────────

       party (supplier)
            │
            ▼
    [1] purchase_request (1)──(N) purchase_request_line ──(N:1) item
         │   status: pending → approved
         │ source_request_id
         ▼
    [2] purchase_order (1)──(N) purchase_order_line ──(N:1) item
         │   tracks: qty_received, qty_invoiced per line
         │   status: draft → sent → partially_received → received
         │ po_id
         ▼
    [3] grn (1)──(N) grn_line ──(N:1) item
         │   ordered_qty vs received_qty vs damaged_qty
         │   ─► inventory_transaction (+qty, purchase_receipt)
         │   ─► batch (CREATE or UPDATE on confirmation)
         │   status: draft → confirmed
         │ grn_id
         ▼
    [4] purchase_invoice (1)──(N) purchase_invoice_line ──(N:1) item
         │   3-way match: PO / GRN / Invoice
         │   status: pending → partially_paid → paid
         │
    ┌────┴──────────────────────────────────────────────┐
    │ invoice_id                                          │
    ▼                                                     ▼
[5] debit_note (1)──(N) debit_note_line         [6] supplier_payment (1)──(N) supplier_payment_allocation
     ─► inventory_transaction (−qty)                      └──(N:1) purchase_invoice
        (purchase_return)                          TRIGGER: updates purchase_invoice.paid_amount
                                                            and purchase_invoice.status
                                                   TRIGGER: updates supplier_data.outstanding
```

---

## 11. Migration Order

Tables must be created in this order to satisfy foreign key dependencies.

### Phase 1 — No upstream FKs
1. `company`
2. `role`
3. `area`
4. `brand`
5. `unit_of_measure` *(self-ref added after)*
6. `item_category` *(self-ref added after)*
7. `tax`
8. `customer_category`
9. `supplier_category`
10. `credit_term`

### Phase 2 — FK to Phase 1
11. `branch` *(FK: company)*
12. `employee` *(FK: branch)*
13. `app_user` *(FK: role, branch, employee — circular with branch.manager_user_id → use DEFERRABLE INITIALLY DEFERRED)*
14. `audit_log` *(FK: app_user, branch)*
15. `document_sequence` *(FK: branch)*

### Phase 3 — Masters with FKs to Phase 2
16. `warehouse` *(FK: branch)*
17. `bin` *(FK: warehouse)*
18. `route` *(FK: area)*
19. `coa_account` *(FK: branch — self-ref, create table first then add self-ref FK)*
20. `payment_method` *(FK: coa_account)*
21. `salesperson` *(FK: employee)*
22. `purchaser` *(FK: employee)*
23. `price_list` *(FK: branch)*

### Phase 4 — Inventory + Party
24. `item` *(FK: item_category, brand, unit_of_measure)*
25. `party`
26. `customer_data` *(FK: party, customer_category, area, route, price_list, salesperson, credit_term)*
27. `supplier_data` *(FK: party, supplier_category, purchaser, credit_term, area)*
28. `customer_branch` *(FK: party)*
29. `item_supplier` *(FK: item, party)*
30. `item_uom` *(FK: item, unit_of_measure)*
31. `batch` *(FK: item, warehouse, bin)*
32. `serial` *(FK: item, warehouse, bin, batch)*
33. `inventory_transaction` *(FK: item, warehouse, bin, batch, serial, branch)*
34. `inventory_closing` *(FK: warehouse)*
35. `inventory_closing_line` *(FK: inventory_closing, item, warehouse)*
36. `price_list_item` *(FK: price_list, item)*
37. `promotion`
38. `promotion_item` *(FK: promotion, item)*

### Phase 5 — Purchase Documents (chain order)
39. `purchase_request` *(FK: branch, app_user)*
40. `purchase_request_line` *(FK: purchase_request, item)*
41. `purchase_order` *(FK: branch, party, purchaser, purchase_request)*
42. `purchase_order_line` *(FK: purchase_order, item, unit_of_measure)*
43. `grn` *(FK: branch, party, purchase_order, warehouse)*
44. `grn_line` *(FK: grn, item, unit_of_measure, purchase_order_line, batch, bin)*
45. `purchase_invoice` *(FK: branch, party, purchaser, purchase_order, grn, coa_account)*
46. `purchase_invoice_line` *(FK: purchase_invoice, item, unit_of_measure, purchase_order_line, grn_line)*
47. `supplier_payment` *(FK: branch, party, coa_account)*
48. `supplier_payment_allocation` *(FK: supplier_payment, purchase_invoice)*
49. `debit_note` *(FK: branch, party, purchase_invoice)*
50. `debit_note_line` *(FK: debit_note, item, unit_of_measure, grn_line)*

### Phase 6 — Sales Documents (chain order)
51. `quotation` *(FK: branch, party, salesperson)*
52. `quotation_line` *(FK: quotation, item, unit_of_measure)*
53. `sales_order` *(FK: branch, party, quotation, salesperson)*
54. `sales_order_line` *(FK: sales_order, item, unit_of_measure)*
55. `delivery_note` *(FK: branch, party, warehouse)*
56. `delivery_note_line` *(FK: delivery_note, item, unit_of_measure, batch, serial, bin)*
57. `sales_invoice` *(FK: branch, party, delivery_note, sales_order, quotation, salesperson, coa_account)*
58. `sales_invoice_line` *(FK: sales_invoice, item, unit_of_measure, batch, serial)*
59. `sales_return` *(FK: branch, party, sales_invoice)*
60. `sales_return_line` *(FK: sales_return, item, unit_of_measure, batch, serial, sales_invoice_line)*
61. `credit_note` *(FK: branch, party, sales_invoice, sales_return)*
62. `credit_note_line` *(FK: credit_note, item)*
63. `sales_payment` *(FK: branch, party, coa_account)*
64. `sales_payment_allocation` *(FK: sales_payment, sales_invoice)*

### Phase 7 — Materialized View + Triggers
65. `item_stock_balance` materialized view
66. All 18 triggers (10 sales + 8 purchase)

---

## 12. ZATCA Notes

| Rule | Detail |
|---|---|
| `zatca_invoice_uuid` | Globally unique UUID per invoice. Use `DEFAULT gen_random_uuid()`. Required in the TLV QR code. |
| Invoice chaining | ZATCA requires each invoice to contain the cryptographic hash of the **previous invoice** (same document type, same device). `zatca_hash` stores this invoice's XML hash. The chain is assembled by the ZATCA service layer by reading the `zatca_hash` of the immediately preceding invoice by `invoice_number` within the same `branch_id`. Not a DB FK. |
| B2B invoices | ZATCA **Clearance** mode. Must be cleared via ZATCA API **before** the invoice is sent to the customer. |
| B2C invoices | ZATCA **Reporting** mode. Must be reported to ZATCA within **24 hours** of issuance. |
| Sales Returns | Map to **"Credit Invoice"** in ZATCA terminology. Both `sales_return` and the associated `credit_note` carry ZATCA UUID + XML. |
| VAT number format | 15 digits, starts and ends with `3`. DB check: `vat_number ~ '^3[0-9]{13}3$'` |
| Customer snapshot | Customer address, VAT number, and national address are denormalized onto `sales_invoice` at creation time. ZATCA auditors compare the invoice XML against the snapshot — the customer's current master data is irrelevant to past invoices. |
| Phase-2 device cert | Stored in `company.zatca_cert` (PEM). Private key stored encrypted in `company.zatca_private_key`. The signing service reads these at runtime. |

---

## 13. Table Purpose Reference

A plain-English explanation of every table — what it stores, why it exists, and whether it is needed immediately or can be deferred.

---

### Organization & Auth

#### `app_user`
Stores the login accounts of everyone who uses the system — accountants, sales staff, warehouse managers, admins. Every document in the system records `created_by` pointing here, so you always know who created, approved, or voided something. This table is **required from day one** — you cannot track accountability without it.

#### `role` + `role_permission`
`role` defines named permission sets (e.g. "Sales Manager", "Accountant"). `role_permission` stores what each role can do per module — create, view, update, delete, approve, export. These tables are **global / cross-cutting** and have no direct impact on the sales flow itself. They are needed when you build the authentication and access-control layer. **Defer until you implement login and user management.**

#### `document_sequence`
A counter table that generates sequential, non-duplicate document numbers like `INV-2026-0001`. Without it, two users saving an invoice at the same time could both receive the same number. The table uses a database row lock (`SELECT … FOR UPDATE`) so only one document gets each number. One row per document type per branch per year. **Required from day one** — every document in the system depends on it.

---

### Inventory

#### `serial`
One row per individually serialized unit of an item (e.g. laptop with serial `SN-ABC-123`). Only relevant when `item.tracking_type = 'serial'`. When that item is sold, its `status` changes to `sold`; on return it becomes `returned`. If none of your items require serial tracking, this table can be **deferred**. If you sell electronics, appliances, or any item that needs unit-level traceability, keep it from the start.

#### `inventory_transaction`
An **insert-only ledger** — every stock movement (sale, return, transfer, adjustment) writes one row here with a positive (stock in) or negative (stock out) quantity. Current stock for any item = `SUM(qty)` on this table filtered by item + warehouse. This is the single source of truth for inventory levels and the full stock movement history. **Required from day one.** You cannot have inventory tracking without it.

#### `inventory_closing` + `inventory_closing_line`
Used for periodic physical stock counts (year-end or ad-hoc). `inventory_closing` is the count session header. `inventory_closing_line` is one row per item showing the system quantity (from `inventory_transaction`) versus the physical quantity actually counted by warehouse staff. The variance is used to post adjusting journal entries. The frontend already has this at `/inventory/closing`. **Defer until you are ready to do physical stock counts.** It does not affect the daily sales flow.

---

### Pricing

#### `price_list`
A named pricing configuration — e.g. "Retail SAR", "Wholesale SAR", "Export USD". A customer is linked to one price list via `customer_data.price_list_id`. When a sales invoice is created, the system looks up the customer's price list to auto-fill the item rate. The frontend has a Price Lists page at `/sales/pricing`. **Keep from the start** — the header table is lightweight and needed for customer linking.

#### `price_list_item`
The actual per-item prices on a price list, with optional tier pricing (e.g. unit price drops from SAR 50 to SAR 45 when ordered qty ≥ 10). If all your customers currently pay the same price from `item.selling_price`, you can **defer this table**. Add it when you need differentiated pricing per customer segment.

---

### Warehouses & Bins

#### `warehouse`
A physical or logical storage location for inventory — e.g. "Main Warehouse – Riyadh", "East Branch – Dammam". The Delivery Notes page already requires selecting a warehouse (stock is deducted from it). The Stock Overview shows stock levels per warehouse. **Required from day one** — the sales and inventory flow depends on it.

#### `bin`
A sub-location within a warehouse — e.g. Rack A, Shelf B-3, Floor Zone C. Useful in large warehouses where the picker needs to know the exact location of an item. For smaller operations, warehouses alone are sufficient. **Defer until you need precise bin-level location tracking.** The `bin_id` FK on `batch`, `serial`, and `delivery_note_line` is nullable, so the system works fine without bins.

---

### Promotions

#### `promotion` + `promotion_item`
The frontend already has a Promotions tab on the Price Lists page (`/sales/pricing`) with four types:

| Type | What it does |
|---|---|
| **Item Discount** | Fixed % or flat amount off a specific item |
| **Bill Discount** | Document-level discount when the bill total exceeds a threshold (e.g. 10% off bills over SAR 500) |
| **Buy 1 Get 1** | Buy one unit, get one free |
| **Mix Combo** | Bundle pricing — buy items A + B + C together at a special price |

The tables exist in the schema and are ready. However, the **application logic to automatically apply promotions at invoice creation time has not been built yet** on the frontend. The database tables are simple to create now. Mark the promotion-application logic as a future feature.

---

### Line Tables (`*_line`)

Every sales document has two tables: a **header** (one row per document) and a **line table** (one row per item on that document). They are always required together — you cannot store a quotation without its lines.

| Header | Line table | What the line stores |
|---|---|---|
| `quotation` | `quotation_line` | Each item quoted: qty, rate, VAT snapshot, line total |
| `sales_order` | `sales_order_line` | Each item ordered; also tracks `qty_delivered` and `qty_invoiced` as delivery notes and invoices are created |
| `delivery_note` | `delivery_note_line` | Each item: `ordered_qty` vs `delivered_qty` — supports partial delivery (deliver less than ordered) |
| `sales_invoice` | `sales_invoice_line` | Each item sold: qty, rate, discount, VAT rate snapshot, taxable value, VAT amount, line total — this is the **legally taxable record** |
| `sales_return` | `sales_return_line` | Each item being returned with qty and rate |
| `credit_note` | `credit_note_line` | Each item in the credit note |

Line tables are kept separate (not JSON columns) because you need to query, filter, and report at the item level — e.g. "total qty sold of item X this month", "which invoices contain item Y". That is not possible with a JSON column.

The same rule applies to all purchase documents:

| Header | Line table | What the line stores |
|---|---|---|
| `purchase_request` | `purchase_request_line` | Each item requested: name, qty, required date, perishable flag |
| `purchase_order` | `purchase_order_line` | Each item ordered from supplier; tracks `qty_received` and `qty_invoiced` as GRNs and invoices are created |
| `grn` | `grn_line` | Each item: `ordered_qty` vs `received_qty` vs `damaged_qty`; batch/expiry data captured here |
| `purchase_invoice` | `purchase_invoice_line` | Each item billed by supplier; traces back to PO line and GRN line for 3-way match |
| `debit_note` | `debit_note_line` | Each item returned to supplier with qty and rate |

---

### Purchase Masters

#### `supplier_data`
One-to-one extension of `party` for supplier-specific fields — mirrors what `customer_data` does for customers. Stores credit terms, outstanding AP balance, banking details, and document attachments. `outstanding` is STORED and updated by the supplier payment trigger so you never have to sum across invoices at query time. **Required from day one** if you are implementing the purchase flow.

#### `purchaser`
Internal buyer / procurement staff. Assigned to suppliers and purchase orders for accountability and commission tracking. Mirrors `salesperson` on the sales side. **Required from day one** of the purchase flow if you track who places each order.

#### `supplier_category`
Groups suppliers by category (e.g. Raw Materials, Packaging, Services). Lightweight table — no downstream FK dependencies. **Keep from the start** — it is a simple master that makes supplier filtering and reporting meaningful.

---

### Purchase Documents

#### `purchase_request`
An internal requisition raised by a department (IT, Production, HR, etc.) before a PO is sent to any supplier. Includes perishable-item tracking with risk levels (Safe / Warning / Critical / Expired). Provides an approval trail — only approved requests should become POs. **Required if you want an approval workflow before spending.** Can be skipped if buyers create POs directly.

#### `grn` + `grn_line`
Goods Received Note — the physical proof that goods arrived. `grn_line` records `ordered_qty` vs `received_qty` vs `damaged_qty` and captures batch/expiry data for perishable items on receipt. On GRN confirmation a trigger fires to: (1) insert a `purchase_receipt` row in `inventory_transaction`, (2) update `purchase_order_line.qty_received`, and (3) create or update the `batch` record. **Required from day one** — without it you have no stock-in events and no 3-way match.

#### `purchase_invoice` (Purchase Billing)
The supplier's bill. Links back to the PO and GRN for 3-way matching. Stores `three_way_match_status` (matched / mismatched) so the AP team can see discrepancies before approving payment. `outstanding_balance` is STORED and updated by the supplier payment trigger. **Required from day one** of the purchase flow.

#### `supplier_payment` + `supplier_payment_allocation`
Records an outgoing payment to a supplier and allocates it across one or more purchase invoices — exactly the AP equivalent of `sales_payment` / `sales_payment_allocation`. Trigger updates `purchase_invoice.paid_amount` and `supplier_data.outstanding`. **Required from day one** to track what has been paid and what is still owed.

#### `debit_note`
Issued to a supplier for returned goods, short-deliveries, or price corrections. Optionally links to the original purchase invoice. On approval, a trigger fires to write a `purchase_return` row into `inventory_transaction` (negative qty — stock leaves the warehouse). **Build when** returns or adjustments to supplier invoices are part of your workflow.

---

## 14. What to Build Now vs Later

### Sales & Shared

| Table / Feature | Build now? | Reason |
|---|---|---|
| `role`, `role_permission` | **Later** | Global auth concern; no impact on daily flow |
| `app_user` | **Now** (simplified — just name + email + branch) | Every document needs `created_by` |
| `document_sequence` | **Now** | Prevents duplicate document numbers across both flows |
| `warehouse` | **Now** | Required by Delivery Notes, GRNs, and stock tracking |
| `bin` | **Later** | Sub-location detail; all bin FKs are nullable |
| `serial` | **Now if you sell serial-tracked items, else Later** | Depends on your item types |
| `inventory_transaction` | **Now** | Core stock ledger — used by both sales and purchase |
| `inventory_closing` + line | **Later** | Year-end feature; not part of daily flow |
| `price_list` (header only) | **Now** | Customer linking needs it |
| `price_list_item` | **Later** | Only needed when you have differentiated pricing |
| `promotion` + `promotion_item` | **Now** (schema only) | UI exists; application logic to be built later |
| All `*_line` tables | **Always Now** | Inseparable from their header tables |

### Purchase

| Table / Feature | Build now? | Reason |
|---|---|---|
| `supplier_category` | **Now** | Lightweight master; no FK dependencies |
| `purchaser` | **Now** | Assigned to POs; needed for accountability |
| `supplier_data` | **Now** | Required for AP balance tracking and credit terms |
| `purchase_request` + line | **Optional** | Build if you need an internal approval step before POs; skip if buyers create POs directly |
| `purchase_order` + line | **Now** | Core purchase document — tracks what was ordered and at what price |
| `grn` + `grn_line` | **Now** | Required for stock-in events and 3-way match; cannot skip |
| `purchase_invoice` + line | **Now** | AP record — tracks what is owed to each supplier |
| `supplier_payment` + allocation | **Now** | Required to settle invoices and update AP outstanding |
| `debit_note` + line | **When needed** | Build when supplier returns or invoice adjustments are part of your workflow |
