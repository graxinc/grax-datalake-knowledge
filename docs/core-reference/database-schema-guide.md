# Database Schema Guide

Comprehensive reference for GRAX Data Lake database schema, field definitions, and data structures optimized for Amazon Athena queries.

## Database Environment

- **Engine**: Amazon Athena (Version 3)
- **Workgroup**: lakehouse
- **Catalog**: lakehouse
- **Database**: lakehouse
- **Data Format**: Parquet with Snappy compression
- **Partitioning**: By date for performance optimization

## Core Table Structure

### Primary Objects

| Object | Table Name | Primary Use | Record Volume |
|--------|------------|-------------|---------------|
| **Accounts** | `lakehouse.object_account` | Company and organization data | Medium |
| **Contacts** | `lakehouse.object_contact` | Individual contact records | High |
| **Leads** | `lakehouse.object_lead` | Lead management and qualification | High |
| **Opportunities** | `lakehouse.object_opportunity` | Sales pipeline and revenue data | Medium |
| **Cases** | `lakehouse.object_case` | Support and service tracking | High |
| **Activities** | `lakehouse.object_task`, `lakehouse.object_event` | Sales and marketing activities | Very High |

### Data Lake Metadata Fields

**Every table includes these GRAX-specific fields**:

| Field | Type | Purpose | Usage Pattern |
|-------|------|---------|---------------|
| `grax__idseq` | bigint | Version sequence number | Latest record identification |
| `grax__deleted` | boolean | Soft delete flag | Active record filtering |
| `grax__operation` | varchar | Change operation type | Data lineage tracking |
| `grax__datecreated` | timestamp | GRAX ingestion timestamp | Data freshness validation |

**Critical Pattern**: Always filter deleted records

```sql
-- MANDATORY: Filter deleted records
WHERE grax__deleted IS NULL
```

## Field Naming Conventions

### Type Suffixes

| Suffix | Data Type | Athena Type | Usage Example |
|--------|-----------|-------------|---------------|
| `_f` | Number (Float) | `double` | `amount_f`, `annualrevenue_f` |
| `_i` | Number (Integer) | `bigint` | `numberofemployees_i` |
| `_b` | Boolean | `boolean` | `isconverted_b`, `isclosed_b` |
| `_ts` | DateTime | `timestamp` | `createddate_ts`, `lastmodifieddate_ts` |
| `_d` | Date Only | `date` | `closedate_d`, `converteddate_d` |
| No Suffix | Text/String | `varchar` | `name`, `email`, `company` |

### Standard Field Patterns

**Timestamps**:

- `createddate_ts`: Record creation timestamp
- `lastmodifieddate_ts`: Last update timestamp
- `systemmodstamp_ts`: System modification timestamp

**Identifiers**:

- `id`: Salesforce record ID (18-character)
- `accountid`: Foreign key to Account
- `contactid`: Foreign key to Contact
- `ownerid`: User assignment

**Lookups and Relationships**:

- `parentid`: Hierarchical parent relationship
- `masterrecordid`: Merge relationship
- Custom relationship fields follow `relationshipname__c` pattern

## Account Object Schema

### Core Fields

| Field | Type | Description | Analysis Use |
|-------|------|-------------|-------------|
| `id` | varchar | Unique Salesforce Account ID | Primary key, relationships |
| `name` | varchar | Account name | Reporting, search |
| `type` | varchar | Account classification | Segmentation analysis |
| `industry` | varchar | Industry classification | Vertical analysis |
| `annualrevenue_f` | double | Annual revenue amount | Company sizing, segmentation |
| `numberofemployees_f` | double | Employee count | Company sizing |
| `billingcountry` | varchar | Primary country | Geographic analysis |
| `website` | varchar | Company website | Data enrichment |

### Hierarchy Fields

| Field | Type | Description | Corporate Analysis Use |
|-------|------|-------------|------------------------|
| `parentid` | varchar | Immediate parent account ID | Direct hierarchy |
| `ultimate_parent_account__c` | varchar | Top-level parent ID | Corporate family analysis |
| `hierarchy_depth__c` | double | Levels from ultimate parent | Org structure analysis |
| `is_ultimate_parent__c` | boolean | Top of hierarchy flag | Parent identification |

**Corporate Hierarchy Query Pattern**:

```sql
-- Find all subsidiaries of a corporate parent
WITH corporate_family AS (
    SELECT 
        id,
        name,
        ultimate_parent_account__c,
        hierarchy_depth__c
    FROM lakehouse.object_account
    WHERE grax__deleted IS NULL
        AND ultimate_parent_account__c IS NOT NULL
)
SELECT 
    parent.name as parent_company,
    child.name as subsidiary,
    child.hierarchy_depth__c as depth_level
FROM corporate_family child
JOIN lakehouse.object_account parent 
    ON child.ultimate_parent_account__c = parent.id
WHERE parent.grax__deleted IS NULL
ORDER BY parent.name, child.hierarchy_depth__c
```

## Lead Object Schema

### Core Fields

| Field | Type | Description | Qualification Use |
|-------|------|-------------|-------------------|
| `id` | varchar | Unique Salesforce Lead ID | Primary key |
| `email` | varchar | Contact email address | Deduplication, communication |
| `company` | varchar | Company name | Account matching |
| `status` | varchar | Lead qualification status | Funnel analysis |
| `leadsource` | varchar | Lead acquisition channel | Attribution analysis |
| `rating` | varchar | Lead quality rating | Prioritization |
| `isconverted_b` | boolean | Conversion flag | Conversion analysis |
| `converteddate_d` | date | Conversion date | Conversion timing |
| `convertedaccountid` | varchar | Account created from lead | Post-conversion tracking |
| `convertedcontactid` | varchar | Contact created from lead | Post-conversion tracking |
| `convertedopportunityid` | varchar | Opportunity created from lead | Revenue attribution |

### Segmentation Fields

| Field | Type | Description | Segmentation Use |
|-------|------|-------------|------------------|
| `numberofemployees_f` | double | Company size | Company segmentation |
| `annualrevenue_f` | double | Company revenue | Revenue segmentation |
| `industry` | varchar | Industry classification | Vertical analysis |
| `country` | varchar | Geographic location | Territory analysis |

**Lead Qualification Analysis Pattern**:

```sql
-- Lead funnel analysis with stage progression
WITH lead_funnel AS (
    SELECT 
        status,
        COUNT(*) as lead_count,
        COUNT(CASE WHEN isconverted_b = true THEN 1 END) as converted_count,
        ROUND(COUNT(CASE WHEN isconverted_b = true THEN 1 END) * 100.0 / COUNT(*), 2) as conversion_rate
    FROM lakehouse.object_lead
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    GROUP BY status
)
SELECT 
    status,
    lead_count,
    converted_count,
    conversion_rate || '%' as conversion_rate_pct
FROM lead_funnel
ORDER BY lead_count DESC
```

## Opportunity Object Schema

### Core Fields

| Field | Type | Description | Sales Analysis Use |
|-------|------|-------------|-------------------|
| `id` | varchar | Unique Salesforce Opportunity ID | Primary key |
| `name` | varchar | Opportunity name | Reporting |
| `accountid` | varchar | Associated account | Account relationship |
| `amount_f` | double | Opportunity value | Revenue analysis |
| `stagename` | varchar | Sales stage | Pipeline analysis |
| `probability_f` | double | Win probability (0-100) | Forecasting |
| `isclosed_b` | boolean | Closed status | Pipeline vs. closed analysis |
| `iswon_b` | boolean | Won status | Win/loss analysis |
| `closedate_d` | date | Expected/actual close date | Timing analysis |
| `leadsource` | varchar | Opportunity source | Attribution analysis |

### Timing Fields

| Field | Type | Description | Velocity Analysis Use |
|-------|------|-------------|----------------------|
| `createddate_ts` | timestamp | Opportunity creation | Age calculation |
| `lastmodifieddate_ts` | timestamp | Last update | Activity tracking |
| `laststagechangedate_ts` | timestamp | Stage change timing | Stage velocity |

**Sales Velocity Calculation Pattern**:

```sql
-- Calculate average days in each stage
WITH stage_durations AS (
    SELECT 
        stagename,
        AVG(DATE_DIFF('day', 
            createddate_ts, 
            COALESCE(closedate_d, CURRENT_DATE)
        )) as avg_days_in_stage,
        COUNT(*) as opportunity_count
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
    GROUP BY stagename
)
SELECT 
    stagename,
    ROUND(avg_days_in_stage, 1) as avg_days,
    opportunity_count
FROM stage_durations
ORDER BY avg_days DESC
```

## Latest Records Pattern

**Critical**: GRAX stores complete change history. Always use latest records for current state analysis.

### Latest Record Query Pattern

```sql
-- Get latest version of each record
WITH latest_records AS (
    SELECT 
        id,
        MAX(grax__idseq) as latest_seq
    FROM lakehouse.object_account
    WHERE grax__deleted IS NULL
    GROUP BY id
)
SELECT 
    a.id,
    a.name,
    a.type,
    a.industry
FROM lakehouse.object_account a
JOIN latest_records l ON a.id = l.id AND a.grax__idseq = l.latest_seq
WHERE a.grax__deleted IS NULL
```

### Window Function Alternative

```sql
-- Using ROW_NUMBER() window function
WITH current_accounts AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (
            PARTITION BY id 
            ORDER BY grax__idseq DESC
        ) as rn
    FROM lakehouse.object_account
    WHERE grax__deleted IS NULL
)
SELECT 
    id,
    name,
    type,
    industry
FROM current_accounts
WHERE rn = 1
```

## Performance Optimization

### Partition Strategy

**Date-based partitioning** for query performance:

```sql
-- Leverage partitioning with date filters
WHERE createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
    AND grax__deleted IS NULL
```

### Index-Friendly Queries

**Recommended filter order**:

1. Date range filters (leverages partitioning)
1. `grax__deleted IS NULL` (excludes soft-deleted records)
1. Status/category filters (reduces scan volume)
1. Latest record patterns (ensures current state)

### Common Performance Anti-Patterns

**Avoid**:

```sql
-- DON'T: Query without date bounds
SELECT * FROM lakehouse.object_opportunity

-- DON'T: Ignore deleted record filtering  
SELECT * FROM lakehouse.object_account WHERE name = 'ACME Corp'

-- DON'T: Forget latest record patterns
SELECT COUNT(*) FROM lakehouse.object_lead
```

**Do**:

```sql
-- DO: Include date bounds and proper filtering
WITH latest_opportunities AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    stagename,
    COUNT(*) as opportunity_count,
    SUM(amount_f) as total_pipeline
FROM latest_opportunities
WHERE rn = 1
GROUP BY stagename
ORDER BY total_pipeline DESC
```

## Data Quality Considerations

### Required Field Validation

```sql
-- Validate data completeness
SELECT 
    'Accounts' as object_type,
    COUNT(*) as total_records,
    COUNT(name) as records_with_name,
    COUNT(type) as records_with_type,
    COUNT(industry) as records_with_industry
FROM lakehouse.object_account
WHERE grax__deleted IS NULL
    AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
```

### Common Data Quality Issues

| Issue | Detection | Resolution |
|-------|-----------|------------|
| **Duplicate Records** | Multiple `grax__idseq` for same `id` | Use latest records pattern |
| **Missing Required Fields** | NULL values in critical fields | Apply validation filters |
| **Invalid Picklist Values** | Non-standard status/stage names | Reference [Configuration Reference](configuration-reference.md) |
| **Date Inconsistencies** | `closedate_d` before `createddate_ts` | Add date validation logic |

## Integration Patterns

### Cross-Object Relationships

```sql
-- Account to Opportunity relationship with latest records
WITH latest_accounts AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_account
    WHERE grax__deleted IS NULL
),
latest_opportunities AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_opportunity
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    a.name as account_name,
    a.industry,
    COUNT(o.id) as opportunity_count,
    SUM(o.amount_f) as total_pipeline
FROM latest_accounts a
LEFT JOIN latest_opportunities o ON a.id = o.accountid AND o.rn = 1
WHERE a.rn = 1
GROUP BY a.name, a.industry
HAVING COUNT(o.id) > 0
ORDER BY total_pipeline DESC
```

## Custom Fields

### Custom Field Patterns

**Salesforce custom fields** end with `__c` and follow the same type suffix patterns:

- `custom_field__c` (text)
- `custom_number__f` (numeric)
- `custom_date__ts` (datetime)
- `custom_checkbox__b` (boolean)

### Common Custom Fields

| Field Pattern | Purpose | Analysis Use |
|---------------|---------|-------------|
| `*_source__c` | Lead/opportunity attribution | Channel analysis |
| `*_score__f` | Scoring and ranking | Prioritization |
| `*_status__c` | Custom workflow stages | Process analysis |
| `ultimate_parent_account__c` | Corporate hierarchy | Family analysis |

See [Configuration Reference](configuration-reference.md) for organization-specific custom field values and business logic.
