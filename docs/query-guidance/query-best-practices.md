# Query Best Practices

Performance optimization and error prevention guidelines for GRAX Data Lake queries using Amazon Athena.

**Configuration Integration**: All examples reference standardized values from [Configuration Reference](../core-reference/configuration-reference.md). Update these values to match your organization's Salesforce configuration.

## Performance Optimization Guidelines

### Mandatory Query Structure

#### 1. Latest Records Pattern

**ALWAYS** use for current state analysis:

```sql
WITH latest_records AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_[table]
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)  -- Performance boundary
)
SELECT * FROM latest_records WHERE rn = 1
```

#### 2. Deleted Records Filtering

**ALWAYS** include in WHERE clause:

```sql
WHERE grax__deleted IS NULL
```

#### 3. Date Boundaries

**ALWAYS** include reasonable time limits:

```sql
-- For analysis queries
WHERE createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)

-- For current state queries
WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)

-- For testing
WHERE createddate_ts >= DATE_ADD('day', -30, CURRENT_DATE)
LIMIT 100
```

### Query Development Workflow

#### Phase 1: Exploration (Small Scale)

```sql
-- Start with limited scope for development
WITH latest_opportunities AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('day', -30, CURRENT_DATE)  -- Small time window
)
SELECT *
FROM latest_opportunities
WHERE rn = 1
LIMIT 10  -- Small limit for testing
```

#### Phase 2: Validation (Medium Scale)

```sql
-- Expand scope after validation
WITH latest_opportunities AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -6, CURRENT_DATE)  -- Medium time window
)
SELECT 
    stagename,
    COUNT(*) as opportunity_count,
    SUM(amount_f) as total_amount
FROM latest_opportunities
WHERE rn = 1
GROUP BY stagename
ORDER BY total_amount DESC
LIMIT 50  -- Moderate limit
```

#### Phase 3: Production (Full Scale)

```sql
-- Remove limits for production analysis
WITH latest_opportunities AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)  -- Full analysis window
)
SELECT 
    stagename,
    COUNT(*) as opportunity_count,
    SUM(amount_f) as total_amount,
    AVG(amount_f) as avg_amount
FROM latest_opportunities
WHERE rn = 1
GROUP BY stagename
ORDER BY total_amount DESC
-- Remove LIMIT for full analysis
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

```sql
-- Standard implementation for current state
WITH latest_[object] AS (
    SELECT A.* 
    FROM lakehouse.object_[object] A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_[object] B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
)
SELECT * FROM latest_[object]
```

### 2. Progressive Stage Logic

**For funnel analysis using stage progression from [Configuration Reference](../core-reference/configuration-reference.md):**

```sql
-- MCL Logic: Marketing Qualified Lead identification
CASE 
    WHEN status = 'Working' THEN createddate_ts
    ELSE NULL
END as mql_date,

-- SQL Logic: Sales Qualified Lead progression  
CASE 
    WHEN status IN ('SQL', 'SAL') THEN createddate_ts
    WHEN isconverted_b = true AND converteddate_d IS NOT NULL THEN CAST(converteddate_d AS timestamp)
    ELSE NULL
END as sql_date
```

### 3. Unique Column Names in JOINs

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
- Include `'Closed Lost'` in progressive stage counts (per [Configuration Reference](../core-reference/configuration-reference.md))
- Use ambiguous column names in JOINs
- **Reference columns in ORDER BY/WHERE that aren't in your SELECT clause after GROUP BY**

### ALWAYS

- Create derived columns in separate CTEs before referencing them
- Use current status/stage with creation/change dates for date assignments
- Use unique column names in subqueries to prevent ambiguous errors
- Test with LIMIT 10 before running full queries
- Validate that all referenced columns actually exist in the schema
- Reference business values from [Configuration Reference](../core-reference/configuration-reference.md)
- **Include ALL columns you'll need to reference in your final GROUP BY clause**

## Enhanced Debug Checklist

**If you get column resolution errors:**

1. **Check Column Scope**: Verify the column exists in the current CTE/table context

1. **Validate GROUP BY Logic**: After GROUP BY, only grouped columns and aggregates are available

1. **Trace Column Lifecycle**: Follow where columns are created, modified, and referenced

1. **Test Incrementally**: Build complex queries step by step with SELECT * to verify scope

1. **Confirm Field Naming**: Ensure field naming follows `_f`, `_ts`, `_d`, `_b` pattern

1. **Map Dependencies**: Ensure derived columns are created before being referenced

## Quarterly Analysis Error Recovery

**When you encounter "Column cannot be resolved" in quarterly/trend analysis:**

1. **Identify the Missing Column**: Note which column is causing the error

1. **Check Aggregation Context**: Determine if it's after a GROUP BY operation

1. **Restructure Query**: Move column calculations to earlier CTE

1. **Include in GROUP BY**: Add the column to GROUP BY if you need to reference it later

1. **Test Fix**: Use SELECT * to verify column availability before final query

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

### 3. Efficient Date Filtering

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

## Structured CTE Guidelines

### Logical CTE Flow

**Follow this progression for complex analytical queries:**

```sql
-- Step 1: Data Foundation
WITH latest_records AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_[table]
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
),

-- Step 2: Basic Calculations
base_calculations AS (
    SELECT 
        *,
        -- All basic field calculations
        DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) as record_age,
        DATE_TRUNC('quarter', createddate_ts) as created_quarter
    FROM latest_records
    WHERE rn = 1
),

-- Step 3: Business Logic and Segmentation
business_logic AS (
    SELECT 
        *,
        -- Segmentation using values from Configuration Reference
        CASE 
            WHEN record_age > 365 THEN 'Aged'
            WHEN record_age > 180 THEN 'Mature'
            ELSE 'Recent'
        END as age_category
    FROM base_calculations
),

-- Step 4: Filtering (if needed)
filtered_data AS (
    SELECT *
    FROM business_logic
    WHERE [additional_conditions]
),

-- Step 5: Final Aggregation
final_aggregation AS (
    SELECT 
        created_quarter,
        age_category,
        COUNT(*) as record_count
        -- All non-aggregate fields must be in GROUP BY
    FROM filtered_data
    GROUP BY created_quarter, age_category
)

-- Step 6: Final Selection and Formatting
SELECT 
    created_quarter,
    age_category,
    record_count,
    ROUND(record_count * 100.0 / SUM(record_count) OVER (), 2) as percentage
FROM final_aggregation
ORDER BY created_quarter DESC, age_category
```

### Column Lifecycle Tracking

**Track which columns are available at each CTE level:**

1. **Latest Records**: All original fields + `rn`
1. **Base Calculations**: Original fields + calculated fields
1. **Business Logic**: Previous fields + segmentation fields
1. **Filtered Data**: Previous fields (same as business logic)
1. **Final Aggregation**: Only GROUP BY fields + aggregates
1. **Final Selection**: Only aggregated level fields + window functions

## Performance Monitoring

### Query Performance Indicators

**Watch for these warning signs:**

- **Query timeout**: Add more restrictive date filters
- **High data scan volume**: Check partitioning strategy
- **Memory errors**: Reduce aggregation complexity
- **Column resolution errors**: Review CTE scope boundaries

### Performance Optimization Techniques

#### 1. Early Filtering

```sql
-- Filter as early as possible
WITH filtered_base AS (
    SELECT *
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
        AND stagename != 'Closed Lost'  -- Early business filter
),
latest_records AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM filtered_base
)
SELECT * FROM latest_records WHERE rn = 1
```

#### 2. Selective Field Selection

```sql
-- Only select fields you need
WITH latest_opportunities AS (
    SELECT 
        id,
        accountid,
        amount_f,
        stagename,
        createddate_ts,  -- Only required fields
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
-- Instead of SELECT *
SELECT 
    id,
    amount_f,
    stagename
FROM latest_opportunities
WHERE rn = 1
```

#### 3. Appropriate Date Ranges

| Analysis Type | Recommended Range | Performance Impact |
|---------------|-------------------|-------------------|
| **Daily Trends** | 90 days | Low |
| **Monthly Analysis** | 12-24 months | Medium |
| **Quarterly Reports** | 8 quarters (24 months) | Medium-High |
| **Annual Analysis** | 3-5 years | High |
| **Development/Testing** | 30 days | Minimal |

## Integration with Other Documentation

These best practices integrate with:

- **[Database Schema Guide](../core-reference/database-schema-guide.md)** - Field definitions and relationships
- **[Configuration Reference](../core-reference/configuration-reference.md)** - Business values and thresholds
- **[Athena SQL Syntax Guide](athena-sql-syntax-guide.md)** - Proper SQL syntax patterns
- **[Query Templates](query-templates.md)** - Reusable patterns implementing these practices
- **[Customer Fallback Instructions](../troubleshooting/customer-fallback-instructions.md)** - Error recovery strategies

Following these practices ensures reliable, performant queries that provide accurate business insights while preventing common errors and performance issues.
