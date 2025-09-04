# Database Schema Guide

## Overview

The data lake contains complete historical Salesforce data with comprehensive versioning and change tracking capabilities. This guide provides field-level documentation for all major business objects.

## Universal System Fields

Every table contains these mandatory system fields:

| Field Name      | Data Type    | Purpose                     | Usage                                                           |
|-----------------|--------------|-----------------------------|-----------------------------------------------------------------|
| `grax__idseq`   | varchar      | Version sequence number     | Use for chronological ordering and latest record identification |
| `grax__deleted` | timestamp(3) | Deletion marker             | **ALWAYS** filter `IS NULL` for active records                 |

## Field Naming Conventions

### Type Suffixes

| Suffix | Data Type   | Purpose                    | Examples                              |
|--------|-------------|----------------------------|---------------------------------------|
| `_f`   | double      | Numeric calculations       | `amount_f`, `probability_f`           |
| `_ts`  | timestamp   | Date/time operations       | `createddate_ts`, `lastmodifieddate_ts` |
| `_d`   | date        | Date-only fields           | `converteddate_d`, `closedate_d`      |
| `_b`   | boolean     | True/false values          | `isconverted_b`, `isclosed_b`         |
| `_i`   | bigint      | Integer values             | `casenumber_i`, `numberofemployees_i` |

### Standard Fields

| Field Pattern   | Purpose                    | Examples                        |
|-----------------|----------------------------|---------------------------------|
| `id`            | Primary key (all tables)   | Unique record identifier        |
| `*id`           | Foreign keys               | `ownerid`, `accountid`, `contactid` |
| `name`          | Primary name field         | Record display name             |
| `status`        | Status fields              | Current record status           |
| `created*`      | Creation timestamps        | `createddate_ts`, `createdbyid` |
| `lastmodified*` | Modification tracking      | `lastmodifieddate_ts`, `lastmodifiedbyid` |

## Core Business Objects

### object_lead

**Primary Purpose**: Lead qualification and conversion tracking

#### Key Identification Fields

- `id` (varchar) - Unique lead identifier
- `email` (varchar) - Lead email address
- `name` (varchar) - Lead full name

#### Qualification Fields

- `status` (varchar) - Lead status values:
  - `'Open'` - New leads (MCL criteria)
  - `'Working'` - Active leads (MQL criteria)
  - `'Converted'` - Successfully converted
  - `'Disqualified'` - Not viable leads

#### Date Fields

- `createddate_ts` (timestamp) - Lead creation date - **USE FOR MCL DATE**
- `converteddate_d` (date) - Conversion date
- `lastmodifieddate_ts` (timestamp) - Last modification date

#### Conversion Fields

- `isconverted_b` (boolean) - Conversion status
- `convertedopportunityid` (varchar) - Related opportunity after conversion
- `convertedaccountid` (varchar) - Related account after conversion
- `convertedcontactid` (varchar) - Related contact after conversion

#### Company Information

- `company` (varchar) - Company name
- `industry` (varchar) - Industry classification
- `numberofemployees_f` (double) - Company size
- `annualrevenue_f` (double) - Company revenue

### object_opportunity

**Primary Purpose**: Sales pipeline and opportunity management

#### Key Identification Fields

- `id` (varchar) - Unique opportunity identifier
- `name` (varchar) - Opportunity name
- `accountid` (varchar) - Related account ID

#### Stage Management

- `stagename` (varchar) - Current stage values:
  - `'SQL'` - Sales Qualified Lead
  - `'Proof of Value (SQO)'` - Sales Qualified Opportunity
  - `'Proposal'` - Formal proposal delivered
  - `'Contracts'` - Contract negotiation
  - `'Closed Won'` - Successfully closed
  - `'Closed Lost'` - Lost opportunity
- `laststagechangedate_ts` (timestamp) - Most recent stage change date

#### Financial Fields

- `amount_f` (double) - Opportunity value
- `probability_f` (double) - Win probability percentage
- `expectedrevenue_f` (double) - Weighted pipeline value

#### Date Management

- `createddate_ts` (timestamp) - Opportunity creation date
- `closedate_d` (date) - Expected/actual close date
- `lastactivitydate_ts` (timestamp) - Last activity date

#### Ownership

- `ownerid` (varchar) - Opportunity owner (links to object_user)

### object_account

**Primary Purpose**: Customer account management and segmentation

#### Key Identification Fields

- `id` (varchar) - Unique account identifier
- `name` (varchar) - Account name

#### Segmentation Fields

- `type` (varchar) - Account type
- `industry` (varchar) - Industry classification
- `numberofemployees_f` (double) - Company size
- `annualrevenue_f` (double) - Company revenue

#### Contact Information

- `phone` (varchar) - Primary phone number
- `website` (varchar) - Company website

#### Address Fields

- `billingcountry` (varchar) - Billing country
- `billingstate` (varchar) - Billing state/province
- `billingcity` (varchar) - Billing city

#### Date Fields

- `createddate_ts` (timestamp) - Account creation date
- `lastmodifieddate_ts` (timestamp) - Last modification date

### object_contact

**Primary Purpose**: Contact relationship management

#### Key Identification Fields

- `id` (varchar) - Unique contact identifier
- `accountid` (varchar) - Related account ID

#### Personal Information

- `firstname` (varchar) - First name
- `lastname` (varchar) - Last name
- `email` (varchar) - Email address
- `phone` (varchar) - Phone number
- `title` (varchar) - Job title

#### Address Fields

- `mailingcountry` (varchar) - Mailing country
- `mailingstate` (varchar) - Mailing state/province
- `mailingcity` (varchar) - Mailing city

#### Date Fields

- `createddate_ts` (timestamp) - Contact creation date
- `lastmodifieddate_ts` (timestamp) - Last modification date

### object_user

**Primary Purpose**: User management and activity tracking

#### Key Fields

- `id` (varchar) - User ID
- `name` (varchar) - User full name
- `firstname` (varchar) - First name
- `lastname` (varchar) - Last name
- `email` (varchar) - User email
- `username` (varchar) - Username
- `isactive_b` (boolean) - Active status
- `profileid` (varchar) - User profile
- `userroleid` (varchar) - User role
- `managerid` (varchar) - Manager user
- `createddate_ts` (timestamp) - User creation date

### object_case

**Primary Purpose**: Customer support and service tracking

#### Key Fields

- `id` (varchar) - Unique case identifier
- `subject` (varchar) - Case subject
- `status` (varchar) - Case status
- `priority` (varchar) - Case priority
- `type` (varchar) - Case type
- `accountid` (varchar) - Related account
- `contactid` (varchar) - Related contact
- `ownerid` (varchar) - Case owner
- `createddate_ts` (timestamp) - Case creation date
- `isclosed_b` (boolean) - Closed flag

## Latest Records Pattern

**CRITICAL**: Always use this pattern for current state analysis:

```sql
WITH latest_[objectname] AS (
    SELECT A.* 
    FROM lakehouse.object_[objectname] A 
    INNER JOIN (
        SELECT B.Id, MAX(B.grax__idseq) AS Latest 
        FROM lakehouse.object_[objectname] B 
        GROUP BY B.Id
    ) B ON A.Id = B.Id AND A.grax__idseq = B.Latest
    WHERE A.grax__deleted IS NULL
)
```

## Common Join Patterns

### Lead to Opportunity (Conversion Tracking)

```sql
FROM latest_leads l
LEFT JOIN latest_opportunities o ON l.convertedopportunityid = o.id
```

### Opportunity to Account (Account Details)

```sql
FROM latest_opportunities opp
LEFT JOIN latest_accounts acc ON opp.accountid = acc.id
```

### User Ownership (Owner Details)

```sql
FROM latest_opportunities opp  
LEFT JOIN latest_users u ON opp.ownerid = u.id
```

## Segmentation Criteria

### Customer Segmentation

```sql
CASE 
    WHEN numberofemployees_f > 250 OR annualrevenue_f > 100000000 THEN 'Enterprise'
    WHEN numberofemployees_f > 50 OR annualrevenue_f > 10000000 THEN 'SMB'  
    ELSE 'Self Service'
END AS segment
```

### Company Size Categories

```sql
CASE 
    WHEN numberofemployees_f IS NULL THEN 'Unknown'
    WHEN numberofemployees_f <= 50 THEN 'Small (1-50)'
    WHEN numberofemployees_f <= 250 THEN 'Medium (51-250)'
    ELSE 'Large (250+)'
END AS company_size_category
```

## Data Type Handling

### Date Conversions

```sql
-- Convert date to timestamp for calculations
CAST(converteddate_d AS timestamp) AS conversion_timestamp

-- Extract date parts  
DATE_TRUNC('month', createddate_ts) AS creation_month

-- Date arithmetic
DATE_ADD('month', -6, CURRENT_DATE) AS six_months_ago
```

### Null Handling

```sql
-- Safe aggregations
COALESCE(amount_f, 0) AS amount_safe

-- Safe string operations  
COALESCE(industry, 'Unknown') AS industry_category
```

## Performance Guidelines

### Optimization Tips

1. **Filter early**: Apply WHERE conditions in CTEs

1. **Use EXISTS**: More efficient than IN for subqueries

1. **Limit date ranges**: Use specific date filters to reduce scan scope

1. **Prefer typed fields**: Use `_f` and `_ts` fields for calculations

1. **Test with LIMIT**: Always test large queries with row limits first

### Common Anti-Patterns

**Avoid These Patterns:**

```sql
-- Missing grax__deleted filter
SELECT * FROM lakehouse.object_lead  

-- Using wrong field types
WHERE amount = 1000  -- Should use amount_f

-- String comparison on dates
WHERE createddate = '2024-01-01'  -- Should use createddate_ts
```

**Use These Patterns:**

```sql
-- Always filter deleted records
SELECT * FROM lakehouse.object_lead WHERE grax__deleted IS NULL

-- Use proper field types
WHERE amount_f = 1000

-- Use timestamp fields for dates
WHERE createddate_ts >= DATE('2024-01-01')
```
