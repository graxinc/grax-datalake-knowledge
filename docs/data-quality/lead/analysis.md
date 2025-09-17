# Lead Data Quality Analysis

## Overview

This document provides comprehensive data quality analysis patterns for lead records in the GRAX Data Lake. All analysis queries reference configuration values from [Configuration Reference](../../core-reference/configuration-reference.md) to ensure adaptability across different Salesforce implementations.

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
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
)
SELECT * FROM latest_leads
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
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
completeness_analysis AS (
    SELECT 
        -- Critical identification fields
        COUNT(*) as total_leads,
        COUNT(email) as has_email,
        COUNT(CASE WHEN email IS NOT NULL AND email != '' THEN 1 END) as has_valid_email,
        
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
        
    FROM latest_leads
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

### Lead Status Distribution Analysis

Analyzes the distribution of lead status values against expected values from [Configuration Reference](../../core-reference/configuration-reference.md):

```sql
WITH latest_leads AS (
    SELECT 
        id,
        status,
        isconverted_b,
        createddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
status_analysis AS (
    SELECT 
        status,
        COUNT(*) as lead_count,
        COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () as percentage,
        
        -- Expected status values from Configuration Reference
        CASE 
            WHEN status IN ('Open', 'Working', 'Converted', 'Disqualified', 'Existing Customer') 
            THEN 'Expected'
            WHEN status IS NULL 
            THEN 'Missing'
            ELSE 'Unexpected'
        END as status_category,
        
        MIN(createddate_ts) as earliest_use,
        MAX(createddate_ts) as latest_use
        
    FROM latest_leads
    GROUP BY status
)
SELECT 
    status,
    status_category,
    lead_count,
    ROUND(percentage, 2) as percentage,
    earliest_use,
    latest_use
FROM status_analysis
ORDER BY lead_count DESC
```

## Accuracy Analysis

### Email Format Validation

Validates email addresses against business standards and identifies common formatting issues:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        email,
        company,
        leadsource,
        createddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
email_validation AS (
    SELECT 
        id,
        email,
        company,
        leadsource,
        createddate_ts,
        
        -- Basic email format validation
        CASE 
            WHEN email IS NULL THEN 'Missing'
            WHEN email = '' THEN 'Empty'
            WHEN email LIKE '%@%.%' THEN 'Valid Format'
            ELSE 'Invalid Format'
        END as email_format_status,
        
        -- Test email detection
        CASE 
            WHEN LOWER(email) LIKE '%test%' THEN true
            WHEN LOWER(email) LIKE '%example%' THEN true  
            WHEN LOWER(email) LIKE '%dummy%' THEN true
            WHEN LOWER(email) LIKE '%fake%' THEN true
            ELSE false
        END as is_test_email,
        
        -- Domain analysis
        CASE 
            WHEN email LIKE '%@%.%' THEN LOWER(SPLIT_PART(email, '@', 2))
            ELSE NULL
        END as email_domain,
        
        -- Business vs personal email classification
        CASE 
            WHEN LOWER(email) LIKE '%@gmail.com' OR 
                 LOWER(email) LIKE '%@yahoo.com' OR 
                 LOWER(email) LIKE '%@hotmail.com' OR
                 LOWER(email) LIKE '%@outlook.com' OR
                 LOWER(email) LIKE '%@aol.com'
            THEN 'Personal'
            WHEN email LIKE '%@%.%' THEN 'Business'
            ELSE 'Unknown'
        END as email_type
        
    FROM latest_leads
    WHERE email IS NOT NULL
)
SELECT 
    email_format_status,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM email_validation
GROUP BY email_format_status

UNION ALL

SELECT 
    'Test Emails' as email_format_status,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / (SELECT COUNT(*) FROM email_validation), 2) as percentage
FROM email_validation
WHERE is_test_email = true

UNION ALL

SELECT 
    email_type as email_format_status,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM email_validation
WHERE email_type IN ('Personal', 'Business')
GROUP BY email_type
```

### Firmographic Data Validation

Validates company sizing and revenue data against business logic and expected ranges:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        company,
        industry,
        numberofemployees_f,
        annualrevenue_f,
        country,
        createddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
firmographic_validation AS (
    SELECT 
        id,
        company,
        industry,
        numberofemployees_f,
        annualrevenue_f,
        country,
        
        -- Employee count validation (using thresholds from Configuration Reference)
        CASE 
            WHEN numberofemployees_f IS NULL THEN 'Missing'
            WHEN numberofemployees_f <= 0 THEN 'Invalid'
            WHEN numberofemployees_f > 10000000 THEN 'Unrealistic'
            WHEN numberofemployees_f <= 50 THEN 'SMB Range'
            WHEN numberofemployees_f <= 250 THEN 'Mid-Market Range'
            ELSE 'Enterprise Range'
        END as employee_count_category,
        
        -- Revenue validation (using thresholds from Configuration Reference)
        CASE 
            WHEN annualrevenue_f IS NULL THEN 'Missing'
            WHEN annualrevenue_f <= 0 THEN 'Invalid'
            WHEN annualrevenue_f > 1000000000000 THEN 'Unrealistic'
            WHEN annualrevenue_f <= 10000000 THEN 'SMB Range'
            WHEN annualrevenue_f <= 100000000 THEN 'Mid-Market Range'
            ELSE 'Enterprise Range'
        END as revenue_category,
        
        -- Employee-Revenue consistency check
        CASE 
            WHEN numberofemployees_f IS NULL OR annualrevenue_f IS NULL THEN 'Insufficient Data'
            WHEN numberofemployees_f <= 50 AND annualrevenue_f > 100000000 THEN 'Inconsistent - Small Team High Revenue'
            WHEN numberofemployees_f > 1000 AND annualrevenue_f < 1000000 THEN 'Inconsistent - Large Team Low Revenue'
            ELSE 'Reasonable'
        END as consistency_check
        
    FROM latest_leads
)
SELECT 
    'Employee Count Distribution' as analysis_type,
    employee_count_category as category,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM firmographic_validation
GROUP BY employee_count_category

UNION ALL

SELECT 
    'Revenue Distribution' as analysis_type,
    revenue_category as category,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM firmographic_validation
GROUP BY revenue_category

UNION ALL

SELECT 
    'Consistency Analysis' as analysis_type,
    consistency_check as category,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM firmographic_validation
GROUP BY consistency_check
```

## Consistency Analysis

### Lead Conversion Consistency

Analyzes consistency between lead conversion status and related conversion data:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        email,
        status,
        isconverted_b,
        converteddate_d,
        convertedaccountid,
        convertedcontactid,
        convertedopportunityid,
        createddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
conversion_consistency AS (
    SELECT 
        id,
        status,
        isconverted_b,
        converteddate_d,
        
        -- Conversion consistency validation
        CASE 
            WHEN isconverted_b = true AND converteddate_d IS NOT NULL 
                 AND convertedaccountid IS NOT NULL AND convertedcontactid IS NOT NULL
            THEN 'Fully Consistent'
            
            WHEN isconverted_b = true AND converteddate_d IS NULL
            THEN 'Missing Conversion Date'
            
            WHEN isconverted_b = true AND convertedaccountid IS NULL
            THEN 'Missing Account Link'
            
            WHEN isconverted_b = true AND convertedcontactid IS NULL  
            THEN 'Missing Contact Link'
            
            WHEN isconverted_b = false AND (converteddate_d IS NOT NULL 
                 OR convertedaccountid IS NOT NULL OR convertedcontactid IS NOT NULL)
            THEN 'Inconsistent - Has Conversion Data But Not Converted'
            
            WHEN isconverted_b IS NULL
            THEN 'Missing Conversion Status'
            
            ELSE 'Consistent'
        END as conversion_consistency,
        
        -- Status alignment with conversion (using values from Configuration Reference)
        CASE 
            WHEN isconverted_b = true AND status != 'Converted'
            THEN 'Status Mismatch - Converted But Wrong Status'
            
            WHEN isconverted_b = false AND status = 'Converted'
            THEN 'Status Mismatch - Not Converted But Status Says Converted'
            
            ELSE 'Status Aligned'
        END as status_alignment
        
    FROM latest_leads
)
SELECT 
    conversion_consistency,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM conversion_consistency
GROUP BY conversion_consistency

UNION ALL

SELECT 
    status_alignment as conversion_consistency,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM conversion_consistency
WHERE status_alignment != 'Status Aligned'
GROUP BY status_alignment
```

### Lead-to-Opportunity Relationship Validation

Validates the relationship integrity between converted leads and their associated opportunities:

```sql
WITH latest_leads AS (
    SELECT 
        id as lead_id,
        email,
        company,
        isconverted_b,
        convertedopportunityid,
        convertedaccountid,
        convertedcontactid,
        converteddate_d,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
latest_opportunities AS (
    SELECT 
        id as opportunity_id,
        name as opportunity_name,
        accountid,
        stagename,
        createddate_ts as opp_created_date,
        grax__idseq
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
converted_leads AS (
    SELECT * FROM latest_leads WHERE isconverted_b = true
),
relationship_validation AS (
    SELECT 
        cl.lead_id,
        cl.email,
        cl.company,
        cl.convertedopportunityid,
        cl.convertedaccountid,
        cl.converteddate_d,
        lo.opportunity_id,
        lo.opportunity_name,
        lo.accountid as opp_accountid,
        lo.stagename,
        
        -- Relationship validation
        CASE 
            WHEN cl.convertedopportunityid IS NULL THEN 'Missing Opportunity Link'
            WHEN lo.opportunity_id IS NULL THEN 'Broken Opportunity Link'
            WHEN cl.convertedaccountid != lo.accountid THEN 'Account Mismatch'
            ELSE 'Valid Relationship'
        END as relationship_status
        
    FROM converted_leads cl
    LEFT JOIN latest_opportunities lo ON cl.convertedopportunityid = lo.opportunity_id
)
SELECT 
    relationship_status,
    COUNT(*) as converted_lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM relationship_validation
GROUP BY relationship_status
ORDER BY converted_lead_count DESC
```

## Integrity Analysis

### Historical Data Pattern Analysis

Identifies unusual patterns in lead creation and modification that may indicate data quality issues:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        email,
        company,
        status,
        leadsource,
        createddate_ts,
        lastmodifieddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
temporal_analysis AS (
    SELECT 
        id,
        email,
        company,
        status,
        leadsource,
        createddate_ts,
        lastmodifieddate_ts,
        
        -- Creation date validation
        CASE 
            WHEN createddate_ts IS NULL THEN 'Missing Creation Date'
            WHEN createddate_ts > CURRENT_TIMESTAMP THEN 'Future Creation Date'
            WHEN createddate_ts < TIMESTAMP '2000-01-01 00:00:00' THEN 'Invalid Creation Date'
            ELSE 'Valid Creation Date'
        END as creation_date_status,
        
        -- Modification patterns
        CASE 
            WHEN lastmodifieddate_ts IS NULL THEN 'Missing Modification Date'
            WHEN lastmodifieddate_ts < createddate_ts THEN 'Modification Before Creation'
            WHEN lastmodifieddate_ts > CURRENT_TIMESTAMP THEN 'Future Modification Date'
            ELSE 'Valid Modification Date'
        END as modification_date_status,
        
        -- Age-based analysis
        DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) as lead_age_days,
        
        CASE 
            WHEN DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) > 1095 THEN 'Very Old (3+ years)'
            WHEN DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) > 730 THEN 'Old (2-3 years)'
            WHEN DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) > 365 THEN 'Aging (1-2 years)'
            WHEN DATE_DIFF('day', createddate_ts, CURRENT_TIMESTAMP) > 90 THEN 'Mature (3-12 months)'
            ELSE 'Recent (< 3 months)'
        END as lead_age_category
        
    FROM latest_leads
)
SELECT 
    'Creation Date Analysis' as analysis_type,
    creation_date_status as status_category,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM temporal_analysis
GROUP BY creation_date_status

UNION ALL

SELECT 
    'Modification Date Analysis' as analysis_type,
    modification_date_status as status_category,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM temporal_analysis
GROUP BY modification_date_status

UNION ALL

SELECT 
    'Lead Age Distribution' as analysis_type,
    lead_age_category as status_category,
    COUNT(*) as lead_count,
    ROUND(100.0 * COUNT(*) / SUM(COUNT(*)) OVER (), 2) as percentage
FROM temporal_analysis
GROUP BY lead_age_category
```

## Quality Trend Analysis

### Monthly Data Quality Trends

Tracks data quality metrics over time to identify improvement or degradation patterns:

```sql
WITH latest_leads AS (
    SELECT 
        id,
        email,
        company,
        status,
        numberofemployees_f,
        annualrevenue_f,
        createddate_ts,
        grax__idseq
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL 
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    QUALIFY ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) = 1
),
monthly_quality AS (
    SELECT 
        DATE_TRUNC('month', createddate_ts) as creation_month,
        COUNT(*) as total_leads,
        
        -- Completeness metrics
        COUNT(email) as leads_with_email,
        COUNT(CASE WHEN email LIKE '%@%.%' THEN 1 END) as leads_with_valid_email,
        COUNT(company) as leads_with_company,
        COUNT(status) as leads_with_status,
        
        -- Firmographic completeness
        COUNT(numberofemployees_f) as leads_with_employee_count,
        COUNT(annualrevenue_f) as leads_with_revenue,
        
        -- Data quality percentages
        ROUND(100.0 * COUNT(email) / COUNT(*), 2) as email_completion_rate,
        ROUND(100.0 * COUNT(CASE WHEN email LIKE '%@%.%' THEN 1 END) / COUNT(*), 2) as valid_email_rate,
        ROUND(100.0 * COUNT(company) / COUNT(*), 2) as company_completion_rate,
        ROUND(100.0 * COUNT(numberofemployees_f) / COUNT(*), 2) as employee_count_rate,
        ROUND(100.0 * COUNT(annualrevenue_f) / COUNT(*), 2) as revenue_completion_rate
        
    FROM latest_leads
    GROUP BY DATE_TRUNC('month', createddate_ts)
)
SELECT 
    creation_month,
    total_leads,
    email_completion_rate,
    valid_email_rate,
    company_completion_rate,
    employee_count_rate,
    revenue_completion_rate,
    
    -- Month-over-month trends
    LAG(email_completion_rate) OVER (ORDER BY creation_month) as prev_email_rate,
    email_completion_rate - LAG(email_completion_rate) OVER (ORDER BY creation_month) as email_rate_change,
    
    LAG(company_completion_rate) OVER (ORDER BY creation_month) as prev_company_rate,
    company_completion_rate - LAG(company_completion_rate) OVER (ORDER BY creation_month) as company_rate_change
    
FROM monthly_quality
ORDER BY creation_month DESC
```

This comprehensive lead data quality analysis framework provides customers with detailed insights into their lead data integrity, enabling proactive data quality management and continuous improvement initiatives.
