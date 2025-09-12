# Sales Process Analysis

## Sales Stage Definitions

These definitions use the standardized configuration values from [Configuration Reference](../core-reference/configuration-reference.md). Organizations with different Salesforce configurations should update the Configuration Reference document to match their specific values.

### Lead Qualification Stages

#### MCL (Marketing Contacted Lead)

**Definition**: Lead that has demonstrated initial interest and meets basic qualification criteria.

**Qualification Criteria** (from [Configuration Reference](../core-reference/configuration-reference.md)):

- Lead Status = `'Open - Not Contacted'` (as defined in Lead Status Configuration)
- **Date Assignment**: Always use lead creation date (`createddate_ts`)

**Query Pattern**:

```sql
-- Using configuration values from Configuration Reference
CASE WHEN status = 'Open - Not Contacted' THEN createddate_ts ELSE NULL END as mcl_date
```

#### MQL (Marketing Qualified Lead)

**Definition**: Lead qualified by marketing with demonstrated interest and fit for sales engagement.

**Qualification Criteria** (from [Configuration Reference](../core-reference/configuration-reference.md)):

1. **Primary**: Lead Status = `'MQL'`

1. **Fallback**: Lead converted but never achieved `'MQL'` status

**Date Assignment Logic**:

```sql
-- Using lead status values from Configuration Reference
CASE 
    WHEN status = 'MQL' THEN createddate_ts
    WHEN isconverted_b = true THEN CAST(converteddate_d AS timestamp)
    ELSE NULL
END as mql_date
```

### Opportunity Stages

These stages follow the progression logic defined in [Opportunity Stage Configuration](../core-reference/configuration-reference.md#opportunity-stage-values).

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
-- Using stage exclusion logic from Configuration Reference
COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count
```

#### Development Stage

**Definition**: Opportunity in active development with needs analysis and value proposition work.

**Qualification Criteria** (from [Configuration Reference](../core-reference/configuration-reference.md)):

- Current stage = `'Needs Analysis'`, `'Value Proposition'`, or beyond
- **Excludes**: `'Closed Lost'` opportunities

**Count Logic**:

```sql
-- Using stage progression logic from Configuration Reference
COUNT(CASE WHEN stagename IN ('Needs Analysis', 'Value Proposition', 'Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won') 
      THEN 1 END) as development_count
```

#### Advanced Stage

**Definition**: Opportunity in advanced stages with decision maker identification and perception analysis.

**Qualification Criteria**:

- Current stage = `'Id. Decision Makers'`, `'Perception Analysis'`, or beyond
- **Excludes**: `'Closed Lost'` opportunities

**Count Logic**:

```sql
-- Using stage names from Configuration Reference
COUNT(CASE WHEN stagename IN ('Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won')
      THEN 1 END) as advanced_count
```

#### Proposal Stage

**Definition**: Formal proposal delivered with detailed solution and pricing.

**Qualification Criteria**:

- Current stage = `'Proposal/Price Quote'` OR beyond (as defined in Configuration Reference)
- **Excludes**: `'Closed Lost'` opportunities

**Count Logic**:

```sql
-- Using stage names from Configuration Reference
COUNT(CASE WHEN stagename IN ('Proposal/Price Quote', 'Negotiation/Review', 'Closed Won')
      THEN 1 END) as proposal_count
```

#### Negotiation Stage

**Definition**: Contract negotiation and final review phase.

**Qualification Criteria**:

- Current stage = `'Negotiation/Review'` OR `'Closed Won'`

**Count Logic**:

```sql
-- Using stage names from Configuration Reference
COUNT(CASE WHEN stagename IN ('Negotiation/Review', 'Closed Won')
      THEN 1 END) as negotiation_count
```

#### Closed Won

**Definition**: Successfully closed opportunity with executed contracts.

**Count Logic**:

```sql
-- Using stage name from Configuration Reference
COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as closed_won_count
```

## Sequential Stage Progression Logic

### Fundamental Rule

Every **ACTIVE** opportunity must be counted in all stages it has logically passed through. **`'Closed Lost'` opportunities are excluded from progressive stage counts** as they represent failed progression (per [Stage Progression Logic](../core-reference/configuration-reference.md#opportunity-stage-values)).

### Progression Flow

```text
MCL → MQL → SQL → Development → Advanced → Proposal → Negotiation → Closed Won
                                                                   ↘ Closed Lost (EXCLUDED)
```

### Complete Sequential Funnel Query

```sql
-- Using latest records pattern and configuration values from Configuration Reference
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
        -- Using lead status values from Configuration Reference
        COUNT(CASE WHEN status = 'Open - Not Contacted' THEN 1 END) as mcl_count,
        COUNT(CASE 
            WHEN status = 'MQL' THEN 1
            WHEN isconverted_b = true THEN 1
        END) as mql_count
    FROM latest_lead
    WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
opportunity_metrics AS (
    SELECT 
        -- Using stage progression logic from Configuration Reference
        -- SQL: All opportunities except Closed Lost
        COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count,
        -- Development: Reached development stages or beyond (excluding Closed Lost)
        COUNT(CASE WHEN stagename IN ('Needs Analysis', 'Value Proposition', 'Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won') 
              THEN 1 END) as development_count,
        -- Advanced: Reached advanced stages or beyond
        COUNT(CASE WHEN stagename IN ('Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won')
              THEN 1 END) as advanced_count,
        -- Proposal: Reached Proposal or beyond
        COUNT(CASE WHEN stagename IN ('Proposal/Price Quote', 'Negotiation/Review', 'Closed Won')
              THEN 1 END) as proposal_count,
        -- Negotiation: Reached Negotiation or beyond
        COUNT(CASE WHEN stagename IN ('Negotiation/Review', 'Closed Won')
              THEN 1 END) as negotiation_count,
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
    om.development_count,
    om.advanced_count,
    om.proposal_count,
    om.negotiation_count,
    om.closed_won_count,
    om.closed_lost_count,
    -- Conversion rates
    ROUND(lm.mql_count * 100.0 / NULLIF(lm.mcl_count, 0), 2) as mcl_to_mql_rate,
    ROUND(om.development_count * 100.0 / NULLIF(om.sql_count, 0), 2) as sql_to_development_rate,
    ROUND(om.closed_won_count * 100.0 / NULLIF(om.sql_count, 0), 2) as sql_to_won_rate
FROM lead_metrics lm
CROSS JOIN opportunity_metrics om
```

## Validation Queries

### Sequential Validation Check

```sql
-- This query validates the stage progression logic from Configuration Reference
WITH funnel_counts AS (
    SELECT 
        -- Using stage names and progression logic from configuration reference
        COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as sql_count,
        COUNT(CASE WHEN stagename IN ('Needs Analysis', 'Value Proposition', 'Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won') THEN 1 END) as development_count,
        COUNT(CASE WHEN stagename IN ('Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won') THEN 1 END) as advanced_count,
        COUNT(CASE WHEN stagename IN ('Proposal/Price Quote', 'Negotiation/Review', 'Closed Won') THEN 1 END) as proposal_count,
        COUNT(CASE WHEN stagename IN ('Negotiation/Review', 'Closed Won') THEN 1 END) as negotiation_count,
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
    development_count,
    advanced_count,
    proposal_count,
    negotiation_count,
    closed_won_count,
    -- Validation checks
    CASE 
        WHEN development_count > sql_count THEN 'ERROR: Development > SQL'
        WHEN advanced_count > development_count THEN 'ERROR: Advanced > Development' 
        WHEN proposal_count > advanced_count THEN 'ERROR: Proposal > Advanced'
        WHEN negotiation_count > proposal_count THEN 'ERROR: Negotiation > Proposal'
        WHEN closed_won_count > negotiation_count THEN 'ERROR: Closed Won > Negotiation'
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
        -- Using lead status logic from Configuration Reference
        CASE 
            WHEN status = 'MQL' THEN createddate_ts
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
        -- Using stage names from Configuration Reference
        CASE 
            WHEN stagename IN ('Needs Analysis', 'Value Proposition', 'Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won') 
            THEN laststagechangedate_ts
            ELSE NULL
        END as development_date
    FROM latest_opportunity
    WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
      AND stagename != 'Closed Lost'  -- Using exclusion logic from configuration reference
)
SELECT 
    AVG(DATE_DIFF('day', sql_date, development_date)) as avg_sql_to_development_days,
    AVG(DATE_DIFF('day', createddate_ts, CAST(closedate_d AS timestamp))) as avg_sql_to_close_days
FROM opportunity_with_dates
WHERE development_date IS NOT NULL
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
        -- Using segmentation thresholds from Configuration Reference
        CASE 
            WHEN l.numberofemployees_f >= 1000 OR l.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN l.numberofemployees_f BETWEEN 200 AND 999 OR l.annualrevenue_f BETWEEN 10000000 AND 100000000 THEN 'Mid-Market'
            ELSE 'SMB'
        END as segment
    FROM latest_lead l
    WHERE l.createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
opportunity_segmentation AS (
    SELECT 
        o.*,
        -- Using segmentation thresholds from Configuration Reference
        CASE 
            WHEN a.numberofemployees_f >= 1000 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f BETWEEN 200 AND 999 OR a.annualrevenue_f BETWEEN 10000000 AND 100000000 THEN 'Mid-Market'
            ELSE 'SMB'
        END as segment
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    WHERE o.createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    ls.segment,
    -- Using lead status values from Configuration Reference
    COUNT(CASE WHEN ls.status = 'Open - Not Contacted' THEN 1 END) as mcl_count,
    COUNT(CASE WHEN ls.status = 'MQL' OR ls.isconverted_b = true THEN 1 END) as mql_count,
    -- Using stage names and progression logic from Configuration Reference
    COUNT(CASE WHEN os.stagename != 'Closed Lost' THEN 1 END) as sql_count,
    COUNT(CASE WHEN os.stagename IN ('Needs Analysis', 'Value Proposition', 'Id. Decision Makers', 'Perception Analysis', 'Proposal/Price Quote', 'Negotiation/Review', 'Closed Won') THEN 1 END) as development_count,
    COUNT(CASE WHEN os.stagename = 'Closed Won' THEN 1 END) as closed_won_count,
    -- Conversion rates by segment
    ROUND(COUNT(CASE WHEN ls.status = 'MQL' OR ls.isconverted_b = true THEN 1 END) * 100.0 / 
          NULLIF(COUNT(CASE WHEN ls.status = 'Open - Not Contacted' THEN 1 END), 0), 2) as mcl_to_mql_rate,
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
      -- Using stage exclusion logic from Configuration Reference
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
    -- Using stage ordering from Configuration Reference
    CASE stagename
        WHEN 'Prospecting' THEN 1
        WHEN 'Qualification' THEN 2
        WHEN 'Needs Analysis' THEN 3  
        WHEN 'Value Proposition' THEN 4
        WHEN 'Id. Decision Makers' THEN 5
        WHEN 'Perception Analysis' THEN 6
        WHEN 'Proposal/Price Quote' THEN 7
        WHEN 'Negotiation/Review' THEN 8
        ELSE 9
    END
```

## Configuration Adaptation

For organizations with different Salesforce implementations, update the [Configuration Reference](../core-reference/configuration-reference.md) document with your specific values:

- **Lead Status Values**: Update the lead qualification stage values
- **Opportunity Stage Names**: Modify stage names to match your sales process  
- **Segmentation Thresholds**: Adjust employee/revenue thresholds for company sizing
- **Customer Types**: Update account classification values

This comprehensive sales process analysis framework ensures consistent, accurate tracking of lead progression and opportunity advancement while maintaining proper sequential logic and excluding failed progressions from advancement metrics.
