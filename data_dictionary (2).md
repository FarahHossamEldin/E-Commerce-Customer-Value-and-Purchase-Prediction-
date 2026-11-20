# Data Dictionary: Olist Brazilian E-Commerce (Processed)

**Source:** Olist Brazilian E-Commerce Public Dataset (Kaggle, `olistbr/brazilian-ecommerce`)  
**Files:** Nine CSV files in `data/processed/`

## Important scope note

This dictionary describes the processed CSV files based on the cleaning work completed so far. It does **not** describe proposed future transformations.

- No derived analytical columns have been added in the cleaning work documented here.
- Only two datasets were modified:
  - `products`: four zero values in `product_weight_g` were converted to missing values (`NaN`).
  - `geolocation`: 261,831 exact duplicate rows were removed; 738,332 rows remain.
- The other datasets were reviewed but their rows and values were preserved.
- Potential anomalies were documented rather than automatically corrected or deleted.
- ZIP-code prefixes remain integer columns in the current DataFrames; they have not been converted to text.

## Relationships between datasets

The CSV files do not contain database-enforced foreign-key constraints. The following logical relationships were checked using matching ID columns:

```text
customers 1 ── N orders
orders    1 ── N order_items
orders    1 ── N payments
orders    1 ── N reviews
products  1 ── N order_items
sellers   1 ── N order_items
```

`geolocation` can be related to `customers` and `sellers` through ZIP-code prefixes. The geolocation table has not been aggregated to one row per ZIP prefix; it contains distinct source records after exact duplicate removal.

### Relationship validation results

| Relationship / check | Result |
|---|---:|
| Orders with a matching customer | 99,441 of 99,441 |
| Orders with at least one matching order item | 98,666 of 99,441 |
| Orders without order items | 775 |
| Orders with a matching payment record | 99,440 of 99,441 |
| Orders without payment records | 1 |
| Order items with a matching product | 112,650 of 112,650 |
| Order items with a matching seller | 112,650 of 112,650 |
| Reviews with a matching order | All matched |

The 775 orders without order items had these statuses: 603 `unavailable`, 164 `canceled`, 5 `created`, 2 `invoiced`, and 1 `shipped`. They were retained.

One delivered order had no payment record. It was retained and documented as an anomaly.

---

## customers

**File:** `olist_customers_dataset.csv`  
**Rows:** 99,441  
**Columns:** 5

| Column | Type in current DataFrame | Description | Notes |
|---|---|---|---|
| `customer_id` | string | Customer record identifier used to link to `orders` | Unique; no duplicates found |
| `customer_unique_id` | string | Identifier representing the real customer across records | 3,345 repeated values; retained because repeat customers matter |
| `customer_zip_code_prefix` | integer | Customer ZIP-code prefix | No missing values |
| `customer_city` | string | Customer city | No missing values |
| `customer_state` | string | Customer state abbreviation | No missing values |

## orders

**File:** `olist_orders_dataset.csv`  
**Rows:** 99,441  
**Columns:** 8

| Column | Type | Description | Notes |
|---|---|---|---|
| `order_id` | text | Unique order identifier | Used to link order-related tables |
| `customer_id` | text | Customer record identifier | Links to `customers.customer_id`; all 99,441 matched |
| `order_status` | text | Current order status | Includes delivered, shipped, canceled, unavailable, created, invoiced, processing, and approved |
| `order_purchase_timestamp` | datetime | Timestamp when the order was placed | |
| `order_approved_at` | datetime, nullable | Payment approval timestamp | Missing values retained |
| `order_delivered_carrier_date` | datetime, nullable | Timestamp when the order was handed to the carrier | Missing values and chronology anomalies retained |
| `order_delivered_customer_date` | datetime, nullable | Timestamp when the order was delivered to the customer | Missing values and chronology anomalies retained |
| `order_estimated_delivery_date` | datetime | Estimated delivery date | |

**Documented timestamp findings:** 166 carrier timestamps were earlier than purchase timestamps, and 24 customer-delivery timestamps were earlier than carrier timestamps. These anomalies were investigated and documented; timestamps were not changed. Counts may overlap.

## order_items

**File:** `olist_order_items_dataset.csv`  
**Rows:** 112,650  
**Columns:** 7

| Column | Type | Description | Notes |
|---|---|---|---|
| `order_id` | text | Order identifier | Links to `orders.order_id` |
| `order_item_id` | integer | Item sequence within an order | Composite key with `order_id`; no duplicate pairs found |
| `product_id` | text | Product identifier | Links to `products.product_id`; all items matched |
| `seller_id` | text | Seller identifier | Links to `sellers.seller_id`; all items matched |
| `shipping_limit_date` | datetime | Deadline for the seller to hand the item to the carrier | |
| `price` | numeric | Item price in Brazilian reais (BRL) | No negative values found; high values reviewed and retained |
| `freight_value` | numeric | Freight charge in BRL | Zero values retained; may represent free shipping |

No derived `item_total`, `is_price_outlier`, or `is_freight_outlier` columns have been added.

## payments

**File:** `olist_order_payments_dataset.csv`  
**Rows:** 103,886  
**Columns:** 5

| Column | Type | Description | Notes |
|---|---|---|---|
| `order_id` | text | Order identifier | Links to `orders.order_id` |
| `payment_sequential` | integer | Sequence number of a payment within an order | |
| `payment_type` | text | Payment method | Includes `credit_card`, `boleto`, `voucher`, `debit_card`, and `not_defined` |
| `payment_installments` | integer | Number of installments | Two records have zero installments; retained |
| `payment_value` | numeric | Payment amount in BRL | Zero-value records retained |

Three `not_defined` records and zero-value payment records were investigated and retained. No payment rows or values were changed.

## reviews

**File:** `olist_order_reviews_dataset.csv`  
**Columns:** 7

| Column | Type | Description | Notes |
|---|---|---|---|
| `review_id` | text | Review identifier | 814 repeated IDs found; not treated as a unique key |
| `order_id` | text | Order identifier | Links to `orders.order_id`; all reviews matched |
| `review_score` | integer | Customer rating from 1 to 5 | All observed scores were within the valid range |
| `review_comment_title` | text, nullable | Optional review title | Missing values retained |
| `review_comment_message` | text, nullable | Optional review message | Missing values retained |
| `review_creation_date` | datetime | Date the review survey was sent | |
| `review_answer_timestamp` | datetime | Timestamp when the customer answered | |

There were no exact duplicate rows. Repeated `review_id` values were retained because they were associated with different orders. No “latest review per order” filtering was performed.

## products

**File:** `olist_products_dataset.csv`  
**Columns:** 9

| Column | Type | Description | Notes |
|---|---|---|---|
| `product_id` | text | Unique product identifier | No duplicate IDs found |
| `product_category_name` | text, nullable | Product category in Portuguese | Missing values retained |
| `product_name_lenght` | numeric, nullable | Product-name length | Original source column spelling retained |
| `product_description_lenght` | numeric, nullable | Product-description length | Original source column spelling retained |
| `product_photos_qty` | numeric, nullable | Number of product photos | |
| `product_weight_g` | numeric, nullable | Product weight in grams | Four zero values converted to missing values |
| `product_length_cm` | numeric, nullable | Product package length in centimeters | |
| `product_height_cm` | numeric, nullable | Product package height in centimeters | |
| `product_width_cm` | numeric, nullable | Product package width in centimeters | |

Missing product metadata and physical measurements were retained. No `unknown` category values or English-category column were added during the documented cleaning stage.

## sellers

**File:** `olist_sellers_dataset.csv`  
**Rows:** 3,095  
**Columns:** 4

| Column | Type in current DataFrame | Description | Notes |
|---|---|---|---|
| `seller_id` | string | Unique seller identifier | No duplicate IDs found |
| `seller_zip_code_prefix` | integer | Seller ZIP-code prefix | No non-positive values found |
| `seller_city` | string | Seller city | |
| `seller_state` | string | Seller state abbreviation | |

No missing values were found.

## geolocation

**File:** `olist_geolocation_dataset.csv`  
**Rows after exact-duplicate removal:** 738,332  
**Columns:** 5

| Column | Type in current DataFrame | Description | Notes |
|---|---|---|---|
| `geolocation_zip_code_prefix` | integer | ZIP-code prefix associated with a geographic record | Multiple records may share a prefix |
| `geolocation_lat` | float | Latitude | All values passed the valid latitude range check |
| `geolocation_lng` | float | Longitude | All values passed the valid longitude range check |
| `geolocation_city` | string | City associated with the geographic record | |
| `geolocation_state` | string | State abbreviation | |

A total of 261,831 exact duplicate rows were removed. No missing values or invalid coordinate ranges were found. Coordinates were not averaged, and the table was not reduced to one row per ZIP prefix.

## category_translation

**File:** `product_category_name_translation.csv`  
**Rows:** 71  
**Columns:** 2

| Column | Type | Description | Notes |
|---|---|---|---|
| `product_category_name` | text | Product category name in Portuguese | No duplicate category names found |
| `product_category_name_english` | text | English translation of the category | |

No missing values or exact duplicate rows were found. This table remains separate; its translations have not been merged into `products`.

---

## Notes for future analysis

- Statistical outlier treatment (for example, IQR-based flagging, capping, or transformation) has not yet been performed systematically. Extreme values reviewed during cleaning were retained unless there was clear evidence of invalid data.
- Additional derived features should be documented separately when they are created for EDA or modeling.
- Before using a column in a database, confirm its actual pandas/CSV type in the final exported files. ZIP-code prefixes may be better represented as text in a future schema if preserving leading zeros is required.
