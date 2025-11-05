# dbt-databricks-scd2-data-quality

This repository showcases a production-style ELT workflow implemented with **dbt Core** on **Databricks SQL Warehouse**, structured around the **Medallion architecture**. Raw operational data lands in **Bronze** with minimal handling, is standardized and conformed in **Silver**, and is shaped into analytics marts and historized dimensions in **Gold** using **SCD Type‑2** snapshots. The project includes **data quality tests** (generic, singular, and custom generic), **seeds** for lookups, and **macros** for reusable logic. It is designed to be recruiter‑friendly and deployment‑ready, with clear configuration and places to add screenshots for lineage, tests, snapshots, and successful builds.


---

## Description

This project demonstrates how to **transform raw data into trustworthy, analytics‑ready models** using dbt on Databricks in a way that mirrors real production systems. Configuration lives in `dbt_project.yml` and `profiles.yml`, with the latter reading credentials from environment variables for safety. The **Bronze** layer materializes raw sources as lightweight views or tables to preserve fidelity and observability. The **Silver** layer introduces business semantics, joining facts and dimensions, deriving fields, and enforcing robust data tests. The **Gold** layer crystallizes analytics marts and applies **SCD2 snapshots** to track attribute changes over time.

Before running, configure a Databricks **SQL Warehouse** and export three environment variables—`DBT_HOST`, `DBT_HTTP_PATH`, and `DBT_TOKEN`—then run `dbt debug`, `dbt deps`, and `dbt build` to execute seeds, models, tests, and snapshots in the correct order. Add a screenshot of your Databricks connection details as `docs/images/databricks_sql_warehouse.png`, and a successful build as `docs/images/dbt_build_success.png`.

```bash
# virtual environment + install (example with uv)
uv venv && . .venv/bin/activate           # Windows: .venv\Scripts\activate
uv pip install --upgrade pip dbt-core dbt-databricks

# Databricks connection (example)
export DBT_HOST="<your-host>"
export DBT_HTTP_PATH="<your-sql-warehouse-http-path>"
export DBT_TOKEN="<your-personal-access-token>"

# verify and run
dbt debug && dbt deps
dbt build       # seeds + run + tests + snapshots
```

---

## Bronze

The **Bronze** layer focuses on **fidelity and traceability**: it brings source data into the lakehouse with minimal transformation, typically as **views** so that the storage footprint remains lean while preserving the raw grain for audits and reprocessing. Sources are declared in a YAML file so lineage is explicit and discoverable in `dbt docs`. Within Bronze models, we avoid business rules and heavy reshaping to ensure downstream teams can always reason back to the original data.

A typical sources file (e.g., `models/sources.yml`) declares tables such as `fact_sales`, `dim_product`, `dim_customer`, `dim_store`, and `dim_date` under a `src` source with catalog and schema aligned to Unity Catalog. A basic Bronze model like `bronze_sales.sql` simply selects from the declared source, keeping column names and types intact wherever possible. Even at Bronze, we enforce **key integrity** using generic tests in a properties file (e.g., `models/bronze/properties.yml`) to ensure primary keys are `unique` and `not_null`. For soft‑domain rules that may fluctuate (e.g., permitted `country` values), we attach `accepted_values` with `severity: warn` so teams are alerted without blocking builds:

```yaml
# models/sources.yml
version: 2

sources:
  - name: src
    catalog: "{{ target.catalog }}"
    schema: "source"
    tables:
      - name: fact_sales
      - name: dim_product
      - name: dim_customer
      - name: dim_store
      - name: dim_date
```

```sql
-- models/bronze/bronze_sales.sql
select
  *
from {{ source('src', 'fact_sales') }}
```

```yaml
# models/bronze/properties.yml
version: 2

models:
  - name: bronze_sales
    config:
      materialized: view
      schema: bronze
    columns:
      - name: sales_id
        tests: [unique, not_null]

  - name: bronze_store
    config:
      materialized: view
      schema: bronze
    columns:
      - name: store_id
        tests: [unique, not_null]
      - name: country
        tests:
          - accepted_values:
              values: ["USA", "Canada"]
              config:
                severity: warn
```

---

## Silver

The **Silver** layer applies **business semantics** by joining facts and dimensions, standardizing data types, deriving metrics, and enforcing stronger data quality guarantees. This is where you typically consolidate entity logic (e.g., product/category, customer attributes, payment methods) into models that analysts and downstream systems can consume directly. Reusable business logic lives in **macros**—for example, a simple `multiply` macro can encapsulate revenue math and keep SQL tidy. When performance or SLA requires it, Silver models can adopt **incremental materialization** by defining a `unique_key` and an `updated_at` (or a filter on a date surrogate key) to process **only new or changed rows** after the first full build. That said, keeping Silver as **tables** rebuilt nightly is often sufficient and operationally simpler for many teams.

Below, `silver_salesinfo.sql` demonstrates how facts from `bronze_sales` join with `bronze_product` and `bronze_customer`, deriving `gross_amount` from unit price and quantity, and selecting a clean set of analytics-ready columns. To keep **quantities and monetary fields sane**, we combine **singular tests** (zero rows when passing) and a **custom generic** test (`generic_non_negative`) attached via YAML to relevant numeric columns.

```sql
-- macros/multiply.sql
{% macro multiply(a, b) %}
  ({{ a }} * {{ b }})
{% endmacro %}
```

```sql
-- models/silver/silver_salesinfo.sql
with s as (
  select
    sales_id,
    product_sk,
    customer_sk,
    {{ multiply('unit_price','quantity') }} as gross_amount,
    discount,
    payment_method,
    date_sk
  from {{ ref('bronze_sales') }}
),
p as (
  select product_sk, category
  from {{ ref('bronze_product') }}
),
c as (
  select customer_sk, gender
  from {{ ref('bronze_customer') }}
)
select
  s.sales_id,
  p.category,
  c.gender,
  s.payment_method,
  s.gross_amount,
  s.discount
from s
join p using (product_sk)
join c using (customer_sk)
```

```sql
-- tests/non_negative_amounts.sql  (singular test)
select *
from {{ ref('silver_salesinfo') }}
where gross_amount < 0
```

```sql
-- tests/generic/generic_non_negative.sql  (custom generic)
{% test generic_non_negative(model, column_name) %}
select *
from {{ model }}
where {{ column_name }} < 0
{% endtest %}
```

```yaml
# models/silver/properties.yml
version: 2

models:
  - name: silver_salesinfo
    columns:
      - name: gross_amount
        tests:
          - generic_non_negative
```

 <img src="https://github.com/pninad9/dbt-databricks-scd2-data-quality/blob/8b4d62a3ee66aef898b45f486a91fc527e19cf0b/screenshot/Silver%20Lin.png" />


---

## Gold

The **Gold** layer delivers **analytics marts** and **historized dimensions** ready for BI, reverse ETL, or ML features. It aggregates Silver models into measures and dimensions aligned with business concepts. Where attributes change over time and the organization needs to answer “**what was true then**?”, we use **dbt snapshots** to implement **SCD Type‑2** patterns.

A common approach is to first define a **“latest” view** of upstream item records (e.g., `source_gold_items.sql`) by partitioning and ranking each `id` by `updated_at`, selecting the most recent version. This “clean latest” feed then becomes the input to the snapshot, which compares successive runs and writes a historized table with `dbt_valid_from` and `dbt_valid_to` for temporal queries. Over time, this produces a tidy, queryable history that BI tools and analysts can use to reconstruct past states of the world.

```sql
-- models/gold/source_gold_items.sql
with ranked as (
  select
    id,
    name,
    category,
    updated_at,
    row_number() over (partition by id order by updated_at desc) as rn
  from {{ source('src', 'items') }}
)
select id, name, category, updated_at
from ranked
where rn = 1
```

```sql
-- snapshots/items_snapshot.sql  (SCD2)
{% snapshot items_snapshot %}
{{
  config(
    target_schema='gold',
    unique_key='id',
    strategy='timestamp',
    updated_at='updated_at',
    invalidate_hard_deletes=True
  )
}}
select id, name, category, updated_at
from {{ ref('source_gold_items') }}
{% endsnapshot %}
```

```bash
dbt snapshot
```

 <img src="https://github.com/pninad9/dbt-databricks-scd2-data-quality/blob/8b4d62a3ee66aef898b45f486a91fc527e19cf0b/screenshot/SCD%202.png" /> — query showing two versions of the same `id` with different `dbt_valid_from` / `dbt_valid_to`.  
<img src="https://github.com/pninad9/dbt-databricks-scd2-data-quality/blob/8b4d62a3ee66aef898b45f486a91fc527e19cf0b/screenshot/DBT%20Bulid.png" — successful `dbt build` including snapshots.

---

## Final Word

This repository is more than a set of SQL files—it is a compact **template for reliable, evolvable analytics**. By separating **Bronze** fidelity, **Silver** semantics, and **Gold** analytics and history, it scales across teams while keeping code modular and testable. Seeds and macros reduce duplication, generic and singular tests prevent regression, and snapshots answer time‑travel questions that businesses ask every week. To extend this further, add **source freshness checks**, publish **dbt docs** for discoverability, and wire a CI workflow (or dbt Cloud job) so every pull request builds and tests before merging. If you use Azure Data Factory or another orchestrator for extract/load, include pipeline JSONs under `adf/` and reference dbt runs in your jobs for end‑to‑end automation.

