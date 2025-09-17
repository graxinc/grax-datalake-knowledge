# Lead Data Quality Remediation

## Overview

This document provides comprehensive remediation strategies for lead data quality issues in the GRAX Data Lake. All remediation approaches reference configuration values from [Configuration Reference](../../core-reference/configuration-reference.md) and provide actionable solutions that can be implemented through automated processes, workflow improvements, and user training initiatives.

## Remediation Framework Architecture

### Core Remediation Principles

**Immediate Impact**: Prioritize fixes that provide immediate improvement to lead qualification and conversion processes.

**Preventive Focus**: Address root causes to prevent future data quality issues rather than only fixing existing problems.

**Scalable Solutions**: Implement remediation strategies that scale across growing lead volumes and diverse data sources.

**Measurable Outcomes**: Ensure all remediation efforts can be quantified using the scoring framework from [Lead Data Quality Scoring](./scoring.md).

## Critical Field Remediation Strategies

### Email Field Quality Remediation

Addresses email format validation, test email detection, and business vs personal email classification issues:

#### Immediate Remediation Queries

```sql
-- Identify leads requiring email remediation
WITH latest_leads AS (
    SELECT 
        id,
        email,
        firstname,
        lastname,
        company,
        leadsource,
        createddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
email_issues AS (
    SELECT 
        id,
        email,
        company,
        leadsource,
        
        -- Issue classification
        CASE 
            WHEN email IS NULL OR email = '' THEN 'Missing Email'
            WHEN email NOT LIKE '%@%.%' THEN 'Invalid Format'
            WHEN LOWER(email) LIKE '%test%' OR LOWER(email) LIKE '%example%' 
                 OR LOWER(email) LIKE '%dummy%' OR LOWER(email) LIKE '%fake%' THEN 'Test Email'
            WHEN LOWER(email) LIKE '%@gmail.com' OR LOWER(email) LIKE '%@yahoo.com' 
                 OR LOWER(email) LIKE '%@hotmail.com' OR LOWER(email) LIKE '%@outlook.com' THEN 'Personal Email'
            ELSE 'Valid Business Email'
        END as email_issue_type,
        
        -- Remediation priority
        CASE 
            WHEN email IS NULL OR email = '' THEN 1
            WHEN email NOT LIKE '%@%.%' THEN 1
            WHEN LOWER(email) LIKE '%test%' OR LOWER(email) LIKE '%example%' THEN 2
            WHEN LOWER(email) LIKE '%@gmail.com' THEN 3
            ELSE 4
        END as remediation_priority,
        
        -- Suggested remediation action
        CASE 
            WHEN email IS NULL OR email = '' THEN 'Data Collection Required'
            WHEN email NOT LIKE '%@%.%' THEN 'Format Correction Required'
            WHEN LOWER(email) LIKE '%test%' OR LOWER(email) LIKE '%example%' THEN 'Replace Test Email'
            WHEN LOWER(email) LIKE '%@gmail.com' THEN 'Verify Business Context'
            ELSE 'No Action Required'
        END as remediation_action
        
    FROM latest_leads
    WHERE email IS NULL OR email = '' OR email NOT LIKE '%@%.%' OR
          LOWER(email) LIKE '%test%' OR LOWER(email) LIKE '%example%' OR
          LOWER(email) LIKE '%@gmail.com' OR LOWER(email) LIKE '%@yahoo.com'
)
SELECT 
    email_issue_type,
    COUNT(*) as affected_leads,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM latest_leads), 2) as percentage_of_leads,
    remediation_action,
    STRING_AGG(id, ', ' ORDER BY remediation_priority LIMIT 10) as sample_lead_ids
FROM email_issues
GROUP BY email_issue_type, remediation_priority, remediation_action
ORDER BY remediation_priority, affected_leads DESC
```

### Lead Conversion Consistency Fixes

Addresses mismatches between conversion status and related conversion data:

#### Conversion Data Repair Scripts

```sql
WITH conversion_issues AS (
    SELECT 
        id,
        email,
        status,
        isconverted_b,
        converteddate_d,
        convertedaccountid,
        convertedcontactid,
        convertedopportunityid,
        
        -- Conversion consistency analysis using Configuration Reference status values
        CASE 
            WHEN isconverted_b = true AND status != 'Converted' THEN 'Status Should Be Converted'
            WHEN isconverted_b = false AND status = 'Converted' THEN 'Status Should Not Be Converted'
            WHEN isconverted_b = true AND converteddate_d IS NULL THEN 'Missing Conversion Date'
            WHEN isconverted_b = true AND convertedaccountid IS NULL THEN 'Missing Account Reference'
            WHEN isconverted_b = true AND convertedcontactid IS NULL THEN 'Missing Contact Reference'
            WHEN isconverted_b = false AND (converteddate_d IS NOT NULL OR convertedaccountid IS NOT NULL) 
            THEN 'Has Conversion Data But Not Marked Converted'
            ELSE 'Conversion Data Consistent'
        END as conversion_issue_type,
        
        -- Remediation recommendations
        CASE 
            WHEN isconverted_b = true AND status != 'Converted' THEN 'Update Status to Converted'
            WHEN isconverted_b = false AND status = 'Converted' THEN 'Update Status to Working or Disqualified'
            WHEN isconverted_b = true AND converteddate_d IS NULL THEN 'Add Conversion Date'
            WHEN isconverted_b = true AND convertedaccountid IS NULL THEN 'Link to Account Record'
            WHEN isconverted_b = true AND convertedcontactid IS NULL THEN 'Link to Contact Record'
            WHEN isconverted_b = false AND converteddate_d IS NOT NULL THEN 'Clear Conversion Date'
            ELSE 'No Action Required'
        END as recommended_action,
        
        -- Priority level for remediation
        CASE 
            WHEN isconverted_b = true AND (convertedaccountid IS NULL OR convertedcontactid IS NULL) THEN 1
            WHEN isconverted_b = true AND status != 'Converted' THEN 2
            WHEN isconverted_b = false AND status = 'Converted' THEN 2
            ELSE 3
        END as remediation_priority
        
    FROM latest_leads
    WHERE NOT (
        (isconverted_b = true AND status = 'Converted' AND converteddate_d IS NOT NULL 
         AND convertedaccountid IS NOT NULL AND convertedcontactid IS NOT NULL) OR
        (isconverted_b = false AND status != 'Converted' AND converteddate_d IS NULL 
         AND convertedaccountid IS NULL AND convertedcontactid IS NULL)
    )
)
SELECT 
    conversion_issue_type,
    COUNT(*) as affected_leads,
    recommended_action,
    remediation_priority,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM latest_leads), 2) as percentage_of_total,
    STRING_AGG(id, ', ' ORDER BY remediation_priority LIMIT 10) as sample_lead_ids
FROM conversion_issues
GROUP BY conversion_issue_type, recommended_action, remediation_priority
ORDER BY remediation_priority, affected_leads DESC
```

## Process Improvement Remediation

### Lead Source Quality Enhancement

Improves lead source classification and tracking for better attribution analysis:

#### Lead Source Standardization and Enhancement

```sql
WITH lead_source_analysis AS (
    SELECT 
        leadsource,
        COUNT(*) as lead_count,
        ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage,
        
        -- Lead source quality assessment
        CASE 
            WHEN leadsource IS NULL OR leadsource = '' THEN 'Missing Lead Source'
            WHEN LOWER(leadsource) IN ('web', 'website', 'online', 'internet') THEN 'Generic Web Source'
            WHEN LOWER(leadsource) LIKE '%test%' THEN 'Test Lead Source'
            WHEN LENGTH(leadsource) < 3 THEN 'Abbreviated Source'
            ELSE 'Specific Lead Source'
        END as source_quality_category,
        
        -- Standardization suggestions
        CASE 
            WHEN LOWER(leadsource) IN ('web', 'website', 'online', 'internet') THEN 'Website - Organic'
            WHEN LOWER(leadsource) LIKE '%google%' THEN 'Search - Google'
            WHEN LOWER(leadsource) LIKE '%linkedin%' THEN 'Social - LinkedIn'
            WHEN LOWER(leadsource) LIKE '%email%' THEN 'Email Campaign'
            WHEN LOWER(leadsource) LIKE '%referral%' THEN 'Referral Program'
            WHEN LOWER(leadsource) LIKE '%event%' OR LOWER(leadsource) LIKE '%trade%' THEN 'Event Marketing'
            ELSE leadsource
        END as standardized_lead_source,
        
        -- Conversion rate by source (for quality assessment)
        ROUND(100.0 * COUNT(CASE WHEN isconverted_b = true THEN 1 END) / COUNT(*), 2) as conversion_rate
        
    FROM latest_leads
    GROUP BY leadsource
)
SELECT 
    source_quality_category,
    COUNT(*) as unique_sources,
    SUM(lead_count) as total_affected_leads,
    ROUND(SUM(percentage), 2) as total_percentage,
    ROUND(AVG(conversion_rate), 2) as avg_conversion_rate,
    
    -- Recommended action
    CASE 
        WHEN source_quality_category = 'Missing Lead Source' THEN 'Implement Lead Source Tracking'
        WHEN source_quality_category = 'Generic Web Source' THEN 'Enhance Source Attribution Granularity'
        WHEN source_quality_category = 'Test Lead Source' THEN 'Remove Test Sources from Production'
        ELSE 'Standardize Source Naming'
    END as recommended_action,
    
    STRING_AGG(
        '"' || COALESCE(leadsource, 'NULL') || '" -> "' || standardized_lead_source || '"',
        '; ' ORDER BY lead_count DESC LIMIT 3
    ) as sample_standardizations
FROM lead_source_analysis
GROUP BY source_quality_category
ORDER BY total_affected_leads DESC
```

## Automated Remediation Implementation

### Priority Remediation Queue

Creates executable remediation workflows that can be scheduled for automatic data quality improvements:

```sql
WITH remediation_queue AS (
    SELECT 
        ll.id,
        ll.email,
        ll.firstname,
        ll.lastname,
        ll.company,
        ll.status,
        
        -- Calculate overall remediation priority score
        (
            CASE WHEN ll.email IS NULL OR ll.email = '' THEN 20 ELSE 0 END +
            CASE WHEN ll.email NOT LIKE '%@%.%' THEN 15 ELSE 0 END +
            CASE WHEN ll.firstname IS NULL OR ll.lastname IS NULL THEN 10 ELSE 0 END +
            CASE WHEN ll.company IS NULL OR ll.company = '' THEN 8 ELSE 0 END +
            CASE WHEN ll.status NOT IN ('Open', 'Working', 'Converted', 'Disqualified', 'Existing Customer') THEN 5 ELSE 0 END +
            CASE WHEN ll.isconverted_b = true AND ll.convertedaccountid IS NULL THEN 15 ELSE 0 END
        ) as remediation_priority_score,
        
        -- Estimated improvement in quality score
        (
            CASE WHEN ll.email IS NULL THEN 25 WHEN ll.email NOT LIKE '%@%.%' THEN 20 ELSE 0 END +
            CASE WHEN ll.firstname IS NULL OR ll.lastname IS NULL THEN 15 ELSE 0 END +
            CASE WHEN ll.company IS NULL THEN 20 ELSE 0 END +
            CASE WHEN ll.isconverted_b = true AND ll.convertedaccountid IS NULL THEN 20 ELSE 0 END
        ) as potential_score_improvement
        
    FROM latest_leads ll
    WHERE ll.email IS NULL OR ll.email = '' OR ll.email NOT LIKE '%@%.%' OR
          ll.firstname IS NULL OR ll.lastname IS NULL OR
          ll.company IS NULL OR ll.company = '' OR
          ll.status NOT IN ('Open', 'Working', 'Converted', 'Disqualified', 'Existing Customer') OR
          (ll.isconverted_b = true AND ll.convertedaccountid IS NULL)
)
SELECT 
    id,
    email,
    firstname,
    lastname,
    company,
    status,
    remediation_priority_score,
    potential_score_improvement,
    
    -- ROI estimate for remediation effort
    CASE 
        WHEN potential_score_improvement >= 50 THEN 'High ROI - Immediate Action'
        WHEN potential_score_improvement >= 25 THEN 'Medium ROI - Scheduled Remediation'
        ELSE 'Low ROI - Maintenance Priority'
    END as remediation_roi_category
    
FROM remediation_queue
WHERE remediation_priority_score > 0
ORDER BY remediation_priority_score DESC, potential_score_improvement DESC
LIMIT 500
```

### Remediation Impact Measurement

Tracks the effectiveness of remediation efforts and provides continuous improvement metrics:

#### Before/After Quality Measurement

```sql
WITH remediation_impact AS (
    SELECT 
        'Pre-Remediation' as measurement_period,
        COUNT(*) as total_leads,
        ROUND(AVG(
            CASE WHEN email IS NOT NULL AND email LIKE '%@%.%' THEN 100 ELSE 0 END
        ), 2) as avg_email_quality,
        ROUND(AVG(
            CASE WHEN firstname IS NOT NULL AND lastname IS NOT NULL THEN 100 ELSE 0 END
        ), 2) as avg_name_quality,
        ROUND(AVG(
            CASE WHEN company IS NOT NULL AND company != '' THEN 100 ELSE 0 END
        ), 2) as avg_company_quality,
        COUNT(CASE WHEN 
            email IS NOT NULL AND email LIKE '%@%.%' AND
            firstname IS NOT NULL AND lastname IS NOT NULL AND
            company IS NOT NULL AND company != '' THEN 1 
        END) as complete_records
    FROM latest_leads
)
SELECT 
    measurement_period,
    total_leads,
    avg_email_quality,
    avg_name_quality,
    avg_company_quality,
    complete_records,
    ROUND(100.0 * complete_records / total_leads, 2) as complete_records_percentage
FROM remediation_impact
```

## Training and Process Recommendations

### User Training Focus Areas

Based on data quality analysis results, provides specific training recommendations:

#### Training Priority Matrix

```sql
WITH training_needs AS (
    SELECT 
        'Email Data Quality' as training_topic,
        (SELECT COUNT(*) FROM latest_leads WHERE email IS NULL OR email NOT LIKE '%@%.%') as affected_records,
        'Sales and Marketing Teams' as target_audience,
        'High' as priority_level,
        'Email format validation, business vs personal email identification' as training_content
        
    UNION ALL
    
    SELECT 
        'Company Information Standards',
        (SELECT COUNT(*) FROM latest_leads WHERE company IS NULL OR company = ''),
        'Sales Development Representatives',
        'High',
        'Company name standardization, firmographic data collection best practices'
        
    UNION ALL
    
    SELECT 
        'Lead Status Management',
        (SELECT COUNT(*) FROM latest_leads WHERE status NOT IN ('Open', 'Working', 'Converted', 'Disqualified', 'Existing Customer')),
        'Sales Operations Team',
        'Medium',
        'Status value standardization using Configuration Reference values'
        
    UNION ALL
    
    SELECT 
        'Lead Conversion Process',
        (SELECT COUNT(*) FROM latest_leads WHERE isconverted_b = true AND (convertedaccountid IS NULL OR convertedcontactid IS NULL)),
        'Sales Representatives',
        'High',
        'Proper lead conversion workflow, relationship linking requirements'
)
SELECT 
    training_topic,
    affected_records,
    ROUND(100.0 * affected_records / (SELECT COUNT(*) FROM latest_leads), 2) as percentage_impact,
    target_audience,
    priority_level,
    training_content,
    
    -- ROI estimation for training investment
    CASE 
        WHEN affected_records > (SELECT COUNT(*) * 0.20 FROM latest_leads) THEN 'High ROI Training Investment'
        WHEN affected_records > (SELECT COUNT(*) * 0.10 FROM latest_leads) THEN 'Medium ROI Training Investment'
        ELSE 'Low ROI Training Investment'
    END as training_roi_assessment
FROM training_needs
ORDER BY 
    CASE priority_level 
        WHEN 'High' THEN 1 
        WHEN 'Medium' THEN 2 
        ELSE 3 
    END,
    affected_records DESC
```

## Remediation Success Metrics

### Key Performance Indicators

Defines measurable outcomes for successful data quality remediation:

#### Quality Improvement Tracking

```sql
WITH quality_kpis AS (
    SELECT 
        'Data Completeness' as kpi_category,
        'Email Field Population Rate' as kpi_name,
        ROUND(100.0 * COUNT(CASE WHEN email IS NOT NULL AND email != '' THEN 1 END) / COUNT(*), 2) as current_value,
        95.0 as target_value,
        '%' as unit_measure
    FROM latest_leads
    
    UNION ALL
    
    SELECT 
        'Data Completeness',
        'Complete Name Information Rate',
        ROUND(100.0 * COUNT(CASE WHEN firstname IS NOT NULL AND lastname IS NOT NULL THEN 1 END) / COUNT(*), 2),
        90.0,
        '%'
    FROM latest_leads
    
    UNION ALL
    
    SELECT 
        'Data Accuracy',
        'Valid Email Format Rate',
        ROUND(100.0 * COUNT(CASE WHEN email LIKE '%@%.%' THEN 1 END) / COUNT(*), 2),
        98.0,
        '%'
    FROM latest_leads
    
    UNION ALL
    
    SELECT 
        'Data Consistency',
        'Conversion Data Integrity Rate',
        ROUND(100.0 * COUNT(CASE 
            WHEN isconverted_b = true AND converteddate_d IS NOT NULL AND convertedaccountid IS NOT NULL 
            THEN 1 END) / COUNT(CASE WHEN isconverted_b = true THEN 1 END), 2),
        95.0,
        '%'
    FROM latest_leads
    
    UNION ALL
    
    SELECT 
        'Overall Quality',
        'Records Meeting Minimum Quality Standards',
        ROUND(100.0 * COUNT(CASE 
            WHEN email IS NOT NULL AND email LIKE '%@%.%' AND
                 firstname IS NOT NULL AND lastname IS NOT NULL AND
                 company IS NOT NULL AND company != ''
            THEN 1 END) / COUNT(*), 2),
        85.0,
        '%'
    FROM latest_leads
)
SELECT 
    kpi_category,
    kpi_name,
    current_value,
    target_value,
    current_value - target_value as gap_to_target,
    unit_measure,
    
    -- Performance status
    CASE 
        WHEN current_value >= target_value THEN 'Target Achieved'
        WHEN current_value >= target_value * 0.9 THEN 'Near Target'
        WHEN current_value >= target_value * 0.8 THEN 'Improvement Needed'
        ELSE 'Significant Gap'
    END as performance_status,
    
    -- Priority for improvement
    CASE 
        WHEN current_value < target_value * 0.8 THEN 'High Priority'
        WHEN current_value < target_value * 0.9 THEN 'Medium Priority'
        ELSE 'Low Priority'
    END as improvement_priority
    
FROM quality_kpis
ORDER BY 
    CASE improvement_priority 
        WHEN 'High Priority' THEN 1 
        WHEN 'Medium Priority' THEN 2 
        ELSE 3 
    END,
    gap_to_target
```

### Remediation Implementation Roadmap

Provides a structured approach to implementing data quality improvements:

#### 30-60-90 Day Remediation Plan

```sql
WITH implementation_phases AS (
    SELECT 
        'Phase 1: Immediate Fixes (0-30 days)' as phase,
        'Critical Data Issues' as focus_area,
        'Fix missing email addresses, correct invalid email formats, resolve conversion data inconsistencies' as activities,
        (SELECT COUNT(*) FROM latest_leads WHERE email IS NULL OR email NOT LIKE '%@%.%' OR 
         (isconverted_b = true AND convertedaccountid IS NULL)) as affected_records,
        'Data Operations Team' as responsible_team,
        'Automated scripts and manual data correction' as implementation_method
        
    UNION ALL
    
    SELECT 
        'Phase 2: Process Improvements (30-60 days)',
        'Data Collection Enhancement',
        'Implement lead source standardization, enhance name field validation, establish geographic data standards',
        (SELECT COUNT(*) FROM latest_leads WHERE leadsource IS NULL OR firstname IS NULL OR lastname IS NULL),
        'Sales Operations and Marketing Teams',
        'Workflow automation and validation rules'
        
    UNION ALL
    
    SELECT 
        'Phase 3: Preventive Measures (60-90 days)',
        'Quality Maintenance',
        'Deploy automated quality monitoring, implement user training programs, establish ongoing quality metrics',
        (SELECT COUNT(*) FROM latest_leads),
        'All User Teams',
        'Training programs and quality dashboards'
)
SELECT 
    phase,
    focus_area,
    activities,
    affected_records,
    responsible_team,
    implementation_method,
    
    -- Success criteria
    CASE 
        WHEN phase LIKE 'Phase 1%' THEN 'Reduce critical quality issues by 80%'
        WHEN phase LIKE 'Phase 2%' THEN 'Improve overall completion rates by 15%'
        ELSE 'Maintain quality standards above 90%'
    END as success_criteria,
    
    -- Estimated effort
    CASE 
        WHEN phase LIKE 'Phase 1%' THEN 'High - 40 hours'
        WHEN phase LIKE 'Phase 2%' THEN 'Medium - 60 hours'
        ELSE 'Medium - 30 hours'
    END as estimated_effort
    
FROM implementation_phases
ORDER BY phase
```

This comprehensive lead data quality remediation framework provides customers with actionable strategies to improve their lead data quality systematically while ensuring measurable outcomes and sustainable improvements over time.
