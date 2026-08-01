# From Flat File to Relational Schema: Design Rationale

## Starting point

The raw export (`raw_logistics_export.csv`) has one row per product on
one order — 29 columns, roughly 1.9 million rows at the 750,000-order
scale. Every row repeats the full customer record, the full product
record, the full warehouse record, and the full shipment record,
regardless of how many times that customer, product, warehouse, or
shipment already appears elsewhere in the file. This is normal for a
system export; it is not usable as a database design as-is, for three
concrete reasons.

**Redundancy.** A customer who has placed 40 orders has their name,
email, city, and region repeated on every one of those rows. If that
customer's email is wrong, it has to be corrected in 40 places, not one.

**Update, insert, and delete anomalies.** Because customer and product
data only exists attached to an order row, there is no way to record a
new customer before they place their first order, and no way to remove
a discontinued product without also losing every historical order line
that mentions it.

**No enforceable integrity.** Nothing in a flat file stops a row from
referencing a `customer_id` or `product_id` that doesn't exist elsewhere
in the file. Consistency depends entirely on whoever generated the
export having done everything correctly upstream.

## How the tables were identified

The method was to look at each column in the raw file and ask what it
actually depends on — not what row it happens to sit on, but what
real-world entity determines its value. This is functional dependency
analysis, and it's the basis for normalization.

Columns like `customer_name`, `email`, `customer_city`, and
`customer_region` all depend only on `customer_id` — a given customer
has exactly one name, one email, one city, regardless of which order
you're looking at. The same is true for `product_name`, `category`, and
`unit_price` depending only on `product_id`; for `warehouse_name`,
`warehouse_city`, and `warehouse_region` depending only on
`warehouse_id`; and for `carrier_name` depending only on `carrier_id`.
None of these values actually depend on the order — they were just
carried along on every row because the file is flat.

Once those groups were identified, each became its own table:
`customers`, `products`, `warehouses`, `carriers`. This removes the
redundancy directly — each customer, product, warehouse, and carrier now
exists exactly once, and every other table refers to it by ID instead of
repeating its details.

What's left after pulling those out is the data that genuinely belongs
to the transaction itself: which customer ordered from which warehouse
and when (`orders`), which products and quantities were on that order
(`order_items`), and how and when it shipped (`shipments`). These three
remain, because unlike the reference data above, they don't reduce to a
single lookup — a customer places many orders, an order contains many
products, and each of those facts is genuinely tied to a specific
transaction rather than to one recurring entity.

## Why `order_items` is a separate table, not a column on `orders`

An order can contain more than one product. Cramming a variable-length
list of products into a single `orders` row isn't representable in a
relational table, where each column holds one value. The standard
solution is a bridge (junction) table: `order_items` has one row per
product per order, and connects `orders` to `products` through two
foreign keys. This is also the point where `unit_price` and `line_total`
live — deliberately kept here rather than always looking them up from
`products`, because the price at the time of purchase needs to stay
fixed even if the catalog price changes later. That's an intentional,
reasoned exception to strict normalization, not an oversight.

## Why `shipments` is separate from `orders`, even though it's 1:1

Every order has exactly one shipment in this dataset, so it might seem
like shipment columns could just live on the `orders` table. They were
kept separate for two reasons. First, conceptually, an order and its
shipment are different things with different lifecycles — an order is
placed once and rarely changes after that; a shipment's status,
carrier, and delivery dates get updated repeatedly as it moves. Keeping
them apart means those frequent updates don't touch the core order
record at all. Second, this separation is what made it possible to
partition `orders` and `shipments` independently by their own dates
(`order_date` and `ship_date`), which wouldn't be possible if they were
one table.

## What level of normalization this reaches

The result satisfies third normal form (3NF): every non-key column in
each table depends on that table's key, the whole key, and nothing but
the key. First normal form is satisfied because every column holds a
single atomic value (no repeating groups, which the raw file already
avoided by having one row per line item). Second normal form is
satisfied because no column depends on only part of a composite key —
this is exactly the problem that was fixed by moving `customer_name`,
`product_name`, and similar columns out of the order-line grain and into
their own tables. Third normal form is satisfied because no column
depends on another non-key column instead of the key itself — for
example, `warehouse_region` depends on `warehouse_id`, not on
`order_id`, so it belongs in `warehouses`, not `orders`.

The one deliberate departure from strict normalization is the
`unit_price` kept on `order_items`, discussed above — a considered
trade-off for historical accuracy, not a missed dependency.

## Where partitioning fits into this

Partitioning `orders` and `shipments` by year was a separate decision
made after normalization, for a different reason: physical query
performance, not logical correctness. It required Postgres' composite
primary key rule (the partition column must be part of the primary
key), which is why `order_items` carries a denormalized `order_date`
column purely to support its foreign key into `orders`. That column
doesn't violate normalization on its own terms — it isn't functionally
independent information, it's a structural necessity introduced by a
storage-layer optimization, and it's documented as such in the schema
and data dictionary.
