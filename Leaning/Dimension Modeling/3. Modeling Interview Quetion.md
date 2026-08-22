# Interview

**Question → How to think → Answer → Example → Edge cases → Interview communication**

---

# 1. You are asked to design a sales warehouse. What is the first thing you do?

### How a professional thinks

A junior engineer starts drawing:

```text
fact_sales
dim_customer
dim_product
```

A strong engineer **doesn't start with tables**.

The first question is:

> **What business process are we modeling, and what questions must the model answer?**

Then determine:

1. Business process
2. Grain
3. Measures
4. Dimensions
5. Historical requirements
6. Source systems
7. Expected query patterns
8. Scale
9. Data quality requirements

### Answer

Suppose the business wants:

* revenue by product
* revenue by customer
* revenue by region
* daily sales
* discount analysis
* historical customer segmentation

I'd first identify **order line item** as the likely grain:

> One row represents one product line within one order.

Then:

```text
fact_sales
-----------
order_id
order_line_id
date_key
customer_key
product_key
store_key
quantity
gross_amount
discount_amount
net_amount
```

Dimensions:

```text
dim_customer
dim_product
dim_date
dim_store
```

If customer segmentation changes historically, I'd consider SCD Type 2 for `dim_customer`.

### Edge case

What if one order has multiple currencies?

Then `amount = 100` is meaningless without currency context.

You may need:

```text
transaction_currency
exchange_rate
base_currency_amount
```

### Professional communication

Say:

> "Before finalizing the schema, I'd establish the business grain and metric definitions. Otherwise we risk designing a technically valid schema that produces incorrect business results."

---

# 2. Why is grain more important than choosing star vs snowflake?

### How to think

Star vs snowflake is a structural decision.

**Grain is a semantic decision.**

If grain is wrong, the entire model can produce incorrect numbers.

### Example

Suppose:

```text
Order 1001
Laptop = ₹50,000
Mouse = ₹2,000
```

If your fact is order-level:

```text
1001 | ₹52,000
```

If your fact is order-line-level:

```text
1001 | Laptop | ₹50,000
1001 | Mouse  | ₹2,000
```

Now suppose someone joins order-level data to order-line-level data.

The ₹52,000 can appear twice.

You get:

```text
₹104,000
```

instead of:

```text
₹52,000
```

### Professional rule

Before adding a column to a fact table ask:

> **"Can this attribute have multiple values at this grain?"**

If yes, something is wrong.

### Interview answer

> "I establish grain before selecting the dimensional structure because grain determines which dimensions, measures, keys, and relationships are valid. Star versus snowflake primarily affects organization and consumption, whereas incorrect grain can directly produce incorrect business metrics."

---

# 3. You have an order table and an order-item table. Should they become one fact table or two?

### Scenario

Source:

```text
orders
------
order_id
customer_id
order_date
order_total
shipping_fee
```

```text
order_items
-----------
order_id
product_id
quantity
unit_price
discount
```

### How to think

They have different grains.

`orders`:

> one row per order

`order_items`:

> one row per order line

Combining them blindly creates a mixed-grain fact.

### Better approach

Potentially:

```text
fact_order
----------
order_id
customer_key
date_key
order_total
shipping_fee
```

and:

```text
fact_order_item
---------------
order_id
order_line_id
product_key
quantity
unit_price
discount
net_amount
```

### But don't automatically create two facts.

Ask:

> "What analytics are required?"

If almost every analytical use case is product-level, perhaps the order-level information can be derived from the item fact.

For example:

```sql
SUM(net_amount)
GROUP BY order_id
```

might be sufficient.

### Edge case

Shipping fee exists only at order level.

If you copy `shipping_fee` onto every order item:

```text
Laptop → ₹100 shipping
Mouse  → ₹100 shipping
Keyboard → ₹100 shipping
```

then:

```sql
SUM(shipping_fee)
```

becomes ₹300 instead of ₹100.

You need an allocation strategy if shipping must exist at item grain.

### Professional answer

> "I wouldn't combine them simply because they belong to the same business process. I'd first identify their grains. If order-level and item-level measures coexist, I either maintain separate facts or explicitly define an allocation/derivation strategy."

---

# 4. A business asks for "revenue." What questions should you ask before modeling it?

This is a very good senior-level question.

"Revenue" sounds simple but can have multiple meanings.

Ask:

### 1. Gross or net?

```text
gross sales
- discount
- return
- tax
```

Which one?

### 2. Order date or payment date?

A customer places an order on:

```text
Jan 31
```

Payment occurs:

```text
Feb 1
```

Which month gets revenue?

### 3. Return treatment?

If:

```text
Sale = ₹1,000
Return = ₹1,000
```

is revenue:

```text
₹1,000
```

or:

```text
₹0
```

### 4. Currency?

```text
USD
EUR
INR
```

Do we report transaction currency or converted currency?

### 5. Cancelled orders?

Do they count?

### 6. Partial refunds?

Suppose:

```text
Order = ₹10,000
Refund = ₹3,000
```

What is net revenue?

### Professional thinking

> **A metric definition is part of the data model.**

You shouldn't simply create:

```text
revenue
```

without defining its semantic meaning.

---

# 5. A dimension changes frequently. Should you always use SCD Type 2?

### Answer

No.

SCD Type 2 is powerful but expensive.

Suppose customer status changes every day:

```text
Silver
Gold
Silver
Gold
Platinum
...
```

Type 2 could generate enormous numbers of rows.

Ask:

> Does historical state actually matter?

If not:

```text
Type 1
```

may be better.

If historical reporting matters:

```text
Type 2
```

If only current and previous state matter:

```text
Type 3
```

### Edge case

What if an attribute changes 100 times per day?

Type 2 may create:

```text
customer × hundreds of versions
```

You need to question whether that attribute belongs in the dimension at all.

Maybe it is actually an **event stream** or transactional fact.

### Professional insight

> **Frequent change is sometimes a signal that you've modeled the business concept at the wrong grain.**

That's an excellent interview observation.

---

# 6. A customer moves from Hyderabad to Bangalore. Which address should historical sales use?

This is a classic SCD Type 2 scenario.

Suppose:

```text
2025 sale → Hyderabad
2026 sale → Bangalore
```

If historical reporting matters, the 2025 sale should still be associated with the Hyderabad version.

Dimension:

```text
customer_sk | customer_id | city       | start | end
1           | 101         | Hyderabad  | 2025  | 2026
2           | 101         | Bangalore  | 2026  | NULL
```

Facts:

```text
2025 sale → customer_sk 1
2026 sale → customer_sk 2
```

### Edge case

What if a sale arrives late?

Sale date:

```text
2025-10-10
```

But it arrives in the warehouse in:

```text
2026-08
```

You cannot simply use the customer's current dimension version.

You need **effective-date lookup**.

### Professional answer

> "For historical reporting, I would resolve the dimension key based on the business event's effective timestamp, not the warehouse ingestion timestamp."

That's an important distinction.

---

# 7. A fact arrives before its dimension record. What do you do?

This is the **late-arriving dimension** problem.

Example:

```text
Order arrives
customer_id = 5001
```

But `customer 5001` isn't yet present in the dimension.

If the fact requires:

```text
customer_key
```

you have a problem.

### Possible solution

Create an **unknown/inferred member**.

Example:

```text
customer_sk = -1
customer_id = 5001
status = UNKNOWN
```

Then later update the dimension when customer information arrives.

Or use an inferred member:

```text
customer_sk = 90001
customer_id = 5001
customer_name = NULL
```

Then enrich it later.

### Why not simply reject the fact?

Because rejecting the fact can cause:

* lost revenue
* incomplete reporting
* reconciliation issues

### Professional thinking

You need a policy for:

```text
missing dimension
late dimension
invalid key
unknown customer
deleted customer
```

### Interview answer

> "I wouldn't allow referential-integrity issues to silently drop business events. I'd typically use an unknown or inferred dimension member, capture the exception, and reconcile the dimension when the authoritative record arrives."

---

# 8. What happens if a source system changes the meaning of a column?

This is more dangerous than a schema change.

Suppose:

Initially:

```text
status = 1 → active
status = 0 → inactive
```

Then the application changes:

```text
status = 1 → pending
status = 2 → active
```

The column still exists.

Your pipeline still runs.

But your warehouse is now **wrong**.

### Professional insight

Schema validation alone doesn't protect against semantic changes.

You need:

* data contracts
* source documentation
* accepted-value tests
* anomaly detection
* reconciliation
* business validation

Example:

```text
status IN ('active', 'inactive')
```

If suddenly:

```text
pending
```

appears, the pipeline should alert.

### Key lesson

> **A pipeline can be technically successful while producing semantically incorrect data.**

This is one of the most important data-engineering concepts.

---

# 9. A dimension has 500 attributes. Should you keep everything in one dimension?

Not necessarily.

Ask:

> Are all attributes used at the same grain and for the same analytical purpose?

Suppose `dim_customer` contains:

```text
name
email
phone
address
income
credit_score
marketing_preferences
device_preferences
employment
social_profile
...
```

This can become unwieldy.

### Possible approaches

Separate dimensions or satellites based on:

* business domain
* change frequency
* security
* ownership
* access patterns

For example:

```text
dim_customer
dim_customer_contact
dim_customer_marketing
dim_customer_risk
```

### But don't over-normalize.

If you split everything into 50 tables, analysts will hate the model.

### Professional decision

Consider:

* frequency of use
* change frequency
* security requirements
* ownership
* query complexity
* storage
* semantic clarity

---

# 10. When would you choose a periodic snapshot instead of a transaction fact?

### Scenario

You have bank accounts.

Transactions:

```text
Deposit +₹10,000
Withdrawal -₹2,000
Deposit +₹5,000
```

You could calculate balance from transactions.

But the business asks:

> "What was the balance of every account at the end of every day?"

A daily snapshot is appropriate.

```text
fact_account_daily
------------------
date_key
account_key
closing_balance
```

### Why?

Because the business question asks for **state**, not just events.

### Transaction fact

Answers:

> What happened?

### Periodic snapshot

Answers:

> What was the state at a regular point in time?

### Edge case

If you only store snapshots, you may lose transaction-level detail.

Therefore a mature system may have both:

```text
fact_transaction
fact_account_daily
```

The snapshot can be derived from transactions.

---

# 11. When would an accumulating snapshot be better than event modeling?

Consider order fulfillment:

```text
Order created
Payment
Packing
Shipping
Delivery
```

An event table:

```text
fact_order_event
----------------
order_id
event_type
event_timestamp
```

is excellent for detailed event analysis.

But a business dashboard might repeatedly ask:

```text
average time from order to delivery
average payment-to-shipping time
orders currently stuck in packing
```

An accumulating snapshot:

```text
fact_order_lifecycle
--------------------
order_id
created_at
paid_at
packed_at
shipped_at
delivered_at
```

can make those queries much simpler.

### Professional answer

Don't choose one universally.

You can have:

```text
raw/event-level history
        ↓
accumulating snapshot
```

The event model preserves detail; the accumulating snapshot optimizes process analytics.

---

# 12. Two teams want different definitions of "customer." How do you model it?

This is not merely a SQL problem.

Suppose:

Marketing says:

> Customer = anyone who registered.

Finance says:

> Customer = anyone who completed a paid transaction.

Support says:

> Customer = anyone with an active account.

You now have competing semantics.

### Bad approach

Create:

```text
dim_customer
```

and silently choose one definition.

### Better approach

Create a shared business definition or explicitly model different concepts:

```text
registered_user
paying_customer
active_customer
```

Potentially:

```text
dim_user
```

plus status/relationship structures.

### Professional communication

Say:

> "Before implementing the model, I'd get the business owners to agree on the canonical entity definition. If the concepts are genuinely different, I would model them separately rather than forcing different semantics into one customer dimension."

### Key principle

> **Data modeling is also semantic governance.**

---

# 13. Your fact table has 20 billion rows. Would you automatically aggregate it?

No.

First ask:

> What queries are actually slow?

Suppose users query:

```text
daily revenue by country
```

You might introduce:

```text
agg_daily_country_sales
```

But if users also need:

```text
individual order analysis
```

you cannot replace the detailed fact.

### Possible architecture

```text
Detailed Fact
     ↓
Aggregated Tables
     ↓
BI
```

Keep the detailed fact as the source of truth and use aggregates for common workloads.

### But aggregates have risks

You now have:

```text
fact_sales
agg_sales
```

Both must remain consistent.

If transformations differ, numbers can diverge.

### Professional answer

> "I would introduce aggregate tables based on demonstrated workload patterns rather than row count alone. I'd preserve the atomic fact for detailed analysis and establish a clear lineage and reconciliation strategy for derived aggregates."

---

# 14. You have a many-to-many relationship between customers and accounts. How do you model it?

Suppose:

```text
Customer A → Account X
Customer B → Account X
```

and:

```text
Customer A → Account Y
```

A simple foreign key doesn't capture that cleanly.

You need a bridge:

```text
bridge_customer_account
-----------------------
customer_key
account_key
relationship_type
effective_date
end_date
```

### Why this matters

Suppose account balance is:

```text
₹100,000
```

and two customers are associated with it.

If you join:

```text
account → customer
```

and sum balance by customer, you might accidentally produce:

```text
₹100,000
+
₹100,000
=
₹200,000
```

even though the actual account balance is ₹100,000.

### Professional thinking

Many-to-many relationships are dangerous because they can cause **measure multiplication**.

You need to define:

* ownership
* allocation percentage
* relationship type
* effective dates

Maybe:

```text
Customer A = 60%
Customer B = 40%
```

Then allocation is explicit.

---

# 15. A fact contains NULL foreign keys. Is that a data-quality failure?

Not necessarily.

You must understand why.

Suppose:

```text
pickup_location_key = NULL
```

For a ride where the GPS wasn't available.

That might be a genuine unknown.

But if:

```text
95% of customer_key values = NULL
```

then something is probably broken.

### Professional approach

Distinguish:

```text
Unknown
Not applicable
Missing
Not yet available
Invalid
```

Don't treat all NULLs as one semantic state.

A dimensional model may use explicit members:

```text
-1 = Unknown
-2 = Not Applicable
-3 = Not Yet Available
```

depending on the warehouse conventions.

### Key lesson

> **NULL is a technical representation; business meaning must be defined separately.**

---

# 16. Should dates be stored as timestamps or surrogate date keys?

For analytical models, a common pattern is:

```text
date_key = 20260823
```

with:

```text
dim_date
--------
date_key
calendar_date
day
month
quarter
year
week
fiscal_period
holiday_flag
```

Then facts use:

```text
date_key
```

### Why?

The date dimension can provide standardized business calendar attributes.

For example:

```text
2026-08-23
```

might map to:

```text
FY2026
Q2
Week 34
Sunday
Holiday = No
```

### But timestamps are still important.

Don't replace event timestamps with date keys.

A fact may need:

```text
created_at
completed_at
```

for precise event ordering.

The date key is useful for analytical dimensionalization.

### Professional answer

> "I generally preserve the original timestamp for event-level accuracy and may also derive date keys for dimensional reporting. These solve different problems."

---

# 17. A source deletes a customer. Should the warehouse delete the customer too?

Usually, **not automatically**.

You need to understand the business meaning of deletion.

Possible scenarios:

### Physical deletion

The source says:

> Delete this customer permanently.

### Business deactivation

The account becomes inactive but history must remain.

### Privacy deletion

The customer requests data deletion.

### Data correction

The source accidentally deleted the record.

These are very different.

### Analytical warehouse often needs historical preservation.

You may use:

```text
is_active
end_date
deleted_flag
```

depending on requirements.

### Privacy edge case

If regulations require personal data deletion, you may need to remove or anonymize PII while preserving non-identifying analytical facts.

For example:

```text
customer_key
purchase_amount
date_key
```

could potentially remain while:

```text
name
email
phone
address
```

are removed or anonymized.

### Professional thinking

> "I wouldn't treat source deletion as an automatic warehouse deletion. I'd first distinguish business lifecycle, technical deletion, and privacy deletion requirements."

---

# 18. Your product hierarchy changes. How do you preserve historical reporting?

Suppose today:

```text
Electronics
  → Laptop
```

Tomorrow the company restructures:

```text
Computers
  → Laptop
```

What should old sales show?

This is a business decision.

Possibilities:

### Historical classification

Old sales remain under:

```text
Electronics
```

New sales use:

```text
Computers
```

### Restated classification

All historical sales are reclassified to:

```text
Computers
```

These produce different reports.

### SCD Type 2

The product/category dimension can preserve historical versions.

### Professional question

Ask the business:

> "Do you want reporting based on the hierarchy that existed when the transaction occurred, or the current hierarchy applied retrospectively?"

That one question prevents huge reporting disagreements.

---

# 19. Your model is correct but analysts keep writing incorrect SQL. What do you change?

This is an excellent real-world scenario.

Suppose your model technically works, but analysts repeatedly:

* double-count orders
* join facts together
* misuse measures
* filter the wrong date
* misunderstand SCDs

The problem may be the **model usability**, not just analyst skill.

### Possible improvements

Create:

```text
semantic layer
```

or curated marts.

For example:

```text
fact_sales
```

might be technically correct but difficult.

You could provide:

```text
sales_mart
```

with carefully defined measures and dimensions.

Also provide:

* data dictionary
* column descriptions
* grain documentation
* certified metrics
* example queries
* join guidance

### Strong principle

> **A good data model should make the correct behavior obvious.**

If every analyst needs a 30-page document to avoid double counting, the model may need improvement.

---

# 20. You inherit a warehouse with a terrible schema. How would you redesign it without breaking everyone?

This is a very realistic senior data-engineering scenario.

Don't immediately replace everything.

### Step 1 — Understand consumers

Identify:

```text
dashboards
reports
ML models
analysts
APIs
downstream pipelines
```

### Step 2 — Identify pain points

Examples:

* duplicate metrics
* slow queries
* inconsistent definitions
* excessive joins
* incorrect aggregations
* missing history

### Step 3 — Profile actual usage

Find:

```text
most-used tables
most-used columns
most expensive queries
most common joins
```

### Step 4 — Define target model

For example:

```text
Legacy warehouse
       ↓
Canonical model
       ↓
Semantic marts
```

### Step 5 — Migrate incrementally

Don't break everything at once.

Potentially:

```text
v1 legacy
v2 canonical
```

Then migrate dashboards gradually.

### Step 6 — Validate

Run old and new models side by side.

Compare:

```text
revenue
orders
customers
daily totals
monthly totals
```

### Step 7 — Deprecate

Once consumers migrate:

```text
legacy → deprecated → removed
```

### Professional communication

Say:

> "I would treat the existing warehouse as a production dependency, not simply as technical debt. I'd first map consumers and metric dependencies, define a target canonical model, build compatibility layers where necessary, validate old versus new outputs, migrate consumers incrementally, and only then deprecate the legacy structures."

That sounds like someone who has actually worked with production systems.

---

# Final Mental Model: How a Pro Approaches Any Data Modeling Problem

When you receive **any** modeling question, mentally run this sequence:

```text
                 BUSINESS
                    ↓
        What decisions are needed?
                    ↓
          What is the process?
                    ↓
             WHAT IS THE GRAIN?
                    ↓
        ┌───────────┴───────────┐
        ↓                       ↓
      FACTS                 DIMENSIONS
        ↓                       ↓
 What happened?          Who/what/where?
        ↓                       ↓
   Measures             Attributes
                            ↓
                  Does history matter?
                            ↓
                       SCD strategy
                            ↓
                 Star / Snowflake?
                            ↓
              Normalize / Denormalize?
                            ↓
                 Query patterns?
                            ↓
                     Scale / Cost?
                            ↓
                 Data quality rules?
                            ↓
                Consumer experience?
                            ↓
                 BUSINESS OUTPUT
```

And before finalizing the model, ask yourself these **10 professional questions**:

1. **What exactly does one row represent?**
2. **Can any measure be duplicated through a join?**
3. **What happens when a dimension changes?**
4. **Do we need historical values or only current state?**
5. **What happens when data arrives out of order?**
6. **What happens when a dimension arrives after the fact?**
7. **What happens when the source changes or deletes data?**
8. **How will analysts actually query this?**
9. **What happens when the data becomes 100× larger?**
10. **Can I explain this model clearly to a business person, backend engineer, analyst, and platform engineer?**

--------------------
# Author: Md Shaukat Ali
