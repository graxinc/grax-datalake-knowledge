# Comprehensive Pipeline Analysis

This analysis pattern provides a complete assessment of sales pipeline health, velocity, and performance metrics using GRAX's complete historical data to enable strategic decision-making.

## Executive Summary Query

Generate key pipeline metrics for executive reporting:

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
active_pipeline AS (
    SELECT 
        COUNT(*) as total_opportunities,
        SUM(amount_f) as total_pipeline_value,
        AVG(amount_f) as avg_deal_size,
        SUM(amount_f * probability_f / 100) as weighted_pipeline_value
    FROM latest_opportunity
    WHERE stagename NOT IN ('Closed Won', 'Closed Lost')
      AND amount_f IS NOT NULL
      AND amount_f > 0
      AND createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
),
closed_analysis AS (
    SELECT 
        COUNT(*) as total_closed,
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as won_count,
        AVG(DATE_DIFF('day', createddate_ts, CAST(closedate_d AS timestamp))) as avg_sales_cycle
    FROM latest_opportunity
    WHERE stagename IN ('Closed Won', 'Closed Lost')
      AND closedate_d >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    ap.total_opportunities,
    ROUND(ap.total_pipeline_value, 0) as total_pipeline_value,
    ROUND(ap.avg_deal_size, 0) as avg_deal_size,
    ROUND(ap.weighted_pipeline_value, 0) as weighted_pipeline_value,
    ROUND(ca.won_count * 100.0 / NULLIF(ca.total_closed, 0), 1) as overall_win_rate_pct,
    ROUND(ca.avg_sales_cycle, 0) as avg_sales_cycle_days
FROM active_pipeline ap
CROSS JOIN closed_analysis ca;
```

## Pipeline Health Assessment

Analyze current pipeline status with risk indicators:

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
pipeline_analysis AS (
    SELECT 
        o.id,
        o.name,
        o.stagename,
        o.amount_f,
        o.probability_f,
        o.createddate_ts,
        o.closedate_d,
        o.laststagechangedate_ts,
        -- Using segmentation from Configuration Reference
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment,
        -- Calculate opportunity age and stage duration
        DATE_DIFF('day', o.createddate_ts, CURRENT_TIMESTAMP) as opportunity_age_days,
        DATE_DIFF('day', o.laststagechangedate_ts, CURRENT_TIMESTAMP) as days_in_current_stage,
        -- Risk indicators
        CASE 
            WHEN DATE_DIFF('day', o.laststagechangedate_ts, CURRENT_TIMESTAMP) > 90 THEN 'Stalled'
            WHEN DATE_DIFF('day', o.createddate_ts, CURRENT_TIMESTAMP) > 365 THEN 'Aged'
            WHEN o.closedate_d < CURRENT_DATE AND o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN 'Overdue'
            ELSE 'Healthy'
        END as risk_status
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    WHERE o.stagename NOT IN ('Closed Won', 'Closed Lost')
      AND o.createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
      AND o.amount_f IS NOT NULL
      AND o.amount_f > 0
)
SELECT 
    stagename,
    customer_segment,
    risk_status,
    COUNT(*) as opportunity_count,
    ROUND(SUM(amount_f), 0) as pipeline_value,
    ROUND(AVG(amount_f), 0) as avg_deal_size,
    ROUND(SUM(amount_f * probability_f / 100), 0) as weighted_pipeline_value,
    ROUND(AVG(opportunity_age_days), 0) as avg_opportunity_age_days,
    ROUND(AVG(days_in_current_stage), 0) as avg_days_in_current_stage,
    ROUND(AVG(probability_f), 1) as avg_probability_pct
FROM pipeline_analysis
GROUP BY stagename, customer_segment, risk_status
-- Using stage ordering from Configuration Reference
ORDER BY 
    CASE stagename
        WHEN 'SQL' THEN 1
        WHEN 'Proof of Value (SQO)' THEN 2
        WHEN 'Proposal' THEN 3
        WHEN 'Contracts' THEN 4
        ELSE 5
    END,
    customer_segment,
    risk_status;
```

## Sales Funnel Analysis

Comprehensive funnel progression by customer segment:

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
      AND A.createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
      AND A.amount_f IS NOT NULL
      AND A.amount_f > 0
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
funnel_analysis AS (
    SELECT 
        -- Using segmentation thresholds from Configuration Reference
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment,
        
        -- SQL: All opportunities except Closed Lost
        COUNT(CASE WHEN o.stagename != 'Closed Lost' THEN 1 END) as sql_count,
        SUM(CASE WHEN o.stagename != 'Closed Lost' THEN o.amount_f ELSE 0 END) as sql_value,
        
        -- SQO: Reached SQO or beyond (excluding Closed Lost)
        COUNT(CASE WHEN o.stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') THEN 1 END) as sqo_count,
        SUM(CASE WHEN o.stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') THEN o.amount_f ELSE 0 END) as sqo_value,
        
        -- Proposal: Reached Proposal or beyond
        COUNT(CASE WHEN o.stagename IN ('Proposal', 'Contracts', 'Closed Won') THEN 1 END) as proposal_count,
        SUM(CASE WHEN o.stagename IN ('Proposal', 'Contracts', 'Closed Won') THEN o.amount_f ELSE 0 END) as proposal_value,
        
        -- Contracts: Reached Contracts or beyond
        COUNT(CASE WHEN o.stagename IN ('Contracts', 'Closed Won') THEN 1 END) as contracts_count,
        SUM(CASE WHEN o.stagename IN ('Contracts', 'Closed Won') THEN o.amount_f ELSE 0 END) as contracts_value,
        
        -- Closed Won: Successfully completed
        COUNT(CASE WHEN o.stagename = 'Closed Won' THEN 1 END) as closed_won_count,
        SUM(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f ELSE 0 END) as closed_won_value
        
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    GROUP BY 
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END
)
SELECT 
    customer_segment,
    sql_count,
    ROUND(sql_value, 0) as sql_value,
    sqo_count,
    ROUND(sqo_value, 0) as sqo_value,
    proposal_count,
    ROUND(proposal_value, 0) as proposal_value,
    contracts_count,
    ROUND(contracts_value, 0) as contracts_value,
    closed_won_count,
    ROUND(closed_won_value, 0) as closed_won_value,
    
    -- Conversion rates (count-based)
    ROUND(sqo_count * 100.0 / NULLIF(sql_count, 0), 1) as sql_to_sqo_rate_pct,
    ROUND(proposal_count * 100.0 / NULLIF(sqo_count, 0), 1) as sqo_to_proposal_rate_pct,
    ROUND(contracts_count * 100.0 / NULLIF(proposal_count, 0), 1) as proposal_to_contracts_rate_pct,
    ROUND(closed_won_count * 100.0 / NULLIF(contracts_count, 0), 1) as contracts_to_won_rate_pct,
    ROUND(closed_won_count * 100.0 / NULLIF(sql_count, 0), 1) as overall_win_rate_pct
    
FROM funnel_analysis
ORDER BY sql_count DESC;
```

## Sales Velocity Analysis

Historical performance trends with sales cycle analysis:

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
closed_opportunities AS (
    SELECT 
        o.id,
        o.stagename,
        o.amount_f,
        o.createddate_ts,
        o.closedate_d,
        DATE_DIFF('day', o.createddate_ts, CAST(o.closedate_d AS timestamp)) as sales_cycle_days,
        EXTRACT(year FROM o.closedate_d) as close_year,
        EXTRACT(quarter FROM o.closedate_d) as close_quarter,
        CASE 
            WHEN o.stagename = 'Closed Won' THEN 'Won'
            WHEN o.stagename = 'Closed Lost' THEN 'Lost'
            ELSE 'Other'
        END as outcome
    FROM latest_opportunity o
    WHERE o.stagename IN ('Closed Won', 'Closed Lost')
      AND o.closedate_d IS NOT NULL
      AND o.closedate_d >= DATE_ADD('month', -12, CURRENT_DATE)
      AND o.amount_f IS NOT NULL
      AND o.amount_f > 0
)
SELECT 
    close_year,
    close_quarter,
    outcome,
    COUNT(*) as deal_count,
    ROUND(SUM(amount_f), 0) as total_value,
    ROUND(AVG(amount_f), 0) as avg_deal_size,
    ROUND(AVG(sales_cycle_days), 0) as avg_sales_cycle_days,
    ROUND(MIN(sales_cycle_days), 0) as min_sales_cycle_days,
    ROUND(MAX(sales_cycle_days), 0) as max_sales_cycle_days,
    -- Win rate by period
    ROUND(
        SUM(CASE WHEN outcome = 'Won' THEN 1 ELSE 0 END) * 100.0 / 
        COUNT(*), 1
    ) as period_win_rate_pct
FROM closed_opportunities
GROUP BY close_year, close_quarter, outcome
ORDER BY close_year DESC, close_quarter DESC, outcome;
```

## Pipeline Forecasting

Project future performance based on current pipeline and velocity:

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
current_pipeline AS (
    SELECT 
        stagename,
        COUNT(*) as opportunity_count,
        SUM(amount_f) as pipeline_value,
        SUM(amount_f * probability_f / 100) as weighted_value,
        AVG(probability_f) as avg_probability
    FROM latest_opportunity
    WHERE stagename NOT IN ('Closed Won', 'Closed Lost')
      AND amount_f IS NOT NULL
      AND amount_f > 0
      AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    GROUP BY stagename
),
historical_conversion AS (
    SELECT 
        'SQL' as stage,
        COUNT(CASE WHEN stagename != 'Closed Lost' THEN 1 END) as entered_stage,
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as won_from_stage
    FROM latest_opportunity
    WHERE closedate_d >= DATE_ADD('month', -6, CURRENT_DATE)
      AND stagename IN ('Closed Won', 'Closed Lost')
    UNION ALL
    SELECT 
        'Proof of Value (SQO)' as stage,
        COUNT(CASE WHEN stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won') THEN 1 END),
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END)
    FROM latest_opportunity
    WHERE closedate_d >= DATE_ADD('month', -6, CURRENT_DATE)
      AND stagename IN ('Closed Won', 'Closed Lost')
    UNION ALL
    SELECT 
        'Proposal' as stage,
        COUNT(CASE WHEN stagename IN ('Proposal', 'Contracts', 'Closed Won') THEN 1 END),
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END)
    FROM latest_opportunity
    WHERE closedate_d >= DATE_ADD('month', -6, CURRENT_DATE)
      AND stagename IN ('Closed Won', 'Closed Lost')
    UNION ALL
    SELECT 
        'Contracts' as stage,
        COUNT(CASE WHEN stagename IN ('Contracts', 'Closed Won') THEN 1 END),
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END)
    FROM latest_opportunity
    WHERE closedate_d >= DATE_ADD('month', -6, CURRENT_DATE)
      AND stagename IN ('Closed Won', 'Closed Lost')
)
SELECT 
    cp.stagename,
    cp.opportunity_count,
    ROUND(cp.pipeline_value, 0) as pipeline_value,
    ROUND(cp.weighted_value, 0) as weighted_value,
    ROUND(cp.avg_probability, 1) as avg_probability_pct,
    ROUND(hc.won_from_stage * 100.0 / NULLIF(hc.entered_stage, 0), 1) as historical_win_rate_pct,
    -- Forecasted wins based on historical conversion
    ROUND(cp.opportunity_count * (hc.won_from_stage * 1.0 / NULLIF(hc.entered_stage, 0)), 0) as forecasted_wins,
    -- Forecasted revenue based on historical conversion
    ROUND(cp.pipeline_value * (hc.won_from_stage * 1.0 / NULLIF(hc.entered_stage, 0)), 0) as forecasted_revenue
FROM current_pipeline cp
LEFT JOIN historical_conversion hc ON cp.stagename = hc.stage
-- Using stage ordering from Configuration Reference
ORDER BY 
    CASE cp.stagename
        WHEN 'SQL' THEN 1
        WHEN 'Proof of Value (SQO)' THEN 2
        WHEN 'Proposal' THEN 3
        WHEN 'Contracts' THEN 4
        ELSE 5
    END;
```

## Risk Assessment

Identify at-risk opportunities requiring immediate attention:

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
risk_assessment AS (
    SELECT 
        o.id,
        o.name,
        a.name as account_name,
        o.stagename,
        o.amount_f,
        o.probability_f,
        o.closedate_d,
        DATE_DIFF('day', o.createddate_ts, CURRENT_TIMESTAMP) as opportunity_age_days,
        DATE_DIFF('day', o.laststagechangedate_ts, CURRENT_TIMESTAMP) as days_in_current_stage,
        -- Risk scoring
        CASE 
            WHEN DATE_DIFF('day', o.laststagechangedate_ts, CURRENT_TIMESTAMP) > 120 THEN 'Critical - Stalled >4 months'
            WHEN DATE_DIFF('day', o.laststagechangedate_ts, CURRENT_TIMESTAMP) > 90 THEN 'High - Stalled >3 months'
            WHEN o.closedate_d < CURRENT_DATE AND o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN 'High - Past Due'
            WHEN DATE_DIFF('day', o.createddate_ts, CURRENT_TIMESTAMP) > 365 THEN 'Medium - Aged >1 year'
            WHEN DATE_DIFF('day', o.laststagechangedate_ts, CURRENT_TIMESTAMP) > 60 THEN 'Medium - Stalled >2 months'
            ELSE 'Low'
        END as risk_level
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    WHERE o.stagename NOT IN ('Closed Won', 'Closed Lost')
      AND o.amount_f IS NOT NULL
      AND o.amount_f > 0
)
SELECT 
    id,
    name as opportunity_name,
    account_name,
    stagename,
    ROUND(amount_f, 0) as opportunity_value,
    probability_f as probability_pct,
    closedate_d as expected_close_date,
    opportunity_age_days,
    days_in_current_stage,
    risk_level
FROM risk_assessment
WHERE risk_level != 'Low'
ORDER BY 
    CASE risk_level
        WHEN 'Critical - Stalled >4 months' THEN 1
        WHEN 'High - Past Due' THEN 2
        WHEN 'High - Stalled >3 months' THEN 3
        WHEN 'Medium - Aged >1 year' THEN 4
        WHEN 'Medium - Stalled >2 months' THEN 5
        ELSE 6
    END,
    amount_f DESC;
```

## Configuration Adaptation

For organizations with different Salesforce implementations, update the [Configuration Reference](/docs/core-reference/configuration-reference.md) document with your specific values:

- **Opportunity Stage Names**: Update stage names to match your sales process stages
- **Segmentation Thresholds**: Adjust employee count and revenue thresholds for Enterprise/SMB/Self-Service classification
- **Risk Thresholds**: Modify time-based risk indicators (30, 60, 90+ days) to match your sales cycle expectations
- **Field Names**: Update if using custom field names for amounts, probabilities, or dates

## Analysis Methodology

### Historical Intelligence Approach

This comprehensive analysis leverages GRAX's complete opportunity change history to provide insights that traditional CRM reporting cannot deliver:

1. **True Conversion Rates**: Based on complete funnel progression history, not just current snapshots
1. **Accurate Sales Cycles**: Calculated from actual opportunity creation to close dates across full lifecycle
1. **Risk Prediction**: Identifies stalled opportunities using historical pattern recognition
1. **Velocity Trends**: Shows how sales performance changes over time with complete context

### Strategic Framework

**Enable "Adapt Faster" Decision Making**:

```text
Current Pipeline Assessment → Historical Performance Analysis → Risk Identification → Forecasted Outcomes → Strategic Recommendations
```

This comprehensive pipeline analysis enables organizations to make data-driven decisions faster than competitors by leveraging complete historical intelligence for strategic advantage.
