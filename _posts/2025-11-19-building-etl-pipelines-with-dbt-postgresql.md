---
title: "Building ETL Pipelines with dbt and PostgreSQL"
date: 2025-11-19
categories: [Blog]
tags: [Data Engineering, ETL, dbt, PostgreSQL, SQL]
---

# Building ETL Pipelines with dbt and PostgreSQL

Data engineering is the backbone of modern data-driven organizations. Whether you're building a business intelligence platform, machine learning pipeline, or analytics dashboard, robust ETL (Extract, Transform, Load) processes are essential. In this post, we'll explore how to build production-ready data pipelines using dbt (data build tool) and PostgreSQL.

## Why dbt + PostgreSQL?

**dbt** has revolutionized data transformation by bringing software engineering best practices to analytics:
- Version control for your transformations
- Testing and documentation
- Modularity and reusability
- Clear dependency management

**PostgreSQL** provides a robust, feature-rich database that's:
- Open source and cost-effective
- Supports advanced SQL features
- Excellent performance for analytical workloads
- Rich ecosystem of extensions

Together, they create a powerful stack for data transformation and analytics.

## Architecture Overview

Our ETL pipeline follows this structure:

```
Raw Data (Sources)
       ↓
   Staging Layer (dbt models)
       ↓
   Intermediate Layer (business logic)
       ↓
   Marts Layer (final analytics tables)
       ↓
   BI Tools / Reports
```

## Setting Up Your Environment

### 1. PostgreSQL Installation

```bash
# Install PostgreSQL (macOS)
brew install postgresql@14

# Start PostgreSQL service
brew services start postgresql@14

# Create your database
createdb analytics_db
```

### 2. dbt Installation and Setup

```bash
# Install dbt with PostgreSQL adapter
pip install dbt-postgres

# Initialize dbt project
dbt init my_analytics_project
cd my_analytics_project
```

### 3. Configure dbt Profile

Edit `~/.dbt/profiles.yml`:

```yaml
my_analytics_project:
  target: dev
  outputs:
    dev:
      type: postgres
      host: localhost
      user: your_username
      password: your_password
      port: 5432
      dbname: analytics_db
      schema: analytics
      threads: 4
      keepalives_idle: 0
```

## Building Your First ETL Pipeline

### Step 1: Define Source Data

Create `models/staging/sources.yml`:

```yaml
version: 2

sources:
  - name: raw_data
    database: analytics_db
    schema: public
    tables:
      - name: raw_users
        description: "Raw user data from application database"
        columns:
          - name: user_id
            description: "Unique user identifier"
            tests:
              - unique
              - not_null
          - name: created_at
            description: "User registration timestamp"
            
      - name: raw_transactions
        description: "Raw transaction events"
        columns:
          - name: transaction_id
            tests:
              - unique
              - not_null
          - name: user_id
            tests:
              - not_null
              - relationships:
                  to: source('raw_data', 'raw_users')
                  field: user_id
```

### Step 2: Create Staging Models

Staging models clean and standardize raw data.

`models/staging/stg_users.sql`:

```sql
{{ config(materialized='view') }}

with source as (
    select * from {{ source('raw_data', 'raw_users') }}
),

cleaned as (
    select
        user_id,
        lower(trim(email)) as email,
        lower(trim(username)) as username,
        created_at,
        updated_at,
        -- Parse JSON if needed
        (metadata::json->>'country')::varchar as country,
        (metadata::json->>'subscription_tier')::varchar as subscription_tier
    from source
    where user_id is not null
)

select * from cleaned
```

`models/staging/stg_transactions.sql`:

```sql
{{ config(materialized='view') }}

with source as (
    select * from {{ source('raw_data', 'raw_transactions') }}
),

cleaned as (
    select
        transaction_id,
        user_id,
        transaction_amount,
        -- Handle currency conversion
        case 
            when currency = 'USD' then transaction_amount
            when currency = 'EUR' then transaction_amount * 1.1
            when currency = 'GBP' then transaction_amount * 1.25
            else transaction_amount
        end as amount_usd,
        transaction_date,
        transaction_type,
        status
    from source
    where status != 'cancelled'
)

select * from cleaned
```

### Step 3: Intermediate Business Logic

`models/intermediate/int_user_transaction_summary.sql`:

```sql
{{ config(materialized='ephemeral') }}

with users as (
    select * from {{ ref('stg_users') }}
),

transactions as (
    select * from {{ ref('stg_transactions') }}
),

user_stats as (
    select
        user_id,
        count(*) as total_transactions,
        sum(amount_usd) as total_spent,
        avg(amount_usd) as avg_transaction_value,
        min(transaction_date) as first_transaction_date,
        max(transaction_date) as last_transaction_date,
        count(distinct date_trunc('month', transaction_date)) as active_months
    from transactions
    group by user_id
)

select
    u.user_id,
    u.email,
    u.username,
    u.country,
    u.subscription_tier,
    u.created_at as user_created_at,
    coalesce(s.total_transactions, 0) as total_transactions,
    coalesce(s.total_spent, 0) as lifetime_value,
    s.avg_transaction_value,
    s.first_transaction_date,
    s.last_transaction_date,
    s.active_months
from users u
left join user_stats s on u.user_id = s.user_id
```

### Step 4: Marts for Analytics

`models/marts/core/fct_user_metrics.sql`:

```sql
{{ config(
    materialized='table',
    indexes=[
      {'columns': ['user_id']},
      {'columns': ['country']},
    ]
) }}

with user_summary as (
    select * from {{ ref('int_user_transaction_summary') }}
),

final as (
    select
        user_id,
        email,
        country,
        subscription_tier,
        user_created_at,
        total_transactions,
        lifetime_value,
        avg_transaction_value,
        
        -- Customer segmentation
        case
            when lifetime_value >= 10000 then 'VIP'
            when lifetime_value >= 1000 then 'High Value'
            when lifetime_value >= 100 then 'Medium Value'
            else 'Low Value'
        end as customer_segment,
        
        -- Engagement metrics
        case
            when active_months >= 12 then 'Very Active'
            when active_months >= 6 then 'Active'
            when active_months >= 3 then 'Moderate'
            else 'Inactive'
        end as engagement_level,
        
        -- Calculate days since last transaction
        current_date - last_transaction_date as days_since_last_transaction,
        
        -- Churn risk indicator
        case
            when current_date - last_transaction_date > 90 then 'High Risk'
            when current_date - last_transaction_date > 30 then 'Medium Risk'
            else 'Low Risk'
        end as churn_risk
        
    from user_summary
)

select * from final
```

### Step 5: Testing Your Models

`models/marts/core/schema.yml`:

```yaml
version: 2

models:
  - name: fct_user_metrics
    description: "User metrics and customer analytics"
    columns:
      - name: user_id
        description: "Unique user identifier"
        tests:
          - unique
          - not_null
      
      - name: lifetime_value
        description: "Total revenue from user"
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
      
      - name: customer_segment
        description: "Customer value segmentation"
        tests:
          - accepted_values:
              values: ['VIP', 'High Value', 'Medium Value', 'Low Value']
```

## Running Your Pipeline

```bash
# Test source data
dbt test --select source:*

# Run staging models
dbt run --select staging.*

# Run all models
dbt run

# Generate documentation
dbt docs generate
dbt docs serve
```

## Advanced Patterns

### Incremental Models

For large datasets, use incremental models:

```sql
{{ config(
    materialized='incremental',
    unique_key='transaction_id'
) }}

select * from {{ source('raw_data', 'raw_transactions') }}

{% if is_incremental() %}
    where transaction_date > (select max(transaction_date) from {{ this }})
{% endif %}
```

### Macros for Reusability

Create `macros/date_spine.sql`:

```sql
{% macro date_spine(start_date, end_date) %}
    select
        date::date as date
    from generate_series(
        '{{ start_date }}'::date,
        '{{ end_date }}'::date,
        '1 day'::interval
    ) as date
{% endmacro %}
```

Use it in models:

```sql
with dates as (
    {{ date_spine('2020-01-01', 'current_date') }}
)
select * from dates
```

## Performance Optimization

### 1. Materialization Strategy
- **Views** for simple transformations
- **Tables** for complex, frequently queried data
- **Incremental** for large datasets
- **Ephemeral** for intermediate steps

### 2. Indexing

```sql
{{ config(
    indexes=[
        {'columns': ['user_id'], 'type': 'btree'},
        {'columns': ['created_at'], 'type': 'brin'},
    ]
) }}
```

### 3. Partitioning

For time-series data:

```sql
{{ config(
    materialized='table',
    partition_by={
        'field': 'transaction_date',
        'data_type': 'date',
        'granularity': 'month'
    }
) }}
```

## Monitoring and Alerting

### dbt Tests as Data Quality Checks

```yaml
# Custom test for data freshness
tests:
  - dbt_utils.recency:
      datepart: day
      field: updated_at
      interval: 1
```

### Integration with Airflow

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG('dbt_pipeline', 
         start_date=datetime(2025, 1, 1),
         schedule_interval='@daily') as dag:
    
    dbt_run = BashOperator(
        task_id='dbt_run',
        bash_command='cd /path/to/dbt && dbt run'
    )
    
    dbt_test = BashOperator(
        task_id='dbt_test',
        bash_command='cd /path/to/dbt && dbt test'
    )
    
    dbt_run >> dbt_test
```

## Best Practices

1. **Layer your models** - Staging → Intermediate → Marts
2. **Test everything** - Data quality is critical
3. **Document as you go** - Future you will thank you
4. **Use version control** - Treat SQL like code
5. **Monitor performance** - Track query times and costs
6. **Implement CI/CD** - Automate testing and deployment

## Conclusion

Building ETL pipelines with dbt and PostgreSQL provides a modern, maintainable approach to data transformation. The combination of SQL-based transformations, built-in testing, and version control makes it an excellent choice for data teams of any size.

Start simple, test thoroughly, and iterate. Your data pipeline will evolve with your business needs, and dbt's modular approach makes that evolution manageable.

## Resources

- [dbt Documentation](https://docs.getdbt.com/)
- [PostgreSQL Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)
- [dbt Utils Package](https://github.com/dbt-labs/dbt-utils)
- [Analytics Engineering Guide](https://www.getdbt.com/analytics-engineering/)

---

*Building data pipelines? Let's discuss best practices and challenges!*
