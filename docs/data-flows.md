# Data flows

One row per flow: where the data comes from, what it is, who uses it, how often
it moves, and in which format.

Sources: the course brief for Albert's Marketplace, and the column headers of
the nightly export (`legacy_csv`, release v1.0.0). Rows marked *assumed* are not
in the brief and need confirming.

| # | Source | Data | Consumer | Frequency | Format | Notes |
|---|---|---|---|---|---|---|
| 1 | Customers, through the web shop | Accounts (`customer`: name, email, phone, `pwd`), carts, orders and order items, payments, saved cards (`payment_details`: `card_no`, `cvv`, expiry, billing address), shipping addresses, reviews, wishlists | PostgreSQL (shop database) | Continuous, per transaction | SQL writes | The only writer of customer and order data |
| 2 | PostgreSQL | Product catalogue, prices, daily deals, reviews, the customer's account, saved cards and addresses at checkout | Web shop | Continuous, per page view | SQL reads | The shop and everything below share one database |
| 3 | PostgreSQL | Any of the tables, including orders, payments and customers | Analysts | Ad hoc, during the day | SQL result sets | Runs against the same tables checkout writes to |
| 4 | PostgreSQL | Full export, one file per table (27 files), including `customer.pwd` and `payment_details.card_no` / `cvv` | Analysts' spreadsheets | Nightly batch | CSV | The brief says 25 tables; the export holds 27. Each file starts with an unnamed index column |
| 5 | Analysts' spreadsheets and direct queries (flows 3 and 4) | Revenue figures | Readers of the revenue reports | Not stated in the brief | Spreadsheets | The reports disagree on revenue |
| 6 | *Requested, does not exist yet* | Units sold per product, per country, per day | ML team | Daily | Table | Country is not on the order: it has to come from the buyer's shipping address (`customer_shipping` → `shipping_details.country`) |
| 7 | *Assumed:* sellers | Products, prices, stock (`product`, `seller_products`) | PostgreSQL | Continuous | SQL writes | `seller` and `seller_products` tables exist, but the brief does not say how products get in |

## Under the matrix, answer

### Which flows carry personal data?

Flows 1 to 4. Flow 1 writes it, flow 2 reads it back to the shop, and flows 3
and 4 put it in front of analysts who do not need it:

- **Identity and contact:** `customer.fname`, `lname`, `email`, `phone`, and the
  addresses in `shipping_details`.
- **Credentials:** `customer.pwd`. Every value we sampled is 12 characters with
  no hash format, so the passwords look like plain text.
- **Payment card data:** `payment_details.card_no`, `cvv`, `expiry_date`,
  `billing_address`. The card industry's rules (PCI DSS) forbid storing a CVV
  after authorisation, and flow 4 copies it into a file every night.

Flow 5 should carry only aggregates. Flow 6 does not need personal data: country
is enough, and the table never has to hold a customer.

### Which consumers read directly from a system that also serves customers?

- **Analysts (flow 3):** their ad hoc queries run on the database the shop
  writes to and reads from, so a heavy query competes with checkout.
- **The nightly export (flow 4):** a full dump of every table, read from the
  same database. It is quieter at night, but the shop is still open.
- **The ML team (flow 6)** would be the third, if their table were built with a
  query on PostgreSQL.

### Where could the same figure be computed twice, in two different ways?

**Revenue**, in at least three ways:

1. **Live vs nightly:** an analyst querying PostgreSQL at 3 pm and a spreadsheet
   built from last night's CSV see different sets of orders. Nothing records
   which one a report used.
2. **Price at purchase vs current price:** `order_items.price_at_purchase` ×
   `qty` versus `product.price` × `qty`. In the v1.0.0 export the two totals
   differ (39,199,901 vs 39,199,921), so a report using the catalogue price is
   wrong, and the gap grows every time a price changes.
3. **Payments vs order lines:** `payment.amount` versus the sum of the order
   lines. They match in this export, but nothing enforces it. Discounts
   (`orders.discount_id`, `daily_deals`) and returns (`returns`) would make them
   diverge: both tables are empty in this export, and neither definition says
   how to treat them.

**Units sold** has the same problem: `order_items.qty` counts what was ordered,
`shipment` what left the warehouse, and `returns` what came back. The ML
team's table has to choose one, and say which.
