# Data Dictionary — Sankofa Freight Networks Data Warehouse

| Table | Approx. row count |
|---|---|
| `warehouses` | 15 |
| `carriers` | 8 |
| `products` | 500 |
| `customers` | 5,000 |
| `orders` | 750,000 |
| `order_items` | ~1,875,000 |
| `shipments` | 750,000 |

---

## `warehouses`
Not partitioned — small, stable reference table.

| Column | Type | Nullable | Key | Description |
|---|---|---|---|---|
| `warehouse_id` | INTEGER | No | PK | Unique identifier for the warehouse |
| `warehouse_name` | VARCHAR(150) | No | | Warehouse display name (e.g. "Kumasi Distribution Center") |
| `warehouse_city` | VARCHAR(100) | No | | City the warehouse is located in |
| `warehouse_region` | VARCHAR(50) | No | | One of Ghana's 16 administrative regions |

## `carriers`
Not partitioned — small, stable reference table.

| Column | Type | Nullable | Key | Description |
|---|---|---|---|---|
| `carrier_id` | INTEGER | No | PK | Unique identifier for the shipping carrier |
| `carrier_name` | VARCHAR(150) | No | Unique | Carrier company name |

## `products`
Not partitioned — small, stable reference table.

| Column | Type | Nullable | Key | Description |
|---|---|---|---|---|
| `product_id` | INTEGER | No | PK | Unique identifier for the product |
| `product_name` | VARCHAR(255) | No | | Product display name |
| `category` | VARCHAR(100) | No | | Product category (e.g. Electronics, Apparel) |
| `unit_price` | NUMERIC(10,2) | No | | Current catalog price; must be ≥ 0 |

## `customers`
Not partitioned — grows slowly relative to orders.

| Column | Type | Nullable | Key | Description |
|---|---|---|---|---|
| `customer_id` | INTEGER | No | PK | Unique identifier for the customer |
| `customer_name` | VARCHAR(150) | No | | Customer full name |
| `email` | VARCHAR(255) | **Yes** | | Customer email; some raw records legitimately have none |
| `customer_city` | VARCHAR(100) | Yes | | City the customer is based in |
| `customer_region` | VARCHAR(50) | Yes | | One of Ghana's 16 administrative regions |
| `customer_country` | VARCHAR(50) | No | | Defaults to `'Ghana'` |

## `orders` — PARTITIONED by RANGE on `order_date` (yearly)
Partitions: `orders_2024`, `orders_2025`, `orders_2026`, `orders_default` (catch-all).

| Column | Type | Nullable | Key | Description |
|---|---|---|---|---|
| `order_id` | INTEGER | No | PK (composite) | Order identifier — unique in combination with `order_date` |
| `customer_id` | INTEGER | No | FK → `customers.customer_id` | Customer who placed the order |
| `warehouse_id` | INTEGER | No | FK → `warehouses.warehouse_id` | Warehouse the order is fulfilled from |
| `order_date` | DATE | No | PK (composite), **partition key** | Date the order was placed |
| `order_status` | VARCHAR(20) | No | | One of `Completed`, `Processing`, `Cancelled` |

> **Why the composite key?** Postgres requires a partitioned table's
> primary key to include the partition column, so `order_id` alone can't
> be the PK once `orders` is partitioned by `order_date`.

## `order_items` — NOT partitioned, but carries `order_date` for its FK
One row per product per order (the junction table between `orders` and `products`).

| Column | Type | Nullable | Key | Description |
|---|---|---|---|---|
| `order_item_id` | INTEGER | No | PK | Unique identifier for the order line item |
| `order_id` | INTEGER | No | FK (composite, with `order_date`) → `orders` | Parent order |
| `order_date` | DATE | No | FK (composite, with `order_id`) → `orders` | Denormalized copy of the parent order's date — required because `orders`' primary key is composite |
| `product_id` | INTEGER | No | FK → `products.product_id` | Product ordered |
| `quantity` | INTEGER | No | | Units ordered; must be > 0 |
| `unit_price` | NUMERIC(10,2) | No | | Price **at time of order** (kept even if the product's catalog price later changes) |
| `line_total` | NUMERIC(12,2) | No | | `quantity × unit_price` |

## `shipments` — PARTITIONED by RANGE on `ship_date` (yearly)
Partitions: `shipments_2024`, `shipments_2025`, `shipments_2026`, `shipments_default` (catch-all).

| Column | Type | Nullable | Key | Description |
|---|---|---|---|---|
| `shipment_id` | INTEGER | No | PK (composite) | Shipment identifier — unique in combination with `ship_date` |
| `order_id` | INTEGER | No | | Order this shipment fulfills (1:1 with orders) |
| `carrier_id` | INTEGER | No | FK → `carriers.carrier_id` | Carrier handling the shipment |
| `service_level` | VARCHAR(20) | No | | One of `Standard`, `Express`, `Overnight`, `Freight` |
| `ship_date` | DATE | No | PK (composite), **partition key** | Date the order was shipped |
| `expected_delivery_date` | DATE | No | | Date delivery was expected |
| `actual_delivery_date` | DATE | **Yes** | | Actual delivery date; NULL means still in transit or cancelled |
| `delivery_status` | VARCHAR(20) | No | | One of `Delivered`, `In Transit`, `Delayed`, `Cancelled` |
| `shipping_cost` | NUMERIC(10,2) | No | | Cost of the shipment; must be ≥ 0 |

---

## Indexes (beyond primary keys)

| Index | Table | Columns | Purpose |
|---|---|---|---|
| `idx_orders_customer_id` | orders | customer_id | Fast lookup of a customer's orders |
| `idx_orders_warehouse_id` | orders | warehouse_id | Fast lookup of a warehouse's orders |
| `idx_orders_warehouse_date` | orders | warehouse_id, order_date | "Orders per warehouse over time" reporting |
| `idx_order_items_order_id` | order_items | order_id | Fast lookup of an order's line items |
| `idx_order_items_product_id` | order_items | product_id | Fast lookup of a product's order history |
| `idx_shipments_order_id` | shipments | order_id | Fast lookup of an order's shipment |
| `idx_shipments_carrier_id` | shipments | carrier_id | Fast lookup of a carrier's shipments |
| `idx_shipments_delivery_status` | shipments | delivery_status | Filtering by delivery status for reporting |

## View

**`vw_order_line_items_full`** — pre-joins all 7 tables into one row per
order line item (order + customer + product + warehouse + shipment +
carrier details), so analysts don't need to remember every join.
