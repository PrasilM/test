# SQL Query Refactoring Analysis

## Project Overview

This documentation analyzes the refactoring of SQL queries from the existing files `third.sql` and `bkgs.sql` into a new consolidated query in `new_code.sql`. The primary objective was to create a new query that produces the same output as `bkgs.sql` while leveraging the structure and data sources from `third.sql`.

## Files Analyzed

1. **bkgs.sql**: Original query that produces the desired output
2. **third.sql**: Source of additional data structure and table relationships
3. **new_code.sql**: The refactored query combining elements from both sources

## Changes and Transformations

### Source Table Changes
- **Original**: `bkgs.sql` primarily queried from `non_published_analytics.apla_planning.bookings_aggregate`
- **New**: `new_code.sql` uses `published_domain.apla_sales_order.so_integrated_latest_v` as the main source

### Column Name Mapping
The refactoring involved mapping column names between the different schemas:

| bkgs.sql (Original) | new_code.sql (New) |
|---------------------|-------------------|
| book.product_code | sobill.product_cd |
| book.business_season_year_code | sobill.business_season_cd |
| book.month_per_code | sobill.month_period_cd |
| book.customer_owner_group_code | sobill.owner_group_nbr |
| book.customer_sold_to_code | sobill.sold_to_customer_cd |
| book.active_authorized_futures_contracts_amount_usd | sobill.aaf_contracts_amt_usd |
| book.gross_futures_units | sobill.gross_futures_qty |

### Added Joins
The new query adds two important left joins that weren't present in the original:

1. **Price Data Join**:
   ```sql
   left join (
       select distinct prc.product_cd as product_cd,
                       prc.business_season_cd as business_season_cd,
                       prc.country_nm as country_nm,
                       max(prc.regional_msrp_amt) over(...) as country_retail_price_local,
                       max(prc.regional_msrp_usd_amt) over(...) as country_retail_price_usd,
                       max(prc.regional_wholesale_amt) over(...) as country_wholesale_price_local,
                       max(prc.regional_wholesale_usd_price_amt) over(...) as country_wholesale_price_usd
       from published_domain.apla_sales_order.so_integrated_latest_v prc
   ) price
   ```

2. **MSRP Data Join**:
   ```sql
   left join (
       select distinct apla_msrp.product_code as product_code,
                       apla_msrp.season_year as season_year,
                       apla_msrp.region_name as region_name,
                       apla_msrp.price_effective_begin_date as price_effective_begin_date,
                       apla_msrp.price_effective_end_date as price_effective_end_date,
                       apla_msrp.msrp_usd_ex_tax as msrp_usd_ex_tax
       from published_domain.apla_product.product_price_season_flat_v apla_msrp
   ) msrp
   ```

### Filter Condition Changes
The WHERE clause was modified to reference the new table structure:

**Original**:
```sql
where book.season_sort_sequence_number >= (select max(season_sort_sequence_number) - 20 from non_published_analytics.apla_planning.bookings_aggregate)
and book.season_sort_sequence_number <= (select max(season_sort_sequence_number) - 1 from non_published_analytics.apla_planning.bookings_aggregate)
```

**New**:
```sql
where sobill.season_sort_sequence_nbr >= (select max(season_sort_sequence_nbr) - 20 from published_domain.apla_sales_order.so_integrated_latest_v)
and sobill.season_sort_sequence_nbr <= (select max(season_sort_sequence_nbr) - 1 from published_domain.apla_sales_order.so_integrated_latest_v)
```

### Calculation Changes
The calculation for `net_ship_units` was adapted to use the new column names:

**Original**:
```sql
sum(cast((book.gross_futures_units - book.nike_cancels_units - book.customer_cancels_units + book.at_once_units) as decimal(38,6))) as net_ship_units
```

**New**:
```sql
sum(cast((sobill.gross_futures_qty - sobill.nike_reject_cancel_qty - sobill.customer_reject_qty + sobill.at_once_qty) as decimal(38,6))) as net_ship_units
```

## Key Technical Decisions

1. **Direct Source Data**: The refactoring moves from using an aggregated analytics table to directly accessing the source data tables, which can provide more up-to-date information.

2. **Column Naming Convention**: The new query maintains the output column names from `bkgs.sql` while sourcing the data from differently named fields in the source tables.

3. **Additional Data Enrichment**: The new query incorporates additional price data through the joins that may enhance the dataset beyond what was available in the original query.

4. **Data Type Consistency**: All numeric columns maintain the same precision and scale in their casts to ensure data consistency.

## Expected Benefits

1. **Data Freshness**: By sourcing from `published_domain` tables directly, the data should be more current than the aggregated tables.

2. **Maintainability**: Consolidating logic from multiple queries into a single, well-structured query should improve long-term maintainability.

3. **Performance**: The optimization of joins and the use of window functions for price data may improve query performance.

## Validation Requirements

To ensure the refactored query produces identical results to the original:

1. Run both queries against the same time period data
2. Compare row counts to ensure completeness
3. Compare key metrics (totals for units and amounts) to ensure accuracy
4. Perform detailed row-by-row comparison for a sample of data

## Next Steps

1. Performance testing of the new query
2. Validation of all calculated metrics against the original query
3. Implementation in production systems
4. Documentation update in data dictionaries and query repositories
