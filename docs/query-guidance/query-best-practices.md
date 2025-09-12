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

## Critical GROUP BY Rules

### Common GROUP BY Column Resolution Error

❌ **ERROR PATTERN:**

```sql
WITH latest_lead AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
quarterly_trends AS (
    SELECT 
        DATE_TRUNC('quarter', createddate_ts) as quarter,
        -- Segmentation using values from Configuration Reference
        CASE 
            WHEN numberofemployees_f < 200 THEN 'SMB'
            WHEN numberofemployees_f BETWEEN 200 AND 999 THEN 'Mid-Market'
            ELSE 'Enterprise'
        END as customer_segment,
        COUNT(*) as lead_count
    FROM latest_lead
    WHERE rn = 1
    GROUP BY DATE_TRUNC('quarter', createddate_ts)
    -- ERROR: customer_segment not in GROUP BY but used in SELECT
)
SELECT 
    quarter,
    customer_segment,  -- This will FAIL
    lead_count
FROM quarterly_trends
ORDER BY quarter DESC, customer_segment  -- This will also FAIL
```

**Error Message**: `COLUMN_NOT_FOUND: line XX:XX: Column 'customer_segment' cannot be resolved or requester is not authorized to access requested resources`

✅ **CORRECT APPROACH:**

```sql
WITH latest_lead AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
enriched_lead AS (
    SELECT 
        *,
        -- Calculate segmentation BEFORE aggregation
        CASE 
            WHEN numberofemployees_f < 200 THEN 'SMB'
            WHEN numberofemployees_f BETWEEN 200 AND 999 THEN 'Mid-Market'
            ELSE 'Enterprise'
        END as customer_segment
    FROM latest_lead
    WHERE rn = 1
),
quarterly_trends AS (
    SELECT 
        DATE_TRUNC('quarter', createddate_ts) as quarter,
        customer_segment,  -- Now included in GROUP BY
        COUNT(*) as lead_count
    FROM enriched_lead
    GROUP BY 
        DATE_TRUNC('quarter', createddate_ts),
        customer_segment  -- MUST include all SELECT columns in GROUP BY
)
SELECT 
    quarter,
    customer_segment,
    lead_count
FROM quarterly_trends
ORDER BY quarter DESC, customer_segment
```

### Quarterly Trend Analysis Template

**Complete working pattern for quarterly performance analysis:**

```sql
WITH latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('quarter', -8, CURRENT_DATE)
),
enriched_opportunity AS (
    -- Step 1: Calculate all fields needed for later analysis
    SELECT 
        *,
        DATE_TRUNC('quarter', createddate_ts) as created_quarter,
        -- Segmentation using thresholds from Configuration Reference
        CASE 
            WHEN amount_f >= 100000 THEN 'Enterprise Deal'
            WHEN amount_f >= 25000 THEN 'Mid-Market Deal'
            ELSE 'SMB Deal'
        END as deal_category,
        -- Stage classification using values from Configuration Reference
        CASE 
            WHEN stagename IN ('Prospecting', 'Qualification') THEN 'Early Stage'
            WHEN stagename IN ('Needs Analysis', 'Value Proposition') THEN 'Development'
            WHEN stagename IN ('Proposal/Price Quote', 'Negotiation/Review') THEN 'Late Stage'
            WHEN stagename = 'Closed Won' THEN 'Won'
            WHEN stagename = 'Closed Lost' THEN 'Lost'
            ELSE 'Other'
        END as stage_category
    FROM latest_opportunity
    WHERE rn = 1
),
quarterly_aggregation AS (
    -- Step 2: Aggregation with ALL dimensions in GROUP BY
    SELECT 
        created_quarter,
        deal_category,
        stage_category,
        COUNT(*) as opportunity_count,
        SUM(amount_f) as total_amount,
        AVG(amount_f) as avg_amount
    FROM enriched_opportunity
    GROUP BY 
        created_quarter,
        deal_category,
        stage_category  -- All non-aggregate SELECT columns MUST be in GROUP BY
)
-- Step 3: Final SELECT can reference any grouped columns
SELECT 
    created_quarter,
    deal_category,
    stage_category,
    opportunity_count,
    ROUND(total_amount, 0) as total_amount,
    ROUND(avg_amount, 0) as avg_amount
FROM quarterly_aggregation
ORDER BY 
    created_quarter DESC,
    deal_category,
    stage_category
```

## Error Prevention Rules

### NEVER Do These Things

1. **Query without deleted record filtering**:

   ```sql
   -- WRONG: Will include deleted records
   SELECT * FROM lakehouse.object_lead
   ```

1. **Query without latest record patterns for current state**:

   ```sql
   -- WRONG: Will return historical versions
   SELECT COUNT(*) FROM lakehouse.object_lead WHERE grax__deleted IS NULL
   ```

1. **Query without date boundaries**:

   ```sql
   -- WRONG: Will scan entire table
   SELECT * FROM lakehouse.object_opportunity WHERE grax__deleted IS NULL
   ```

1. **Reference columns in ORDER BY/WHERE that aren't in your SELECT clause after GROUP BY**:

   ```sql
   -- WRONG: customer_segment not in final SELECT or GROUP BY
   SELECT quarter, COUNT(*) FROM table GROUP BY quarter ORDER BY customer_segment
   ```

1. **Use hardcoded configuration values**:

   ```sql
   -- WRONG: Hardcoded status values
   WHERE status IN ('Open', 'Working')
   ```

### ALWAYS Do These Things

1. **Filter deleted records first**:

   ```sql
   WHERE grax__deleted IS NULL
   ```

1. **Use latest records pattern for current state**:

   ```sql
   ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
   WHERE rn = 1
   ```

1. **Include date boundaries for performance**:

   ```sql
   WHERE createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
   ```

1. **Include ALL columns you'll need to reference in your final GROUP BY clause**:

   ```sql
   GROUP BY quarter, customer_segment, stage_category  -- All SELECT columns
   ```

1. **Reference Configuration Reference for business values**:

   ```sql
   -- Using lead status values from Configuration Reference
   WHERE status IN ('MQL', 'SAL', 'SQL')
   ```

1. **Use LIMIT during development**:

   ```sql
   LIMIT 100  -- Remove for production
   ```

## Enhanced Debug Checklist

### When Building Complex Aggregations

1. **✅ Plan Your CTEs**:
   - Level 1: Get latest records and basic filtering
   - Level 2: Add all calculations and enrichments
   - Level 3: Perform aggregation
   - Level 4: Final formatting and selection

1. **✅ Test Each CTE Level**:

   ```sql
   -- Test intermediate results
   WITH enriched_data AS (
       SELECT *, [calculations] FROM base_table
   )
   SELECT * FROM enriched_data LIMIT 10  -- Verify calculations
   ```

1. **✅ Verify Column Scope**:

   ```sql
   -- Before final aggregation, check available columns
   WITH test_scope AS (
       SELECT column1, column2, COUNT(*) FROM table GROUP BY column1, column2
   )
   SELECT * FROM test_scope LIMIT 5  -- See what columns exist
   ```

### Quarterly Analysis Error Recovery

**If you get "Column 'X' cannot be resolved" during quarterly analysis:**

1. **✅ Identify the Missing Column**: Which column is causing the error?

1. **✅ Check CTE Scope**: Is the column available in the current CTE context?

1. **✅ Move Calculation Earlier**: Add the calculation to an earlier CTE before aggregation:

   ```sql
   WITH enriched_data AS (
       SELECT 
           *,
           CASE WHEN amount_f > 50000 THEN 'Large' ELSE 'Small' END as size_category
       FROM base_table
   ),
   aggregated_data AS (
       SELECT 
           quarter,
           size_category,  -- Now available for GROUP BY
           COUNT(*) as count
       FROM enriched_data
       GROUP BY quarter, size_category  -- Include in GROUP BY
   )
   SELECT * FROM aggregated_data
   ```

1. **✅ Test Scope Step by Step**: Use SELECT * to verify columns at each level

1. **✅ Update GROUP BY**: Ensure all non-aggregate columns are grouped

### Scope Boundary Testing

**Before building complex queries, test scope boundaries:**

```sql
-- Step 1: Test base data availability
WITH base_data AS (
    SELECT * FROM table WHERE conditions
)
SELECT * FROM base_data LIMIT 5;

-- Step 2: Test enrichment availability
WITH enriched_data AS (
    SELECT *, [calculated_fields] FROM base_data
)
SELECT * FROM enriched_data LIMIT 5;

-- Step 3: Test aggregation availability
WITH aggregated_data AS (
    SELECT group_fields, COUNT(*) FROM enriched_data GROUP BY group_fields
)
SELECT * FROM aggregated_data LIMIT 5;
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
        COUNT(*) as record_count,
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
