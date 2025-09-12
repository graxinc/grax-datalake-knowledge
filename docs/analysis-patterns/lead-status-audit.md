# Lead Status Audit

## Overview

This guide provides a framework for identifying leads that should be classified as "Existing Customer" but are currently marked as active prospects. This analysis prevents unnecessary sales outreach to existing customers and ensures accurate lead qualification metrics.

**Configuration Dependencies**: All lead status values, account types, and classification logic in this document use standardized values from [Configuration Reference](../core-reference/configuration-reference.md). Organizations with different Salesforce configurations should update that document to match their specific values.

## Business Problem

**Issue**: Active leads may actually be from existing customer organizations, leading to:

- Wasted sales resources on existing customers
- Potential customer confusion from duplicate outreach  
- Inaccurate funnel metrics and conversion rates
- Missed opportunities for account expansion vs new acquisition

**Goal**: Identify leads where the email domain matches existing customer accounts to enable proper lead status correction.

## Customer Account Definition

An account is considered an "Existing Customer" based on **Account Type** field values defined in [Configuration Reference](../core-reference/configuration-reference.md):

**Customer Types** (from Account Classification Configuration):

- `Type = 'Customer - Direct'` - Primary customer accounts
- `Type = 'Customer - Channel'` - Customer subsidiary accounts

**Exclusions**:

- Accounts with `Type = 'Prospect'` are **NEVER** considered existing customers
- Annual revenue is **NOT** used for customer identification

## Lead Classification Logic

A lead should be classified as "Existing Customer" if:

1. Lead Status ≠ `'Closed - Converted'` (current misclassification)

1. Lead is currently active (MCL or MQL status using values from [Configuration Reference](../core-reference/configuration-reference.md))

1. Lead email domain matches an existing customer account domain

1. Matching account has `Type = 'Customer - Direct'` OR `Type = 'Customer - Channel'`

1. Matching account does NOT have `Type = 'Prospect'`

## Core Analysis Query

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
latest_contact AS (
    SELECT A.* 
    FROM lakehouse.object_contact A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_contact B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
-- Extract email domains from leads
lead_domains AS (
    SELECT 
        l.*,
        LOWER(TRIM(SUBSTRING(l.email, POSITION('@' IN l.email) + 1))) as email_domain
    FROM latest_lead l
    WHERE l.email IS NOT NULL 
      AND l.email LIKE '%@%'
      -- Using status exclusion logic from Configuration Reference
      AND l.status NOT IN ('Closed - Converted', 'Closed - Not Converted')
      -- Using active lead criteria from Configuration Reference
      AND (l.status IN ('Open - Not Contacted', 'Working - Contacted', 'MQL', 'SAL', 'SQL'))
),
-- Extract domains from customer accounts using account types from Configuration Reference
account_domains AS (
    SELECT DISTINCT
        a.id as account_id,
        a.name as account_name,
        a.type as account_type,
        -- Extract domain from website field
        LOWER(TRIM(REPLACE(REPLACE(REPLACE(a.website, 'http://', ''), 'https://', ''), 'www.', ''))) as account_domain
    FROM latest_account a
    -- Using customer types from Configuration Reference
    WHERE a.type IN ('Customer - Direct', 'Customer - Channel')  -- Only customer types
      AND a.type != 'Prospect'                                   -- Explicitly exclude prospects
      AND a.website IS NOT NULL
      AND LENGTH(TRIM(a.website)) > 0
),
-- Alternative domain extraction from customer account contacts
contact_domains AS (
    SELECT DISTINCT
        c.accountid,
        LOWER(TRIM(SUBSTRING(c.email, POSITION('@' IN c.email) + 1))) as contact_domain
    FROM latest_contact c
    WHERE c.email IS NOT NULL 
      AND c.email LIKE '%@%'
      AND c.accountid IN (
          SELECT a.id 
          FROM latest_account a 
          -- Using customer types from Configuration Reference
          WHERE a.type IN ('Customer - Direct', 'Customer - Channel')  -- Only customer types
            AND a.type != 'Prospect'                                   -- Explicitly exclude prospects
      )
),
-- Combine customer domains from multiple sources
all_customer_domains AS (
    SELECT account_id, account_name, account_type, account_domain as domain
    FROM account_domains
    WHERE account_domain IS NOT NULL AND LENGTH(account_domain) > 0
    
    UNION
    
    SELECT 
        a.id as account_id, 
        a.name as account_name, 
        a.type as account_type,
        cd.contact_domain as domain
    FROM latest_account a
    INNER JOIN contact_domains cd ON a.id = cd.accountid
    -- Using customer types from Configuration Reference
    WHERE a.type IN ('Customer - Direct', 'Customer - Channel')  -- Only customer types
      AND a.type != 'Prospect'                                   -- Explicitly exclude prospects
)
-- Final results: Leads that should be marked as Existing Customer
SELECT 
    ld.id as lead_id,
    ld.name as lead_name,
    ld.email as lead_email,
    ld.company as lead_company,
    ld.status as current_lead_status,
    ld.email_domain,
    ld.createddate_ts as lead_created_date,
    
    -- Matching customer account info
    acd.account_id,
    acd.account_name,
    acd.account_type,
    acd.domain as matching_domain,
    
    -- Validation using account types from Configuration Reference
    CASE 
        WHEN acd.account_type NOT IN ('Customer - Direct', 'Customer - Channel') THEN 'ERROR: Non-customer matched'
        WHEN acd.account_type = 'Prospect' THEN 'ERROR: Prospect matched (should be impossible)'
        ELSE 'Valid Customer Match'
    END as match_validation,
    
    -- Customer classification using types from Configuration Reference
    CASE 
        WHEN acd.account_type = 'Customer - Direct' THEN 'Direct Customer Account'
        WHEN acd.account_type = 'Customer - Channel' THEN 'Channel Customer Account'
        ELSE 'Other Customer Type'
    END as customer_classification,
    
    -- Age analysis
    DATE_DIFF('day', ld.createddate_ts, CURRENT_TIMESTAMP) as days_since_lead_created
    
FROM lead_domains ld
INNER JOIN all_customer_domains acd ON ld.email_domain = acd.domain
-- Final safety check using account types from Configuration Reference
WHERE acd.account_type IN ('Customer - Direct', 'Customer - Channel')
  AND acd.account_type != 'Prospect'
ORDER BY 
    acd.account_type, 
    ld.createddate_ts DESC
```

## Validation Queries

### Account Type Distribution Check

```sql
-- Verify account type distribution for domain matching candidates
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
latest_contact AS (
    SELECT A.* 
    FROM lakehouse.object_contact A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_contact B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
),
all_accounts_with_domains AS (
    SELECT 
        type as account_type,
        COUNT(*) as account_count,
        COUNT(CASE WHEN website IS NOT NULL AND LENGTH(TRIM(website)) > 0 THEN 1 END) as with_website,
        COUNT(CASE 
            WHEN id IN (
                SELECT DISTINCT accountid 
                FROM latest_contact 
                WHERE email IS NOT NULL
            ) THEN 1 
        END) as with_contacts
    FROM latest_account
    GROUP BY type
)
SELECT 
    account_type,
    account_count,
    with_website,
    with_contacts,
    -- Flag which types are included in customer matching using configuration from Configuration Reference
    CASE 
        WHEN account_type IN ('Customer - Direct', 'Customer - Channel') THEN 'INCLUDED in customer matching'
        WHEN account_type = 'Prospect' THEN 'EXCLUDED from customer matching'
        ELSE 'OTHER - Review inclusion criteria'
    END as matching_inclusion
FROM all_accounts_with_domains
ORDER BY account_count DESC
```

## Summary Analysis

### Misclassified Leads by Customer Type

```sql
WITH misclassified_leads AS (
    -- Core analysis results (use complete query above)
    SELECT 'placeholder' as customer_classification -- Replace with actual query
)
SELECT 
    customer_classification,
    COUNT(*) as lead_count,
    -- Using lead status values from Configuration Reference
    COUNT(CASE WHEN current_lead_status = 'Open - Not Contacted' THEN 1 END) as mcl_leads,
    COUNT(CASE WHEN current_lead_status = 'MQL' THEN 1 END) as mql_leads,
    AVG(days_since_lead_created) as avg_days_old,
    -- Validation check
    COUNT(CASE WHEN match_validation LIKE '%ERROR%' THEN 1 END) as validation_errors
FROM misclassified_leads
GROUP BY customer_classification
ORDER BY lead_count DESC
```

## Business Impact Analysis

### Revenue Protection

- Prevents sales outreach conflicts with existing customer success teams
- Ensures prospect accounts remain in active sales pipeline
- Eliminates false positive customer classifications
- Reduces risk of customer confusion from properly classified customers

### Process Efficiency

- Eliminates wasted sales development resources on actual existing customers
- Preserves sales opportunities with ALL prospect accounts
- Improves lead qualification accuracy using Account Type governance from [Configuration Reference](../core-reference/configuration-reference.md)
- Enables better territory and quota planning

### Data Governance

- Establishes Account Type as single source of truth for customer classification (per [Configuration Reference](../core-reference/configuration-reference.md))
- Simplifies customer identification process
- Removes dependency on financial metrics for customer status
- Creates clear, auditable classification rules

## Implementation Recommendations

### Immediate Actions

1. **Execute Core Analysis**: Run the main query to identify all misclassified leads

1. **Validate Results**: Use validation queries to confirm logic correctness

1. **Update Lead Status**: Change identified leads from active status to `'Closed - Converted'` (using status value from [Configuration Reference](../core-reference/configuration-reference.md))

1. **Document Process**: Record classification rules and validation steps

### Ongoing Monitoring

1. **Weekly Audits**: Run analysis weekly to catch new misclassifications

1. **Process Integration**: Build checks into lead import and qualification workflows

1. **Training Updates**: Educate sales teams on proper customer identification using values from [Configuration Reference](../core-reference/configuration-reference.md)

1. **Metric Adjustment**: Update funnel reports to reflect corrected classifications

## Configuration Adaptation

For organizations with different Salesforce implementations, update the [Configuration Reference](../core-reference/configuration-reference.md) document with your specific values:

### Lead Status Audit Configuration Updates

1. **Lead Status Values**: Update `'Open - Not Contacted'`, `'MQL'`, `'Closed - Converted'` to match your lead qualification stages
1. **Account Types**: Modify `'Customer - Direct'`, `'Customer - Channel'`, `'Prospect'` with your account classification values
1. **Active Lead Criteria**: Adjust which status values indicate active leads requiring audit
1. **Exclusion Logic**: Update which account types should be excluded from customer matching

### Validation Process

1. **Update Configuration**: Modify values in [Configuration Reference](../core-reference/configuration-reference.md)
1. **Test Analysis**: Execute the core analysis query with your configuration
1. **Validate Logic**: Ensure classification results make sense for your business context
1. **Document Changes**: Record customizations for audit trail purposes

This comprehensive audit framework ensures accurate lead classification that respects the distinction between confirmed customers and active prospects while maintaining data integrity and supporting effective sales operations across different Salesforce implementations.
