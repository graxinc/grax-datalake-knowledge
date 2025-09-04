# Query Best Practices

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
    -- Create the mql_date calculation here
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
    -- MQL Date Logic based on CURRENT status
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
        -- Calculate MQL date based on current status
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
COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count
```

**SQO Stage**: Reached SQO or beyond (excluding Closed Lost)

```sql
COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') 
      THEN 1 END) as sqo_count
```

**Proposal Stage**: Reached Proposal or beyond

```sql
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
- Include Closed Lost in progressive stage counts
- Use ambiguous column names in JOINs

### ALWAYS

- Create derived columns in separate CTEs before referencing them
- Use current status/stage with creation/change dates for date assignments
- Use unique column names in subqueries to prevent ambiguous errors
- Test with LIMIT 10 before running full queries
- Validate that all referenced columns actually exist in the schema

## Quick Debug Checklist

**If you get column resolution errors:**

1. Check if the column exists in the table schema
2. Verify you're using the correct CTE that contains the column
3. Confirm field naming follows `_f`, `_ts`, `_d`, `_b` pattern
4. Ensure you're not referencing derived columns before they're created
5. Use `SELECT *` temporarily to see what columns are actually available

## Performance Optimization

### 1. Query Structure

- **Filter early**: Apply WHERE conditions in CTEs
- **Use specific date ranges**: Avoid full table scans
- **Test incrementally**: Build complex queries step by step

### 2. Data Type Usage

- **Use typed fields**: Prefer `_f`, `_ts`, `_d`, `_b` suffixes
- **Cast appropriately**: Convert dates to timestamps when needed
- **Handle nulls**: Use COALESCE for safe operations

### 3. Join Optimization

- **Use table aliases**: Keep queries readable and avoid ambiguity
- **Apply filters before joins**: Reduce data volume early
- **Use appropriate join types**: LEFT JOIN vs INNER JOIN based on requirements

Following these best practices will ensure reliable, performant queries that accurately represent business logic and avoid common pitfalls in the data lake environment.
