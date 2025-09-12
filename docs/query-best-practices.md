# Query Best Practices

## Overview

This guide provides essential patterns for reliable query formation and error prevention in the GRAX Data Lake environment. Following these practices ensures consistent results and optimal performance.

**Configuration Dependencies**: All business-specific values referenced in examples use standardized configuration from [Configuration Reference](./configuration-reference.md). Organizations should update that document to match their specific values rather than modifying individual queries.

## Critical Query Formation Rules

### 1. Never Reference Non-Existent Columns

**The Problem**: Referencing columns that don't exist in the current scope

❌ **PROBLEM PATTERN:**

```sql
-- This FAILS - mql_date doesn't exist in the latest_lead table
SELECT 
    createddate_ts,
    mql_date  -- ERROR: Column doesn't exist
FROM latest_lead
```

✅ **SOLUTION PATTERN:**

```sql
-- Create the derived column first, then reference it
SELECT 
    createddate_ts,
    -- Create the mql_date calculation here using values from docs/configuration-reference.md
    CASE 
        WHEN status = 'Working' THEN createddate_ts
        ELSE NULL
    END as mql_date
FROM latest_lead
```

### 2. Proper Historical Date Logic

❌ **WRONG APPROACH:**

```sql
-- This assumes historical data exists in current record - IT DOESN'T
SELECT MIN(CASE WHEN status = 'Working' THEN some_historical_date END) as mql_date
```

✅ **CORRECT APPROACH:**

```sql
-- Use creation date with status logic for date assignment
SELECT 
    id,
    createddate_ts as lead_created,
    -- MQL Date Logic using status values from docs/configuration-reference.md
    CASE 
        WHEN status = 'Working' THEN createddate_ts
        WHEN isconverted_b = true AND converteddate_d IS NOT NULL THEN CAST(converteddate_d AS timestamp)
        ELSE NULL
    END as mql_date
FROM latest_lead
```

### 3. Mandatory CTE Structure for Complex Analysis

**ALWAYS use this pattern for funnel analysis:**

```sql
WITH month_series AS (
    -- Time period generation
    SELECT DATE_TRUNC('month', DATE_ADD('month', -month_offset, CURRENT_DATE)) as report_month
    FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11)) AS months(month_offset)
),
latest_lead AS (
    -- Standard latest records pattern
    SELECT A.* 
    FROM lakehouse.object_lead A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_lead B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
-- CREATE derived columns in separate CTE
lead_with_dates AS (
    SELECT 
        *,
        -- Calculate MQL date using status values from docs/configuration-reference.md
        CASE 
            WHEN status = 'Working' THEN createddate_ts
            WHEN isconverted_b = true THEN CAST(converteddate_d AS timestamp)
            ELSE NULL
        END as calculated_mql_date
    FROM latest_lead
),
mcl_analysis AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as mcl_month,
        COUNT(*) as mcl_count
    FROM lead_with_dates  -- Use the CTE with derived columns
    -- Using lead status from docs/configuration-reference.md
    WHERE status = 'Open'
    GROUP BY DATE_TRUNC('month', createddate_ts)
),
mql_analysis AS (
    SELECT 
        DATE_TRUNC('month', calculated_mql_date) as mql_month,  -- Now this exists!
        COUNT(*) as mql_count
    FROM lead_with_dates  -- Use the CTE with derived columns
    WHERE calculated_mql_date IS NOT NULL
    GROUP BY DATE_TRUNC('month', calculated_mql_date)
)
-- Continue with final SELECT and JOINs
```

## GROUP BY Column Resolution Error Prevention

### 4. Critical GROUP BY Rules

**The Problem**: After GROUP BY operations, only grouped columns and aggregated functions are available in subsequent operations.

❌ **COMMON ERROR PATTERN:**

```sql
-- This FAILS because customer_segment is referenced in ORDER BY but not in final SELECT GROUP BY
WITH quarterly_data AS (
    SELECT 
        DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) as close_quarter,
        CASE 
            WHEN a.numberofemployees_f > 250 THEN 'Enterprise'
            ELSE 'SMB'
        END as customer_segment,
        o.stagename,
        o.amount_f
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
),
quarterly_performance AS (
    SELECT 
        close_quarter,
        customer_segment,  -- Grouped column
        COUNT(*) as total_opportunities,
        SUM(amount_f) as won_revenue
    FROM quarterly_data
    GROUP BY close_quarter, customer_segment
)
SELECT 
    CAST(close_quarter AS date) as quarter,
    total_opportunities,
    won_revenue
FROM quarterly_performance
ORDER BY close_quarter DESC, customer_segment  -- ERROR: customer_segment not in SELECT
```

✅ **CORRECT APPROACH:**

```sql
-- Include ALL columns you need to reference in the final SELECT
WITH quarterly_data AS (
    SELECT 
        DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) as close_quarter,
        CASE 
            WHEN a.numberofemployees_f > 250 THEN 'Enterprise'
            ELSE 'SMB'
        END as customer_segment,
        o.stagename,
        o.amount_f
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
),
quarterly_performance AS (
    SELECT 
        close_quarter,
        customer_segment,  -- Include in GROUP BY
        COUNT(*) as total_opportunities,
        SUM(amount_f) as won_revenue
    FROM quarterly_data
    GROUP BY close_quarter, customer_segment
)
SELECT 
    CAST(close_quarter AS date) as quarter,
    customer_segment,  -- Include in SELECT if you want to reference it
    total_opportunities,
    won_revenue
FROM quarterly_performance
ORDER BY close_quarter DESC, customer_segment  -- Now this works
```

### 5. Quarterly Trend Analysis Template

**Safe pattern for quarterly performance analysis:**

```sql
WITH latest_opportunity AS (
    SELECT A.* 
    FROM lakehouse.object_opportunity A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_opportunity B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
latest_account AS (
    SELECT A.* 
    FROM lakehouse.object_account A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_account B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
enriched_opportunities AS (
    -- Step 1: Create ALL derived columns BEFORE any aggregation
    SELECT 
        o.id,
        o.stagename,
        o.amount_f,
        o.createddate_ts,
        o.closedate_d,
        DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) as close_quarter,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment,
        DATE_DIFF('day', o.createddate_ts, CAST(o.closedate_d AS timestamp)) as sales_cycle_days
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    WHERE o.stagename IN ('Closed Won', 'Closed Lost')
      AND o.closedate_d IS NOT NULL
      AND CAST(o.closedate_d AS timestamp) >= DATE_ADD('month', -24, CURRENT_DATE)
      AND o.amount_f IS NOT NULL
),
quarterly_aggregation AS (
    -- Step 2: Perform aggregation with ALL needed grouping columns
    SELECT 
        close_quarter,
        customer_segment,  -- MUST include in GROUP BY if you want to reference later
        COUNT(*) as total_opportunities,
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as won_opportunities,
        SUM(CASE WHEN stagename = 'Closed Won' THEN amount_f ELSE 0 END) as won_revenue,
        AVG(CASE WHEN stagename = 'Closed Won' THEN sales_cycle_days END) as avg_won_cycle_days
    FROM enriched_opportunities
    GROUP BY close_quarter, customer_segment  -- Include ALL columns you'll reference
)
-- Step 3: Final SELECT with all columns available
SELECT 
    CAST(close_quarter AS date) as quarter,
    customer_segment,  -- Available because it was in GROUP BY
    total_opportunities,
    won_opportunities,
    ROUND(won_opportunities * 100.0 / total_opportunities, 1) as win_rate_pct,
    ROUND(won_revenue, 0) as won_revenue,
    ROUND(avg_won_cycle_days, 0) as avg_won_cycle_days
FROM quarterly_aggregation
ORDER BY close_quarter DESC, customer_segment  -- Both columns are available
```

## Universal Query Requirements

### 1. Latest Records Pattern (MANDATORY)

For ALL current state analysis, use this pattern:

```sql
WITH latest_[objectname] AS (
    SELECT A.* 
    FROM lakehouse.object_[objectname] A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_[objectname] B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
)
SELECT * FROM latest_[objectname]
```

### 2. System Field Requirements

**ALWAYS include:**

- `WHERE grax__deleted IS NULL` - Filter out deleted records
- Use `grax__idseq` for version control when needed

### 3. Field Naming Validation

**Before writing any query, verify:**

- ✅ Use `_f` for numeric fields (`amount_f`, not `amount`)
- ✅ Use `_ts` for timestamps (`createddate_ts`, not `createddate`)
- ✅ Use `_d` for dates (`converteddate_d`)
- ✅ Use `_b` for booleans (`isconverted_b`, not `isconverted`)
- ✅ Use `_i` for integers (`numberofemployees_i`)

## Stage Date Assignment Logic

### Lead Qualification Dates

**MCL Date**: Always use lead creation date

```sql
createddate_ts as mcl_date
```

**MQL Date**: Use status-based logic with creation date

```sql
-- Using lead status values from docs/configuration-reference.md
CASE 
    WHEN status = 'Working' THEN createddate_ts
    WHEN isconverted_b = true THEN CAST(converteddate_d AS timestamp)
    ELSE NULL
END as mql_date
```

### Opportunity Stage Dates

**SQL Date**: Always use opportunity creation date

```sql
createddate_ts as sql_date  -- ALL opportunities are SQL at creation
```

**SQO Date**: Use stage change date with fallback

```sql
-- Using stage names from docs/configuration-reference.md
CASE 
    WHEN stagename = 'Proof of Value (SQO)' THEN laststagechangedate_ts
    WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won') THEN laststagechangedate_ts
    ELSE NULL
END as sqo_date
```

## Progressive Stage Logic

### Sequential Stage Qualification

**SQL Stage**: All opportunities except Closed Lost

```sql
-- Using stage exclusion logic from docs/configuration-reference.md
COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count
```

**SQO Stage**: Reached SQO or beyond (excluding Closed Lost)

```sql
-- Using stage progression logic from docs/configuration-reference.md
COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') 
      THEN 1 END) as sqo_count
```

**Proposal Stage**: Reached Proposal or beyond

```sql
-- Using stage names from docs/configuration-reference.md
COUNT(CASE WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won')
      THEN 1 END) as proposal_count
```

### Velocity Calculations

**Use existing date fields:**

```sql
-- Calculate velocity between known dates
AVG(DATE_DIFF('day', createddate_ts, 
    CASE WHEN isconverted_b = true THEN CAST(converteddate_d AS timestamp) 
         ELSE CURRENT_TIMESTAMP END
)) as lead_to_conversion_days
```

## JOIN Safety and Performance

### 1. Unique Column Names in JOINs

**Always use unique column names to prevent ambiguous errors:**

```sql
-- Good pattern with unique names
LEFT JOIN mcl_analysis mcl ON ms.report_month = mcl.mcl_month
LEFT JOIN mql_analysis mql ON ms.report_month = mql.mql_month  
LEFT JOIN sql_analysis sql ON ms.report_month = sql.sql_month
```

### 2. Early Filtering

**Apply filters in CTEs to reduce data volume:**

```sql
WITH filtered_opportunities AS (
    SELECT * 
    FROM latest_opportunity
    WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
      -- Using stage exclusion logic from docs/configuration-reference.md
      AND stagename != 'Closed Lost'
)
-- Then use filtered_opportunities in subsequent analysis
```

### 3. Efficient Existence Checks

**Use EXISTS instead of IN for better performance:**

```sql
AND EXISTS (
    SELECT 1 FROM lakehouse.object_related
    WHERE object_related.parent_id = main_table.id
      AND object_related.grax__deleted IS NULL
)
```

## Error Prevention Rules

### NEVER

- Reference columns that don't exist in the current table/CTE
- Assume historical stage dates exist in latest records
- Forget `WHERE grax__deleted IS NULL`
- Include `'Closed Lost'` in progressive stage counts (per [Configuration Reference](./configuration-reference.md))
- Use ambiguous column names in JOINs
- **Reference columns in ORDER BY/WHERE that aren't in your SELECT clause after GROUP BY**

### ALWAYS

- Create derived columns in separate CTEs before referencing them
- Use current status/stage with creation/change dates for date assignments
- Use unique column names in subqueries to prevent ambiguous errors
- Test with LIMIT 10 before running full queries
- Validate that all referenced columns actually exist in the schema
- Reference business values from [Configuration Reference](./configuration-reference.md)
- **Include ALL columns you'll need to reference in your final GROUP BY clause**

## Enhanced Debug Checklist

**If you get column resolution errors:**

1. **Check Column Scope**: Verify the column exists in the current CTE/table context

2. **Validate GROUP BY Logic**: After GROUP BY, only grouped columns and aggregates are available

3. **Trace Column Lifecycle**: Follow where columns are created, modified, and referenced

4. **Test Incrementally**: Build complex queries step by step with SELECT * to verify scope

5. **Confirm Field Naming**: Ensure field naming follows `_f`, `_ts`, `_d`, `_b` pattern

6. **Map Dependencies**: Ensure derived columns are created before being referenced

## Quarterly Analysis Error Recovery

**When you encounter "Column cannot be resolved" in quarterly/trend analysis:**

1. **Identify the Missing Column**: Note which column is causing the error

2. **Check Aggregation Context**: Determine if it's after a GROUP BY operation

3. **Restructure Query**: Move column calculations to earlier CTE

4. **Include in GROUP BY**: Add the column to GROUP BY if you need to reference it later

5. **Test Fix**: Use SELECT * to verify column availability before final query

**Example Recovery Process:**

```sql
-- Step 1: If this fails
ORDER BY close_quarter DESC, customer_segment  -- ERROR: customer_segment not resolved

-- Step 2: Check if customer_segment is in the final SELECT
SELECT 
    CAST(close_quarter AS date) as quarter,
    -- customer_segment missing here
    total_opportunities

-- Step 3: Add missing column to SELECT
SELECT 
    CAST(close_quarter AS date) as quarter,
    customer_segment,  -- Add this
    total_opportunities

-- Step 4: Ensure it's in the GROUP BY of the source CTE
GROUP BY close_quarter, customer_segment  -- Must include both
```

## Performance Optimization

### 1. Query Structure

- **Filter early**: Apply WHERE conditions in CTEs
- **Use specific date ranges**: Avoid full table scans
- **Test incrementally**: Build complex queries step by step
- **Structure CTEs logically**: Calculations → Filtering → Aggregation → Final SELECT

### 2. Data Type Usage

- **Use typed fields**: Prefer `_f`, `_ts`, `_d`, `_b` suffixes
- **Cast appropriately**: Convert dates to timestamps when needed
- **Handle nulls**: Use COALESCE for safe operations

### 3. Join Optimization

- **Use table aliases**: Keep queries readable and avoid ambiguity
- **Apply filters before joins**: Reduce data volume early
- **Use appropriate join types**: LEFT JOIN vs INNER JOIN based on requirements

## Configuration-Dependent Patterns

### Segmentation Logic

```sql
-- Using thresholds from docs/configuration-reference.md
CASE 
    WHEN numberofemployees_f > 250 OR annualrevenue_f > 100000000 THEN 'Enterprise'
    WHEN numberofemployees_f > 50 OR annualrevenue_f > 10000000 THEN 'SMB'
    ELSE 'Self Service'
END as customer_segment
```

### Lead Status Filtering

```sql
-- Using lead status values from docs/configuration-reference.md
WHERE status IN ('Open', 'Working')  -- Active leads only
```

### Opportunity Stage Filtering

```sql
-- Using stage names from docs/configuration-reference.md
WHERE stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won')
```

### Account Classification

```sql
-- Using account types from docs/configuration-reference.md
WHERE type IN ('Customer', 'Customer - Subsidiary') AND type != 'Prospect'
```

## Configuration Adaptation

For organizations with different Salesforce implementations:

### Update Process

1. **Modify Configuration**: Update [Configuration Reference](./configuration-reference.md) with your specific values
1. **Test Query Patterns**: Ensure all patterns work with your configuration
1. **Validate Results**: Confirm business logic produces expected outcomes
1. **Document Changes**: Record any pattern modifications needed

### Key Areas for Adaptation

- **Lead Status Values**: Update status names used in filtering and logic
- **Opportunity Stage Names**: Modify stage progression patterns
- **Account Classification**: Adjust customer identification logic
- **Segmentation Thresholds**: Update employee/revenue limits
- **Field Names**: Adapt if using custom field naming

Following these best practices ensures reliable, performant queries that accurately represent business logic while maintaining adaptability across different Salesforce implementations through centralized configuration management.
