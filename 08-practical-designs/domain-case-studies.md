# Domain Case Studies

These four designs apply the same dimensional principles to different business realities. Each section is usable on its own, but the comparison is instructive: e-commerce emphasizes header/line grain, SaaS separates behavior from contracted state, HR makes historical context unavoidable, and banking exposes semi-additivity, bridges, and currency semantics.

## At a glance

| Domain | Atomic event | State snapshot | Historical dimension | Characteristic risk |
|---|---|---|---|---|
| E-commerce | Order line, shipment line, payment, return line | Open-order or inventory snapshot | Customer, product | Multiplying headers, lines, shipments, and payments |
| SaaS | Product event, invoice line, payment | Subscription/account month | Account, plan, user | Treating activity, entitlement, and revenue as one grain |
| HR | Employee movement, payroll line | Employee-assignment month | Employee, position, organization | Rewriting history with today's department |
| Banking | Account transaction | Account-day/month balance | Account, customer, product | Summing balances over time or double counting joint accounts |

## E-commerce: orders, shipments, payments, and returns

### The modeling problem

An online order looks like one object in the application, but analytics sees several processes:

- an order contains product lines;
- lines may ship separately from different locations;
- one payment may cover several lines or several payments may cover one order;
- discounts, tax, and freight can exist at header or line grain;
- returns occur after sale and may refer to only part of a line.

Putting all of this into one table produces row multiplication and ambiguous amounts.

### Conformed dimensions

Use shared Date, Customer, Product, Channel, Promotion, Currency, Geography, Fulfillment Location, and Order Status definitions where applicable. Date plays multiple roles: order, promised, shipped, delivered, paid, and returned.

Customer and Product often need Type 2 history. A sold order line should retain the customer segment and product attributes valid at order time when historical-as-was reporting is required.

### Order-line transaction fact

#### Grain

> **One row represents one product line on one accepted order.**

```text
fact_order_line
  order_date_key            FK
  customer_key              FK
  product_key               FK
  channel_key               FK
  promotion_key             FK
  transaction_currency_key  FK
  order_number              DD
  order_line_number         DD
  ordered_quantity               -- additive
  gross_line_amount              -- additive
  line_discount_amount           -- additive
  allocated_order_discount       -- additive
  allocated_freight_amount       -- additive
  tax_amount                     -- additive under stated tax semantics
  net_line_amount                -- additive
  line_count                     -- additive, constant 1
```

Header descriptive context is copied as foreign keys onto the line fact. `order_number` remains in the fact as a degenerate dimension for grouping and drill-through.

Header charges must not be repeated unchanged on every line. Allocate them with an approved driver such as gross value, quantity, weight, or volume, and reconcile allocated values to the header total. Retain the original header value in a reconciliation structure when required.

### Shipment-line transaction fact

#### Grain

> **One row represents one product quantity from one order line shipped in one shipment event.**

```text
fact_shipment_line
  shipped_date_key       FK
  promised_date_key      FK
  delivered_date_key     FK
  customer_key           FK
  product_key            FK
  fulfillment_location_key FK
  carrier_key            FK
  order_number           DD
  order_line_number      DD
  shipment_number        DD
  shipped_quantity            -- additive
  shipping_cost_amount        -- additive
  shipped_weight_kg           -- additive
  promised_to_ship_days       -- additive; normally averaged
  ship_to_delivery_days       -- additive; normally averaged
  shipment_line_count         -- additive
```

One order line may produce several shipment rows. That is expected, not a duplicate. Order and shipment facts can be compared only after each is aggregated to compatible conformed dimensions.

### Payment transaction fact

#### Grain

> **One row represents one posted payment, refund, chargeback, or payment adjustment.**

```text
fact_payment
  posting_date_key           FK
  customer_key               FK
  payment_method_key         FK
  transaction_currency_key   FK
  rate_type_key              FK
  order_number               DD
  payment_transaction_id     DD
  payment_event_type_key     FK
  amount_transaction_currency -- additive
  amount_group_currency       -- additive
  payment_event_count         -- additive
```

The payment fact is normally order-level, not line-level. Do not join it directly to order lines and sum the payment amount. For product profitability, use a governed allocation or aggregate order lines and payments separately at order grain before combining.

### Return-line transaction fact

#### Grain

> **One row represents one returned product quantity from one original order line in one return event.**

Keep return reason, disposition, product condition, and return channel as dimensions. Negative sales rows can work for simple reversals, but a distinct return process is more expressive when operations need receipt, inspection, refund, and restocking context.

### Safe cross-process comparison

```sql
with ordered as (
  select order_number, product_key, sum(ordered_quantity) as ordered_qty
  from fact_order_line
  group by order_number, product_key
),
shipped as (
  select order_number, product_key, sum(shipped_quantity) as shipped_qty
  from fact_shipment_line
  group by order_number, product_key
)
select
  coalesce(o.order_number, s.order_number) as order_number,
  coalesce(o.product_key, s.product_key) as product_key,
  o.ordered_qty,
  s.shipped_qty
from ordered o
full outer join shipped s
  on s.order_number = o.order_number
 and s.product_key = o.product_key;
```

Both facts are reduced to the same row headers before combination. Joining their raw rows would multiply split shipments.

### Tradeoffs

- Allocated line economics enable product/customer profitability but add policy and reconciliation work.
- Separate process facts create more models but preserve truthful grains.
- An accumulating order-line fulfillment snapshot can simplify current pipeline analysis, while transaction facts remain the audit trail.
- A consolidated order-profit fact is useful only after revenue, discounts, fulfillment cost, payment fees, and returns share a defensible line-level allocation.

### Red flags

- One row per order containing repeated product columns.
- Full order freight copied to every line.
- Payments joined directly to lines and summed.
- Shipment rows called duplicate order lines.
- Current product category used to rewrite old sales without an explicit current-state perspective.
- `DISTINCT` added to hide line/shipment multiplication.

## SaaS: behavior, subscriptions, and recurring revenue

### The modeling problem

SaaS analytics combines at least three distinct meanings:

- **behavior:** users performed product events;
- **entitlement/state:** an account or subscription had a plan, seats, and status;
- **financial activity:** invoices, credits, and payments were posted.

Daily active users, licensed seats, monthly recurring revenue, and cash collected do not share a natural grain.

### Core dimensions

Conform Date, Time, Account, User, Product/Feature, Plan, Subscription Status, Geography, Acquisition Channel, Currency, and Sales Segment.

`dim_account` may be Type 2 for segment, region, owner, or customer-success tier. `dim_plan` should distinguish commercial plan versions when entitlement or pricing changes materially. Current account reporting is an alternate perspective, not a replacement for event-time history.

### Product event fact

#### Grain

> **One row represents one user performing one governed product event at one event timestamp.**

```text
fact_product_event
  event_date_key     FK
  event_time_key     FK
  account_key        FK
  user_key           FK
  feature_key        FK
  event_type_key     FK
  device_key         FK
  event_id           DD
  event_count             -- additive
  usage_quantity          -- additive only when the event definition permits
  duration_seconds        -- additive with overlap/session rules
```

Event names need a governed contract. A client click, server-side accepted action, and completed workflow must not all be labeled `feature_used`.

### Subscription monthly snapshot

#### Grain

> **One row represents one subscription at the close of one billing month.**

```text
fact_subscription_monthly_snapshot
  month_end_date_key   FK
  account_key          FK
  subscription_key     FK
  plan_key             FK
  subscription_status_key FK
  transaction_currency_key FK
  active_subscription_count -- additive within one month
  licensed_seats            -- semi-additive across time
  active_seats              -- semi-additive across time
  mrr_transaction_currency  -- semi-additive across time
  mrr_group_currency        -- semi-additive across time
  discount_mrr              -- semi-additive across time
```

MRR is state at a month boundary. Sum it across accounts for one month; do not sum twelve month-end MRR values and call the result annual recurring revenue. ARR may be a governed transformation of current MRR, while recognized revenue belongs to a financial process.

### Subscription lifecycle snapshot

#### Grain

> **One row represents one subscription lifecycle from initial activation to terminal cancellation.**

Possible milestones include trial start, activation, first payment, upgrade, downgrade, cancellation requested, and ended. An accumulating snapshot is useful only for a small stable set of milestones. Repeated upgrades and pauses remain transaction events or state versions.

### Invoice-line fact

#### Grain

> **One row represents one charge or credit line on one issued invoice.**

```text
fact_invoice_line
  invoice_date_key     FK
  service_period_start_date_key FK
  service_period_end_date_key   FK
  account_key          FK
  subscription_key     FK
  plan_key             FK
  charge_type_key      FK
  currency_key         FK
  invoice_number       DD
  invoice_line_number  DD
  billed_amount             -- additive
  tax_amount                -- additive
  credited_amount           -- additive
  invoice_line_count        -- additive
```

Invoice, payment, recognized revenue, and MRR are related but not interchangeable. Keep their processes distinct and conform their account, subscription, currency, and date definitions.

### Useful metrics and their grains

| Metric | Source | Aggregation caution |
|---|---|---|
| Daily active users | Product event | Distinct users must be recomputed for each period |
| Feature adoption rate | Events plus eligible account/subscription state | Numerator and denominator need aligned population and period |
| Seat utilization | Subscription snapshot | `SUM(active_seats) / SUM(licensed_seats)`, not average row percentages |
| Month-end MRR | Subscription snapshot | Filter one month; semi-additive across time |
| Churned subscriptions | Subscription events/lifecycle | Define logo, subscription, gross, and net churn separately |
| Cash collected | Payment fact | Payment date and currency policy differ from revenue date |

### Tradeoffs

- Event data gives behavioral depth but needs identity, bot, session, and schema governance.
- Monthly snapshots make recurring state simple but cannot explain every intra-month change.
- A daily subscription snapshot gives more precision at much greater volume; timespan state is another option when changes are sparse.
- Keeping financial and product facts separate requires drill-across but prevents false causal joins.

### Red flags

- A universal `saas_fact` mixing clicks, subscriptions, invoices, and payments.
- Summing MRR over months.
- Defining an active account solely as one with any event, without an eligibility or bot policy.
- Joining raw events to subscription snapshots and multiplying snapshot measures by event count.
- Using the current plan to classify historical usage without saying so.
- Calling invoice value “revenue” or payment value “MRR.”

## HR: employee history and headcount

### The modeling problem

HR questions mix current profiles, historical assignments, organizational hierarchies, employee movements, payroll, and periodic headcount. Today's employee row cannot correctly answer where someone worked last year.

### Employee dimension with Type 2 history

```text
dim_employee
  employee_key          PK  -- one historical version
  durable_employee_key      -- one person/employment identity across versions
  source_employee_number
  employee_status
  organization_business_id
  department_name
  position_name
  manager_durable_employee_key
  location_name
  employment_type
  valid_from_timestamp
  valid_to_timestamp
  is_current
```

#### Dimension grain

> **One row represents one version of one employee profile valid during one non-overlapping interval.**

Name corrections may be Type 1; department, position, manager, location, employment type, and status are common Type 2 candidates when their history has analytical value. The choice is attribute-specific and owned by HR governance.

If an employee number can be reused after rehire, it is not a safe durable identity. Define whether rehire continues the same durable employee or creates a new employment relationship.

### Headcount monthly snapshot

#### Grain

> **One row represents one employee assignment active at the close of one calendar month.**

```text
fact_employee_assignment_monthly_snapshot
  month_end_date_key  FK
  employee_key        FK
  organization_key    FK
  department_key      FK
  position_key        FK
  manager_key         FK
  location_key        FK
  employment_status_key FK
  assignment_id       DD
  assignment_count         -- additive within one month
  headcount                 -- additive within one month; normally 1 or allocation
  full_time_equivalent      -- additive within one month
  base_salary_amount        -- semi-additive across time; access controlled
  tenure_days               -- non-additive
```

Use assignment grain, not employee grain, when simultaneous assignments are legitimate. If the business wants each person to total one across assignments, add governed allocation weights or maintain a separate person-level snapshot.

Headcount is not additive across month ends. Twelve monthly rows for one employee are not twelve employees.

### Employee movement fact

#### Grain

> **One row represents one effective employee-assignment movement event.**

Movement types include hire, rehire, transfer, promotion, leave, return, termination, and manager change. This transaction fact preserves the event trail that a monthly snapshot compresses.

```text
fact_employee_movement
  effective_date_key     FK
  employee_key           FK
  from_organization_key  FK
  to_organization_key    FK
  from_position_key      FK
  to_position_key        FK
  movement_type_key      FK
  movement_id            DD
  movement_count              -- additive
  compensation_change_amount  -- additive only under governed currency/time semantics
```

### Payroll line fact

#### Grain

> **One row represents one pay element for one employee assignment in one payroll result.**

Gross pay, tax, benefit, bonus, and deduction lines should be classified by a pay-element dimension. Currency and pay-period roles are explicit. Restrict sensitive access independently of model usability.

### Organization hierarchy

A stable fixed-depth hierarchy can be flattened into named attributes such as Team, Department, Division, and Company. A variable-depth reporting structure may require a hierarchy bridge. The manager relationship can be recursive, but historical manager analysis must respect effective time.

### Late corrections

HR changes are often retroactive. A transfer received in April may be effective in March. The process must:

1. correct non-overlapping employee Type 2 intervals;
2. resolve movements and snapshots to the corrected versions;
3. rebuild affected month-end snapshots and aggregates;
4. disclose restatement under the organization's reporting policy.

Processing date and effective date answer different questions. Preserve both when operational latency matters.

### Tradeoffs

- Type 2 history gives correct as-was reporting but produces several rows per employee and requires durable-key counts.
- Monthly snapshots simplify headcount trends but repeat long-lived assignments.
- Recursive and matrix organizations may need bridges and allocation policies.
- Privacy, row-level security, and purpose limitation are architectural requirements, not optional reporting filters.

### Red flags

- Employee number used as both durable identity and version key.
- Current department joined to old payroll or learning facts by natural key.
- `COUNT(*)` on a Type 2 employee dimension presented as employee count.
- Headcount summed across months.
- Multiple concurrent assignments counted as several people without disclosure.
- Generic hierarchy levels with no business names.
- Retroactive changes applied to the dimension but not to affected snapshots.

## Banking: transactions, balances, joint ownership, and currencies

### The modeling problem

Banking combines high-volume postings, end-of-period balances, changing account/customer relationships, heterogeneous products, and strict currency and audit rules. It is an ideal demonstration of why event and state models coexist.

### Dimensions

Conform Date, Time, Account, Customer, Product, Branch, Channel, Transaction Type, Currency, Rate Type, Geography, and Risk Segment.

`dim_account` holds descriptive account context and may use Type 2 for product, branch, risk class, or status when history matters. Natural account numbers can change or be reused; use warehouse-controlled keys. Heterogeneous mortgage, card, deposit, and loan attributes may require common supertype plus subtype dimensions rather than one extremely sparse account table.

### Account transaction fact

#### Grain

> **One row represents one posted accounting transaction entry on one account.**

```text
fact_account_transaction
  posting_date_key          FK
  value_date_key            FK
  posting_time_key          FK
  account_key               FK
  product_key               FK
  branch_key                FK
  channel_key               FK
  transaction_type_key      FK
  transaction_currency_key  FK
  rate_type_key             FK
  transaction_id            DD
  transaction_count              -- additive
  debit_amount_local             -- additive
  credit_amount_local            -- additive
  net_amount_local               -- additive
  net_amount_group_currency      -- additive
  fee_amount_local               -- additive
  exchange_rate_applied          -- non-additive descriptive measurement
```

Posting date and value date are different roles. Reversals should be explicit transactions or governed restatements; deleting a posting obscures the accounting trail.

### Daily account balance snapshot

#### Grain

> **One row represents one account at the close of one banking day.**

```text
fact_account_daily_snapshot
  snapshot_date_key       FK
  account_key             FK
  product_key             FK
  branch_key              FK
  account_currency_key    FK
  account_status_key      FK
  ending_balance_local         -- semi-additive across time
  ending_balance_group         -- semi-additive across time
  available_balance_local      -- semi-additive across time
  average_daily_balance_local  -- non-additive across periods
  debit_amount_during_day      -- additive
  credit_amount_during_day     -- additive
  transaction_count_during_day -- additive
  account_snapshot_count       -- additive within one date
```

Balance and flow measures intentionally coexist, but their names expose different time semantics. Summing debit amounts across days is valid; summing closing balance across days is not.

### Account-customer bridge

Joint accounts create a genuine multivalued customer relationship.

```text
bridge_account_customer
  account_key
  customer_key
  valid_from_date_key
  valid_to_date_key
  ownership_role_key
  allocation_weight
```

#### Bridge grain

> **One row represents one customer's relationship to one account during one validity interval.**

For allocated reporting, weights for an account and date sum to `1.0`. A two-holder joint account might allocate 0.5 to each, though the business may define legal ownership differently. Without weights, associating the full balance with every holder is an impact report and will overstate portfolio totals.

At card-transaction grain, the initiating cardholder may be single-valued and can be a normal fact foreign key. The bridge is still needed for account-level balances. Pattern choice follows the fact grain.

### Currency policy

Store both transaction/account currency and a governed standard currency when group reporting needs reproducible comparison. Record the rate type and event-date conversion used. Closing balances may use closing rates while flows use transaction or average rates; these must not share an unlabeled `converted_amount`.

### Example: point-in-time portfolio balance

```sql
select
    d.full_date,
    p.product_family,
    sum(f.ending_balance_group) as portfolio_balance_group
from fact_account_daily_snapshot f
join dim_date d    on d.date_key = f.snapshot_date_key
join dim_product p on p.product_key = f.product_key
where d.full_date = date '2026-09-30'
group by d.full_date, p.product_family;
```

The one-date filter is essential. A trend query returns one balance per date rather than summing dates together.

### Tradeoffs

- Transactions preserve movements; snapshots make balance trends simple. Maintaining both costs storage and reconciliation effort.
- Type 2 account and customer dimensions preserve historical context but make bridge maintenance time-sensitive.
- Weighted ownership preserves totals but may imply an allocation that differs from legal or behavioral impact.
- Standard currency improves comparability but does not eliminate local-currency audit needs.

### Red flags

- Closing balances summed across dates.
- Current account product used for all historical transactions.
- Joint account balance copied fully to each holder and then totaled.
- Transaction and balance rows mixed in one fact.
- Exchange rates looked up at query time without rate-date/type governance.
- Account natural number used as the warehouse primary key across source migrations.
- Fact tables joined directly on account and date, multiplying transactions by snapshot rows.

## Cross-domain lessons

### Similar nouns do not imply similar grains

“Customer,” “account,” “user,” and “employee” can appear in many fact tables. The surrounding process still determines the row:

- one order line;
- one product event;
- one employee assignment-month;
- one account-day.

Conformed dimensions integrate those rows; they do not erase grain differences.

### Events and state answer different questions

Events explain change and support audit. Snapshots make state at a boundary easy to query. Accumulating snapshots expose current lifecycle progress. Most real domains need more than one.

### Historical and current perspectives both need names

A fact foreign key to a Type 2 version gives historical-as-was context. A durable-key/current-row view gives today's classification. Neither is universally correct; expose both deliberately.

### Ratios need populations

Adoption, conversion, compliance, utilization, margin percentage, and pass rate require explicit numerators and denominators at compatible grains. Store additive components and govern the population.

### Many-to-many relationships require an analytical policy

Joint owners, employee assignments, product promotions, and course skills may need bridges. Decide whether a report allocates to preserve totals or measures impact and permits overcounting.

## Domain design review

- [ ] Every fact has an explicit one-row statement.
- [ ] Header, line, event, payment, and snapshot grains are separate.
- [ ] Conformed dimensions use consistent meanings across processes.
- [ ] Historical facts use the dimensional version effective at event time.
- [ ] Balances, MRR, headcount, and other states are not summed across time.
- [ ] Ratios retain compatible additive components.
- [ ] Bridges declare their grain, time validity, and weighting policy.
- [ ] Currency values name original/standard basis, rate type, and rate date.
- [ ] Late events and retroactive changes trigger defined restatement behavior.
- [ ] Cross-process analysis aggregates facts separately before combining them.

## Related chapters

- [Grain](../01-foundations/grain.md)
- [Dimensional modeling](../01-foundations/dimensional-modeling.md)
- [Keys](../01-foundations/keys.md)
- [Measures and additivity](../01-foundations/measures.md)
- [Fact table patterns](../02-fact-tables/fact-table-patterns.md)
- [Advanced fact designs](../02-fact-tables/advanced-fact-designs.md)
- [Slowly changing dimensions](../03-dimensions/slowly-changing-dimensions.md)
- [Bridges and many-to-many relationships](../04-relationships/bridges-and-many-to-many.md)
- [Conformance and bus architecture](../05-enterprise-modeling/conformance-and-bus-architecture.md)
- [Late-arriving data](../06-time-and-change/late-arriving-data.md)

## What to remember

1. E-commerce requires separate line, shipment, payment, and return grains; allocation is a governed choice.
2. SaaS behavior, subscription state, and financial activity are different processes with different time semantics.
3. HR needs durable employee identity, Type 2 context, and non-additive headcount snapshots.
4. Banking transactions are flows; balances are semi-additive state; joint ownership requires a bridge policy.
5. Across every domain, facts combine safely only after aggregation to conformed dimensions at a shared grain.
6. Current-state convenience must not silently replace historical correctness.

## Source basis

These designs synthesize Kimball and Ross, *The Data Warehouse Toolkit*, 3rd ed.: order management in Chapter 6, HR in Chapter 9, financial services in Chapter 10, electronic commerce in Chapter 15, insurance in Chapter 16, and the cross-cutting techniques catalog in Chapter 2. SaaS terminology and current platform-oriented implementation choices are later adaptations, not concepts attributed verbatim to the book. See [Sources](../SOURCES.md).
