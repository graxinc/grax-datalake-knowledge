# Business Intelligence Patterns

## Overview

This guide provides advanced analytics patterns for extracting strategic insights from Salesforce data. These patterns focus on revenue intelligence, customer lifecycle analysis, and predictive indicators that drive business decisions.

**Configuration Reference**: All business-specific values used in these patterns are defined in [Configuration Reference](./configuration-reference.md). Organizations with different Salesforce configurations should update that document to match their specific values.

## Revenue Intelligence Patterns

### Pipeline Velocity Analysis

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
opportunity_with_segment AS (
    SELECT 
        o.*,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment,
        -- Calculate opportunity age
        DATE_DIFF('day', o.createddate_ts, CURRENT_TIMESTAMP) as opportunity_age_days,
        -- Calculate days in current stage
        DATE_DIFF('day', o.laststagechangedate_ts, CURRENT_TIMESTAMP) as days_in_current_stage
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    -- Using stage exclusion logic from docs/configuration-reference.md
    WHERE o.stagename NOT IN ('Closed Won', 'Closed Lost')
      AND o.createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
),
velocity_analysis AS (
    SELECT 
        customer_segment,
        stagename,
        COUNT(*) as opportunity_count,
        SUM(amount_f) as pipeline_value,
        AVG(amount_f) as avg_deal_size,
        AVG(opportunity_age_days) as avg_opportunity_age,
        AVG(days_in_current_stage) as avg_days_in_stage,
        AVG(probability_f) as avg_probability,
        -- Risk indicators
        COUNT(CASE WHEN days_in_current_stage > 90 THEN 1 END) as stalled_opportunities,
        COUNT(CASE WHEN opportunity_age_days > 365 THEN 1 END) as aged_opportunities
    FROM opportunity_with_segment
    WHERE amount_f IS NOT NULL
    GROUP BY customer_segment, stagename
)
SELECT 
    customer_segment,
    stagename,
    opportunity_count,
    ROUND(pipeline_value, 0) as pipeline_value,
    ROUND(avg_deal_size, 0) as avg_deal_size,
    ROUND(avg_opportunity_age, 0) as avg_opportunity_age_days,
    ROUND(avg_days_in_stage, 0) as avg_days_in_current_stage,
    ROUND(avg_probability, 1) as avg_probability_pct,
    stalled_opportunities,
    ROUND(stalled_opportunities * 100.0 / opportunity_count, 1) as stall_rate_pct,
    aged_opportunities,
    ROUND(aged_opportunities * 100.0 / opportunity_count, 1) as aging_rate_pct
FROM velocity_analysis
-- Using stage ordering from docs/configuration-reference.md
ORDER BY customer_segment, 
    CASE stagename
        WHEN 'SQL' THEN 1
        WHEN 'Proof of Value (SQO)' THEN 2
        WHEN 'Proposal' THEN 3
        WHEN 'Contracts' THEN 4
        ELSE 5
    END
```

### Win/Loss Analysis

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
closed_opportunities AS (
    SELECT 
        o.id,
        o.name,
        o.stagename,
        o.amount_f,
        o.probability_f,
        o.createddate_ts,
        CAST(o.closedate_d AS timestamp) as closedate_ts,
        DATE_DIFF('day', o.createddate_ts, CAST(o.closedate_d AS timestamp)) as sales_cycle_days,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment,
        a.industry,
        DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) as close_quarter
    FROM latest_opportunity o
    LEFT JOIN latest_account a ON o.accountid = a.id
    -- Using closed stage names from docs/configuration-reference.md
    WHERE o.stagename IN ('Closed Won', 'Closed Lost')
      AND o.closedate_d >= DATE_ADD('month', -24, CURRENT_DATE)
      AND o.amount_f IS NOT NULL
)
SELECT 
    close_quarter,
    customer_segment,
    industry,
    
    -- Volume metrics
    COUNT(*) as total_closed,
    -- Using stage names from docs/configuration-reference.md
    COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) as won_count,
    COUNT(CASE WHEN stagename = 'Closed Lost' THEN 1 END) as lost_count,
    
    -- Win rate analysis
    ROUND(COUNT(CASE WHEN stagename = 'Closed Won' THEN 1 END) * 100.0 / COUNT(*), 1) as win_rate_pct,
    
    -- Revenue metrics
    SUM(CASE WHEN stagename = 'Closed Won' THEN amount_f ELSE 0 END) as won_revenue,
    SUM(CASE WHEN stagename = 'Closed Lost' THEN amount_f ELSE 0 END) as lost_revenue,
    ROUND(AVG(CASE WHEN stagename = 'Closed Won' THEN amount_f END), 0) as avg_won_deal_size,
    ROUND(AVG(CASE WHEN stagename = 'Closed Lost' THEN amount_f END), 0) as avg_lost_deal_size,
    
    -- Sales cycle analysis
    ROUND(AVG(CASE WHEN stagename = 'Closed Won' THEN sales_cycle_days END), 0) as avg_won_cycle_days,
    ROUND(AVG(CASE WHEN stagename = 'Closed Lost' THEN sales_cycle_days END), 0) as avg_lost_cycle_days,
    
    -- Probability analysis
    ROUND(AVG(CASE WHEN stagename = 'Closed Won' THEN probability_f END), 1) as avg_won_probability,
    ROUND(AVG(CASE WHEN stagename = 'Closed Lost' THEN probability_f END), 1) as avg_lost_probability
    
FROM closed_opportunities
GROUP BY close_quarter, customer_segment, industry
HAVING COUNT(*) >= 5  -- Only include segments with meaningful sample size
ORDER BY close_quarter DESC, customer_segment, industry
```

### Revenue Forecasting Analysis

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
latest_user AS (
    SELECT A.* 
    FROM lakehouse.object_user A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_user B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
forecast_data AS (
    SELECT 
        o.id,
        o.stagename,
        o.amount_f,
        o.probability_f,
        CAST(o.closedate_d AS timestamp) as closedate_ts,
        DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) as close_quarter,
        DATE_TRUNC('month', CAST(o.closedate_d AS timestamp)) as close_month,
        u.name as owner_name,
        -- Weighted pipeline calculation
        (o.amount_f * o.probability_f / 100) as weighted_amount,
        -- Risk scoring
        CASE 
            WHEN o.probability_f >= 75 THEN 'High Confidence'
            WHEN o.probability_f >= 50 THEN 'Medium Confidence'
            WHEN o.probability_f >= 25 THEN 'Low Confidence'
            ELSE 'Very Low Confidence'
        END as confidence_level,
        -- Timeline categorization
        CASE 
            WHEN CAST(o.closedate_d AS timestamp) <= DATE_ADD('month', 1, CURRENT_DATE) THEN 'Current Month'
            WHEN CAST(o.closedate_d AS timestamp) <= DATE_ADD('month', 3, CURRENT_DATE) THEN 'Next 3 Months'
            WHEN CAST(o.closedate_d AS timestamp) <= DATE_ADD('month', 6, CURRENT_DATE) THEN 'Next 6 Months'
            ELSE 'Beyond 6 Months'
        END as timeline_category
    FROM latest_opportunity o
    LEFT JOIN latest_user u ON o.ownerid = u.id
    -- Using active pipeline exclusion logic from docs/configuration-reference.md
    WHERE o.stagename NOT IN ('Closed Won', 'Closed Lost')
      AND o.closedate_d IS NOT NULL
      AND o.amount_f IS NOT NULL
      AND o.probability_f IS NOT NULL
)
SELECT 
    close_quarter,
    timeline_category,
    confidence_level,
    COUNT(*) as opportunity_count,
    ROUND(SUM(amount_f), 0) as total_pipeline_value,
    ROUND(SUM(weighted_amount), 0) as weighted_pipeline_value,
    ROUND(AVG(amount_f), 0) as avg_deal_size,
    ROUND(AVG(probability_f), 1) as avg_probability_pct,
    -- Forecast confidence indicators
    ROUND(SUM(weighted_amount) / NULLIF(SUM(amount_f), 0) * 100, 1) as pipeline_confidence_pct,
    COUNT(DISTINCT owner_name) as unique_owners
FROM forecast_data
GROUP BY close_quarter, timeline_category, confidence_level
ORDER BY close_quarter, 
    CASE timeline_category
        WHEN 'Current Month' THEN 1
        WHEN 'Next 3 Months' THEN 2
        WHEN 'Next 6 Months' THEN 3
        WHEN 'Beyond 6 Months' THEN 4
    END,
    CASE confidence_level
        WHEN 'High Confidence' THEN 1
        WHEN 'Medium Confidence' THEN 2
        WHEN 'Low Confidence' THEN 3
        WHEN 'Very Low Confidence' THEN 4
    END
```

## Customer Lifecycle Analysis

### Lead-to-Customer Journey

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
customer_journey AS (
    SELECT 
        l.id as lead_id,
        l.createddate_ts as lead_created,
        l.converteddate_d as lead_converted,
        l.company as lead_company,
        l.leadsource,
        
        o.id as opportunity_id,
        o.createddate_ts as opportunity_created,
        o.stagename as current_stage,
        CAST(o.closedate_d AS timestamp) as opportunity_closedate,
        o.amount_f as opportunity_amount,
        
        a.id as account_id,
        a.name as account_name,
        a.type as account_type,
        
        -- Journey timing calculations
        DATE_DIFF('day', l.createddate_ts, CAST(l.converteddate_d AS timestamp)) as lead_to_conversion_days,
        DATE_DIFF('day', CAST(l.converteddate_d AS timestamp), o.createddate_ts) as conversion_to_opportunity_days,
        DATE_DIFF('day', o.createddate_ts, CAST(o.closedate_d AS timestamp)) as opportunity_lifecycle_days,
        DATE_DIFF('day', l.createddate_ts, CAST(o.closedate_d AS timestamp)) as total_journey_days,
        
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment
        
    FROM latest_lead l
    INNER JOIN latest_opportunity o ON l.convertedopportunityid = o.id
    LEFT JOIN latest_account a ON l.convertedaccountid = a.id
    WHERE l.isconverted_b = true
      AND l.converteddate_d IS NOT NULL
      AND l.createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
)
SELECT 
    customer_segment,
    leadsource,
    current_stage,
    COUNT(*) as journey_count,
    
    -- Journey timing analysis
    ROUND(AVG(lead_to_conversion_days), 0) as avg_lead_to_conversion_days,
    ROUND(AVG(conversion_to_opportunity_days), 0) as avg_conversion_to_opp_days,
    ROUND(AVG(opportunity_lifecycle_days), 0) as avg_opportunity_lifecycle_days,
    ROUND(AVG(total_journey_days), 0) as avg_total_journey_days,
    
    -- Revenue analysis using stage names from docs/configuration-reference.md
    COUNT(CASE WHEN current_stage = 'Closed Won' THEN 1 END) as won_count,
    ROUND(AVG(CASE WHEN current_stage = 'Closed Won' THEN opportunity_amount END), 0) as avg_won_deal_size,
    SUM(CASE WHEN current_stage = 'Closed Won' THEN opportunity_amount ELSE 0 END) as total_won_revenue,
    
    -- Conversion efficiency
    ROUND(COUNT(CASE WHEN current_stage = 'Closed Won' THEN 1 END) * 100.0 / COUNT(*), 1) as lead_to_customer_rate
    
FROM customer_journey
WHERE lead_to_conversion_days IS NOT NULL
  AND opportunity_amount IS NOT NULL
GROUP BY customer_segment, leadsource, current_stage
HAVING COUNT(*) >= 3  -- Only include meaningful sample sizes
ORDER BY customer_segment, leadsource, 
    CASE current_stage
        WHEN 'Closed Won' THEN 1
        WHEN 'Closed Lost' THEN 2
        ELSE 3
    END
```

### Account Growth Analysis

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
account_revenue_history AS (
    SELECT 
        a.id as account_id,
        a.name as account_name,
        a.type as account_type,
        a.industry,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment,
        
        -- First deal metrics using stage names from docs/configuration-reference.md
        MIN(CASE WHEN o.stagename = 'Closed Won' THEN CAST(o.closedate_d AS timestamp) END) as first_purchase_date,
        MIN(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f END) as first_deal_amount,
        
        -- Latest deal metrics
        MAX(CASE WHEN o.stagename = 'Closed Won' THEN CAST(o.closedate_d AS timestamp) END) as latest_purchase_date,
        MAX(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f END) as latest_deal_amount,
        
        -- Aggregate metrics
        COUNT(CASE WHEN o.stagename = 'Closed Won' THEN 1 END) as total_deals,
        SUM(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f ELSE 0 END) as total_revenue,
        AVG(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f END) as avg_deal_size,
        
        -- Recent activity (last 12 months)
        COUNT(CASE WHEN o.stagename = 'Closed Won' 
                   AND CAST(o.closedate_d AS timestamp) >= DATE_ADD('month', -12, CURRENT_DATE) 
                   THEN 1 END) as recent_deals,
        SUM(CASE WHEN o.stagename = 'Closed Won' 
                 AND CAST(o.closedate_d AS timestamp) >= DATE_ADD('month', -12, CURRENT_DATE) 
                 THEN o.amount_f ELSE 0 END) as recent_revenue,
        
        -- Active pipeline using exclusion logic from docs/configuration-reference.md
        COUNT(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN 1 END) as active_opportunities,
        SUM(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN o.amount_f ELSE 0 END) as pipeline_value
        
    FROM latest_account a
    LEFT JOIN latest_opportunity o ON a.id = o.accountid
    -- Using account types from docs/configuration-reference.md
    WHERE a.type IN ('Customer', 'Customer - Subsidiary', 'Prospect')
    GROUP BY a.id, a.name, a.type, a.industry, a.numberofemployees_f, a.annualrevenue_f
),
account_growth_analysis AS (
    SELECT 
        *,
        -- Customer lifecycle stage
        CASE 
            WHEN total_deals = 0 THEN 'Prospect'
            WHEN total_deals = 1 AND recent_deals = 0 THEN 'One-Time Customer'
            WHEN total_deals = 1 AND recent_deals = 1 THEN 'New Customer'
            WHEN total_deals > 1 AND recent_deals > 0 THEN 'Repeat Customer'
            WHEN total_deals > 1 AND recent_deals = 0 THEN 'Dormant Customer'
            ELSE 'Unknown'
        END as lifecycle_stage,
        
        -- Customer tenure
        CASE 
            WHEN first_purchase_date IS NOT NULL 
            THEN DATE_DIFF('month', first_purchase_date, CURRENT_DATE)
            ELSE NULL
        END as customer_tenure_months,
        
        -- Growth indicators
        CASE 
            WHEN latest_deal_amount > first_deal_amount THEN 'Expanding'
            WHEN latest_deal_amount = first_deal_amount THEN 'Stable'
            WHEN latest_deal_amount < first_deal_amount THEN 'Contracting'
            ELSE 'Unknown'
        END as growth_trend,
        
        -- Revenue velocity (annualized)
        CASE 
            WHEN first_purchase_date IS NOT NULL AND customer_tenure_months > 0
            THEN (total_revenue * 12.0 / customer_tenure_months)
            ELSE NULL
        END as annualized_revenue_velocity
        
    FROM (
        SELECT 
            *,
            DATE_DIFF('month', first_purchase_date, CURRENT_DATE) as customer_tenure_months
        FROM account_revenue_history
    ) base
)
SELECT 
    customer_segment,
    lifecycle_stage,
    growth_trend,
    COUNT(*) as account_count,
    
    -- Revenue metrics
    ROUND(SUM(total_revenue), 0) as segment_total_revenue,
    ROUND(AVG(total_revenue), 0) as avg_account_revenue,
    ROUND(SUM(recent_revenue), 0) as segment_recent_revenue,
    ROUND(AVG(recent_revenue), 0) as avg_account_recent_revenue,
    
    -- Deal metrics
    ROUND(AVG(total_deals), 1) as avg_deals_per_account,
    ROUND(AVG(avg_deal_size), 0) as avg_deal_size,
    
    -- Growth metrics
    ROUND(AVG(customer_tenure_months), 0) as avg_tenure_months,
    ROUND(AVG(annualized_revenue_velocity), 0) as avg_revenue_velocity,
    
    -- Pipeline metrics
    ROUND(SUM(pipeline_value), 0) as segment_pipeline_value,
    ROUND(AVG(pipeline_value), 0) as avg_account_pipeline_value
    
FROM account_growth_analysis
WHERE total_deals > 0  -- Focus on actual customers
GROUP BY customer_segment, lifecycle_stage, growth_trend
ORDER BY customer_segment, 
    CASE lifecycle_stage
        WHEN 'New Customer' THEN 1
        WHEN 'Repeat Customer' THEN 2
        WHEN 'One-Time Customer' THEN 3
        WHEN 'Dormant Customer' THEN 4
        ELSE 5
    END,
    growth_trend
```

## Predictive Analytics Patterns

### Churn Risk Analysis

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
latest_case AS (
    SELECT A.* 
    FROM lakehouse.object_case A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_case B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
customer_health_metrics AS (
    SELECT 
        a.id as account_id,
        a.name as account_name,
        a.type as account_type,
        -- Using segmentation thresholds from docs/configuration-reference.md
        CASE 
            WHEN a.numberofemployees_f > 250 OR a.annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN a.numberofemployees_f > 50 OR a.annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as customer_segment,
        
        -- Revenue trends using stage names from docs/configuration-reference.md
        SUM(CASE WHEN o.stagename = 'Closed Won' 
                 AND CAST(o.closedate_d AS timestamp) >= DATE_ADD('month', -12, CURRENT_DATE) 
                 THEN o.amount_f ELSE 0 END) as revenue_last_12m,
        SUM(CASE WHEN o.stagename = 'Closed Won' 
                 AND CAST(o.closedate_d AS timestamp) >= DATE_ADD('month', -24, CURRENT_DATE)
                 AND CAST(o.closedate_d AS timestamp) < DATE_ADD('month', -12, CURRENT_DATE)
                 THEN o.amount_f ELSE 0 END) as revenue_prev_12m,
        
        -- Activity metrics
        MAX(CASE WHEN o.stagename = 'Closed Won' THEN CAST(o.closedate_d AS timestamp) END) as last_purchase_date,
        COUNT(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN 1 END) as active_opportunities,
        SUM(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN o.amount_f ELSE 0 END) as pipeline_value,
        
        -- Support metrics
        COUNT(CASE WHEN c.createddate_ts >= DATE_ADD('month', -6, CURRENT_DATE) THEN 1 END) as recent_cases,
        COUNT(CASE WHEN c.priority IN ('High', 'Critical') 
                   AND c.createddate_ts >= DATE_ADD('month', -6, CURRENT_DATE) 
                   THEN 1 END) as high_priority_cases,
        AVG(CASE WHEN c.isclosed_b = true 
                 AND c.createddate_ts >= DATE_ADD('month', -6, CURRENT_DATE)
                 THEN DATE_DIFF('day', c.createddate_ts, c.lastmodifieddate_ts)
                 END) as avg_case_resolution_days
        
    FROM latest_account a
    LEFT JOIN latest_opportunity o ON a.id = o.accountid
    LEFT JOIN latest_case c ON a.id = c.accountid
    -- Using customer account types from docs/configuration-reference.md
    WHERE a.type IN ('Customer', 'Customer - Subsidiary')
    GROUP BY a.id, a.name, a.type, a.numberofemployees_f, a.annualrevenue_f
),
churn_risk_scoring AS (
    SELECT 
        *,
        -- Revenue trend scoring
        CASE 
            WHEN revenue_prev_12m > 0 AND revenue_last_12m / revenue_prev_12m < 0.5 THEN 3  -- Severe decline
            WHEN revenue_prev_12m > 0 AND revenue_last_12m / revenue_prev_12m < 0.8 THEN 2  -- Moderate decline
            WHEN revenue_last_12m = 0 AND revenue_prev_12m > 0 THEN 4                       -- No recent revenue
            WHEN revenue_last_12m = 0 AND revenue_prev_12m = 0 THEN 3                       -- No revenue history
            ELSE 1  -- Stable or growing
        END as revenue_risk_score,
        
        -- Activity risk scoring
        CASE 
            WHEN last_purchase_date < DATE_ADD('month', -18, CURRENT_DATE) THEN 3  -- Very stale
            WHEN last_purchase_date < DATE_ADD('month', -12, CURRENT_DATE) THEN 2  -- Stale
            WHEN active_opportunities = 0 THEN 2                                    -- No pipeline
            ELSE 1  -- Recent activity
        END as activity_risk_score,
        
        -- Support risk scoring
        CASE 
            WHEN high_priority_cases >= 3 THEN 3                    -- Many critical issues
            WHEN recent_cases >= 10 THEN 2                          -- High support volume
            WHEN avg_case_resolution_days > 14 THEN 2               -- Slow resolution
            WHEN recent_cases = 0 THEN 1                            -- No recent issues
            ELSE 1  -- Normal support levels
        END as support_risk_score,
        
        -- Days since last purchase
        DATE_DIFF('day', last_purchase_date, CURRENT_DATE) as days_since_last_purchase
        
    FROM customer_health_metrics
),
final_risk_analysis AS (
    SELECT 
        *,
        (revenue_risk_score + activity_risk_score + support_risk_score) as total_risk_score,
        
        CASE 
            WHEN (revenue_risk_score + activity_risk_score + support_risk_score) >= 8 THEN 'Critical Risk'
            WHEN (revenue_risk_score + activity_risk_score + support_risk_score) >= 6 THEN 'High Risk'
            WHEN (revenue_risk_score + activity_risk_score + support_risk_score) >= 4 THEN 'Medium Risk'
            ELSE 'Low Risk'
        END as risk_category
        
    FROM churn_risk_scoring
)
SELECT 
    risk_category,
    customer_segment,
    COUNT(*) as account_count,
    ROUND(AVG(total_risk_score), 1) as avg_risk_score,
    
    -- Revenue at risk
    ROUND(SUM(revenue_last_12m), 0) as revenue_at_risk_12m,
    ROUND(AVG(revenue_last_12m), 0) as avg_revenue_per_account,
    
    -- Pipeline at risk
    ROUND(SUM(pipeline_value), 0) as pipeline_at_risk,
    ROUND(AVG(pipeline_value), 0) as avg_pipeline_per_account,
    
    -- Activity indicators
    ROUND(AVG(days_since_last_purchase), 0) as avg_days_since_purchase,
    ROUND(AVG(active_opportunities), 1) as avg_active_opportunities,
    
    -- Support indicators
    ROUND(AVG(recent_cases), 1) as avg_recent_cases,
    ROUND(AVG(high_priority_cases), 1) as avg_high_priority_cases
    
FROM final_risk_analysis
GROUP BY risk_category, customer_segment
ORDER BY 
    CASE risk_category
        WHEN 'Critical Risk' THEN 1
        WHEN 'High Risk' THEN 2
        WHEN 'Medium Risk' THEN 3
        WHEN 'Low Risk' THEN 4
    END,
    customer_segment
```

## Territory and Performance Analysis

### Sales Performance by Owner

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
latest_user AS (
    SELECT A.* 
    FROM lakehouse.object_user A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_user B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
      AND A.isactive_b = true
),
sales_performance AS (
    SELECT 
        u.id as user_id,
        u.name as sales_rep_name,
        u.email as sales_rep_email,
        
        -- Current quarter performance using stage names from docs/configuration-reference.md
        COUNT(CASE WHEN o.stagename = 'Closed Won' 
                   AND DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) = DATE_TRUNC('quarter', CURRENT_DATE)
                   THEN 1 END) as q_won_deals,
        SUM(CASE WHEN o.stagename = 'Closed Won' 
                 AND DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) = DATE_TRUNC('quarter', CURRENT_DATE)
                 THEN o.amount_f ELSE 0 END) as q_won_revenue,
        
        -- Year-to-date performance
        COUNT(CASE WHEN o.stagename = 'Closed Won' 
                   AND EXTRACT(year FROM CAST(o.closedate_d AS timestamp)) = EXTRACT(year FROM CURRENT_DATE)
                   THEN 1 END) as ytd_won_deals,
        SUM(CASE WHEN o.stagename = 'Closed Won' 
                 AND EXTRACT(year FROM CAST(o.closedate_d AS timestamp)) = EXTRACT(year FROM CURRENT_DATE)
                 THEN o.amount_f ELSE 0 END) as ytd_won_revenue,
        
        -- Pipeline metrics using exclusion logic from docs/configuration-reference.md
        COUNT(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN 1 END) as active_pipeline_count,
        SUM(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') THEN o.amount_f ELSE 0 END) as active_pipeline_value,
        SUM(CASE WHEN o.stagename NOT IN ('Closed Won', 'Closed Lost') 
                 THEN (o.amount_f * o.probability_f / 100) ELSE 0 END) as weighted_pipeline_value,
        
        -- Activity and efficiency metrics
        AVG(CASE WHEN o.stagename = 'Closed Won' 
                 THEN DATE_DIFF('day', o.createddate_ts, CAST(o.closedate_d AS timestamp))
                 END) as avg_sales_cycle_days,
        ROUND(COUNT(CASE WHEN o.stagename = 'Closed Won' THEN 1 END) * 100.0 / 
              NULLIF(COUNT(CASE WHEN o.stagename IN ('Closed Won', 'Closed Lost') THEN 1 END), 0), 1) as win_rate_pct,
        
        -- Deal size analysis
        AVG(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f END) as avg_won_deal_size,
        MAX(CASE WHEN o.stagename = 'Closed Won' THEN o.amount_f END) as largest_won_deal,
        
        -- Quarter-over-quarter growth
        SUM(CASE WHEN o.stagename = 'Closed Won' 
                 AND DATE_TRUNC('quarter', CAST(o.closedate_d AS timestamp)) = DATE_ADD('quarter', -1, DATE_TRUNC('quarter', CURRENT_DATE))
                 THEN o.amount_f ELSE 0 END) as prev_q_won_revenue
        
    FROM latest_user u
    LEFT JOIN latest_opportunity o ON u.id = o.ownerid
    WHERE o.createddate_ts >= DATE_ADD('year', -2, CURRENT_DATE)  -- Focus on recent performance
    GROUP BY u.id, u.name, u.email
)
SELECT 
    sales_rep_name,
    
    -- Current performance
    q_won_deals,
    ROUND(q_won_revenue, 0) as q_won_revenue,
    ytd_won_deals,
    ROUND(ytd_won_revenue, 0) as ytd_won_revenue,
    
    -- Pipeline health
    active_pipeline_count,
    ROUND(active_pipeline_value, 0) as active_pipeline_value,
    ROUND(weighted_pipeline_value, 0) as weighted_pipeline_value,
    ROUND(weighted_pipeline_value / NULLIF(active_pipeline_value, 0) * 100, 1) as pipeline_confidence_pct,
    
    -- Efficiency metrics
    ROUND(avg_sales_cycle_days, 0) as avg_sales_cycle_days,
    win_rate_pct,
    ROUND(avg_won_deal_size, 0) as avg_won_deal_size,
    ROUND(largest_won_deal, 0) as largest_won_deal,
    
    -- Growth analysis
    CASE 
        WHEN prev_q_won_revenue > 0 
        THEN ROUND((q_won_revenue - prev_q_won_revenue) / prev_q_won_revenue * 100, 1)
        ELSE NULL
    END as qoq_revenue_growth_pct,
    
    -- Performance tier
    CASE 
        WHEN q_won_revenue >= 1000000 THEN 'Top Performer'
        WHEN q_won_revenue >= 500000 THEN 'High Performer'
        WHEN q_won_revenue >= 100000 THEN 'Standard Performer'
        ELSE 'Developing Performer'
    END as performance_tier
    
FROM sales_performance
WHERE ytd_won_deals > 0 OR active_pipeline_count > 0  -- Focus on active sellers
ORDER BY ytd_won_revenue DESC
```

## Configuration Adaptation

For organizations with different Salesforce implementations:

### Update Configuration Values

Modify the [Configuration Reference](./configuration-reference.md) document to match your organization's specific values:

- **Opportunity Stage Names**: Update to match your sales process stages
- **Segmentation Thresholds**: Adjust employee count and revenue thresholds for Enterprise/SMB/Self-Service classification
- **Account Types**: Update customer classification values  
- **Lead Status Values**: Modify lead qualification stage names
- **Field Names**: Update if using custom field names

### Validation Process

1. **Test Queries**: Execute sample queries with your configuration values
1. **Validate Results**: Ensure data makes sense for your business context
1. **Document Changes**: Record customizations for future reference
1. **Share Updates**: Consider contributing common variations back to the knowledge base

These business intelligence patterns provide comprehensive insights into revenue trends, customer behavior, predictive indicators, and sales performance, enabling data-driven decision making and strategic business optimization across different Salesforce implementations.
