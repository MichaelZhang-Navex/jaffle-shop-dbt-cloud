# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

The Jaffle Shop is a dbt Cloud project modeling data for a fictional restaurant. It demonstrates dbt Cloud features including staging models, marts, semantic models, metrics, and MetricFlow integrations. The project is designed for learning dbt Cloud development patterns and contains synthetic e-commerce data.

## Common Commands

### Development
```bash
# Install dbt packages (run first after cloning)
dbt deps

# Build all models and run tests
dbt build

# Build specific model
dbt build --select model_name

# Run models only
dbt run

# Run tests only
dbt test
```

### Loading Source Data
```bash
# Load sample data using seeds (1 year of data)
dbt seed --full-refresh --vars '{"load_source_data": true}'
```

### Working with Larger Datasets
If you need to generate larger datasets locally (requires Python and dbt Core):

```bash
# Using task runner (if installed)
task load YEARS=6 DB=bigquery

# Or manually
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
pip install dbt-core dbt-[warehouse]
jafgen 6
rm -rf seeds/jaffle-data
mv jaffle-data seeds
dbt seed --full-refresh --vars '{"load_source_data": true}'
```

## Architecture

### Data Flow
1. **Sources** (`models/staging/__sources.yml`): Raw tables in the `raw` schema
   - `raw_customers`, `raw_orders`, `raw_items`, `raw_stores`, `raw_products`, `raw_supplies`

2. **Staging Models** (`models/staging/`): Light transformations on raw data
   - Views that rename columns and apply basic type conversions
   - One staging model per source table (e.g., `stg_orders`, `stg_customers`)

3. **Marts Models** (`models/marts/`): Business logic and final analytics tables
   - Tables materialized for downstream consumption
   - Join staging models and add derived metrics
   - Key marts: `customers`, `orders`, `products`, `supplies`

### Key Architectural Patterns

**Staging Layer Pattern**: All staging models follow the same CTE structure:
```sql
with source as (
    select * from {{ source('ecom', 'raw_table') }}
),
renamed as (
    -- column renaming and transformations
)
select * from renamed
```

**Marts Layer Pattern**: Marts use multiple CTEs to build complex business logic:
- Import staging models via `ref()`
- Create intermediate aggregations
- Join together with final business logic
- Example: `customers.sql` joins `stg_customers` with order summaries

**Cross-Database Compatibility**: Uses adapter dispatch pattern in macros (e.g., `cents_to_dollars`) to support multiple warehouses (Snowflake, BigQuery, Postgres, Fabric).

### Semantic Layer

This project includes MetricFlow semantic models and metrics defined in YAML files:
- **Semantic Models**: Define entities, dimensions, and measures on top of marts (see `models/marts/orders.yml`)
- **Metrics**: Business metrics like `order_total`, `new_customer_orders`, `food_orders`
- **Saved Queries**: Pre-defined metric queries for exports

### Custom Macros

- **`cents_to_dollars`** (`macros/cents_to_dollars.sql`): Converts cent amounts to dollars with warehouse-specific implementations
- **`generate_schema_name`** (`macros/generate_schema_name.sql`): Custom schema naming logic

## Testing Strategy

The project uses multiple testing approaches:

1. **Schema Tests**: Generic tests defined in `.yml` files (uniqueness, not_null, relationships)
2. **Data Tests**: Custom SQL tests in `data-tests/` directory
3. **Unit Tests**: Defined inline in YAML (see `orders.yml` for example)
4. **dbt_utils Tests**: Expression validations (e.g., `expression_is_true` for order total calculations)

## Configuration

### dbt_project.yml
- **Project**: `jaffle_shop`
- **Profile**: `default` (configured in dbt Cloud)
- **Materialization**: Staging = views, Marts = tables
- **Seeds**: `raw` schema, controlled by `load_source_data` variable
- **Timezone**: America/Los_Angeles (for dbt_date package)

### Dependencies
Defined in `packages.yml`:
- `dbt-labs/dbt_utils` (v1.1.1)
- `godatadriven/dbt_date` (v0.10.0)
- `dbt-labs/dbt-audit-helper` (main branch)

### Code Quality
SQLFluff configuration in `.sqlfluff`:
- Templater: dbt-cloud
- Lowercase keywords, identifiers, and functions
- Trailing commas
- 4-space indentation
- Max line length: 80 characters

## Environment Configuration

This project is designed for dbt Cloud but can run locally with dbt Cloud CLI:
- Development in IDE or Cloud CLI
- Production Environment builds to `prod` schema on `main` branch
- Optional Staging Environment for pre-production testing on `staging` branch
