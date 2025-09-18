# Lead Data Quality Analysis

## Overview

This document provides comprehensive data quality analysis patterns for lead records in the GRAX Data Lake. All analysis queries reference configuration values from [Configuration Reference](../../core-reference/configuration-reference.md) to ensure adaptability across different Salesforce implementations.

**IMPORTANT**: All queries use Athena-compatible SQL patterns with proper latest record selection using window functions and filtering rather than QUALIFY syntax.

## Lead Record Foundation Analysis

### Complete Lead Dataset Query

This foundational query establishes the baseline dataset for all lead data quality analysis:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        email,
        firstname,
        lastname,
        company,
        status,
        leadsource,
        industry,
        country,
        state,
        city,
        phone,
        website,
        title,
        numberofemployees_f,
        annualrevenue_f,
        description,
        isconverted_b,
        converteddate_d,
        convertedaccountid,
        convertedcontactid,
        convertedopportunityid,
        createddate_ts,
        lastmodifieddate_ts,
        grax__idseq,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as row_num
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
current_leads AS (
    SELECT * FROM latest_leads WHERE row_num = 1
)
SELECT * FROM current_leads
```

## Completeness Analysis

### Critical Field Population Analysis

Evaluates the completeness of essential fields required for effective lead qualification and conversion:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        email,
        firstname,
        lastname,
        company,
        status,
        leadsource,
        industry,
        country,
        state,
        city,
        phone,
        website,
        title,
        numberofemployees_f,
        annualrevenue_f,
        grax__idseq,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as row_num
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
current_leads AS (
    SELECT * FROM latest_leads WHERE row_num = 1
),
completeness_analysis AS (
    SELECT 
        -- Critical identification fields
        COUNT(*) as total_leads,
        COUNT(email) as has_email,
        COUNT(CASE WHEN email IS NOT NULL AND email != '' AND email LIKE '%@%.%' THEN 1 END) as has_valid_email,
        
        -- Name completeness
        COUNT(firstname) as has_firstname,
        COUNT(lastname) as has_lastname,
        COUNT(CASE WHEN firstname IS NOT NULL AND lastname IS NOT NULL THEN 1 END) as has_full_name,
        
        -- Company information
        COUNT(company) as has_company,
        COUNT(CASE WHEN company IS NOT NULL AND company != '' THEN 1 END) as has_valid_company,
        
        -- Lead qualification fields (using values from Configuration Reference)
        COUNT(CASE WHEN status IS NOT NULL THEN 1 END) as has_status,
        COUNT(leadsource) as has_lead_source,
        
        -- Geographic information
        COUNT(country) as has_country,
        COUNT(state) as has_state,
        COUNT(city) as has_city,
        
        -- Contact information
        COUNT(phone) as has_phone,
        COUNT(website) as has_website,
        COUNT(title) as has_title,
        
        -- Firmographic data
        COUNT(industry) as has_industry,
        COUNT(numberofemployees_f) as has_employee_count,
        COUNT(annualrevenue_f) as has_revenue
        
    FROM current_leads
)
SELECT 
    total_leads,
    
    -- Critical field completeness percentages
    ROUND(100.0 * has_email / total_leads, 2) as email_completion_rate,
    ROUND(100.0 * has_valid_email / total_leads, 2) as valid_email_rate,
    
    -- Name field completeness
    ROUND(100.0 * has_firstname / total_leads, 2) as firstname_completion_rate,
    ROUND(100.0 * has_lastname / total_leads, 2) as lastname_completion_rate,
    ROUND(100.0 * has_full_name / total_leads, 2) as full_name_completion_rate,
    
    -- Company information completeness
    ROUND(100.0 * has_valid_company / total_leads, 2) as company_completion_rate,
    
    -- Qualification field completeness
    ROUND(100.0 * has_status / total_leads, 2) as status_completion_rate,
    ROUND(100.0 * has_lead_source / total_leads, 2) as lead_source_completion_rate,
    
    -- Geographic completeness
    ROUND(100.0 * has_country / total_leads, 2) as country_completion_rate,
    ROUND(100.0 * has_state / total_leads, 2) as state_completion_rate,
    ROUND(100.0 * has_city / total_leads, 2) as city_completion_rate,
    
    -- Contact information completeness
    ROUND(100.0 * has_phone / total_leads, 2) as phone_completion_rate,
    ROUND(100.0 * has_website / total_leads, 2) as website_completion_rate,
    ROUND(100.0 * has_title / total_leads, 2) as title_completion_rate,
    
    -- Firmographic completeness
    ROUND(100.0 * has_industry / total_leads, 2) as industry_completion_rate,
    ROUND(100.0 * has_employee_count / total_leads, 2) as employee_count_completion_rate,
    ROUND(100.0 * has_revenue / total_leads, 2) as revenue_completion_rate
    
FROM completeness_analysis
```

## Lead Status Progression Analysis

### Historical Status Progression Tracking for Open Leads

This analysis tracks the complete status progression history for leads that are currently open, providing insights into lead nurturing effectiveness, sales process efficiency, and potential bottlenecks in the qualification pipeline.

**Business Value**: Identifies leads stuck in qualification stages, measures progression velocity, and reveals process inefficiencies that impact conversion rates.

```sql
WITH all_lead_history AS (
    SELECT 
        id as lead_id,
        status,
        createddate_ts,
        lastmodifieddate_ts,
        grax__idseq,
        LAG(status) OVER (PARTITION BY id ORDER BY grax__idseq) as previous_status,
        LAG(lastmodifieddate_ts) OVER (PARTITION BY id ORDER BY grax__idseq) as previous_modified_date
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
-- Get current status for filtering to open leads only
current_lead_status AS (
    SELECT 
        lead_id,
        status as current_status,
        ROW_NUMBER() OVER (PARTITION BY lead_id ORDER BY grax__idseq DESC) as row_num
    FROM all_lead_history
),
current_open_leads AS (
    SELECT lead_id, current_status
    FROM current_lead_status 
    WHERE row_num = 1 
      -- Using lead status values from Configuration Reference
      AND current_status IN ('Open', 'Working')
),
-- Track all status changes for currently open leads
status_progressions AS (
    SELECT 
        h.lead_id,
        h.status,
        h.previous_status,
        h.lastmodifieddate_ts as status_change_date,
        h.previous_modified_date,
        -- Calculate time in previous status (in days)
        CASE 
            WHEN h.previous_status IS NOT NULL AND h.previous_modified_date IS NOT NULL 
            THEN DATE_DIFF('day', h.previous_modified_date, h.lastmodifieddate_ts)
            ELSE NULL
        END as days_in_previous_status,
        -- Determine if this is a forward or backward progression
        CASE 
            WHEN h.previous_status = 'Open' AND h.status = 'Working' THEN 'Forward'
            WHEN h.previous_status = 'Working' AND h.status = 'Open' THEN 'Backward'
            WHEN h.previous_status IS NULL THEN 'Initial'
            ELSE 'Other'
        END as progression_direction,
        ROW_NUMBER() OVER (PARTITION BY h.lead_id ORDER BY h.grax__idseq) as progression_sequence
    FROM all_lead_history h
    INNER JOIN current_open_leads c ON h.lead_id = c.lead_id
    WHERE h.previous_status IS NOT NULL OR h.status IS NOT NULL
),
-- Calculate current time in status for open leads
current_status_duration AS (
    SELECT 
        c.lead_id,
        c.current_status,
        MAX(h.lastmodifieddate_ts) as last_status_change,
        DATE_DIFF('day', MAX(h.lastmodifieddate_ts), CURRENT_DATE) as days_in_current_status
    FROM current_open_leads c
    INNER JOIN all_lead_history h ON c.lead_id = h.lead_id
    GROUP BY c.lead_id, c.current_status
)
SELECT 
    -- Lead Status Progression Summary
    'Lead Status Progression Summary for Open Leads' as analysis_type,
    
    -- Current status distribution
    COUNT(DISTINCT csd.lead_id) as total_open_leads,
    COUNT(DISTINCT CASE WHEN csd.current_status = 'Open' THEN csd.lead_id END) as mcl_leads,
    COUNT(DISTINCT CASE WHEN csd.current_status = 'Working' THEN csd.lead_id END) as mql_leads,
    
    -- Progression metrics
    COUNT(DISTINCT CASE WHEN sp.progression_direction = 'Forward' THEN sp.lead_id END) as leads_with_forward_progression,
    COUNT(DISTINCT CASE WHEN sp.progression_direction = 'Backward' THEN sp.lead_id END) as leads_with_regression,
    
    -- Time in status analysis
    ROUND(AVG(csd.days_in_current_status), 1) as avg_days_in_current_status,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY csd.days_in_current_status), 1) as median_days_in_current_status,
    
    -- Long-stale lead identification (using industry best practice thresholds)
    COUNT(CASE WHEN csd.current_status = 'Open' AND csd.days_in_current_status > 30 THEN 1 END) as stale_mcl_leads_30_days,
    COUNT(CASE WHEN csd.current_status = 'Open' AND csd.days_in_current_status > 90 THEN 1 END) as stale_mcl_leads_90_days,
    COUNT(CASE WHEN csd.current_status = 'Working' AND csd.days_in_current_status > 14 THEN 1 END) as stale_mql_leads_14_days,
    COUNT(CASE WHEN csd.current_status = 'Working' AND csd.days_in_current_status > 30 THEN 1 END) as stale_mql_leads_30_days,
    
    -- Velocity analysis
    ROUND(AVG(CASE WHEN sp.status = 'Working' AND sp.previous_status = 'Open' 
              THEN sp.days_in_previous_status END), 1) as avg_days_mcl_to_mql
              
FROM current_status_duration csd
LEFT JOIN status_progressions sp ON csd.lead_id = sp.lead_id

UNION ALL

-- Detailed progression patterns
SELECT 
    'Status Progression Patterns' as analysis_type,
    progression_direction,
    COUNT(*) as progression_count,
    COUNT(DISTINCT lead_id) as unique_leads,
    ROUND(AVG(days_in_previous_status), 1) as avg_days_in_previous_status,
    ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY days_in_previous_status), 1) as median_days_in_previous_status,
    NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL, NULL
FROM status_progressions
WHERE days_in_previous_status IS NOT NULL
GROUP BY progression_direction

ORDER BY analysis_type, progression_direction
```

### Lead Velocity Analysis by Segmentation

Analyzes progression velocity across different lead segments to identify optimization opportunities:

```sql
WITH latest_leads AS (
    SELECT 
        id as lead_id,
        status,
        leadsource,
        industry,
        numberofemployees_f,
        annualrevenue_f,
        createddate_ts,
        lastmodifieddate_ts,
        -- Company segmentation using thresholds from Configuration Reference
        CASE 
            WHEN numberofemployees_f > 250 OR annualrevenue_f > 100000000 THEN 'Enterprise'
            WHEN numberofemployees_f > 50 OR annualrevenue_f > 10000000 THEN 'SMB'
            ELSE 'Self Service'
        END as company_segment,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as row_num
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
current_open_leads AS (
    SELECT * FROM latest_leads 
    WHERE row_num = 1 
      -- Using status values from Configuration Reference
      AND status IN ('Open', 'Working')
),
-- Calculate time in current status by segment
segment_velocity AS (
    SELECT 
        company_segment,
        status as current_status,
        leadsource,
        industry,
        COUNT(*) as lead_count,
        ROUND(AVG(DATE_DIFF('day', lastmodifieddate_ts, CURRENT_DATE)), 1) as avg_days_in_status,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY DATE_DIFF('day', lastmodifieddate_ts, CURRENT_DATE)), 1) as median_days_in_status,
        -- Industry best practice benchmarks
        COUNT(CASE WHEN status = 'Open' AND DATE_DIFF('day', lastmodifieddate_ts, CURRENT_DATE) > 30 THEN 1 END) as leads_exceeding_mcl_sla,
        COUNT(CASE WHEN status = 'Working' AND DATE_DIFF('day', lastmodifieddate_ts, CURRENT_DATE) > 14 THEN 1 END) as leads_exceeding_mql_sla
    FROM current_open_leads
    GROUP BY company_segment, status, leadsource, industry
)
SELECT 
    company_segment,
    current_status,
    SUM(lead_count) as total_leads,
    ROUND(AVG(avg_days_in_status), 1) as segment_avg_days_in_status,
    ROUND(AVG(median_days_in_status), 1) as segment_median_days_in_status,
    SUM(leads_exceeding_mcl_sla) as total_stale_mcl,
    SUM(leads_exceeding_mql_sla) as total_stale_mql,
    -- Calculate percentage of leads exceeding SLA
    ROUND(100.0 * SUM(leads_exceeding_mcl_sla) / NULLIF(SUM(CASE WHEN current_status = 'Open' THEN lead_count END), 0), 2) as mcl_sla_breach_rate,
    ROUND(100.0 * SUM(leads_exceeding_mql_sla) / NULLIF(SUM(CASE WHEN current_status = 'Working' THEN lead_count END), 0), 2) as mql_sla_breach_rate
FROM segment_velocity
GROUP BY company_segment, current_status
ORDER BY company_segment, current_status
```

### Lead Source Performance Analysis

Evaluates lead source effectiveness in driving progression through qualification stages:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        status,
        leadsource,
        isconverted_b,
        converteddate_d,
        createddate_ts,
        lastmodifieddate_ts,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as row_num
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
),
current_leads AS (
    SELECT * FROM latest_leads WHERE row_num = 1
),
source_performance AS (
    SELECT 
        leadsource,
        -- Current status distribution
        COUNT(*) as total_leads,
        COUNT(CASE WHEN status = 'Open' THEN 1 END) as mcl_count,
        COUNT(CASE WHEN status = 'Working' THEN 1 END) as mql_count,
        COUNT(CASE WHEN isconverted_b = true THEN 1 END) as converted_count,
        COUNT(CASE WHEN status = 'Disqualified' THEN 1 END) as disqualified_count,
        
        -- Progression rates
        ROUND(100.0 * COUNT(CASE WHEN status = 'Working' OR isconverted_b = true THEN 1 END) / 
              NULLIF(COUNT(*), 0), 2) as mcl_to_mql_rate,
        ROUND(100.0 * COUNT(CASE WHEN isconverted_b = true THEN 1 END) / 
              NULLIF(COUNT(*), 0), 2) as overall_conversion_rate,
        
        -- Velocity analysis for currently open leads (using industry best practices)
        ROUND(AVG(CASE WHEN status IN ('Open', 'Working') 
                  THEN DATE_DIFF('day', lastmodifieddate_ts, CURRENT_DATE) END), 1) as avg_days_in_current_status,
        COUNT(CASE WHEN status = 'Open' AND DATE_DIFF('day', lastmodifieddate_ts, CURRENT_DATE) > 30 THEN 1 END) as stale_mcl_count,
        COUNT(CASE WHEN status = 'Working' AND DATE_DIFF('day', lastmodifieddate_ts, CURRENT_DATE) > 14 THEN 1 END) as stale_mql_count
        
    FROM current_leads
    WHERE leadsource IS NOT NULL
    GROUP BY leadsource
)
SELECT 
    leadsource,
    total_leads,
    mcl_count,
    mql_count,
    converted_count,
    disqualified_count,
    mcl_to_mql_rate,
    overall_conversion_rate,
    avg_days_in_current_status,
    stale_mcl_count,
    stale_mql_count,
    -- Quality scoring based on industry best practices
    CASE 
        WHEN overall_conversion_rate >= 15 AND mcl_to_mql_rate >= 25 THEN 'High Quality'
        WHEN overall_conversion_rate >= 8 AND mcl_to_mql_rate >= 15 THEN 'Medium Quality'
        ELSE 'Low Quality'
    END as source_quality_rating,
    -- SLA compliance rates
    ROUND(100.0 * (mcl_count - stale_mcl_count) / NULLIF(mcl_count, 0), 2) as mcl_sla_compliance_rate,
    ROUND(100.0 * (mql_count - stale_mql_count) / NULLIF(mql_count, 0), 2) as mql_sla_compliance_rate
FROM source_performance
WHERE total_leads >= 10  -- Only analyze sources with meaningful volume
ORDER BY overall_conversion_rate DESC, total_leads DESC
```

## Industry Best Practice Benchmarks

### Lead Status SLA Standards

**MCL (Marketing Contacted Lead) Standards**:

- **Target Response Time**: 24-48 hours for initial contact
- **Qualification Timeline**: 30 days maximum in Open status
- **Escalation Threshold**: 90 days triggers lead recycling review

**MQL (Marketing Qualified Lead) Standards**:

- **Sales Response Time**: 5 minutes to 2 hours for high-priority leads
- **Qualification Timeline**: 14 days maximum in Working status
- **Progression Requirement**: 30 days maximum before conversion or disqualification

### Performance Benchmarks

**Lead Source Quality Metrics**:

- **High Quality Sources**: >15% overall conversion rate, >25% MCL-to-MQL rate
- **Medium Quality Sources**: 8-15% conversion rate, 15-25% MCL-to-MQL rate  
- **Low Quality Sources**: <8% conversion rate, <15% MCL-to-MQL rate

**Velocity Benchmarks**:

- **MCL Progression**: 7-21 days average time from Open to Working
- **MQL Progression**: 5-14 days average time from Working to Conversion
- **Total Lead Lifecycle**: 30-60 days from creation to conversion

This comprehensive lead data quality analysis framework provides customers with detailed insights into their lead data integrity, status progression efficiency, and performance benchmarking against industry standards. All patterns use Athena-compatible SQL syntax and integrate with the established Configuration Reference for adaptability across different Salesforce implementations.
