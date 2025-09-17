# Lead Data Quality Scoring

## Overview

This document provides quantitative scoring methodologies for lead data quality in the GRAX Data Lake. All scoring algorithms reference configuration values from [Configuration Reference](../../core-reference/configuration-reference.md) and use standardized 0-100 scoring scales for consistency across different data quality dimensions.

## Scoring Framework Architecture

### Core Scoring Principles

**Standardized Scale**: All scoring uses 0-100 scale where:

- **90-100**: Excellent quality, no action required
- **80-89**: Good quality, minor improvements recommended
- **70-79**: Acceptable quality, moderate improvements needed
- **60-69**: Poor quality, significant improvements required
- **Below 60**: Critical quality issues, immediate action required

**Weighted Scoring**: Different fields and quality dimensions receive weights based on business impact and importance for lead qualification and conversion processes.

**Adaptable Thresholds**: All scoring thresholds reference [Configuration Reference](../../core-reference/configuration-reference.md) values to ensure adaptability across different Salesforce implementations.

## Field-Level Quality Scoring

### Critical Field Scoring Algorithm

Evaluates individual field quality based on completeness, format validation, and business logic compliance:

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
        phone,
        numberofemployees_f,
        annualrevenue_f,
        isconverted_b,
        converteddate_d,
        convertedaccountid,
        convertedcontactid,
        createddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
field_scoring AS (
    SELECT 
        id,
        
        -- Email field scoring (Weight: 25%)
        CASE 
            WHEN email IS NULL OR email = '' THEN 0
            WHEN email NOT LIKE '%@%.%' THEN 20
            WHEN LOWER(email) LIKE '%test%' OR LOWER(email) LIKE '%example%' 
                 OR LOWER(email) LIKE '%dummy%' THEN 40
            WHEN LOWER(email) LIKE '%@gmail.com' OR LOWER(email) LIKE '%@yahoo.com' 
                 OR LOWER(email) LIKE '%@hotmail.com' THEN 75
            ELSE 100
        END as email_score,
        
        -- Name field scoring (Weight: 15%)
        CASE 
            WHEN firstname IS NULL AND lastname IS NULL THEN 0
            WHEN firstname IS NULL OR lastname IS NULL THEN 50
            WHEN firstname = '' OR lastname = '' THEN 60
            WHEN LENGTH(firstname) < 2 OR LENGTH(lastname) < 2 THEN 70
            ELSE 100
        END as name_score,
        
        -- Company field scoring (Weight: 20%)
        CASE 
            WHEN company IS NULL OR company = '' THEN 0
            WHEN LENGTH(company) < 3 THEN 40
            WHEN LOWER(company) LIKE '%test%' OR LOWER(company) LIKE '%example%' THEN 50
            ELSE 100
        END as company_score,
        
        -- Status field scoring (Weight: 10%) - using values from Configuration Reference
        CASE 
            WHEN status IS NULL THEN 0
            WHEN status NOT IN ('Open', 'Working', 'Converted', 'Disqualified', 'Existing Customer') THEN 30
            ELSE 100
        END as status_score,
        
        -- Geographic information scoring (Weight: 8%)
        CASE 
            WHEN country IS NULL OR country = '' THEN 0
            WHEN state IS NULL OR state = '' THEN 60
            ELSE 100
        END as geographic_score,
        
        -- Contact information scoring (Weight: 7%)
        CASE 
            WHEN phone IS NULL OR phone = '' THEN 0
            WHEN LENGTH(phone) < 10 THEN 50
            WHEN phone NOT REGEXP '^[0-9\-\+\(\)\s\.]+$' THEN 70
            ELSE 100
        END as phone_score,
        
        -- Firmographic scoring (Weight: 15%)
        CASE 
            WHEN numberofemployees_f IS NULL AND annualrevenue_f IS NULL THEN 0
            WHEN numberofemployees_f IS NULL OR annualrevenue_f IS NULL THEN 40
            WHEN numberofemployees_f <= 0 OR annualrevenue_f <= 0 THEN 20
            WHEN numberofemployees_f > 10000000 OR annualrevenue_f > 1000000000000 THEN 30
            -- Consistency check using thresholds from Configuration Reference
            WHEN (numberofemployees_f <= 50 AND annualrevenue_f > 100000000) OR
                 (numberofemployees_f > 1000 AND annualrevenue_f < 1000000) THEN 60
            ELSE 100
        END as firmographic_score
        
    FROM latest_leads
),
weighted_scoring AS (
    SELECT 
        id,
        email_score,
        name_score,
        company_score,
        status_score,
        geographic_score,
        phone_score,
        firmographic_score,
        
        -- Calculate overall field quality score with business-based weighting
        ROUND(
            (email_score * 0.25) +
            (name_score * 0.15) +
            (company_score * 0.20) +
            (status_score * 0.10) +
            (geographic_score * 0.08) +
            (phone_score * 0.07) +
            (firmographic_score * 0.15)
        , 2) as overall_field_score,
        
        -- Quality grade assignment
        CASE 
            WHEN (email_score * 0.25 + name_score * 0.15 + company_score * 0.20 + 
                  status_score * 0.10 + geographic_score * 0.08 + phone_score * 0.07 + 
                  firmographic_score * 0.15) >= 90 THEN 'A - Excellent'
            WHEN (email_score * 0.25 + name_score * 0.15 + company_score * 0.20 + 
                  status_score * 0.10 + geographic_score * 0.08 + phone_score * 0.07 + 
                  firmographic_score * 0.15) >= 80 THEN 'B - Good'
            WHEN (email_score * 0.25 + name_score * 0.15 + company_score * 0.20 + 
                  status_score * 0.10 + geographic_score * 0.08 + phone_score * 0.07 + 
                  firmographic_score * 0.15) >= 70 THEN 'C - Acceptable'
            WHEN (email_score * 0.25 + name_score * 0.15 + company_score * 0.20 + 
                  status_score * 0.10 + geographic_score * 0.08 + phone_score * 0.07 + 
                  firmographic_score * 0.15) >= 60 THEN 'D - Poor'
            ELSE 'F - Critical'
        END as quality_grade
        
    FROM field_scoring
)
SELECT * FROM weighted_scoring
```

## Record-Level Quality Scoring

### Comprehensive Record Quality Assessment

Combines field-level scores with relationship integrity and business logic validation:

```sql
WITH record_quality_scoring AS (
    SELECT 
        id,
        
        -- Data Completeness Score (40% weight)
        ROUND((
            CASE WHEN email IS NOT NULL AND email != '' THEN 20 ELSE 0 END +
            CASE WHEN firstname IS NOT NULL AND lastname IS NOT NULL THEN 15 ELSE 0 END +
            CASE WHEN company IS NOT NULL AND company != '' THEN 15 ELSE 0 END +
            CASE WHEN leadsource IS NOT NULL THEN 10 ELSE 0 END +
            CASE WHEN numberofemployees_f IS NOT NULL OR annualrevenue_f IS NOT NULL THEN 10 ELSE 0 END +
            CASE WHEN status IS NOT NULL THEN 10 ELSE 0 END
        ) * 0.40, 2) as completeness_score,
        
        -- Data Accuracy Score (30% weight)
        ROUND((
            CASE 
                WHEN email IS NOT NULL AND email LIKE '%@%.%' AND 
                     NOT (LOWER(email) LIKE '%test%' OR LOWER(email) LIKE '%example%') THEN 30
                WHEN email IS NOT NULL THEN 15
                ELSE 0 
            END +
            -- Status validation using Configuration Reference values
            CASE 
                WHEN status IN ('Open', 'Working', 'Converted', 'Disqualified', 'Existing Customer') THEN 30
                WHEN status IS NOT NULL THEN 15
                ELSE 0 
            END +
            CASE 
                WHEN numberofemployees_f IS NOT NULL AND numberofemployees_f > 0 AND 
                     numberofemployees_f <= 10000000 THEN 20
                WHEN numberofemployees_f IS NOT NULL THEN 10
                ELSE 0 
            END +
            CASE 
                WHEN annualrevenue_f IS NOT NULL AND annualrevenue_f > 0 AND 
                     annualrevenue_f <= 1000000000000 THEN 20
                WHEN annualrevenue_f IS NOT NULL THEN 10
                ELSE 0 
            END
        ) * 0.30, 2) as accuracy_score,
        
        -- Data Consistency Score (20% weight)
        ROUND((
            -- Conversion consistency
            CASE 
                WHEN isconverted_b = true AND converteddate_d IS NOT NULL AND 
                     convertedaccountid IS NOT NULL AND convertedcontactid IS NOT NULL THEN 50
                WHEN isconverted_b = false AND converteddate_d IS NULL AND 
                     convertedaccountid IS NULL AND convertedcontactid IS NULL THEN 50
                WHEN isconverted_b IS NOT NULL THEN 25
                ELSE 0 
            END +
            -- Status-conversion alignment
            CASE 
                WHEN isconverted_b = true AND status = 'Converted' THEN 25
                WHEN isconverted_b = false AND status != 'Converted' THEN 25
                WHEN status IS NOT NULL THEN 10
                ELSE 0 
            END +
            -- Employee-revenue consistency
            CASE 
                WHEN numberofemployees_f IS NOT NULL AND annualrevenue_f IS NOT NULL THEN
                    CASE 
                        WHEN (numberofemployees_f <= 50 AND annualrevenue_f > 100000000) OR
                             (numberofemployees_f > 1000 AND annualrevenue_f < 1000000) THEN 10
                        ELSE 25
                    END
                ELSE 0 
            END
        ) * 0.20, 2) as consistency_score,
        
        -- Data Integrity Score (10% weight)
        ROUND((
            -- Temporal integrity
            CASE 
                WHEN createddate_ts IS NOT NULL AND createddate_ts <= CURRENT_TIMESTAMP AND 
                     createddate_ts >= TIMESTAMP '2000-01-01 00:00:00' THEN 50
                WHEN createddate_ts IS NOT NULL THEN 25
                ELSE 0 
            END +
            -- Modification integrity  
            CASE 
                WHEN lastmodifieddate_ts IS NOT NULL AND lastmodifieddate_ts >= createddate_ts AND
                     lastmodifieddate_ts <= CURRENT_TIMESTAMP THEN 50
                WHEN lastmodifieddate_ts IS NOT NULL THEN 25
                ELSE 0 
            END
        ) * 0.10, 2) as integrity_score
        
    FROM latest_leads
)
SELECT 
    id,
    completeness_score,
    accuracy_score,
    consistency_score,
    integrity_score,
    
    -- Calculate overall record quality score
    ROUND(completeness_score + accuracy_score + consistency_score + integrity_score, 2) as overall_record_score,
    
    -- Assign record quality grade
    CASE 
        WHEN (completeness_score + accuracy_score + consistency_score + integrity_score) >= 90 THEN 'A - Excellent'
        WHEN (completeness_score + accuracy_score + consistency_score + integrity_score) >= 80 THEN 'B - Good'
        WHEN (completeness_score + accuracy_score + consistency_score + integrity_score) >= 70 THEN 'C - Acceptable'
        WHEN (completeness_score + accuracy_score + consistency_score + integrity_score) >= 60 THEN 'D - Poor'
        ELSE 'F - Critical'
    END as record_quality_grade
    
FROM record_quality_scoring
```

## Object-Level Quality Scoring

### Aggregate Lead Data Quality Metrics

Provides overall quality assessment for the entire lead object with distribution analysis:

```sql
WITH object_level_metrics AS (
    SELECT 
        -- Overall object quality metrics
        COUNT(*) as total_lead_records,
        ROUND(AVG(overall_record_score), 2) as avg_object_quality_score,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY overall_record_score), 2) as median_object_quality_score,
        MIN(overall_record_score) as min_record_score,
        MAX(overall_record_score) as max_record_score,
        ROUND(STDDEV_POP(overall_record_score), 2) as score_standard_deviation,
        
        -- Quality grade distribution
        COUNT(CASE WHEN record_quality_grade = 'A - Excellent' THEN 1 END) as excellent_records,
        COUNT(CASE WHEN record_quality_grade = 'B - Good' THEN 1 END) as good_records,
        COUNT(CASE WHEN record_quality_grade = 'C - Acceptable' THEN 1 END) as acceptable_records,
        COUNT(CASE WHEN record_quality_grade = 'D - Poor' THEN 1 END) as poor_records,
        COUNT(CASE WHEN record_quality_grade = 'F - Critical' THEN 1 END) as critical_records,
        
        -- Quality percentages
        ROUND(100.0 * COUNT(CASE WHEN record_quality_grade = 'A - Excellent' THEN 1 END) / COUNT(*), 2) as excellent_percentage,
        ROUND(100.0 * COUNT(CASE WHEN record_quality_grade = 'B - Good' THEN 1 END) / COUNT(*), 2) as good_percentage,
        ROUND(100.0 * COUNT(CASE WHEN record_quality_grade = 'C - Acceptable' THEN 1 END) / COUNT(*), 2) as acceptable_percentage,
        ROUND(100.0 * COUNT(CASE WHEN record_quality_grade = 'D - Poor' THEN 1 END) / COUNT(*), 2) as poor_percentage,
        ROUND(100.0 * COUNT(CASE WHEN record_quality_grade = 'F - Critical' THEN 1 END) / COUNT(*), 2) as critical_percentage,
        
        -- Overall object quality grade
        CASE 
            WHEN AVG(overall_record_score) >= 90 THEN 'A - Excellent'
            WHEN AVG(overall_record_score) >= 80 THEN 'B - Good'
            WHEN AVG(overall_record_score) >= 70 THEN 'C - Acceptable'
            WHEN AVG(overall_record_score) >= 60 THEN 'D - Poor'
            ELSE 'F - Critical'
        END as overall_object_quality_grade
        
    FROM record_quality_scoring
)
SELECT * FROM object_level_metrics
```

### Quality Trend Scoring Over Time

Tracks quality score trends to identify improvement or degradation patterns:

```sql
WITH monthly_quality_trends AS (
    SELECT 
        DATE_TRUNC('month', l.createddate_ts) as creation_month,
        COUNT(*) as monthly_lead_count,
        ROUND(AVG(r.overall_record_score), 2) as monthly_avg_quality_score,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY r.overall_record_score), 2) as monthly_median_quality_score,
        
        -- Monthly quality grade distribution
        COUNT(CASE WHEN r.record_quality_grade = 'A - Excellent' THEN 1 END) as monthly_excellent_count,
        COUNT(CASE WHEN r.record_quality_grade = 'F - Critical' THEN 1 END) as monthly_critical_count,
        
        -- Monthly quality percentages
        ROUND(100.0 * COUNT(CASE WHEN r.record_quality_grade = 'A - Excellent' THEN 1 END) / COUNT(*), 2) as monthly_excellent_percentage,
        ROUND(100.0 * COUNT(CASE WHEN r.record_quality_grade = 'F - Critical' THEN 1 END) / COUNT(*), 2) as monthly_critical_percentage
        
    FROM lakehouse.object_lead l
    JOIN record_quality_scoring r ON l.id = r.id
    WHERE l.grax__deleted IS NULL 
        AND l.createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    GROUP BY DATE_TRUNC('month', l.createddate_ts)
),
trend_analysis AS (
    SELECT 
        creation_month,
        monthly_lead_count,
        monthly_avg_quality_score,
        monthly_excellent_percentage,
        monthly_critical_percentage,
        
        -- Month-over-month trend calculations
        monthly_avg_quality_score - LAG(monthly_avg_quality_score, 1) OVER (ORDER BY creation_month) as mom_score_change,
        
        -- 3-month moving average for smoother trend identification
        AVG(monthly_avg_quality_score) OVER (ORDER BY creation_month ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) as three_month_avg_score,
        
        -- Quality trend classification
        CASE 
            WHEN monthly_avg_quality_score - LAG(monthly_avg_quality_score, 1) OVER (ORDER BY creation_month) > 2 THEN 'Improving'
            WHEN monthly_avg_quality_score - LAG(monthly_avg_quality_score, 1) OVER (ORDER BY creation_month) < -2 THEN 'Declining'
            ELSE 'Stable'
        END as quality_trend_direction
        
    FROM monthly_quality_trends
)
SELECT 
    creation_month,
    monthly_lead_count,
    monthly_avg_quality_score,
    monthly_excellent_percentage,
    monthly_critical_percentage,
    ROUND(mom_score_change, 2) as mom_score_change,
    ROUND(three_month_avg_score, 2) as three_month_avg_score,
    quality_trend_direction
FROM trend_analysis
ORDER BY creation_month DESC
```

## Quality Scoring Summary Report

### Executive Quality Dashboard Metrics

Provides high-level quality metrics suitable for executive reporting and dashboard presentation:

```sql
WITH executive_summary AS (
    SELECT 
        'Lead Object Quality Summary' as metric_category,
        
        -- Overall quality metrics from object_level_metrics
        total_lead_records,
        avg_object_quality_score as overall_avg_quality_score,
        overall_object_quality_grade as overall_quality_grade,
        
        -- Quality distribution
        excellent_percentage as excellent_quality_percentage,
        good_percentage as good_quality_percentage,
        critical_percentage as critical_quality_percentage,
        
        -- Key performance indicators
        CASE 
            WHEN avg_object_quality_score >= 85 THEN 'High Quality'
            WHEN avg_object_quality_score >= 75 THEN 'Moderate Quality'
            ELSE 'Improvement Needed'
        END as quality_assessment,
        
        CASE 
            WHEN critical_percentage <= 5 THEN 'Low Risk'
            WHEN critical_percentage <= 15 THEN 'Moderate Risk'
            ELSE 'High Risk'
        END as data_risk_level
        
    FROM object_level_metrics
),
actionable_insights AS (
    SELECT 
        -- Priority improvement areas
        CASE 
            WHEN (SELECT critical_percentage FROM object_level_metrics) > 15 THEN 
                'URGENT: Address critical quality issues affecting ' || 
                CAST((SELECT critical_percentage FROM object_level_metrics) AS VARCHAR) || '% of leads'
            WHEN (SELECT poor_percentage FROM object_level_metrics) > 25 THEN 
                'HIGH: Improve poor quality records affecting ' || 
                CAST((SELECT poor_percentage FROM object_level_metrics) AS VARCHAR) || '% of leads'
            WHEN (SELECT acceptable_percentage FROM object_level_metrics) > 35 THEN 
                'MEDIUM: Enhance acceptable quality records for better lead qualification'
            ELSE 'LOW: Maintain current high quality standards with minor optimizations'
        END as priority_recommendation,
        
        -- Success metrics
        CASE 
            WHEN (SELECT excellent_percentage FROM object_level_metrics) >= 60 THEN 
                'Excellent: ' || CAST((SELECT excellent_percentage FROM object_level_metrics) AS VARCHAR) || '% of leads meet highest quality standards'
            WHEN (SELECT excellent_percentage + good_percentage FROM object_level_metrics) >= 80 THEN 
                'Good: ' || CAST((SELECT excellent_percentage + good_percentage FROM object_level_metrics) AS VARCHAR) || '% of leads meet good+ quality standards'
            ELSE 'Improvement Opportunity: Focus on increasing high-quality lead percentage'
        END as success_metric
)
SELECT 
    es.metric_category,
    es.total_lead_records,
    es.overall_avg_quality_score,
    es.overall_quality_grade,
    es.excellent_quality_percentage,
    es.good_quality_percentage,
    es.critical_quality_percentage,
    es.quality_assessment,
    es.data_risk_level,
    ai.priority_recommendation,
    ai.success_metric
FROM executive_summary es
CROSS JOIN actionable_insights ai
```

## Quality Score Benchmarking

### Industry and Historical Benchmarks

Compares current quality scores against industry benchmarks and historical performance:

```sql
WITH quality_benchmarks AS (
    SELECT 
        -- Current performance
        (SELECT avg_object_quality_score FROM object_level_metrics) as current_quality_score,
        (SELECT excellent_percentage FROM object_level_metrics) as current_excellent_percentage,
        (SELECT critical_percentage FROM object_level_metrics) as current_critical_percentage,
        
        -- Industry benchmarks (configurable values)
        85.0 as industry_avg_quality_score,
        45.0 as industry_excellent_percentage,
        8.0 as industry_critical_percentage,
        
        -- Historical comparison (6 months ago)
        COALESCE((
            SELECT AVG(monthly_avg_quality_score) 
            FROM trend_analysis 
            WHERE creation_month >= DATE_ADD('month', -9, CURRENT_DATE)
                AND creation_month < DATE_ADD('month', -6, CURRENT_DATE)
        ), 0) as historical_quality_score,
        
        -- Performance vs benchmarks
        CASE 
            WHEN (SELECT avg_object_quality_score FROM object_level_metrics) >= 85.0 THEN 'Above Industry Average'
            WHEN (SELECT avg_object_quality_score FROM object_level_metrics) >= 80.0 THEN 'Near Industry Average'
            ELSE 'Below Industry Average'
        END as industry_comparison,
        
        -- Improvement trajectory
        CASE 
            WHEN (SELECT avg_object_quality_score FROM object_level_metrics) > 
                 COALESCE((SELECT AVG(monthly_avg_quality_score) FROM trend_analysis 
                          WHERE creation_month >= DATE_ADD('month', -9, CURRENT_DATE)
                              AND creation_month < DATE_ADD('month', -6, CURRENT_DATE)), 0) + 2
            THEN 'Significant Improvement'
            WHEN (SELECT avg_object_quality_score FROM object_level_metrics) > 
                 COALESCE((SELECT AVG(monthly_avg_quality_score) FROM trend_analysis 
                          WHERE creation_month >= DATE_ADD('month', -9, CURRENT_DATE)
                              AND creation_month < DATE_ADD('month', -6, CURRENT_DATE)), 0)
            THEN 'Improving'
            WHEN (SELECT avg_object_quality_score FROM object_level_metrics) < 
                 COALESCE((SELECT AVG(monthly_avg_quality_score) FROM trend_analysis 
                          WHERE creation_month >= DATE_ADD('month', -9, CURRENT_DATE)
                              AND creation_month < DATE_ADD('month', -6, CURRENT_DATE)), 0) - 2
            THEN 'Declining'
            ELSE 'Stable'
        END as improvement_trajectory
)
SELECT 
    current_quality_score,
    industry_avg_quality_score,
    ROUND(current_quality_score - industry_avg_quality_score, 2) as industry_gap,
    historical_quality_score,
    ROUND(current_quality_score - historical_quality_score, 2) as historical_improvement,
    current_excellent_percentage,
    industry_excellent_percentage,
    ROUND(current_excellent_percentage - industry_excellent_percentage, 2) as excellent_gap,
    current_critical_percentage,
    industry_critical_percentage,
    ROUND(current_critical_percentage - industry_critical_percentage, 2) as critical_gap,
    industry_comparison,
    improvement_trajectory
FROM quality_benchmarks
```

This comprehensive lead data quality scoring framework enables customers to quantify their data quality posture, track improvements over time, and benchmark against industry standards while maintaining adaptability across different Salesforce implementations.
