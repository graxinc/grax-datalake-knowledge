# Business Intelligence Patterns

Advanced analytics and reporting templates that transform GRAX Data Lake information into strategic business intelligence for executive decision-making.

**Configuration Integration**: All patterns reference standardized values from [Configuration Reference](../core-reference/configuration-reference.md). Organizations should update that centralized document rather than modifying individual queries.

## Executive Dashboard Patterns

### Key Performance Indicators (KPI) Overview

```sql
WITH latest_lead AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
latest_account AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_account
    WHERE grax__deleted IS NULL
)
SELECT 
    -- Lead Generation Metrics
    COUNT(l.id) as total_leads_12m,
    COUNT(CASE WHEN l.status = 'MQL' THEN 1 END) as mql_count,
    ROUND(COUNT(CASE WHEN l.status = 'MQL' THEN 1 END) * 100.0 / NULLIF(COUNT(l.id), 0), 2) as lead_to_mql_rate,
    
    -- Pipeline Metrics
    COUNT(o.id) as total_opportunities_12m,
    SUM(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN o.amount_f ELSE 0 END) as active_pipeline_value,
    SUM(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f ELSE 0 END) as closed_won_revenue,
    
    -- Conversion Metrics
    COUNT(CASE WHEN l.isconverted_b = true THEN 1 END) as lead_conversions,
    COUNT(CASE WHEN o.stagename = 'Closed Won' THEN 1 END) as won_opportunities,
    ROUND(COUNT(CASE WHEN o.stagename = 'Closed Won' THEN 1 END) * 100.0 / 
          NULLIF(COUNT(CASE WHEN o.stagename != 'Closed Lost' THEN 1 END), 0), 2) as win_rate,
    
    -- Account Growth
    COUNT(DISTINCT a.id) as total_active_accounts
FROM latest_lead l
FULL OUTER JOIN latest_opportunity o ON l.convertedopportunityid = o.id AND o.rn = 1
FULL OUTER JOIN latest_account a ON COALESCE(l.convertedaccountid, o.accountid) = a.id AND a.rn = 1
WHERE l.rn = 1
```

### Monthly Trend Analysis

```sql
WITH month_series AS (
    SELECT 
        DATE_TRUNC('month', DATE_ADD('month', -month_offset, CURRENT_DATE)) as report_month,
        month_offset
    FROM (VALUES (0),(1),(2),(3),(4),(5),(6),(7),(8),(9),(10),(11)) AS months(month_offset)
),
latest_lead AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
),
lead_metrics AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as metric_month,
        COUNT(*) as leads_created,
        COUNT(CASE WHEN status = 'MQL' THEN 1 END) as mqls_created,
        COUNT(CASE WHEN isconverted_b = true THEN 1 END) as leads_converted
    FROM latest_lead
    WHERE rn = 1
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    GROUP BY DATE_TRUNC('month', createddate_ts)
),
opportunity_metrics AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as metric_month,
        COUNT(*) as opportunities_created,
        SUM(amount_f) as pipeline_created,
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as deals_won,
        SUM(CASE WHEN stagename = 'Closed Won' THEN amount_f ELSE 0 END) as revenue_won
    FROM latest_opportunity
    WHERE rn = 1
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    GROUP BY DATE_TRUNC('month', createddate_ts)
)
SELECT 
    ms.report_month,
    COALESCE(lm.leads_created, 0) as leads_created,
    COALESCE(lm.mqls_created, 0) as mqls_created,
    COALESCE(lm.leads_converted, 0) as leads_converted,
    COALESCE(om.opportunities_created, 0) as opportunities_created,
    COALESCE(om.pipeline_created, 0) as pipeline_created,
    COALESCE(om.deals_won, 0) as deals_won,
    COALESCE(om.revenue_won, 0) as revenue_won,
    -- Calculate month-over-month growth
    ROUND((COALESCE(lm.leads_created, 0) - 
           LAG(COALESCE(lm.leads_created, 0), 1) OVER (ORDER BY ms.report_month)) * 100.0 / 
          NULLIF(LAG(COALESCE(lm.leads_created, 0), 1) OVER (ORDER BY ms.report_month), 0), 2) as lead_growth_mom
FROM month_series ms
LEFT JOIN lead_metrics lm ON ms.report_month = lm.metric_month
LEFT JOIN opportunity_metrics om ON ms.report_month = om.metric_month
ORDER BY ms.report_month DESC
```

## Segmentation Analysis Patterns

### Customer Segment Performance

```sql
WITH latest_account AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn,
        -- Using segmentation thresholds from Configuration Reference
        CASE 
            WHEN numberofemployees_f >= 1000 OR annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN numberofemployees_f BETWEEN 200 AND 999 OR annualrevenue_f BETWEEN 10000000 AND 100000000 THEN 'Mid-Market'
            ELSE 'SMB'
        END as customer_segment
    FROM lakehouse.object_account
    WHERE grax__deleted IS NULL
),
latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
segment_performance AS (
    SELECT 
        a.customer_segment,
        COUNT(DISTINCT a.id) as account_count,
        COUNT(o.id) as opportunity_count,
        SUM(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN o.amount_f ELSE 0 END) as active_pipeline,
        SUM(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f ELSE 0 END) as won_revenue,
        COUNT(CASE WHEN o.stagename = 'Closed Won' THEN 1 END) as deals_won,
        COUNT(CASE WHEN o.stagename = 'Closed Lost' THEN 1 END) as deals_lost,
        AVG(o.amount_f) as avg_deal_size,
        AVG(DATE_DIFF('day', o.createddate_ts, 
            CASE WHEN o.stagename IN ('Closed Won', 'Closed Lost') 
                 THEN CAST(o.closedate_d AS timestamp) 
                 ELSE CURRENT_TIMESTAMP END)) as avg_sales_cycle_days
    FROM latest_account a
    LEFT JOIN latest_opportunity o ON a.id = o.accountid AND o.rn = 1
    WHERE a.rn = 1
    GROUP BY a.customer_segment
)
SELECT 
    customer_segment,
    account_count,
    opportunity_count,
    active_pipeline,
    won_revenue,
    deals_won,
    deals_lost,
    ROUND(deals_won * 100.0 / NULLIF(deals_won + deals_lost, 0), 2) as win_rate_pct,
    ROUND(avg_deal_size, 0) as avg_deal_size,
    ROUND(avg_sales_cycle_days, 0) as avg_sales_cycle_days,
    ROUND(won_revenue / NULLIF(account_count, 0), 0) as revenue_per_account
FROM segment_performance
ORDER BY won_revenue DESC
```

## Sales Velocity Analysis

### Stage Velocity and Bottleneck Identification

```sql
WITH latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn,
        -- Calculate time in current stage
        DATE_DIFF('day', 
            COALESCE(laststagechangedate_ts, createddate_ts), 
            CASE WHEN stagename IN ('Closed Won', 'Closed Lost') 
                 THEN CAST(closedate_d AS timestamp)
                 ELSE CURRENT_TIMESTAMP END) as days_in_current_stage
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
stage_analysis AS (
    SELECT 
        stagename,
        COUNT(*) as opportunity_count,
        AVG(days_in_current_stage) as avg_days_in_stage,
        APPROX_PERCENTILE(days_in_current_stage, 0.5) as median_days_in_stage,
        APPROX_PERCENTILE(days_in_current_stage, 0.9) as p90_days_in_stage,
        COUNT(CASE WHEN days_in_current_stage > 90 THEN 1 END) as stale_opportunities,
        AVG(amount_f) as avg_opportunity_value,
        -- Stage progression calculation
        COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as progressed_to_won,
        COUNT(CASE WHEN stagename = 'Closed Lost' THEN 1 END) as progressed_to_lost
    FROM latest_opportunity
    WHERE rn = 1
    GROUP BY stagename
)
SELECT 
    stagename,
    opportunity_count,
    ROUND(avg_days_in_stage, 1) as avg_days_in_stage,
    median_days_in_stage,
    p90_days_in_stage,
    stale_opportunities,
    ROUND(stale_opportunities * 100.0 / opportunity_count, 2) as stale_opportunity_pct,
    ROUND(avg_opportunity_value, 0) as avg_opportunity_value,
    progressed_to_won,
    progressed_to_lost,
    ROUND(progressed_to_won * 100.0 / NULLIF(progressed_to_won + progressed_to_lost, 0), 2) as stage_conversion_rate,
    -- Bottleneck indicator
    CASE 
        WHEN avg_days_in_stage > 60 AND stale_opportunities * 100.0 / opportunity_count > 25 
        THEN 'HIGH PRIORITY - Bottleneck Detected'
        WHEN avg_days_in_stage > 45 OR stale_opportunities * 100.0 / opportunity_count > 15
        THEN 'MEDIUM PRIORITY - Monitor Closely'
        ELSE 'Normal Flow'
    END as bottleneck_status
FROM stage_analysis
ORDER BY 
    CASE stagename
        WHEN 'Prospecting' THEN 1
        WHEN 'Qualification' THEN 2
        WHEN 'Needs Analysis' THEN 3  
        WHEN 'Value Proposition' THEN 4
        WHEN 'Id. Decision Makers' THEN 5
        WHEN 'Perception Analysis' THEN 6
        WHEN 'Proposal/Price Quote' THEN 7
        WHEN 'Negotiation/Review' THEN 8
        WHEN 'Closed Won' THEN 9
        WHEN 'Closed Lost' THEN 10
        ELSE 11
    END
```

## Attribution Analysis Patterns

### Lead Source Performance Through Revenue

```sql
WITH latest_lead AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
),
source_attribution AS (
    SELECT 
        COALESCE(l.leadsource, o.leadsource, 'Unknown') as lead_source,
        COUNT(DISTINCT l.id) as total_leads,
        COUNT(DISTINCT CASE WHEN l.isconverted_b = true THEN l.id END) as converted_leads,
        COUNT(DISTINCT o.id) as total_opportunities,
        COUNT(DISTINCT CASE WHEN o.stagename = 'Closed Won' THEN o.id END) as won_opportunities,
        SUM(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN o.amount_f ELSE 0 END) as active_pipeline,
        SUM(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f ELSE 0 END) as won_revenue,
        AVG(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f END) as avg_won_deal_size,
        AVG(CASE WHEN l.isconverted_b = true THEN DATE_DIFF('day', l.createddate_ts, CAST(l.converteddate_d AS timestamp)) END) as avg_lead_conversion_days,
        AVG(CASE WHEN o.stagename = 'Closed Won' THEN DATE_DIFF('day', o.createddate_ts, CAST(o.closedate_d AS timestamp)) END) as avg_sales_cycle_days
    FROM latest_lead l
    FULL OUTER JOIN latest_opportunity o ON l.convertedopportunityid = o.id AND o.rn = 1
    WHERE l.rn = 1
    GROUP BY COALESCE(l.leadsource, o.leadsource, 'Unknown')
)
SELECT 
    lead_source,
    total_leads,
    converted_leads,
    total_opportunities,
    won_opportunities,
    active_pipeline,
    won_revenue,
    -- Conversion and efficiency metrics
    ROUND(converted_leads * 100.0 / NULLIF(total_leads, 0), 2) as lead_conversion_rate,
    ROUND(won_opportunities * 100.0 / NULLIF(total_opportunities, 0), 2) as opportunity_win_rate,
    ROUND(won_revenue / NULLIF(total_leads, 0), 0) as revenue_per_lead,
    ROUND(avg_won_deal_size, 0) as avg_won_deal_size,
    ROUND(avg_lead_conversion_days, 0) as avg_lead_conversion_days,
    ROUND(avg_sales_cycle_days, 0) as avg_sales_cycle_days,
    -- ROI indicators (assuming cost data available)
    ROUND(won_revenue / NULLIF(converted_leads, 0), 0) as revenue_per_conversion,
    -- Performance ranking
    RANK() OVER (ORDER BY won_revenue DESC) as revenue_rank,
    RANK() OVER (ORDER BY converted_leads * 100.0 / NULLIF(total_leads, 0) DESC) as conversion_rank
FROM source_attribution
WHERE total_leads > 0
ORDER BY won_revenue DESC
```

## Forecasting Patterns

### Pipeline Forecasting with Historical Conversion

```sql
WITH latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn,
        DATE_TRUNC('month', closedate_d) as forecast_month
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND stagename NOT IN ('Closed Won', 'Closed Lost')
        AND closedate_d >= CURRENT_DATE
        AND closedate_d <= DATE_ADD('month', 6, CURRENT_DATE)
),
historical_conversion AS (
    SELECT 
        stagename,
        COUNT(*) as historical_total,
        COUNT(CASE WHEN stagename_final = 'Closed Won' THEN 1 END) as historical_won,
        ROUND(COUNT(CASE WHEN stagename_final = 'Closed Won' THEN 1 END) * 100.0 / COUNT(*), 2) as historical_conversion_rate
    FROM (
        SELECT 
            o1.id,
            o1.stagename,
            o2.stagename as stagename_final
        FROM lakehouse.object_opportunity o1
        JOIN (
            SELECT 
                id,
                stagename,
                ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
            FROM lakehouse.object_opportunity
            WHERE grax__deleted IS NULL
                AND stagename IN ('Closed Won', 'Closed Lost')
                AND createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
        ) o2 ON o1.id = o2.id AND o2.rn = 1
        WHERE o1.grax__deleted IS NULL
            AND o1.stagename NOT IN ('Closed Won', 'Closed Lost')
            AND o1.createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
    )
    GROUP BY stagename
),
pipeline_forecast AS (
    SELECT 
        o.forecast_month,
        o.stagename,
        COUNT(*) as opportunity_count,
        SUM(o.amount_f) as pipeline_value,
        AVG(o.probability_f) as avg_probability,
        h.historical_conversion_rate,
        -- Forecast calculations
        SUM(o.amount_f * o.probability_f / 100) as probability_weighted_value,
        SUM(o.amount_f * h.historical_conversion_rate / 100) as history_weighted_value,
        SUM(o.amount_f * ((o.probability_f + h.historical_conversion_rate) / 2) / 100) as blended_forecast_value
    FROM latest_opportunity o
    LEFT JOIN historical_conversion h ON o.stagename = h.stagename
    WHERE o.rn = 1
    GROUP BY o.forecast_month, o.stagename, h.historical_conversion_rate
)
SELECT 
    forecast_month,
    stagename,
    opportunity_count,
    ROUND(pipeline_value, 0) as pipeline_value,
    ROUND(avg_probability, 1) as avg_probability_pct,
    ROUND(historical_conversion_rate, 1) as historical_conversion_rate_pct,
    ROUND(probability_weighted_value, 0) as probability_weighted_forecast,
    ROUND(history_weighted_value, 0) as history_weighted_forecast,
    ROUND(blended_forecast_value, 0) as blended_forecast,
    -- Confidence indicators
    CASE 
        WHEN ABS(probability_weighted_value - history_weighted_value) / NULLIF(pipeline_value, 0) < 0.1 
        THEN 'High Confidence'
        WHEN ABS(probability_weighted_value - history_weighted_value) / NULLIF(pipeline_value, 0) < 0.25
        THEN 'Medium Confidence'
        ELSE 'Low Confidence - Large Variance'
    END as forecast_confidence
FROM pipeline_forecast
ORDER BY forecast_month, pipeline_value DESC
```

## Competitive Analysis Patterns

### Win/Loss Analysis by Competitor

```sql
WITH latest_opportunity AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND stagename IN ('Closed Won', 'Closed Lost')
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
competitor_analysis AS (
    SELECT 
        COALESCE(competitor__c, 'Unknown/No Competition') as competitor,
        stagename as outcome,
        COUNT(*) as deal_count,
        SUM(amount_f) as total_value,
        AVG(amount_f) as avg_deal_size,
        AVG(DATE_DIFF('day', createddate_ts, CAST(closedate_d AS timestamp))) as avg_sales_cycle_days
    FROM latest_opportunity
    WHERE rn = 1
    GROUP BY COALESCE(competitor__c, 'Unknown/No Competition'), stagename
)
SELECT 
    competitor,
    SUM(deal_count) as total_deals,
    SUM(CASE WHEN outcome = 'Closed Won' THEN deal_count ELSE 0 END) as deals_won,
    SUM(CASE WHEN outcome = 'Closed Lost' THEN deal_count ELSE 0 END) as deals_lost,
    ROUND(SUM(CASE WHEN outcome = 'Closed Won' THEN deal_count ELSE 0 END) * 100.0 / 
          SUM(deal_count), 2) as win_rate_pct,
    ROUND(SUM(CASE WHEN outcome = 'Closed Won' THEN total_value ELSE 0 END), 0) as won_revenue,
    ROUND(SUM(CASE WHEN outcome = 'Closed Lost' THEN total_value ELSE 0 END), 0) as lost_revenue,
    ROUND(AVG(CASE WHEN outcome = 'Closed Won' THEN avg_deal_size END), 0) as avg_won_deal_size,
    ROUND(AVG(CASE WHEN outcome = 'Closed Lost' THEN avg_deal_size END), 0) as avg_lost_deal_size,
    ROUND(AVG(avg_sales_cycle_days), 0) as avg_sales_cycle_days,
    -- Competitive threat assessment
    CASE 
        WHEN SUM(CASE WHEN outcome = 'Closed Won' THEN deal_count ELSE 0 END) * 100.0 / SUM(deal_count) < 30
        THEN 'High Threat - Low Win Rate'
        WHEN SUM(deal_count) > 10 AND SUM(CASE WHEN outcome = 'Closed Won' THEN deal_count ELSE 0 END) * 100.0 / SUM(deal_count) < 50
        THEN 'Medium Threat - Frequent Competitor'
        ELSE 'Low Threat'
    END as threat_level
FROM competitor_analysis
GROUP BY competitor
ORDER BY total_deals DESC, won_revenue DESC
```

## Configuration Adaptation

These business intelligence patterns integrate with [Configuration Reference](../core-reference/configuration-reference.md) for:

### Customizable Elements

- **Lead Status Values**: MQL qualification criteria and progression stages
- **Opportunity Stages**: Sales process stages and win/loss definitions
- **Segmentation Thresholds**: Company size and revenue classifications
- **Time Periods**: Analysis windows and historical comparison ranges
- **Custom Fields**: Organization-specific fields like competitor tracking

### Adaptation Process

1. **Review Default Values**: Examine all referenced configuration values
2. **Update Configuration**: Modify [Configuration Reference](../core-reference/configuration-reference.md) with your values
3. **Test Patterns**: Validate that business logic aligns with your processes
4. **Customize Calculations**: Adjust formulas for organization-specific metrics

This ensures all business intelligence patterns remain consistent while adapting to different Salesforce implementations and business processes.
