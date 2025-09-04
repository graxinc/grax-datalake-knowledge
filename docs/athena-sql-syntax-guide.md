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

## Common Error Messages and Solutions

### Column Name Ambiguous Errors

When you see: "Column 'month_period' is ambiguous"

- **Cause:** Multiple tables/CTEs have columns with the same name
- **Solution:** Use unique column names in each CTE or fully qualify with table aliases

### Column Resolution Errors

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

### When Queries Fail with Ambiguous Names

1. **Identify the Problem**: Look for duplicate column names across JOINs

1. **Add Unique Aliases**: Give each table/CTE a clear, unique alias

1. **Qualify All Columns**: Prefix every column reference with its table alias

1. **Use Unique Names**: Make subquery column names distinct

1. **Test Incrementally**: Build complex queries step by step

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
