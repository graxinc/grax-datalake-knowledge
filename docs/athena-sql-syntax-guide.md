# Athena SQL Syntax Guide

## Overview

Amazon Athena requires specific SQL syntax patterns that differ from standard SQL databases. This guide provides the correct syntax for common operations and helps prevent common errors.

## Critical Syntax Corrections

### Date Functions

#### Date Difference Functions

❌ **INCORRECT (Standard SQL):**

```sql
DATEDIFF(DAY, start_date, end_date)  -- This will FAIL in Athena
```

✅ **CORRECT (Athena Syntax):**

```sql
DATE_DIFF('day', start_date, end_date)  -- Note the underscore and quoted interval
```

#### Key Differences

- Use `DATE_DIFF` (with underscore) instead of `DATEDIFF`
- Interval must be quoted: `'day'`, `'month'`, `'year'`
- Function signature: `DATE_DIFF('interval', start_date, end_date)`

### Supported Date Intervals

```sql
-- All intervals must be quoted strings
DATE_DIFF('day', date1, date2)
DATE_DIFF('week', date1, date2) 
DATE_DIFF('month', date1, date2)
DATE_DIFF('quarter', date1, date2)
DATE_DIFF('year', date1, date2)
DATE_DIFF('hour', datetime1, datetime2)
DATE_DIFF('minute', datetime1, datetime2)
DATE_DIFF('second', datetime1, datetime2)
```

### Current Timestamp Function

✅ **CORRECT Options:**

```sql
CURRENT_TIMESTAMP  -- Preferred
NOW()              -- Also works
CURRENT_DATE       -- For date-only comparisons
```

### Date Arithmetic Functions

#### Date Addition

```sql
-- Add time to date
DATE_ADD('day', 30, CURRENT_DATE)      -- Add 30 days
DATE_ADD('month', -6, CURRENT_DATE)    -- Subtract 6 months
DATE_ADD('year', 1, some_date)         -- Add 1 year
```

#### Date Truncation

```sql
-- Truncate to specific period
DATE_TRUNC('month', some_timestamp)    -- First day of month
DATE_TRUNC('week', some_timestamp)     -- Beginning of week
DATE_TRUNC('day', some_timestamp)      -- Midnight of day
```

## CTE Column Scope and Availability Rules

### The Problem

Athena throws "Column cannot be resolved" errors when attempting to reference columns that are out of scope or unavailable in the current query context.

### Critical Scope Rules

#### Rule 1: CTE Column Boundaries

❌ **WILL FAIL - Column Out of Scope:**

```sql
WITH base_data AS (
    SELECT id, createddate_ts, amount_f 
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
),
aggregated_data AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as report_month,
        COUNT(*) as opp_count,
        SUM(amount_f) as total_amount
    FROM base_data
    GROUP BY DATE_TRUNC('month', createddate_ts)
)
SELECT 
    report_month,
    opp_count,
    total_amount,
    AVG(createddate_ts)  -- ERROR: createddate_ts not available after GROUP BY
FROM aggregated_data
```

✅ **CORRECT - Calculate Before Aggregation:**

```sql
WITH base_data AS (
    SELECT 
        id, 
        createddate_ts, 
        amount_f,
        -- Calculate derived fields BEFORE aggregation
        DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) as age_days
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
),
aggregated_data AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as report_month,
        COUNT(*) as opp_count,
        SUM(amount_f) as total_amount,
        AVG(age_days) as avg_age_days  -- Now this works
    FROM base_data
    GROUP BY DATE_TRUNC('month', createddate_ts)
)
SELECT * FROM aggregated_data
```

#### Rule 2: GROUP BY Context Limitations

After GROUP BY operations, only the following are available:

- Columns in the GROUP BY clause
- Aggregate functions (COUNT, SUM, AVG, etc.)
- Expressions that were calculated before the GROUP BY

❌ **INVALID After GROUP BY:**

```sql
-- After GROUP BY, individual row columns are not available
SELECT 
    customer_segment,
    COUNT(*) as record_count,
    createddate_ts  -- ERROR: Not in GROUP BY and not aggregated
FROM base_data
GROUP BY customer_segment
```

✅ **VALID After GROUP BY:**

```sql
SELECT 
    customer_segment,
    COUNT(*) as record_count,
    AVG(DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP)) as avg_age  -- Aggregated calculation
FROM base_data
GROUP BY customer_segment
```

#### Rule 3: Calculation Placement Strategy

**Place calculations at the appropriate CTE level:**

```sql
WITH base_calculations AS (
    -- Level 1: Basic field calculations (before any aggregation)
    SELECT 
        *,
        DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) as record_age_days,
        CASE WHEN amount_f > 50000 THEN 'High Value' ELSE 'Standard' END as deal_category
    FROM latest_opportunity
),
segment_calculations AS (
    -- Level 2: Add segmentation logic
    SELECT 
        *,
        CASE 
            WHEN record_age_days > 365 THEN 'Aged'
            WHEN record_age_days > 180 THEN 'Mature'
            ELSE 'Recent'
        END as age_category
    FROM base_calculations
),
aggregated_results AS (
    -- Level 3: Aggregation (only grouped columns and aggregates available)
    SELECT 
        deal_category,
        age_category,
        COUNT(*) as opportunity_count,
        AVG(record_age_days) as avg_age,  -- Using pre-calculated field
        AVG(amount_f) as avg_amount
    FROM segment_calculations
    GROUP BY deal_category, age_category
)
SELECT * FROM aggregated_results
```

### Column Scope Best Practices

#### 1. Early Calculation Pattern

```sql
-- Calculate everything you need before aggregation
WITH enriched_data AS (
    SELECT 
        *,
        -- All calculations needed downstream
        DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) as age_days,
        DATE_TRUNC('month', createddate_ts) as created_month,
        CASE WHEN amount_f > 50000 THEN 'Large' ELSE 'Standard' END as deal_size
    FROM base_table
),
final_aggregation AS (
    SELECT 
        created_month,
        deal_size,
        COUNT(*) as deals,
        AVG(age_days) as avg_age  -- Now available
    FROM enriched_data
    GROUP BY created_month, deal_size
)
SELECT * FROM final_aggregation
```

#### 2. Scope Testing Pattern

```sql
-- Test column availability with SELECT * first
WITH test_scope AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as report_month,
        COUNT(*) as record_count
    FROM base_data
    GROUP BY DATE_TRUNC('month', createddate_ts)
)
SELECT * FROM test_scope  -- See what columns are actually available
```

#### 3. Error Prevention Checklist

Before writing complex aggregations:

1. ✅ **Identify Required Calculations**: List all fields needed in final output
1. ✅ **Plan Calculation Placement**: Put calculations before aggregation steps
1. ✅ **Test Scope Boundaries**: Use SELECT * to verify column availability
1. ✅ **Use Descriptive Names**: Make calculated columns clearly identifiable

## Avoiding Ambiguous Column Name Errors

### The Problem

Athena throws "AMBIGUOUS_NAME" errors when column names appear in multiple tables without proper qualification. This is especially common in complex JOINs with multiple CTEs.

### Mandatory Solutions

#### Always Use Table Aliases

❌ **ERROR-PRONE:**

```sql
SELECT month_period, mcl_count, mql_count
FROM month_series
LEFT JOIN mcl_data ON month_period = month_period  -- AMBIGUOUS!
```

✅ **CORRECT:**

```sql
SELECT ms.month_period, mcl.mcl_count, mql.mql_count
FROM month_series ms
LEFT JOIN mcl_data mcl ON ms.month_period = mcl.month_period
```

#### Qualify ALL Column References in JOINs

❌ **WILL FAIL:**

```sql
LEFT JOIN (
    SELECT month_period, COUNT(*) as mcl_count FROM mcl_data GROUP BY month_period
) mcl ON months.month_period = mcl.month_period  -- months.month_period ambiguous
```

✅ **CORRECT:**

```sql
LEFT JOIN (
    SELECT month_period as mcl_month_period, COUNT(*) as mcl_count 
    FROM mcl_data 
    GROUP BY month_period
) mcl ON ms.month_period = mcl.mcl_month_period
```

#### Use Unique Column Names in Subqueries

❌ **PROBLEMATIC:**

```sql
-- Multiple subqueries with same column name
LEFT JOIN (SELECT month_period, COUNT(*) as count FROM table1 GROUP BY month_period) t1 ON ...
LEFT JOIN (SELECT month_period, COUNT(*) as count FROM table2 GROUP BY month_period) t2 ON ...
```

✅ **BETTER:**

```sql
-- Unique column names per subquery
LEFT JOIN (SELECT month_period as t1_month, COUNT(*) as t1_count FROM table1 GROUP BY month_period) t1 ON ...
LEFT JOIN (SELECT month_period as t2_month, COUNT(*) as t2_count FROM table2 GROUP BY month_period) t2 ON ...
```

#### Preferred Pattern: Pre-Qualify in CTEs

✅ **RECOMMENDED APPROACH:**

```sql
WITH month_series AS (
    SELECT DATE_TRUNC('month', DATE_ADD('month', -month_offset, CURRENT_DATE)) as month_period
    FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11)) AS months(month_offset)
),
mcl_monthly AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as mcl_month,  -- Unique name
        COUNT(*) as mcl_count
    FROM latest_lead
    WHERE (status = 'Open')
    GROUP BY DATE_TRUNC('month', createddate_ts)
),
mql_monthly AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as mql_month,  -- Unique name
        COUNT(*) as mql_count
    FROM latest_lead
    WHERE (status = 'Working')
    GROUP BY DATE_TRUNC('month', createddate_ts)
)
SELECT 
    ms.month_period,
    COALESCE(mcl.mcl_count, 0) as mcl_count,
    COALESCE(mql.mql_count, 0) as mql_count
FROM month_series ms
LEFT JOIN mcl_monthly mcl ON ms.month_period = mcl.mcl_month
LEFT JOIN mql_monthly mql ON ms.month_period = mql.mql_month
ORDER BY ms.month_period DESC
```

## Window Functions Limitations

❌ **AVOID (Not supported in Athena):**

```sql
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY field)  -- This syntax fails
```

✅ **USE INSTEAD:**

```sql
-- Use APPROX_PERCENTILE for median calculations
APPROX_PERCENTILE(field, 0.5) as median_field
```

## Common Query Patterns

### Age Calculation (Days Since Created)

```sql
-- Calculate days since lead was created
SELECT 
    id,
    name,
    createddate_ts,
    DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) as days_old
FROM lakehouse.object_lead
WHERE grax__deleted IS NULL
```

### Time-Based Filtering

```sql
-- Leads created in the last 30 days
SELECT *
FROM lakehouse.object_lead 
WHERE grax__deleted IS NULL
  AND createddate_ts >= DATE_ADD('day', -30, CURRENT_DATE)
  AND createddate_ts < CURRENT_DATE
```

### Monthly Trend Analysis

```sql
-- Group by month with proper date truncation
SELECT 
    DATE_TRUNC('month', createddate_ts) AS month,
    COUNT(*) as lead_count
FROM lakehouse.object_lead
WHERE grax__deleted IS NULL
  AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
GROUP BY DATE_TRUNC('month', createddate_ts)
ORDER BY month DESC
```

### Conversion Time Analysis

```sql
-- Time from lead creation to conversion
SELECT 
    id,
    name,
    createddate_ts,
    converteddate_d,
    DATE_DIFF('day', createddate_ts, CAST(converteddate_d AS timestamp)) as days_to_convert
FROM lakehouse.object_lead
WHERE grax__deleted IS NULL
  AND isconverted_b = true
  AND converteddate_d IS NOT NULL
```

## Error Prevention Checklist

### Before Running Complex Queries

1. ✅ **Date Functions**: Use `DATE_DIFF('day', ...)` not `DATEDIFF(DAY, ...)`

1. ✅ **Intervals**: Quote all date intervals: `'day'`, `'month'`, `'year'`

1. ✅ **System Fields**: Always include `WHERE grax__deleted IS NULL`

1. ✅ **Current Time**: Use `CURRENT_TIMESTAMP` or `CURRENT_DATE`

1. ✅ **Type Casting**: Cast date fields to timestamp when needed: `CAST(converteddate_d AS timestamp)`

1. ✅ **Table Aliases**: Use unique aliases for all tables and CTEs

1. ✅ **Column Qualification**: Prefix all columns with table/alias names in JOINs

1. ✅ **Unique Naming**: Use different column names in subqueries to avoid conflicts

1. ✅ **Window Functions**: Use `APPROX_PERCENTILE` instead of `PERCENTILE_CONT`

1. ✅ **Column Scope**: Place calculations before aggregation steps

1. ✅ **Scope Testing**: Verify column availability with SELECT * when building complex CTEs

## Common Error Messages and Solutions

### Column Name Ambiguous Errors

When you see: "Column 'month_period' is ambiguous"

- **Cause:** Multiple tables/CTEs have columns with the same name
- **Solution:** Use unique column names in each CTE or fully qualify with table aliases

### Column Resolution Errors

When you see: "Column 'createddate_ts' cannot be resolved"

- **Cause:** Referencing a column that's out of scope (often after GROUP BY)
- **Solution:** Move the calculation to a CTE before aggregation or use appropriate aggregate functions

When you see: "Column 'day' cannot be resolved"

- **Cause:** Using `DATEDIFF(DAY, ...)` syntax
- **Solution:** Change to `DATE_DIFF('day', ...)`

### Function Registration Errors

When you see: "Function 'datediff' not registered"

- **Cause:** Using non-Athena date function syntax
- **Solution:** Use Athena-specific functions: `DATE_DIFF`, `DATE_ADD`, `DATE_TRUNC`

### Window Function Syntax Errors

When you see: "mismatched input '('. Expecting: 'BY'"

- **Cause:** Usually indicates window function syntax not supported in Athena
- **Solution:** Replace `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY field)` with `APPROX_PERCENTILE(field, 0.5)`

## Performance Optimization

### Efficient Date Filtering

```sql
-- Use partition-friendly date filters
WHERE createddate_ts >= DATE('2024-01-01')
  AND createddate_ts < DATE('2025-01-01')
```

### Avoid Function Calls on Large Datasets

```sql
-- Instead of calculating age for all records, filter first
WHERE createddate_ts >= DATE_ADD('day', -30, CURRENT_DATE)
```

### Use DATE_TRUNC for Grouping

```sql
-- More efficient than EXTRACT functions
GROUP BY DATE_TRUNC('month', createddate_ts)
```

## Troubleshooting Guide

### When Queries Fail with Column Resolution Errors

1. **Identify the Scope Problem**: Check if the column exists in the current CTE/table context

1. **Check Aggregation Context**: If using GROUP BY, ensure columns are either grouped or aggregated

1. **Move Calculations Earlier**: Place complex calculations before aggregation steps

1. **Test Scope Incrementally**: Use SELECT * to see what columns are available at each step

1. **Use Descriptive Names**: Make calculated columns clearly identifiable

### When Queries Fail with Ambiguous Names

1. **Identify the Problem**: Look for duplicate column names across JOINs

1. **Add Unique Aliases**: Give each table/CTE a clear, unique alias

1. **Qualify All Columns**: Prefix every column reference with its table alias

1. **Use Unique Names**: Make subquery column names distinct

1. **Test Incrementally**: Build complex queries step by step

### Common Fixes for Column Resolution Errors

```sql
-- If this fails with column resolution error:
WITH aggregated AS (
    SELECT 
        customer_segment,
        COUNT(*) as record_count
    FROM base_data
    GROUP BY customer_segment
)
SELECT 
    customer_segment,
    record_count,
    AVG(created_date)  -- ERROR: created_date not available after GROUP BY
FROM aggregated

-- Fix by calculating before aggregation:
WITH calculations AS (
    SELECT 
        *,
        DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) as age_days
    FROM base_data
),
aggregated AS (
    SELECT 
        customer_segment,
        COUNT(*) as record_count,
        AVG(age_days) as avg_age  -- Now this works
    FROM calculations
    GROUP BY customer_segment
)
SELECT * FROM aggregated
```

### Common Fixes for Ambiguous Errors

```sql
-- If this fails with ambiguous error:
SELECT month_period, count1, count2
FROM table1 t1
LEFT JOIN table2 t2 ON t1.month_period = t2.month_period

-- Fix by qualifying all columns:
SELECT t1.month_period, t1.count1, t2.count2
FROM table1 t1
LEFT JOIN table2 t2 ON t1.month_period = t2.month_period

-- Or better yet, use unique column names:
SELECT t1.reporting_month, t1.metric1, t2.metric2
FROM (SELECT month_period as reporting_month, count1 as metric1 FROM table1) t1
LEFT JOIN (SELECT month_period as join_month, count2 as metric2 FROM table2) t2 
  ON t1.reporting_month = t2.join_month
```