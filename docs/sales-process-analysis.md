# Sales Process Analysis

## Sales Stage Definitions

These definitions use the standardized configuration values from [Configuration Reference](./configuration-reference.md). Organizations with different Salesforce configurations should update the Configuration Reference document to match their specific values.

### Lead Qualification Stages

#### MCL (Marketing Contacted Lead)

**Definition**: Lead that has demonstrated initial interest and meets basic qualification criteria.

**Qualification Criteria** (from [Configuration Reference](./configuration-reference.md)):

- Lead Status = `'Open'` (as defined in [Lead Status Configuration](./configuration-reference.md#lead-status-configuration))
- **Date Assignment**: Always use lead creation date (`createddate_ts`)

**Query Pattern**:

```sql
-- Using configuration values from docs/configuration-reference.md
CASE WHEN status = 'Open' THEN createddate_ts ELSE NULL END as mcl_date
```

#### MQL (Marketing Qualified Lead)

**Definition**: Lead qualified by marketing with demonstrated interest and fit for sales engagement.

**Qualification Criteria** (from [Configuration Reference](./configuration-reference.md)):

1. **Primary**: Lead Status = `'Working'`

1. **Fallback**: Lead converted but never achieved `'Working'` status

**Date Assignment Logic**:

```sql
-- Using lead status values from docs/configuration-reference.md
CASE 
    WHEN status = 'Working' THEN createddate_ts
    WHEN isconverted_b = true THEN CAST(converteddate_d AS timestamp)
    ELSE NULL
END as mql_date
```

### Opportunity Stages

These stages follow the progression logic defined in [Opportunity Stage Configuration](./configuration-reference.md#opportunity-stage-configuration).

#### SQL (Sales Qualified Lead)

**Definition**: Opportunity qualified by sales with defined business potential.

**Qualification Criteria**:

- **ALL opportunities are SQL by definition at creation**
- **Excludes**: `'Closed Lost'` opportunities (as defined in Configuration Reference)

**Date Assignment**: Always use opportunity creation date

```sql
createddate_ts as sql_date
```

**Count Logic**:

```sql
-- Using stage exclusion logic from docs/configuration-reference.md
COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count
```

#### SQO (Sales Qualified Opportunity)

**Definition**: Opportunity where proof of value has been demonstrated.

**Qualification Criteria** (from [Configuration Reference](./configuration-reference.md)):

- Current stage = `'Proof of Value (SQO)'` OR beyond
- **Excludes**: `'Closed Lost'` opportunities

**Date Assignment Logic**:

```sql
-- Using stage names from docs/configuration-reference.md
CASE 
    WHEN stagename = 'Proof of Value (SQO)' THEN laststagechangedate_ts
    WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won') THEN laststagechangedate_ts
    ELSE NULL
END as sqo_date
```

**Count Logic**:

```sql
-- Using stage progression logic from docs/configuration-reference.md
COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') 
      THEN 1 END) as sqo_count
```

#### Proposal Stage

**Definition**: Formal proposal delivered with detailed solution and pricing.

**Qualification Criteria**:

- Current stage = `'Proposal'` OR beyond (as defined in Configuration Reference)
- **Excludes**: `'Closed Lost'` opportunities

**Count Logic**:

```sql
-- Using stage names from docs/configuration-reference.md
COUNT(CASE WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won')
      THEN 1 END) as proposal_count
```

#### Contracts Stage

**Definition**: Contract negotiation and legal review phase.

**Qualification Criteria**:

- Current stage = `'Contracts'` OR `'Closed Won'`

**Count Logic**:

```sql
-- Using stage names from docs/configuration-reference.md
COUNT(CASE WHEN stagename IN ('Contracts', 'Closed Won')
      THEN 1 END) as contracts_count
```

#### Closed Won

**Definition**: Successfully closed opportunity with executed contracts.

**Count Logic**:

```sql
-- Using stage name from docs/configuration-reference.md
COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as closed_won_count
```

## Sequential Stage Progression Logic

### Fundamental Rule

Every **ACTIVE** opportunity must be counted in all stages it has logically passed through. **`'Closed Lost'` opportunities are excluded from progressive stage counts** as they represent failed progression (per [Stage Progression Logic](./configuration-reference.md#stage-progression-logic)).

### Progression Flow

```text
MCL → MQL → SQL → SQO → Proposal → Contracts → Closed Won
                                               ↘ Closed Lost (EXCLUDED)
```

### Complete Sequential Funnel Query

```sql
-- Using latest records pattern and configuration values from docs/configuration-reference.md
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
lead_metrics AS (
    SELECT 
        -- Using lead status values from docs/configuration-reference.md
        COUNT(CASE WHEN status = 'Open' THEN 1 END) as mcl_count,
        COUNT(CASE 
            WHEN status = 'Working' THEN 1
            WHEN isconverted_b = true THEN 1
        END) as mql_count
    FROM latest_lead
    WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
opportunity_metrics AS (
    SELECT 
        -- Using stage progression logic from docs/configuration-reference.md
        -- SQL: All opportunities except Closed Lost
        COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count,
        -- SQO: Reached SQO or beyond (excluding Closed Lost)
        COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') 
              THEN 1 END) as sqo_count,
        -- Proposal: Reached Proposal or beyond
        COUNT(CASE WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won')
              THEN 1 END) as proposal_count,
        -- Contracts: Reached Contracts or beyond
        COUNT(CASE WHEN stagename IN ('Contracts', 'Closed Won')
              THEN 1 END) as contracts_count,
        -- Closed Won: Successfully completed
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as closed_won_count,
        -- Closed Lost: Failed progression (tracked separately)
        COUNT(CASE WHEN stagename = 'Closed Lost' THEN 1 END) as closed_lost_count
    FROM latest_opportunity
    WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    lm.mcl_count,
    lm.mql_count,
    om.sql_count,
    om.sqo_count,
    om.proposal_count,
    om.contracts_count,
    om.closed_won_count,
    om.closed_lost_count,
    -- Conversion rates
    ROUND(lm.mql_count * 100.0 / NULLIF(lm.mcl_count, 0), 2) as mcl_to_mql_rate,
    ROUND(om.sqo_count * 100.0 / NULLIF(om.sql_count, 0), 2) as sql_to_sqo_rate,
    ROUND(om.closed_won_count * 100.0 / NULLIF(om.sql_count, 0), 2) as sql_to_won_rate
FROM lead_metrics lm
CROSS JOIN opportunity_metrics om
```

## Validation Queries

### Sequential Validation Check

```sql
-- This query validates the stage progression logic from docs/configuration-reference.md
WITH funnel_counts AS (
    SELECT 
        -- Using stage names and progression logic from configuration reference
        COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count,
        COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') THEN 1 END) as sqo_count,
        COUNT(CASE WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won') THEN 1 END) as proposal_count,
        COUNT(CASE WHEN stagename IN ('Contracts', 'Closed Won') THEN 1 END) as contracts_count,
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as closed_won_count
    FROM (
        SELECT A.* 
        FROM lakehouse.object_opportunity A 
        INNER JOIN (
            SELECT B.Id, MAX(B.grax__idseq) AS Latest 
            FROM lakehouse.object_opportunity B 
            GROUP BY B.Id
        ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
        WHERE A.grax__deleted IS NULL
          AND A.createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    ) latest_opportunity
)
SELECT 
    sql_count,
    sqo_count,
    proposal_count,
    contracts_count,
    closed_won_count,
    -- Validation checks
    CASE 
        WHEN sqo_count > sql_count THEN 'ERROR: SQO > SQL'
        WHEN proposal_count > sqo_count THEN 'ERROR: Proposal > SQO' 
        WHEN contracts_count > proposal_count THEN 'ERROR: Contracts > Proposal'
        WHEN closed_won_count > contracts_count THEN 'ERROR: Closed Won > Contracts'
        ELSE 'VALID: Sequential logic maintained'
    END as validation_result
FROM funnel_counts
```

## Velocity Analysis

### Lead Velocity Calculation

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
lead_with_dates AS (
    SELECT 
        *,
        createddate_ts as mcl_date,
        -- Using lead status logic from docs/configuration-reference.md
        CASE 
            WHEN status = 'Working' THEN createddate_ts
            WHEN isconverted_b = true THEN CAST(converteddate_d AS timestamp)
            ELSE NULL
        END as mql_date
    FROM latest_lead
    WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    AVG(DATE_DIFF('day', mcl_date, mql_date)) as avg_mcl_to_mql_days,
    AVG(DATE_DIFF('day', createddate_ts, CAST(converteddate_d AS timestamp))) as avg_lead_to_conversion_days
FROM lead_with_dates
WHERE mql_date IS NOT NULL
```

### Opportunity Velocity Calculation

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
opportunity_with_dates AS (
    SELECT 
        *,
        createddate_ts as sql_date,
        -- Using stage names from docs/configuration-reference.md
        CASE 
            WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') 
            THEN laststagechangedate_ts
            ELSE NULL
        END as sqo_date
    FROM latest_opportunity
    WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
      AND stagename != 'Closed Lost'  -- Using exclusion logic from configuration reference
)
SELECT 
    AVG(DATE_DIFF('day', sql_date, sqo_date)) as avg_sql_to_sqo_days,
    AVG(DATE_DIFF('day', createddate_ts, CAST(closedate_d AS timestamp))) as avg_sql_to_close_days
FROM opportunity_with_dates
WHERE sqo_date IS NOT NULL
  AND stagename = 'Closed Won'
```

## Customer Segmentation Integration

### Funnel Analysis by Segment

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
lead_segmentation AS (
    SELECT 
        l.*,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN l.numberofemployees_f > 250 OR l.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN l.numberofemployees_f > 50 OR l.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as segment
    FROM latest_lead l
    WHERE l.createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
opportunity_segmentation AS (
    SELECT 
        o.*,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as segment
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    WHERE o.createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    ls.segment,
    -- Using lead status values from docs/configuration-reference.md
    COUNT(CASE WHEN ls.status = 'Open' THEN 1 END) as mcl_count,
    COUNT(CASE WHEN ls.status = 'Working' OR ls.isconverted_b = true THEN 1 END) as mql_count,
    -- Using stage names and progression logic from docs/configuration-reference.md
    COUNT(CASE WHEN os.stagename != 'Closed Lost' THEN 1 END) as sql_count,
    COUNT(CASE WHEN os.stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') THEN 1 END) as sqo_count,
    COUNT(CASE WHEN os.stagename = 'Closed Won' THEN 1 END) as closed_won_count,
    -- Conversion rates by segment
    ROUND(COUNT(CASE WHEN ls.status = 'Working' OR ls.isconverted_b = true THEN 1 END) * 100.0 / 
          NULLIF(COUNT(CASE WHEN ls.status = 'Open' THEN 1 END), 0), 2) as mcl_to_mql_rate,
    ROUND(COUNT(CASE WHEN os.stagename = 'Closed Won' THEN 1 END) * 100.0 / 
          NULLIF(COUNT(CASE WHEN os.stagename != 'Closed Lost' THEN 1 END), 0), 2) as sql_to_won_rate
FROM lead_segmentation ls
LEFT JOIN opportunity_segmentation os ON ls.segment = os.segment
GROUP BY ls.segment
ORDER BY mcl_count DESC
```

## Active Pipeline Analysis

### Current Pipeline Health

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
      -- Using stage exclusion logic from docs/configuration-reference.md
      AND A.stagename NOT IN ('Closed Won', 'Closed Lost')
)
SELECT 
    stagename as current_stage,
    COUNT(*) as opportunity_count,
    SUM(amount_f) as pipeline_value,
    AVG(amount_f) as avg_deal_size,
    AVG(probability_f) as avg_probability,
    SUM(amount_f * probability_f / 100) as weighted_value,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) as stage_percentage,
    -- Age analysis
    AVG(DATE_DIFF('day', createddate_ts, CURRENT_DATE)) as avg_age_days
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

## Configuration Adaptation

For organizations with different Salesforce implementations, update the [Configuration Reference](./configuration-reference.md) document with your specific values:

- **Lead Status Values**: Update the lead qualification stage values
- **Opportunity Stage Names**: Modify stage names to match your sales process  
- **Segmentation Thresholds**: Adjust employee/revenue thresholds for company sizing
- **Customer Types**: Update account classification values

This comprehensive sales process analysis framework ensures consistent, accurate tracking of lead progression and opportunity advancement while maintaining proper sequential logic and excluding failed progressions from advancement metrics.
