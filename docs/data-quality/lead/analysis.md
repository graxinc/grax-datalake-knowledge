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

This comprehensive lead data quality analysis framework provides customers with detailed insights into their lead data integrity, enabling proactive data quality management and continuous improvement initiatives. All patterns use Athena-compatible SQL syntax and integrate with the established Configuration Reference for adaptability across different Salesforce implementations.
