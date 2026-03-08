# Geskap — Firebase Firestore Documentation

> **Last updated:** 2026-03-08  
> **Project:** Geskap (CamaireTech)  
> **Database:** Cloud Firestore  
> **Auth:** Firebase Authentication (email/password + custom tokens)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Common Field Conventions](#2-common-field-conventions)
3. [Collection Reference](#3-collection-reference)
   - [users](#users)
   - [companies](#companies) ← subcollections inside
   - [products](#products)
   - [categories](#categories)
   - [sales](#sales)
   - [expenses](#expenses)
   - [finances](#finances)
   - [financeEntryTypes](#financeentrytypes)
   - [expenseTypes](#expensetypes)
   - [customers](#customers)
   - [customerSources](#customersources)
   - [suppliers](#suppliers)
   - [stockBatches](#stockbatches)
   - [stockChanges](#stockchanges)
   - [stockTransfers](#stocktransfers)
   - [stockReplenishmentRequests](#stockreplenishmentrequests)
   - [shops](#shops)
   - [warehouses](#warehouses)
   - [orders](#orders) ← subcollections inside
   - [productions](#productions)
   - [productionFlows](#productionflows)
   - [productionFlowSteps](#productionflowsteps)
   - [productionCategories](#productioncategories)
   - [charges](#charges)
   - [matieres](#matieres)
   - [purchases](#purchases)
   - [matierePurchases](#matierepurchases)
   - [objectives](#objectives)
   - [notifications](#notifications)
   - [actionRequests](#actionrequests)
   - [hrActors](#hractors)
   - [auditLogs](#auditlogs)
   - [dashboardStats](#dashboardstats)
   - [siteAnalytics](#siteanalytics)
   - [cinetpay\_configs](#cinetpay_configs)
   - [campay\_configs](#campay_configs)
   - [checkout\_settings](#checkout_settings)
   - [sellerSettings](#sellersettings)
4. [Security Rules Summary](#4-security-rules-summary)
5. [Key Indexes](#5-key-indexes)
6. [Data Isolation Pattern](#6-data-isolation-pattern)

---

## 1. Architecture Overview

```
Firestore
├── users/                         Global user profiles (one per Firebase Auth UID)
├── companies/                     Company accounts (each = one business)
│   └── {companyId}/
│       ├── employees/             Embedded employee records (legacy)
│       ├── employeeRefs/          Auth-linked employee references
│       ├── permissionTemplates/   Role/permission presets
│       └── profitPeriodPreference/ Preferred reporting period
├── products/                      Product catalogue (scoped by companyId)
├── categories/                    Product/matiere categories
├── sales/                         Sales transactions
├── expenses/                      Business expenses
├── finances/                      Finance journal (cash flow ledger)
├── financeEntryTypes/             Custom finance entry type labels
├── expenseTypes/                  Custom expense category labels
├── customers/                     Customer profiles
├── customerSources/               Lead source definitions
├── suppliers/                     Supplier/vendor records
├── stockBatches/                  Inventory batches (FIFO/LIFO/CMUP)
├── stockChanges/                  Stock movement audit log
├── stockTransfers/                Inter-location stock transfers
├── stockReplenishmentRequests/    Shop → warehouse restock requests
├── shops/                         Retail shop locations
├── warehouses/                    Warehouse storage locations
├── orders/                        Public/guest checkout orders
│   └── {orderId}/
│       ├── events/                Order status history
│       ├── payments/              Payment records
│       └── notes/                 Internal notes
├── productions/                   Manufacturing/production batches
├── productionFlows/               Reusable production workflow templates
├── productionFlowSteps/           Individual flow step definitions
├── productionCategories/          Production classification categories
├── charges/                       Fixed/custom overhead charges
├── matieres/                      Raw materials (magasin)
├── purchases/                     Finished-goods purchase orders
├── matierePurchases/              Raw material purchase orders
├── objectives/                    Business goals & KPI targets
├── notifications/                 In-app user notifications
├── actionRequests/                Employee permission escalation requests
├── hrActors/                      HR personnel (payroll/scheduling)
├── auditLogs/                     Immutable change history
├── dashboardStats/                Pre-computed dashboard metrics
├── siteAnalytics/                 Public catalogue traffic analytics
├── cinetpay_configs/              CinetPay payment gateway settings
├── campay_configs/                Campay mobile money settings
├── checkout_settings/             Per-user checkout configuration
└── sellerSettings/                Per-user catalogue seller settings
```

---

## 2. Common Field Conventions

All business documents inherit **`BaseModel`**:

| Field | Type | Description |
|---|---|---|
| `id` | `string` | Firestore document ID |
| `companyId` | `string` | Company that owns this record (primary isolation key) |
| `userId` | `string` | Firebase UID of creator (legacy audit field) |
| `createdAt` | `Timestamp` | Firestore server timestamp of creation |
| `updatedAt` | `Timestamp` | Firestore server timestamp of last update |
| `createdBy` | `EmployeeRef?` | Employee who created it (optional in some older docs) |

**Timestamps** are stored as Firestore `Timestamp` objects (`{ seconds, nanoseconds }`).

**Soft deletes** are implemented via `isDeleted: boolean` or `isAvailable: boolean` fields — no documents are hard-deleted in production flows.

---

## 3. Collection Reference

---

### `users`

**Path:** `users/{userId}` (userId = Firebase Auth UID)

One document per registered user. Users can belong to multiple companies.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Firebase Auth UID |
| `username` | string | ✅ | Unique display name |
| `email` | string | ✅ | Login email |
| `photoURL` | string? | | Profile photo URL |
| `phone` | string? | | Phone number |
| `status` | `'active'│'suspended'│'invited'` | ✅ | Account status |
| `companies` | `UserCompanyRef[]` | ✅ | List of companies this user belongs to |
| `lastLogin` | Timestamp? | | Last login time |
| `createdAt` | Timestamp | ✅ | |
| `updatedAt` | Timestamp | ✅ | |

**`UserCompanyRef` (embedded in `companies[]`):**

| Field | Type | Description |
|---|---|---|
| `companyId` | string | Reference to `companies/{companyId}` |
| `name` | string | Company name (denormalised) |
| `description` | string? | |
| `logo` | string? | |
| `role` | `'owner'│'admin'│'manager'│'staff'` | User's role in that company |
| `joinedAt` | Timestamp | |
| `permissionTemplateId` | string? | Active permission template |
| `assignedShops` | string[]? | Shop IDs the user can access |
| `assignedWarehouses` | string[]? | Warehouse IDs the user can access |

**Security:** Authenticated users can read any profile. Only the owner can update their own profile.

---

### `companies`

**Path:** `companies/{companyId}`

Central record for each registered business/company on Geskap.

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Auto-generated: `company_<timestamp>_<random>` |
| `name` | string | ✅ | Business display name |
| `description` | string? | | Short description |
| `location` | string? | | City / physical address |
| `logo` | string? | | Base64 string or URL |
| `website` | string? | | Company website URL |
| `email` | string | ✅ | 🔒 Owner email — **private** |
| `phone` | string | ✅ | 🔒 Primary phone — **private** |
| `phones` | `CompanyPhone[]?` | | 🔒 Additional phone numbers — **private** |
| `report_mail` | string? | | 🔒 Reporting email — **private** |
| `report_time` | string/number? | | 🔒 Daily report send time — **private** |
| `emailReportsEnabled` | boolean? | | 🔒 Auto-report toggle — **private** |
| `niu` | string? | | Tax identification number |
| `timezone` | string? | | IANA timezone e.g. `Africa/Douala` |
| `lowStockThreshold` | number? | | Global threshold for low stock alerts |
| `defaultPeriod` | string? | | Default date range picker period |
| `catalogueColors` | object? | | `{ primary, secondary, tertiary }` — catalogue brand colors |
| `dashboardColors` | object? | | `{ primary, secondary, tertiary, headerText }` |
| `seoSettings` | object? | | `{ metaTitle, metaDescription, metaKeywords[], ogImage, twitterCard }` |
| `employees` | `Record<string, CompanyEmployee>?` | | Quick-read employee mirror |
| `employeeCount` | number? | | Total employee count |
| `companyId` | string | ✅ | Owner's `userId` (legacy) |
| `createdAt` | Timestamp | ✅ | |
| `updatedAt` | Timestamp | ✅ | |

> 🔒 **Privacy:** Fields marked **private** are never exposed by the MCP server.

**Subcollections:**

#### `companies/{companyId}/employeeRefs/{firebaseUid}`

Auth-linked employee records (new architecture).

| Field | Type | Description |
|---|---|---|
| `id` | string | Firebase UID |
| `username` | string | Copied from `users` |
| `email` | string | Copied from `users` |
| `role` | `'admin'│'manager'│'staff'` | Role within the company |
| `addedAt` | Timestamp | |
| `posPin` | string? | Hashed 4-digit PIN for POS sessions |

#### `companies/{companyId}/permissionTemplates/{templateId}`

Role-based permission presets.

#### `companies/{companyId}/profitPeriodPreference/{prefId}`

Stores the preferred profit calculation period per company.

---

### `products`

**Path:** `products/{productId}`

Product catalogue entries. Public read is allowed (catalogue).

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | ✅ | |
| `reference` | string | ✅ | Internal product code |
| `sellingPrice` | number | ✅ | Public selling price |
| `costPrice` | number | ✅ | Cost price (private — used for profit) |
| `cataloguePrice` | number? | | Optional catalogue-visible price |
| `category` | string? | | Category name |
| `images` | string[]? | | Firebase Storage URLs |
| `imagePaths` | string[]? | | Storage paths (for deletion) |
| `description` | string? | | Product description for catalogue |
| `barCode` | string? | | EAN-13 barcode |
| `stock` | number? | | @deprecated — use `stockBatches` |
| `isAvailable` | boolean | ✅ | Display toggle |
| `isDeleted` | boolean? | | Soft delete |
| `isVisible` | boolean? | | Catalogue visibility (default: true) |
| `inventoryMethod` | `'FIFO'│'LIFO'│'CMUP'?` | | Valuation method |
| `enableBatchTracking` | boolean? | | Enable detailed batch tracking |
| `tags` | `ProductTag[]?` | | Variation tags (model, color, size…) |
| `companyId` | string | ✅ | |

---

### `categories`

**Path:** `categories/{categoryId}`

Shared categories for products and raw materials.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `description` | string? | |
| `image` | string? | Storage URL or base64 |
| `imagePath` | string? | Storage path |
| `type` | `'product'│'matiere'` | Never mixed |
| `productCount` | number? | Denormalised count |
| `matiereCount` | number? | Denormalised count |
| `isActive` | boolean? | Soft disable |
| `companyId` | string | |

---

### `sales`

**Path:** `sales/{saleId}`

Individual sale transactions.

| Field | Type | Description |
|---|---|---|
| `products` | `SaleProduct[]` | Line items with costPrice, margin, batch info |
| `totalAmount` | number | |
| `status` | `'commande'│'under_delivery'│'paid'│'draft'│'credit'` | |
| `paymentStatus` | `'pending'│'paid'│'cancelled'` | |
| `customerInfo` | `{ name, phone, quarter? }` | |
| `customerSourceId` | string? | Reference to `customerSources` |
| `paymentMethod` | `'cash'│'mobile_money'│'card'?` | |
| `discountType` | `'amount'│'percentage'?` | |
| `discountValue` | number? | |
| `deliveryFee` | number? | |
| `tax` | number? | |
| `tvaRate` | number? | TVA percentage |
| `totalCost` | number? | Total cost price |
| `totalProfit` | number? | |
| `sourceType` | `'shop'│'warehouse'?` | Where sale originated |
| `shopId` | string? | |
| `warehouseId` | string? | |
| `caisseSessionId` | string? | Linked cashier session |
| `cashierId` | string? | |
| `refunds` | array? | Refund history |
| `isAvailable` | boolean? | Soft delete |
| `companyId` | string | |

---

### `expenses`

**Path:** `expenses/{expenseId}`

Business expense records.

| Field | Type | Description |
|---|---|---|
| `description` | string | |
| `amount` | number | |
| `category` | string | Expense type name |
| `date` | Timestamp? | Business date |
| `expenseNature` | `'exploitation'│'investissement'?` | Default: exploitation |
| `images` | string[]? | Receipts |
| `imagePaths` | string[]? | Storage paths |
| `isAvailable` | boolean? | Soft delete |
| `companyId` | string | |

---

### `finances`

**Path:** `finances/{financeId}`

Cash flow journal. Every income and expense that affects the ledger has an entry here.

| Field | Type | Description |
|---|---|---|
| `sourceType` | `'sale'│'expense'│'manual'│'supplier'│'order'│'matiere'` | |
| `sourceId` | string? | saleId, expenseId, orderId, etc. |
| `type` | string | Entry label e.g. `"sale"`, `"expense"`, `"supplier_debt"` |
| `amount` | number | |
| `description` | string? | |
| `date` | Timestamp | Business date |
| `isDeleted` | boolean | Soft delete |
| `isPending` | boolean? | Pending conversion to sale |
| `supplierId` | string? | |
| `batchId` | string? | Linked stock batch |
| `refundedDebtId` | string? | For supplier refunds |
| `companyId` | string | |

---

### `financeEntryTypes`

**Path:** `financeEntryTypes/{typeId}`

Custom labels for manual finance entries.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `isDefault` | boolean | System default vs user-created |
| `userId` | string? | undefined for global defaults |

---

### `expenseTypes`

**Path:** `expenseTypes/{typeId}`

Custom expense category labels.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `isDefault` | boolean | |
| `userId` | string? | |
| `companyId` | string? | |
| `nature` | `'exploitation'│'investissement'?` | |

---

### `customers`

**Path:** `customers/{customerId}`

Customer contact profiles.

| Field | Type | Description |
|---|---|---|
| `name` | string? | |
| `firstName` | string? | |
| `lastName` | string? | |
| `phone` | string | Primary identifier |
| `quarter` | string? | Neighbourhood |
| `address` | string? | |
| `town` | string? | |
| `birthdate` | string? | ISO date |
| `howKnown` | string? | |
| `customerSourceId` | string? | |
| `primaryShopId` | string? | |
| `companyId` | string | |

---

### `customerSources`

**Path:** `customerSources/{sourceId}`

Definitions of how customers were acquired (e.g. TikTok, Facebook, Referral).

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `description` | string? | |
| `color` | string? | Hex color for charts |
| `isActive` | boolean | |
| `companyId` | string | |

---

### `suppliers`

**Path:** `suppliers/{supplierId}`

Vendor/supplier directory.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `contact` | string | Phone or contact info |
| `location` | string? | |
| `email` | string? | |
| `notes` | string? | |
| `isDeleted` | boolean? | |
| `companyId` | string | |

---

### `stockBatches`

**Path:** `stockBatches/{batchId}`

Individual inventory batches (FIFO / LIFO / CMUP tracking).

| Field | Type | Description |
|---|---|---|
| `type` | `'product'│'matiere'` | |
| `productId` | string? | If product |
| `matiereId` | string? | If raw material |
| `quantity` | number | Total quantity in batch |
| `remainingQuantity` | number | Current available quantity |
| `damagedQuantity` | number? | |
| `costPrice` | number | Cost per unit for this batch |
| `status` | `'active'│'depleted'│'corrected'│'deleted'` | |
| `supplierId` | string? | |
| `isOwnPurchase` | boolean? | |
| `isCredit` | boolean? | Bought on credit |
| `locationType` | `'warehouse'│'shop'│'production'│'global'?` | |
| `warehouseId` | string? | |
| `shopId` | string? | |
| `isDeleted` | boolean? | |
| `companyId` | string | |

---

### `stockChanges`

**Path:** `stockChanges/{changeId}`

Immutable audit log of every stock movement.

| Field | Type | Description |
|---|---|---|
| `type` | `'product'│'matiere'` | |
| `productId` / `matiereId` | string? | |
| `change` | number | `+` restock, `-` sale/consumption |
| `reason` | enum | `'sale'│'restock'│'adjustment'│'creation'│'damage'│...` |
| `saleId` | string? | |
| `supplierId` | string? | |
| `locationType` | `'warehouse'│'shop'│'production'│'global'?` | |
| `adjustmentType` | enum? | For corrections |
| `adjustmentReason` | enum? | |
| `transferId` | string? | |
| `notes` | string? | |
| `companyId` | string | |

> **Write rules:** create only. Updates and deletes are forbidden.

---

### `stockTransfers`

**Path:** `stockTransfers/{transferId}`

Tracks product movements between shops and warehouses.

| Field | Type | Description |
|---|---|---|
| `transferType` | `'warehouse_to_shop'│'shop_to_shop'│'warehouse_to_warehouse'│'shop_to_warehouse'` | |
| `fromWarehouseId` | string? | |
| `fromShopId` | string? | |
| `toWarehouseId` | string? | |
| `toShopId` | string? | |
| `productId` | string | |
| `quantity` | number | |
| `batchIds` | string[] | Batch IDs transferred |
| `status` | `'pending'│'completed'│'cancelled'` | |
| `date` | Timestamp? | |
| `notes` | string? | |
| `companyId` | string | |

---

### `stockReplenishmentRequests`

**Path:** `stockReplenishmentRequests/{requestId}`

Requests from a shop to get stock replenished from a warehouse.

| Field | Type | Description |
|---|---|---|
| `shopId` | string | Requesting shop |
| `productId` | string | |
| `quantity` | number | |
| `requestedBy` | string | User UID |
| `status` | `'pending'│'approved'│'rejected'│'fulfilled'` | |
| `transferId` | string? | If fulfilled |
| `fulfilledAt` | Timestamp? | |
| `rejectedReason` | string? | |
| `notes` | string? | |
| `companyId` | string | |

---

### `shops`

**Path:** `shops/{shopId}`

Retail sales locations.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `location` | string? | |
| `address` | string? | |
| `phone` | string? | |
| `email` | string? | |
| `isDefault` | boolean | Auto-created default shop |
| `isActive` | boolean | |
| `assignedUsers` | string[]? | User IDs with full access |
| `readOnlyUsers` | string[]? | User IDs with read access |
| `managerId` | string? | |
| `catalogueSettings` | object? | `{ isPublic, customDomain, seoSettings }` |
| `companyId` | string | |

---

### `warehouses`

**Path:** `warehouses/{warehouseId}`

Central warehouse storage.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `location` | string? | |
| `address` | string? | |
| `isDefault` | boolean | Auto-created default warehouse |
| `isActive` | boolean | |
| `assignedUsers` | string[]? | |
| `readOnlyUsers` | string[]? | |
| `companyId` | string | |

---

### `orders`

**Path:** `orders/{orderId}`

Public/guest checkout orders from the catalogue site. Can be created unauthenticated.

| Field | Type | Description |
|---|---|---|
| `products` | `SaleProduct[]` | Line items |
| `totalAmount` | number | |
| `status` | `OrderStatus` | Same enum as sales |
| `paymentStatus` | `'pending'│'paid'│'cancelled'` | |
| `customerInfo` | `{ name, phone, quarter? }` | |
| `companyId` | string | |

**Subcollections:**

| Subcollection | Description |
|---|---|
| `orders/{orderId}/events/{eventId}` | Status change history (immutable) |
| `orders/{orderId}/payments/{paymentId}` | Payment records |
| `orders/{orderId}/notes/{noteId}` | Internal notes |

---

### `productions`

**Path:** `productions/{productionId}`

Manufacturing production batches.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `reference` | string | |
| `description` | string? | |
| `images` | string[]? | |
| `categoryId` | string? | |
| `flowId` | string? | Reference to `productionFlows` |
| `status` | `'draft'│'in_progress'│'ready'│'published'│'cancelled'│'closed'` | |
| `stateHistory` | `ProductionStateChange[]` | Full audit trail |
| `articles` | `ProductionArticle[]` | Items being produced |
| `totalArticlesQuantity` | number | |
| `charges` | `ProductionChargeRef[]` | Cost overhead snapshots |
| `calculatedCostPrice` | number | Auto-calculated |
| `validatedCostPrice` | number? | User-confirmed cost |
| `isCostValidated` | boolean | |
| `isPublished` | boolean | All articles published |
| `isClosed` | boolean | |
| `companyId` | string | |

---

### `productionFlows`

**Path:** `productionFlows/{flowId}`

Reusable workflow templates for production.

| Field | Type | Description |
|---|---|---|
| `name` | string | e.g. "Standard Production" |
| `description` | string? | |
| `isDefault` | boolean | |
| `isActive` | boolean | |
| `stepIds` | string[] | Ordered step IDs from `productionFlowSteps` |
| `estimatedDuration` | number? | Total days |
| `stepCount` | number? | |
| `companyId` | string | |

---

### `productionFlowSteps`

**Path:** `productionFlowSteps/{stepId}`

Reusable individual steps referenced by production flows.

| Field | Type | Description |
|---|---|---|
| `name` | string | e.g. "Design", "Cutting", "Sewing" |
| `description` | string? | |
| `image` | string? | |
| `estimatedDuration` | number? | Hours |
| `isActive` | boolean | |
| `usageCount` | number? | |
| `companyId` | string | |

---

### `productionCategories`

**Path:** `productionCategories/{categoryId}`

Classification categories for productions.

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `description` | string? | |
| `image` | string? | |
| `productionCount` | number? | |
| `isActive` | boolean | |
| `companyId` | string | |

---

### `charges`

**Path:** `charges/{chargeId}`

Fixed or custom overhead charges (labour, electricity, transport, etc.).

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `description` | string? | |
| `type` | `'fixed'│'custom'` | Fixed = reusable; custom = one-off |
| `amount` | number | |
| `category` | string? | `'main_oeuvre'│'overhead'│'transport'…` |
| `date` | Timestamp | |
| `isActive` | boolean? | |
| `financeEntryId` | string? | |
| `companyId` | string | |

---

### `matieres`

**Path:** `matieres/{matiereId}`

Raw materials inventory (magasin).

| Field | Type | Description |
|---|---|---|
| `name` | string | |
| `description` | string? | |
| `images` | string[]? | |
| `refCategorie` | string? | Category name |
| `refStock` | string | Reference to stock document ID |
| `unit` | string? | Unit of measurement |
| `costPrice` | number | Last purchase price |
| `barCode` | string? | EAN-13 |
| `isDeleted` | boolean? | |
| `companyId` | string | |

---

### `purchases`

**Path:** `purchases/{purchaseId}`

Finished-goods purchase orders from suppliers.

| Field | Type | Description |
|---|---|---|
| `billNumber` | string | Auto-sequence e.g. `"ACH-0001"` |
| `supplierId` | string | |
| `supplierName` | string | Denormalised |
| `reference` | string | Supplier's receipt/bill number |
| `date` | Timestamp | |
| `items` | `PurchaseItem[]` | Line items with batch refs |
| `totalAmount` | number | |
| `status` | `'received'│'cancelled'` | |
| `paymentStatus` | `'paid'│'credit'` | |
| `locationType` | `'shop'│'warehouse'` | Where stock was received |
| `shopId` / `warehouseId` | string? | |
| `isAvailable` | boolean | |
| `notes` | string? | |
| `companyId` | string | |

---

### `matierePurchases`

**Path:** `matierePurchases/{purchaseId}`

Raw material purchase orders. Same structure as `purchases` but for raw materials.

Fields identical to `purchases` with `items` containing `MatierePurchaseItem` (including `unit`).

---

### `objectives`

**Path:** `objectives/{objectiveId}`

Business KPI goals.

| Field | Type | Description |
|---|---|---|
| `title` | string | |
| `description` | string? | |
| `metric` | string | Stats key being tracked |
| `targetAmount` | number | |
| `periodType` | `'predefined'│'custom'` | |
| `predefined` | string? | `'this_month'│'this_year'…` |
| `startAt` / `endAt` | Timestamp? | For custom periods |
| `isAvailable` | boolean? | |
| `companyId` | string | |

---

### `notifications`

**Path:** `notifications/{notificationId}`

In-app notifications for users.

| Field | Type | Description |
|---|---|---|
| `userId` | string | Target user |
| `type` | enum | `'replenishment_request_created'│'transfer_created'│'stock_low'…` |
| `title` | string | |
| `message` | string | |
| `data` | object? | `{ requestId, transferId, shopId, productId… }` |
| `read` | boolean | |
| `readAt` | Timestamp? | |
| `companyId` | string | |

---

### `actionRequests`

**Path:** `actionRequests/{requestId}`

Permission escalation requests — when a staff member needs a one-time or permanent grant for a restricted action.

| Field | Type | Description |
|---|---|---|
| `requesterId` | string | Employee UID |
| `requesterName` | string | |
| `requestedAction` | string | e.g. `'delete_sale'` |
| `resource` | string | e.g. `'sales'` |
| `resourceId` | string? | Specific record |
| `reason` | string? | |
| `status` | `'pending'│'approved'│'rejected'` | |
| `grantType` | `'one_time'│'permanent'?` | |
| `expiresAt` | Timestamp? | |
| `reviewedBy` | string? | |
| `reviewNote` | string? | |
| `companyId` | string | |

---

### `hrActors`

**Path:** `hrActors/{actorId}`

HR personnel records (payroll, scheduling) — separate from app user accounts.

| Field | Type | Description |
|---|---|---|
| `firstName` | string | |
| `lastName` | string | |
| `phone` | string | Required |
| `email` | string? | |
| `actorType` | `'gardien'│'caissier'│'magasinier'│'livreur'│'manager'│…` | |
| `department` | string? | |
| `salary` | number? | |
| `salaryFrequency` | `'hourly'│'daily'│'weekly'│'biweekly'│'monthly'?` | |
| `contractType` | `'CDI'│'CDD'│'stage'│'freelance'│'interim'?` | |
| `hireDate` | Timestamp | |
| `endDate` | Timestamp? | |
| `status` | `'active'│'inactive'│'archived'` | |
| `linkedUserId` | string? | Optional link to Firebase user |
| `emergencyContact` | object? | `{ name, phone, relationship }` |
| `companyId` | string | |

---

### `auditLogs`

**Path:** `auditLogs/{logId}`

Immutable change log. Insert-only (no updates or deletes allowed by security rules).

| Field | Type | Description |
|---|---|---|
| `action` | string | What was done |
| `targetId` | string | ID of the modified document |
| `before` | object? | Previous state |
| `after` | object? | New state |
| `companyId` | string | |
| `userId` | string | Who did it |
| `createdAt` | Timestamp | |

---

### `dashboardStats`

**Path:** `dashboardStats/{statsId}`

Pre-computed analytics snapshots for dashboard performance.

| Field | Type | Description |
|---|---|---|
| `totalSales` | number | |
| `totalExpenses` | number | |
| `totalProfit` | number | |
| `activeOrders` | number | |
| `completedOrders` | number | |
| `cancelledOrders` | number | |
| `companyId` | string | |

---

### `siteAnalytics`

**Path:** `siteAnalytics/{analyticsId}`

Daily public catalogue traffic data.

| Field | Type | Description |
|---|---|---|
| `date` | Timestamp | |
| `views` | number | |
| `uniqueVisitors` | number | |
| `popularProducts` | `Array<{ productId, productName, views }>` | |
| `referrers` | `Array<{ source, count }>` | |
| `deviceTypes` | `Array<{ type: 'desktop'│'mobile'│'tablet', count }>` | |
| `companyId` | string | |

> **Security:** Public read and write (guest catalogue tracking).

---

### `cinetpay_configs`

**Path:** `cinetpay_configs/{configId}` (configId = userId)

CinetPay payment gateway credentials per seller.

> **Security:** Public read (used during guest checkout). Authenticated write only.

---

### `campay_configs`

**Path:** `campay_configs/{configId}` (configId = userId)

Campay mobile money credentials per seller.

> **Security:** Public read (used during guest checkout). Authenticated write only.

---

### `checkout_settings`

**Path:** `checkout_settings/{userId}`

Per-user checkout configuration (shipping, payment methods, etc.).

> **Security:** Public read. Authenticated write only.

---

### `sellerSettings`

**Path:** `sellerSettings/{userId}`

Per-user catalogue seller settings (business details for public catalogue).

> **Security:** Public read. Authenticated write only.

---

## 4. Security Rules Summary

| Collection | Read | Write |
|---|---|---|
| `users` | Authenticated | Own document only |
| `companies` | **Public** | Authenticated |
| `companies/*/employeeRefs` | Authenticated | Authenticated |
| `companies/*/permissionTemplates` | Authenticated | Authenticated |
| `products` | **Public** | Authenticated |
| `sales` | Authenticated | Authenticated |
| `expenses` | Authenticated | Authenticated |
| `finances` | Authenticated | Authenticated |
| `customers` | Authenticated | Authenticated |
| `suppliers` | Authenticated | Authenticated |
| `stockBatches` | Authenticated | Authenticated |
| `stockChanges` | Authenticated | Create only (no update/delete) |
| `orders` | Authenticated | Create public / Update authenticated |
| `orders/*/events` | Authenticated | Create only (no update/delete) |
| `auditLogs` | Authenticated | Create only (no update/delete) |
| `categories` | **Public** | Authenticated |
| `expenseTypes` | **Public** | Authenticated |
| `financeEntryTypes` | **Public** | Authenticated |
| `siteAnalytics` | **Public** | **Public** (guest tracking) |
| `cinetpay_configs` | **Public** | Authenticated |
| `campay_configs` | **Public** | Authenticated |
| `checkout_settings` | **Public** | Authenticated |
| `sellerSettings` | **Public** | Authenticated |
| All others | Authenticated | Authenticated |

---

## 5. Key Indexes

Key composite indexes (from `firestore.indexes.json`):

| Collection | Fields indexed | Use case |
|---|---|---|
| `products` | `companyId` + `isAvailable` + `name` | Product list filtered by company |
| `sales` | `companyId` + `createdAt DESC` | Recent sales by company |
| `sales` | `companyId` + `status` + `createdAt DESC` | Filtered sale list |
| `expenses` | `companyId` + `createdAt DESC` | Expense list |
| `stockBatches` | `companyId` + `productId` + `status` | FIFO/LIFO batch lookup |
| `stockChanges` | `companyId` + `productId` + `createdAt DESC` | Stock history per product |
| `customers` | `companyId` + `phone` | Customer lookup |
| `orders` | `companyId` + `status` + `createdAt DESC` | Order management |
| `productions` | `companyId` + `status` + `createdAt DESC` | Active productions |
| `notifications` | `userId` + `companyId` + `read` | Unread notifications |

---

## 6. Data Isolation Pattern

All business data is isolated per company using a **`companyId` field** on every document. There are no subcollections for business data (except `companies/*/employeeRefs` and related).

```
// Every query for business data MUST filter by companyId:
query(collection(db, 'sales'),
  where('companyId', '==', currentCompanyId),
  orderBy('createdAt', 'desc')
)
```

**Multi-company users:** A single Firebase user (identified by UID) can belong to multiple companies via the `users/{uid}.companies[]` array. The active company context is managed client-side and stored in local cache (`CompanyManager`).

---

*This document reflects the Geskap production Firestore schema as of 2026-03-08.*
