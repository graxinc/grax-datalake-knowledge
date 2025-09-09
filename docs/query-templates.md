# Query Templates

## Overview

All query templates in this document use the standardized configuration values from [Configuration Reference](./configuration-reference.md). Organizations with different Salesforce configurations should update that document to match their specific values rather than modifying individual queries.

**Configuration Dependencies**: The templates below reference business-specific values defined in [Configuration Reference](./configuration-reference.md) including lead status values, opportunity stage names, account types, and segmentation thresholds.

## Core Query Patterns

### Latest Records Pattern (MANDATORY)

Use this pattern for ALL current state analysis:

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

### Historical Change Detection Pattern

```sql
-- Find when a field first changed to a specific value
WITH field_changes AS (
    SELECT 
        current_rec.id,
        MIN(current_rec.lastmodifieddate_ts) AS change_date
    FROM lakehouse.object_[objectname] current_rec
    WHERE current_rec.grax__deleted IS NULL
      AND current_rec.[fieldname] = '[target_value]'
      AND EXISTS (
          -- Ensure there was a previous record with different value
          SELECT 1 FROM lakehouse.object_[objectname] prev_rec
          WHERE prev_rec.id = current_rec.id
            AND prev_rec.grax__deleted IS NULL
            AND prev_rec.grax__idseq < current_rec.grax__idseq
            AND (prev_rec.[fieldname] != '[target_value]' OR prev_rec.[fieldname] IS NULL)
      )
    GROUP BY current_rec.id
)
SELECT * FROM field_changes
```

### Multi-Object Join Pattern

```sql
WITH latest_[object1] AS (
    -- Latest records pattern for first object
    SELECT A.* 
    FROM lakehouse.object_[object1] A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_[object1] B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
latest_[object2] AS (
    -- Latest records pattern for second object
    SELECT A.* 
    FROM lakehouse.object_[object2] A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_[object2] B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
)
SELECT 
    obj1.*,
    obj2.*
FROM latest_[object1] obj1
LEFT JOIN latest_[object2] obj2 ON obj1.[relationship_field] = obj2.id
```

### Time Series Analysis Pattern

```sql
WITH latest_records AS (
    -- Latest records pattern
    SELECT A.* 
    FROM lakehouse.object_[objectname] A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_[objectname] B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
)
SELECT 
    DATE_TRUNC('[period]', [date_field]_ts) AS time_period,
    COUNT(*) AS record_count,
    SUM([numeric_field]_f) AS total_value,
    AVG([numeric_field]_f) AS avg_value
FROM latest_records
WHERE [date_field]_ts >= DATE_ADD('[period]', -[number], CURRENT_DATE)
GROUP BY DATE_TRUNC('[period]', [date_field]_ts)
ORDER BY time_period DESC
```

## Lead Analysis Templates

### Lead Qualification Analysis

```sql
WITH latest_lead AS (
    SELECT A.* 
    FROM lakehouse.object_lead A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_lead B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
lead_with_qualification AS (
    SELECT 
        *,
        -- Using lead status values from docs/configuration-reference.md
        CASE WHEN status = 'Open' THEN createddate_ts ELSE NULL END as mcl_date,
        -- MQL qualification using configuration values
        CASE 
            WHEN status = 'Working' THEN createddate_ts
            WHEN isconverted_b = true THEN CAST(converteddate_d AS timestamp)
            ELSE NULL
        END as mql_date,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN numberofemployees_f > 250 OR annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN numberofemployees_f > 50 OR annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as segment
    FROM latest_lead
)
SELECT 
    segment,
    COUNT(*) as total_leads,
    COUNT(mcl_date) as mcl_count,
    COUNT(mql_date) as mql_count,
    COUNT(CASE WHEN isconverted_b = true THEN 1 END) as converted_count,
    ROUND(COUNT(mql_date) * 100.0 / NULLIF(COUNT(mcl_date), 0), 2) as mcl_to_mql_rate,
    ROUND(COUNT(CASE WHEN isconverted_b = true THEN 1 END) * 100.0 / NULLIF(COUNT(mql_date), 0), 2) as mql_to_conversion_rate
FROM lead_with_qualification
WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
GROUP BY segment
ORDER BY total_leads DESC
```

### Lead Conversion Funnel

```sql
WITH latest_lead AS (
    SELECT A.* 
    FROM lakehouse.object_lead A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_lead B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
      AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    DATE_TRUNC('month', createddate_ts) as month,
    COUNT(*) as total_leads,
    -- Using lead status values from docs/configuration-reference.md
    COUNT(CASE WHEN status = 'Open' THEN 1 END) as mcl_count,
    COUNT(CASE WHEN status = 'Working' THEN 1 END) as mql_count,
    COUNT(CASE WHEN isconverted_b = true THEN 1 END) as converted_count,
    ROUND(COUNT(CASE WHEN status = 'Working' THEN 1 END) * 100.0 / NULLIF(COUNT(CASE WHEN status = 'Open' THEN 1 END), 0), 2) as mcl_to_mql_rate,
    ROUND(COUNT(CASE WHEN isconverted_b = true THEN 1 END) * 100.0 / NULLIF(COUNT(CASE WHEN status = 'Working' THEN 1 END), 0), 2) as mql_to_conversion_rate
FROM latest_lead
GROUP BY DATE_TRUNC('month', createddate_ts)
ORDER BY month DESC
```

## Opportunity Analysis Templates

### Opportunity Stage Progression

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
      AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    -- Using stage progression logic from docs/configuration-reference.md
    -- SQL: All opportunities except Closed Lost
    COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count,
    -- SQO: Reached SQO or beyond
    COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') THEN 1 END) as sqo_count,
    -- Proposal: Reached Proposal or beyond
    COUNT(CASE WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won') THEN 1 END) as proposal_count,
    -- Contracts: Reached Contracts or beyond
    COUNT(CASE WHEN stagename IN ('Contracts', 'Closed Won') THEN 1 END) as contracts_count,
    -- Closed Won: Successfully completed
    COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as closed_won_count,
    -- Closed Lost: Failed progression (tracked separately)
    COUNT(CASE WHEN stagename = 'Closed Lost' THEN 1 END) as closed_lost_count,
    -- Calculate conversion rates
    ROUND(COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') THEN 1 END) * 100.0 / 
          NULLIF(COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END), 0), 2) as sql_to_sqo_rate,
    ROUND(COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) * 100.0 / 
          NULLIF(COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END), 0), 2) as sql_to_won_rate
FROM latest_opportunity
```

### Pipeline Value Analysis

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
      -- Using active pipeline exclusion logic from docs/configuration-reference.md
      AND stagename NOT IN ('Closed Won', 'Closed Lost')
)
SELECT 
    stagename,
    COUNT(*) as opportunity_count,
    SUM(amount_f) as pipeline_value,
    AVG(amount_f) as avg_deal_size,
    AVG(probability_f) as avg_probability,
    SUM(amount_f * probability_f / 100) as weighted_value,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as stage_percentage
FROM latest_opportunity  
WHERE amount_f IS NOT NULL
GROUP BY stagename
ORDER BY 
    -- Using stage ordering from docs/configuration-reference.md
    CASE stagename
        WHEN 'SQL' THEN 1
        WHEN 'Proof of Value (SQO)' THEN 2  
        WHEN 'Proposal' THEN 3
        WHEN 'Contracts' THEN 4
        ELSE 5
    END
```

## Account Analysis Templates

### Customer Segmentation Analysis

```sql
WITH latest_account AS (
    SELECT A.* 
    FROM lakehouse.object_account A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_account B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
account_with_segmentation AS (
    SELECT 
        *,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN numberofemployees_f > 250 OR annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN numberofemployees_f > 50 OR annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as segment,
        CASE 
            WHEN numberofemployees_f IS NULL AND annualrevenue_f IS NULL THEN 'Unsegmentable'
            ELSE 'Segmentable'
        END as segmentation_status
    FROM latest_account
)
SELECT 
    segment,
    COUNT(*) as account_count,
    AVG(numberofemployees_f) as avg_employees,
    AVG(annualrevenue_f) as avg_revenue,
    COUNT(CASE WHEN segmentation_status = 'Segmentable' THEN 1 END) as segmentable_accounts,
    ROUND(COUNT(CASE WHEN segmentation_status = 'Segmentable' THEN 1 END) * 100.0 / COUNT(*), 2) as segmentable_pct
FROM account_with_segmentation
GROUP BY segment
ORDER BY account_count DESC
```

## Data Quality Templates

### Completeness Analysis

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
SELECT 
    COUNT(*) as total_records,
    -- Field completeness analysis
    COUNT([field1]) as [field1]_populated,
    ROUND(COUNT([field1]) * 100.0 / COUNT(*), 2) as [field1]_completeness_pct,
    COUNT([field2]_f) as [field2]_populated,
    ROUND(COUNT([field2]_f) * 100.0 / COUNT(*), 2) as [field2]_completeness_pct,
    COUNT([field3]_ts) as [field3]_populated,
    ROUND(COUNT([field3]_ts) * 100.0 / COUNT(*), 2) as [field3]_completeness_pct
FROM latest_[objectname]
```

### Duplicate Detection

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
),
email_duplicates AS (
    SELECT 
        email,
        COUNT(*) as duplicate_count,
        STRING_AGG(id, ', ') as duplicate_ids
    FROM latest_[objectname]
    WHERE email IS NOT NULL 
      AND email != ''
    GROUP BY email
    HAVING COUNT(*) > 1
)
SELECT 
    'Email Duplicates' as duplicate_type,
    COUNT(*) as duplicate_groups,
    SUM(duplicate_count - 1) as excess_records
FROM email_duplicates
```

## Monthly Trend Templates

### Complete Funnel Trend Analysis

```sql
WITH month_series AS (
    SELECT DATE_TRUNC('month', DATE_ADD('month', -month_offset, CURRENT_DATE)) as report_month
    FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11)) AS months(month_offset)
),
latest_lead AS (
    SELECT A.* 
    FROM lakehouse.object_lead A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_lead B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
latest_opportunity AS (
    SELECT A.* 
    FROM lakehouse.object_opportunity A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_opportunity B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
mcl_monthly AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as mcl_month,
        COUNT(*) as mcl_count
    FROM latest_lead
    -- Using lead status values from docs/configuration-reference.md
    WHERE status = 'Open'
    GROUP BY DATE_TRUNC('month', createddate_ts)
),
mql_monthly AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as mql_month,
        COUNT(*) as mql_count
    FROM latest_lead
    -- Using lead status values from docs/configuration-reference.md
    WHERE status = 'Working'
    GROUP BY DATE_TRUNC('month', createddate_ts)
),
sql_monthly AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as sql_month,
        COUNT(*) as sql_count
    FROM latest_opportunity
    -- Using stage exclusion logic from docs/configuration-reference.md
    WHERE stagename != 'Closed Lost'
    GROUP BY DATE_TRUNC('month', createddate_ts)
),
sqo_monthly AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as sqo_month,
        COUNT(*) as sqo_count
    FROM latest_opportunity
    -- Using stage progression logic from docs/configuration-reference.md
    WHERE stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won')
    GROUP BY DATE_TRUNC('month', createddate_ts)
)
SELECT 
    ms.report_month,
    COALESCE(mcl.mcl_count, 0) as mcl_count,
    COALESCE(mql.mql_count, 0) as mql_count,
    COALESCE(sql.sql_count, 0) as sql_count,
    COALESCE(sqo.sqo_count, 0) as sqo_count
FROM month_series ms
LEFT JOIN mcl_monthly mcl ON ms.report_month = mcl.mcl_month
LEFT JOIN mql_monthly mql ON ms.report_month = mql.mql_month
LEFT JOIN sql_monthly sql ON ms.report_month = sql.sql_month
LEFT JOIN sqo_monthly sqo ON ms.report_month = sqo.sqo_month
ORDER BY ms.report_month DESC
```

## Performance Optimization Templates

### Efficient Date Filtering

```sql
-- Template for performance-optimized date filtering
WITH latest_records AS (
    SELECT A.* 
    FROM lakehouse.object_[objectname] A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_[objectname] B 
        WHERE B.createddate_ts >= DATE_ADD('month', -[timeframe], CURRENT_DATE)
          AND B.grax__deleted IS NULL
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
      AND A.createddate_ts >= DATE_ADD('month', -[timeframe], CURRENT_DATE)
)
SELECT * FROM latest_records
```

### Early Filtering Pattern

```sql
-- Apply filters as early as possible in CTEs
WITH filtered_data AS (
    SELECT * 
    FROM lakehouse.object_[objectname]
    WHERE grax__deleted IS NULL
      AND createddate_ts >= DATE_ADD('month', -6, CURRENT_DATE)
      AND [other_filters]
),
latest_filtered AS (
    SELECT A.* 
    FROM filtered_data A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM filtered_data B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
)
SELECT * FROM latest_filtered
```

### Efficient Existence Checks

```sql
-- Use EXISTS instead of IN for better performance
WITH latest_parent AS (
    SELECT A.* 
    FROM lakehouse.object_parent A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_parent B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
      AND EXISTS (
          SELECT 1 FROM lakehouse.object_child c
          WHERE c.parent_id = A.id
            AND c.grax__deleted IS NULL
      )
)
SELECT * FROM latest_parent
```

## Validation Templates

### Record Count Validation

```sql
-- Validate data integrity across objects
SELECT 
    'object_lead' AS table_name,
    COUNT(*) AS total_records,
    COUNT(CASE WHEN grax__deleted IS NULL THEN 1 END) AS active_records,
    COUNT(CASE WHEN grax__deleted IS NOT NULL THEN 1 END) AS deleted_records
FROM lakehouse.object_lead

UNION ALL

SELECT 
    'object_opportunity' AS table_name,
    COUNT(*) AS total_records,
    COUNT(CASE WHEN grax__deleted IS NULL THEN 1 END) AS active_records,
    COUNT(CASE WHEN grax__deleted IS NOT NULL THEN 1 END) AS deleted_records
FROM lakehouse.object_opportunity

UNION ALL

SELECT 
    'object_account' AS table_name,
    COUNT(*) AS total_records,
    COUNT(CASE WHEN grax__deleted IS NULL THEN 1 END) AS active_records,
    COUNT(CASE WHEN grax__deleted IS NOT NULL THEN 1 END) AS deleted_records
FROM lakehouse.object_account
```

### Latest Records Validation

```sql
-- Verify latest records pattern is working correctly
WITH latest_check AS (
    SELECT 
        id,
        COUNT(*) as record_count,
        MAX(grax__idseq) as max_seq
    FROM lakehouse.object_[objectname]
    WHERE grax__deleted IS NULL
    GROUP BY id
    HAVING COUNT(*) > 1
)
SELECT 
    COUNT(*) AS ids_with_multiple_latest_records,
    CASE 
        WHEN COUNT(*) = 0 THEN 'VALID: Latest records pattern working correctly'
        ELSE 'ERROR: Multiple latest records found for some IDs'
    END as validation_result
FROM latest_check
```

## Configuration Adaptation

For organizations with different Salesforce implementations, modify the [Configuration Reference](./configuration-reference.md) document with your specific values:

### Critical Updates Needed

1. **Lead Status Values**: Update `'Open'`, `'Working'`, etc. to match your lead qualification stages
1. **Opportunity Stage Names**: Replace `'Proof of Value (SQO)'`, `'Proposal'`, etc. with your sales process stages  
1. **Account Types**: Update `'Customer'`, `'Customer - Subsidiary'`, `'Prospect'` with your account classification values
1. **Segmentation Thresholds**: Modify the 250 employee and $100M revenue thresholds for Enterprise classification
1. **SMB Thresholds**: Adjust the 50 employee and $10M revenue thresholds for SMB classification

### Testing Process

1. **Update Configuration**: Modify values in [Configuration Reference](./configuration-reference.md)
1. **Test Templates**: Execute sample queries from this document
1. **Validate Results**: Ensure data makes sense for your business context
1. **Document Changes**: Record customizations for future reference

This standardized approach ensures all query templates remain consistent and easily adaptable across different Salesforce implementations.
