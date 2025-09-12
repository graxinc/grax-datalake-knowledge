# Configuration Reference

**CRITICAL**: This document serves as the **single source of truth** for all business-specific configuration values used throughout the GRAX Data Lake Knowledge Base. All query templates, documentation examples, and analysis patterns reference values from this document rather than using hardcoded strings.

## Usage Pattern

When creating queries or documentation, reference values from this document using the format:

```markdown
<!-- Reference format -->
See [Configuration Reference: Lead Status Values](../core-reference/configuration-reference.md#lead-status-values) for current status definitions.

<!-- In SQL queries, use comments to reference -->
-- Lead qualification statuses from Configuration Reference
WHERE status IN ('MQL', 'SAL', 'SQL')
```

## Lead Management Configuration

### Lead Status Values

**Source**: Salesforce Lead.Status picklist values

**Standard GRAX Configuration**:

| Status Category | Status Values | Definition | Business Logic |
|----------------|---------------|------------|---------------|
| **Unqualified** | `'Open - Not Contacted'`, `'Working - Contacted'` | Initial lead states | Not yet qualified for sales |
| **Marketing Qualified** | `'MQL'` | Marketing qualified lead | Meets basic qualification criteria |
| **Sales Accepted** | `'SAL'` | Sales accepted lead | Accepted by sales team |
| **Sales Qualified** | `'SQL'` | Sales qualified lead | Qualified for opportunity creation |
| **Closed States** | `'Closed - Converted'`, `'Closed - Not Converted'` | Final disposition | End state resolution |

### Lead Source Classification

**Standard GRAX Configuration**:

| Source Category | Source Values | Channel Type |
|----------------|---------------|-------------|
| **Digital** | `'Web'`, `'Website'`, `'Online'` | Inbound digital |
| **Campaign** | `'Email Campaign'`, `'Webinar'`, `'Trade Show'` | Marketing campaigns |
| **Referral** | `'Partner Referral'`, `'Employee Referral'` | Relationship-based |
| **Direct** | `'Phone Inquiry'`, `'Sales Contact'` | Direct sales |

## Opportunity Management Configuration

### Opportunity Stage Values

**Source**: Salesforce Opportunity.StageName picklist values

**Standard GRAX Configuration**:

| Stage Category | Stage Values | Probability | Pipeline Position |
|---------------|--------------|-------------|------------------|
| **Early Stage** | `'Prospecting'`, `'Qualification'` | 10-25% | Top of funnel |
| **Development** | `'Needs Analysis'`, `'Value Proposition'` | 30-50% | Middle funnel |
| **Advanced** | `'Id. Decision Makers'`, `'Perception Analysis'` | 60-75% | Late funnel |
| **Proposal** | `'Proposal/Price Quote'`, `'Negotiation/Review'` | 80-90% | Near close |
| **Closed Won** | `'Closed Won'` | 100% | Revenue recognition |
| **Closed Lost** | `'Closed Lost'` | 0% | Pipeline exit |

### Win/Loss Categories

**For Analysis Segmentation**:

```sql
-- Win/Loss classification from Configuration Reference
CASE 
    WHEN stagename = 'Closed Won' THEN 'Won'
    WHEN stagename = 'Closed Lost' THEN 'Lost'
    WHEN stagename IN ('Proposal/Price Quote', 'Negotiation/Review') THEN 'Late Stage'
    WHEN stagename IN ('Prospecting', 'Qualification') THEN 'Early Stage'
    ELSE 'Development'
END as pipeline_category
```

## Account Segmentation Configuration

### Account Type Classification

**Standard GRAX Configuration**:

| Type Category | Type Values | Segment Definition |
|--------------|-------------|-------------------|
| **Customer** | `'Customer - Direct'`, `'Customer - Channel'` | Revenue-generating accounts |
| **Prospect** | `'Prospect'`, `'Target'` | Potential customers |
| **Partner** | `'Partner'`, `'Channel Partner'` | Strategic relationships |
| **Competitor** | `'Competitor'` | Market intelligence |

### Company Size Segmentation

**Employee Count Thresholds**:

| Segment | Employee Range | Threshold Logic |
|---------|---------------|----------------|
| **SMB** | 1-199 employees | `numberofemployees_f < 200` |
| **Mid-Market** | 200-999 employees | `numberofemployees_f BETWEEN 200 AND 999` |
| **Enterprise** | 1,000+ employees | `numberofemployees_f >= 1000` |

**Revenue Thresholds**:

| Segment | Revenue Range | Threshold Logic |
|---------|---------------|----------------|
| **Small** | Under $10M | `annualrevenue_f < 10000000` |
| **Medium** | $10M-$100M | `annualrevenue_f BETWEEN 10000000 AND 100000000` |
| **Large** | Over $100M | `annualrevenue_f > 100000000` |

## Industry Classification

### Standard Industry Values

**Source**: Salesforce Account.Industry picklist

**Common GRAX Classifications**:

```sql
-- Industry groupings from Configuration Reference
CASE 
    WHEN industry IN ('Technology', 'Software', 'Computer Software') THEN 'Technology'
    WHEN industry IN ('Banking', 'Financial Services', 'Insurance') THEN 'Financial Services'
    WHEN industry IN ('Healthcare', 'Biotechnology', 'Pharmaceuticals') THEN 'Healthcare & Life Sciences'
    WHEN industry IN ('Manufacturing', 'Automotive', 'Industrial') THEN 'Manufacturing'
    WHEN industry IN ('Retail', 'Consumer Goods', 'E-commerce') THEN 'Retail & Consumer'
    ELSE 'Other'
END as industry_group
```

## Temporal Configuration

### Fiscal Year Settings

**Standard GRAX Configuration**:

| Setting | Value | Usage |
|---------|-------|-------|
| **Fiscal Year Start** | January 1 | `EXTRACT(year FROM date_field)` |
| **Quarter Definition** | Calendar quarters | `DATE_TRUNC('quarter', date_field)` |
| **Analysis Window** | 24 months | `>= DATE_ADD('month', -24, CURRENT_DATE)` |

### Date Aggregation Patterns

| Aggregation | SQL Pattern | Use Case |
|-------------|-------------|----------|
| **Daily** | `DATE_TRUNC('day', date_field)` | Activity tracking |
| **Weekly** | `DATE_TRUNC('week', date_field)` | Trend analysis |
| **Monthly** | `DATE_TRUNC('month', date_field)` | Pipeline reporting |
| **Quarterly** | `DATE_TRUNC('quarter', date_field)` | Business reviews |

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
|--------|----------------|-------------------|---------------------|
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
|---------|-------|----------|
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
1. **Opportunity Stages**: Modify stage names to match organization's sales process
1. **Account Types**: Adjust customer classification values
1. **Segmentation Thresholds**: Change employee/revenue thresholds for company sizing
1. **Field Names**: Update field naming if custom fields are used

### Implementation Process

1. **Identify Differences**: Compare organization's values to defaults
1. **Update Configuration**: Modify values in this document
1. **Test Queries**: Validate all query templates work with new values
1. **Update References**: Ensure all documentation references remain accurate

### Validation Checklist

- [ ] All picklist values match Salesforce configuration
- [ ] Segmentation thresholds align with business definitions
- [ ] Date patterns match fiscal year settings
- [ ] Validation rules prevent data quality issues
- [ ] Performance settings balance accuracy and speed

## Integration with Other Documents

This configuration reference integrates with:

- **[Database Schema Guide](database-schema-guide.md)**: Field definitions and data types
- **[Query Templates](../query-guidance/query-templates.md)**: Reusable patterns using these values
- **[Query Best Practices](../query-guidance/query-best-practices.md)**: Performance optimization using these settings
- **[Business Intelligence Patterns](../analysis-patterns/business-intelligence-patterns.md)**: Analytics using these classifications
- **[Customer Fallback Instructions](../troubleshooting/customer-fallback-instructions.md)**: Adaptation strategies for different configurations
