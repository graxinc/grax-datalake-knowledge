# Configuration Reference

## Overview

This document centralizes all business-specific configuration values used throughout the GRAX Data Lake Knowledge Base. These values are based on GRAX's Salesforce implementation but can be adapted for other organizations by modifying this single reference document.

## Environment Configuration

### Database Environment

| Setting | Value | Description |
|---------|--------|-------------|
| **Engine** | Amazon Athena | Query engine for data lake analysis |
| **Workgroup** | `lakehouse` | Athena workgroup name |
| **Catalog** | `lakehouse` | Data catalog identifier |
| **Database** | `lakehouse` | Target database for all queries |

### Table Naming Convention

| Object Type | Table Name Pattern | Example |
|-------------|-------------------|---------|
| **Lead** | `lakehouse.object_lead` | Lead records and qualification |
| **Opportunity** | `lakehouse.object_opportunity` | Sales opportunities and pipeline |
| **Account** | `lakehouse.object_account` | Customer and prospect accounts |
| **Contact** | `lakehouse.object_contact` | Individual contact records |
| **User** | `lakehouse.object_user` | Salesforce user accounts |
| **Case** | `lakehouse.object_case` | Customer support cases |

## Lead Status Configuration

### Lead Qualification Stages

| Stage | Status Value | Business Definition | Date Assignment |
|-------|-------------|-------------------|-----------------|
| **MCL** | `'Open'` | Marketing Contacted Lead | `createddate_ts` |
| **MQL** | `'Working'` | Marketing Qualified Lead | `createddate_ts` |
| **Converted** | `'Converted'` | Successfully converted leads | `converteddate_d` |
| **Disqualified** | `'Disqualified'` | Not viable prospects | N/A |
| **Existing Customer** | `'Existing Customer'` | From customer organizations | N/A |

### Lead Status Query Patterns

```sql
-- MCL Qualification
status = 'Open'

-- MQL Qualification
status = 'Working' OR (isconverted_b = true AND status != 'Working')

-- Active Lead Filter
status IN ('Open', 'Working')

-- Converted Lead Filter
isconverted_b = true AND converteddate_d IS NOT NULL
```

## Opportunity Stage Configuration

### Sales Pipeline Stages

| Stage | Stage Name | Business Definition | Progressive Count |
|-------|-----------|-------------------|------------------|
| **SQL** | Any except `'Closed Lost'` | Sales Qualified Lead | All non-lost opportunities |
| **SQO** | `'Proof of Value (SQO)'` | Sales Qualified Opportunity | SQO stage or beyond |
| **Proposal** | `'Proposal'` | Formal proposal delivered | Proposal stage or beyond |
| **Contracts** | `'Contracts'` | Contract negotiation | Contracts stage or beyond |
| **Closed Won** | `'Closed Won'` | Successfully closed | Won opportunities only |
| **Closed Lost** | `'Closed Lost'` | Failed progression | Excluded from progression counts |

### Stage Progression Logic

```sql
-- SQL: All opportunities except Closed Lost
stagename != 'Closed Lost'

-- SQO: Reached SQO or beyond (excluding Closed Lost)
stagename IN ('Proof of Value (SQO)', 'Proposal', 'Contracts', 'Closed Won')

-- Proposal: Reached Proposal or beyond
stagename IN ('Proposal', 'Contracts', 'Closed Won')

-- Contracts: Reached Contracts or beyond
stagename IN ('Contracts', 'Closed Won')

-- Closed Won: Successfully completed
stagename = 'Closed Won'
```

### Stage Date Assignment

| Stage | Date Field | Logic |
|-------|-----------|-------|
| **SQL** | `createddate_ts` | Always use opportunity creation |
| **SQO** | `laststagechangedate_ts` | When stage = SQO or beyond |
| **Proposal** | `laststagechangedate_ts` | When stage = Proposal or beyond |
| **Closed Won/Lost** | `closedate_d` | Actual close date |

## Account Classification Configuration

### Customer Types

| Type | Configuration Value | Business Definition | Customer Status |
|------|-------------------|-------------------|-----------------|
| **Customer** | `'Customer'` | Primary customer accounts | Existing Customer |
| **Customer Subsidiary** | `'Customer - Subsidiary'` | Customer subsidiary accounts | Existing Customer |
| **Prospect** | `'Prospect'` | Potential customer accounts | Not Existing Customer |

### Customer Identification Logic

```sql
-- Existing Customer Accounts
type IN ('Customer', 'Customer - Subsidiary') AND type != 'Prospect'

-- Prospect Accounts
type = 'Prospect'

-- Customer Exclusion Filter
type != 'Prospect'
```

## Segmentation Configuration

### Company Size Segmentation

| Segment | Employee Threshold | Revenue Threshold | Logic |
|---------|-------------------|------------------|-------|
| **Enterprise** | `> 250` | `> $100M` | Either threshold |
| **SMB** | `> 50` | `> $10M` | Either threshold |
| **Self Service** | `<= 50` | `<= $10M` | Below both thresholds |

### Segmentation Query Pattern

```sql
CASE 
    WHEN numberofemployees_f > 250 OR annualrevenue_f > 100000000 THEN 'Enterprise'
    WHEN numberofemployees_f > 50 OR annualrevenue_f > 10000000 THEN 'SMB'
    ELSE 'Self Service'
END as customer_segment
```

### Company Size Categories

```sql
CASE 
    WHEN numberofemployees_f IS NULL THEN 'Unknown'
    WHEN numberofemployees_f <= 50 THEN 'Small (1-50)'
    WHEN numberofemployees_f <= 250 THEN 'Medium (51-250)'
    ELSE 'Large (250+)'
END as company_size_category
```

## Field Naming Standards

### Data Type Suffixes

| Suffix | Data Type | Purpose | Examples |
|--------|-----------|---------|----------|
| **_f** | `double` | Numeric calculations | `amount_f`, `probability_f`, `annualrevenue_f` |
| **_ts** | `timestamp` | Date/time operations | `createddate_ts`, `lastmodifieddate_ts` |
| **_d** | `date` | Date-only fields | `converteddate_d`, `closedate_d` |
| **_b** | `boolean` | True/false values | `isconverted_b`, `isclosed_b` |
| **_i** | `bigint` | Integer values | `casenumber_i`, `numberofemployees_i` |

### Universal System Fields

| Field Name | Data Type | Purpose | Usage |
|------------|-----------|---------|-------|
| **grax__idseq** | `varchar` | Version sequence number | Chronological ordering |
| **grax__deleted** | `timestamp(3)` | Deletion marker | `IS NULL` for active records |
| **id** | `varchar` | Primary key | Unique record identifier |

## Time Analysis Configuration

### Standard Time Periods

| Period | Pattern | Usage |
|--------|---------|-------|
| **Daily** | `DATE_TRUNC('day', date_field)` | Daily trend analysis |
| **Weekly** | `DATE_TRUNC('week', date_field)` | Weekly reporting |
| **Monthly** | `DATE_TRUNC('month', date_field)` | Monthly funnel analysis |
| **Quarterly** | `DATE_TRUNC('quarter', date_field)` | Quarterly business reviews |

### Date Range Filters

```sql
-- Last 12 months
date_field >= DATE_ADD('month', -12, CURRENT_DATE)

-- Current quarter
DATE_TRUNC('quarter', date_field) = DATE_TRUNC('quarter', CURRENT_DATE)

-- Year to date
EXTRACT(year FROM date_field) = EXTRACT(year FROM CURRENT_DATE)
```

## Data Quality Standards

### Required Fields for Analysis

| Object | Critical Fields | Segmentation Fields | Qualification Fields |
|--------|----------------|-------------------|-------------------|
| **Lead** | `email`, `company`, `status` | `numberofemployees_f`, `annualrevenue_f` | `createddate_ts`, `converteddate_d` |
| **Opportunity** | `amount_f`, `stagename`, `accountid` | Via account relationship | `createddate_ts`, `closedate_d` |
| **Account** | `name`, `type` | `numberofemployees_f`, `annualrevenue_f` | `createddate_ts` |

### Validation Patterns

```sql
-- Email format validation
email LIKE '%@%.%' AND email NOT LIKE '%@example.%' AND email NOT LIKE '%@test.%'

-- Revenue validation
annualrevenue_f > 0 AND annualrevenue_f <= 1000000000000

-- Employee count validation
numberofemployees_f > 0 AND numberofemployees_f <= 10000000
```

## Performance Optimization Settings

### Query Optimization Defaults

| Setting | Value | Purpose |
|---------|-------|---------|
| **Date Filter Range** | 24 months | Balance performance and completeness |
| **Latest Records** | Mandatory pattern | Ensure current state analysis |
| **Early Filtering** | Apply in CTEs | Reduce data scan volume |
| **Limit Testing** | 10-100 rows | Validate before full execution |

### Index-Friendly Patterns

```sql
-- Date range filtering
WHERE createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)

-- Status filtering
WHERE stagename IN ('Closed Won', 'Closed Lost')

-- Type-based filtering
WHERE grax__deleted IS NULL
```

## Customization Guidelines

### For Different Organizations

1. **Lead Status Values**: Update lead qualification stage values in this document
2. **Opportunity Stages**: Modify stage names to match organization's sales process
3. **Account Types**: Adjust customer classification values
4. **Segmentation Thresholds**: Change employee/revenue thresholds for company sizing
5. **Field Names**: Update field naming if custom fields are used

### Implementation Process

1. **Identify Differences**: Compare organization's values to defaults
2. **Update Configuration**: Modify values in this document
3. **Test Queries**: Validate all query templates work with new values
4. **Document Changes**: Record customizations for future reference

### Validation Queries

```sql
-- Discover actual lead status values
SELECT DISTINCT status, COUNT(*) 
FROM lakehouse.object_lead 
WHERE grax__deleted IS NULL 
GROUP BY status ORDER BY COUNT(*) DESC;

-- Discover opportunity stage names
SELECT DISTINCT stagename, COUNT(*) 
FROM lakehouse.object_opportunity 
WHERE grax__deleted IS NULL 
GROUP BY stagename ORDER BY COUNT(*) DESC;

-- Discover account types
SELECT DISTINCT type, COUNT(*) 
FROM lakehouse.object_account 
WHERE grax__deleted IS NULL 
GROUP BY type ORDER BY COUNT(*) DESC;
```

This configuration reference serves as the single source of truth for all business-specific values used throughout the knowledge base, making it easy to adapt the documentation for different Salesforce implementations.
