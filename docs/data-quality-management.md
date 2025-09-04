# Data Quality Management

## Overview

Data quality directly impacts sales funnel accuracy, customer segmentation precision, and business intelligence reliability. This guide provides comprehensive strategies for identifying, analyzing, and remedying data quality issues across Salesforce objects.

## Data Quality Dimensions

### 1. Completeness

**Definition**: Missing critical fields required for business processes

**Impact Areas**:
- Lead qualification accuracy
- Customer segmentation capability
- Contact information for outreach

### 2. Accuracy

**Definition**: Correctness of data values

**Common Issues**:
- Invalid email formats
- Unrealistic revenue or employee counts
- Outdated company information

### 3. Consistency

**Definition**: Uniformity of data across related records

**Examples**:
- Inconsistent company names across leads and accounts
- Mismatched segmentation classifications
- Conflicting data between related objects

### 4. Validity

**Definition**: Data conforming to defined formats and business rules

**Validation Rules**:
- Email format validation
- Phone number structure
- Realistic value ranges for revenue/employees

### 5. Uniqueness

**Definition**: Absence of duplicate records

**Duplicate Types**:
- Same person with multiple lead records
- Company with multiple account records
- Email addresses appearing multiple times

## Lead Data Quality Analysis

### Completeness Assessment

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
completeness_analysis AS (
    SELECT 
        COUNT(*) as total_leads,
        
        -- Critical Contact Fields
        COUNT(email) as email_populated,
        COUNT(firstname) as firstname_populated,
        COUNT(lastname) as lastname_populated,
        COUNT(phone) as phone_populated,
        
        -- Critical Company Fields for Segmentation
        COUNT(company) as company_populated,
        COUNT(industry) as industry_populated,
        COUNT(annualrevenue_f) as revenue_populated,
        COUNT(numberofemployees_f) as employees_populated,
        
        -- Critical Qualification Fields
        COUNT(status) as status_populated,
        COUNT(leadsource) as source_populated,
        
        -- Geographic Fields
        COUNT(country) as country_populated,
        COUNT(state) as state_populated
        
    FROM latest_lead
)
SELECT 
    total_leads,
    
    -- Calculate Completeness Percentages
    ROUND(email_populated * 100.0 / total_leads, 2) as email_completeness_pct,
    ROUND(firstname_populated * 100.0 / total_leads, 2) as firstname_completeness_pct,
    ROUND(lastname_populated * 100.0 / total_leads, 2) as lastname_completeness_pct,
    ROUND(phone_populated * 100.0 / total_leads, 2) as phone_completeness_pct,
    
    -- Company Information Completeness
    ROUND(company_populated * 100.0 / total_leads, 2) as company_completeness_pct,
    ROUND(industry_populated * 100.0 / total_leads, 2) as industry_completeness_pct,
    ROUND(revenue_populated * 100.0 / total_leads, 2) as revenue_completeness_pct,
    ROUND(employees_populated * 100.0 / total_leads, 2) as employees_completeness_pct,
    
    -- Qualification Fields Completeness
    ROUND(status_populated * 100.0 / total_leads, 2) as status_completeness_pct,
    ROUND(source_populated * 100.0 / total_leads, 2) as source_completeness_pct,
    
    -- Geographic Completeness
    ROUND(country_populated * 100.0 / total_leads, 2) as country_completeness_pct
    
FROM completeness_analysis
```

### Validity Assessment

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
validity_issues AS (
    SELECT 
        id,
        email,
        company,
        annualrevenue_f,
        numberofemployees_f,
        
        -- Email Validity Checks
        CASE 
            WHEN email IS NOT NULL AND email NOT LIKE '%@%.%' THEN 'Invalid Email Format'
            WHEN email LIKE '%@example.com' OR email LIKE '%@test.%' THEN 'Test Email Address'
            ELSE NULL
        END as email_issue,
        
        -- Company Data Validity
        CASE 
            WHEN company IS NOT NULL AND (
                LOWER(company) LIKE '%test%' OR 
                LOWER(company) LIKE '%example%' OR
                LOWER(company) = 'n/a' OR
                LENGTH(TRIM(company)) < 2
            ) THEN 'Invalid Company Name'
            ELSE NULL
        END as company_issue,
        
        -- Revenue Validity
        CASE 
            WHEN annualrevenue_f IS NOT NULL AND annualrevenue_f <= 0 THEN 'Invalid Revenue Amount'
            WHEN annualrevenue_f IS NOT NULL AND annualrevenue_f > 1000000000000 THEN 'Unrealistic Revenue Amount'
            ELSE NULL
        END as revenue_issue,
        
        -- Employee Count Validity
        CASE 
            WHEN numberofemployees_f IS NOT NULL AND numberofemployees_f <= 0 THEN 'Invalid Employee Count'
            WHEN numberofemployees_f IS NOT NULL AND numberofemployees_f > 10000000 THEN 'Unrealistic Employee Count'
            ELSE NULL
        END as employee_issue,
        
        -- Status/Conversion Consistency
        CASE 
            WHEN status = 'Converted' AND isconverted_b != true THEN 'Status/Conversion Flag Mismatch'
            WHEN isconverted_b = true AND converteddate_d IS NULL THEN 'Missing Conversion Date'
            ELSE NULL
        END as status_issue
        
    FROM latest_lead
)
SELECT 
    'Email Issues' as issue_type,
    COUNT(CASE WHEN email_issue IS NOT NULL THEN 1 END) as issue_count,
    ROUND(COUNT(CASE WHEN email_issue IS NOT NULL THEN 1 END) * 100.0 / COUNT(*), 2) as issue_percentage
FROM validity_issues

UNION ALL

SELECT 
    'Company Issues' as issue_type,
    COUNT(CASE WHEN company_issue IS NOT NULL THEN 1 END),
    ROUND(COUNT(CASE WHEN company_issue IS NOT NULL THEN 1 END) * 100.0 / COUNT(*), 2)
FROM validity_issues

UNION ALL

SELECT 
    'Revenue Issues' as issue_type,
    COUNT(CASE WHEN revenue_issue IS NOT NULL THEN 1 END),
    ROUND(COUNT(CASE WHEN revenue_issue IS NOT NULL THEN 1 END) * 100.0 / COUNT(*), 2)
FROM validity_issues

UNION ALL

SELECT 
    'Employee Issues' as issue_type,
    COUNT(CASE WHEN employee_issue IS NOT NULL THEN 1 END),
    ROUND(COUNT(CASE WHEN employee_issue IS NOT NULL THEN 1 END) * 100.0 / COUNT(*), 2)
FROM validity_issues

UNION ALL

SELECT 
    'Status Issues' as issue_type,
    COUNT(CASE WHEN status_issue IS NOT NULL THEN 1 END),
    ROUND(COUNT(CASE WHEN status_issue IS NOT NULL THEN 1 END) * 100.0 / COUNT(*), 2)
FROM validity_issues
```

### Duplicate Detection

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
email_duplicates AS (
    SELECT 
        email,
        COUNT(*) as duplicate_count,
        STRING_AGG(id, ', ') as duplicate_ids
    FROM latest_lead
    WHERE email IS NOT NULL 
      AND email != ''
    GROUP BY email
    HAVING COUNT(*) > 1
),
company_name_duplicates AS (
    SELECT 
        LOWER(TRIM(company)) as normalized_company,
        firstname,
        lastname,
        COUNT(*) as duplicate_count,
        STRING_AGG(id, ', ') as duplicate_ids
    FROM latest_lead
    WHERE company IS NOT NULL 
      AND firstname IS NOT NULL 
      AND lastname IS NOT NULL
    GROUP BY LOWER(TRIM(company)), firstname, lastname
    HAVING COUNT(*) > 1
)
-- Email-based duplicates (highest priority)
SELECT 
    'Email Duplicates' as duplicate_type,
    COUNT(*) as duplicate_groups,
    SUM(duplicate_count - 1) as excess_records
FROM email_duplicates

UNION ALL

-- Company + Name duplicates (medium priority)
SELECT 
    'Company+Name Duplicates' as duplicate_type,
    COUNT(*) as duplicate_groups,
    SUM(duplicate_count - 1) as excess_records
FROM company_name_duplicates
```

## Data Quality Monitoring Framework

### Daily Quality Scorecard

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
daily_metrics AS (
    SELECT 
        CURRENT_DATE as report_date,
        
        -- Lead Quality Metrics
        (SELECT COUNT(*) FROM latest_lead WHERE email IS NULL) as leads_missing_email,
        (SELECT COUNT(*) FROM latest_lead WHERE company IS NULL) as leads_missing_company,
        (SELECT COUNT(*) FROM latest_lead WHERE firstname IS NULL OR lastname IS NULL) as leads_missing_names,
        
        -- Account Segmentation Metrics  
        (SELECT COUNT(*) FROM latest_account 
         WHERE annualrevenue_f IS NULL AND numberofemployees_f IS NULL) as unsegmentable_accounts,
        
        -- Opportunity Pipeline Integrity
        (SELECT COUNT(*) FROM latest_opportunity 
         WHERE amount_f IS NULL AND stagename NOT IN ('Closed Lost')) as opps_missing_amount,
         
        -- Data Validity Issues
        (SELECT COUNT(*) FROM latest_lead 
         WHERE email IS NOT NULL AND email NOT LIKE '%@%.%') as invalid_email_formats
)
SELECT 
    report_date,
    leads_missing_email,
    leads_missing_company,
    leads_missing_names,
    unsegmentable_accounts,
    opps_missing_amount,
    invalid_email_formats,
    
    -- Calculate overall quality score (higher is better)
    CASE 
        WHEN (leads_missing_email + leads_missing_company + unsegmentable_accounts + invalid_email_formats) = 0 
        THEN 100
        ELSE GREATEST(0, 100 - (leads_missing_email + leads_missing_company + unsegmentable_accounts + invalid_email_formats))
    END as daily_quality_score
    
FROM daily_metrics
```

## Data Remediation Priorities

### High Impact Issues (Address First)

1. **Missing Email Addresses**: Prevents lead nurturing and outreach
2. **Invalid Email Formats**: Causes bounce rates and delivery issues
3. **Missing Company Names**: Blocks account matching and segmentation
4. **Incomplete Segmentation Data**: Impacts territory assignment and targeting
5. **Broken Conversion Relationships**: Corrupts funnel analysis

### Medium Impact Issues (Address Second)

1. **Phone Number Formatting**: Standardize for consistent communication
2. **Industry Classification**: Improve targeting and personalization
3. **Geographic Data**: Enable territory-based analysis
4. **Duplicate Records**: Clean up to prevent confusion

### Low Impact Issues (Address Last)

1. **Standardized Company Suffixes**: Aesthetic improvement
2. **Complete Address Information**: Nice-to-have for full profiles
3. **Additional Contact Fields**: Enhanced profiling

## Success Metrics

### Primary KPIs

- **Lead Qualification Accuracy**: % of MCL/MQL with complete segmentation data
- **Conversion Data Integrity**: % of conversions with valid relationships
- **Segmentation Accuracy**: % of accounts properly classified
- **Duplicate Reduction**: Month-over-month reduction in duplicate records

### Secondary KPIs

- **Field Completion Rates**: Trending completion percentages
- **Data Validation Compliance**: % of records passing validation rules
- **Email Deliverability**: Improvement in valid email percentages
- **Contact Reachability**: % of leads with valid phone/email combinations

This comprehensive data quality framework ensures reliable, complete, and actionable customer intelligence throughout the entire sales lifecycle while maintaining data integrity standards that support accurate business decision-making.
