# Multi-Customer Fallback Instructions

## Overview

This knowledge base uses centralized configuration values defined in the [Configuration Reference](../configuration-reference.md). When used by customers with different Salesforce implementations, queries may return zero results due to customer-specific field values, stage names, and data structures that differ from the default configuration.

## Universal Fallback Strategy

### Step 1: Initial Query Attempt

Execute the query using values from the [Configuration Reference](../configuration-reference.md).

### Step 2: Zero Results Detection

If query returns zero results, immediately implement fallback discovery:

```sql
-- Discover actual values in customer's data
SELECT DISTINCT [fieldname], COUNT(*) as count
FROM lakehouse.object_[objectname]
WHERE grax__deleted IS NULL
GROUP BY [fieldname]
ORDER BY count DESC
LIMIT 20
```

### Step 3: Adaptive Query Reconstruction

Rebuild query using discovered values and inform customer of the adaptation.

## Customer-Specific Adaptations Required

### Lead Status Values

**Default Configuration** (from [Configuration Reference](../configuration-reference.md)):

- `'Open'` (MCL)
- `'Working'` (MQL)
- `'Converted'`
- `'Disqualified'`
- `'Existing Customer'`

**Fallback Discovery**:

```sql
-- If lead status query returns zero results, discover actual values
SELECT DISTINCT status, COUNT(*) as lead_count
FROM lakehouse.object_lead
WHERE grax__deleted IS NULL
GROUP BY status
ORDER BY lead_count DESC
```

**Adaptive Response Pattern**:

"I notice this query returned zero results with the standard lead status values from our configuration. Let me check what status values your organization uses..."

### Opportunity Stage Names

**Default Configuration** (from [Configuration Reference](../configuration-reference.md)):

- `'SQL'`
- `'Proof of Value (SQO)'`
- `'Proposal'`
- `'Contracts'`
- `'Closed Won'`
- `'Closed Lost'`

**Fallback Discovery**:

```sql
-- Discover customer's stage names
SELECT DISTINCT stagename, COUNT(*) as opp_count
FROM lakehouse.object_opportunity
WHERE grax__deleted IS NULL
GROUP BY stagename
ORDER BY opp_count DESC
```

### Account Type Classifications

**Default Configuration** (from [Configuration Reference](../configuration-reference.md)):

- `'Customer'`
- `'Customer - Subsidiary'`
- `'Prospect'`

**Fallback Discovery**:

```sql
-- Discover account types
SELECT DISTINCT type, COUNT(*) as account_count
FROM lakehouse.object_account
WHERE grax__deleted IS NULL
  AND type IS NOT NULL
GROUP BY type
ORDER BY account_count DESC
```

### Industry Classifications

**Fallback for Industry Analysis**:

```sql
-- When specific industries return zero results
SELECT DISTINCT industry, COUNT(*) as account_count
FROM lakehouse.object_account
WHERE grax__deleted IS NULL
  AND industry IS NOT NULL
GROUP BY industry
ORDER BY account_count DESC
LIMIT 15
```

## Dynamic Field Discovery Pattern

### Universal Field Type Discovery

```sql
-- Discover what numeric fields are available for segmentation
SELECT 
    COUNT(CASE WHEN annualrevenue_f IS NOT NULL THEN 1 END) as has_revenue,
    COUNT(CASE WHEN numberofemployees_f IS NOT NULL THEN 1 END) as has_employees,
    COUNT(CASE WHEN annualrevenue_f IS NOT NULL OR numberofemployees_f IS NOT NULL THEN 1 END) as segmentable
FROM lakehouse.object_account
WHERE grax__deleted IS NULL
```

## Fallback Communication Patterns

### When Zero Results Detected

**Template Response**:

```text
I notice this analysis returned zero results using our standard configuration values. This suggests your Salesforce instance uses different value configurations than our defaults.

Let me discover your organization's specific values and rebuild the analysis...

[Execute discovery queries]

Based on your data, I see you use these values:
- [List discovered values]

Let me rebuild the analysis using your organization's specific configuration...

[Execute adapted query with customer's values]
```

### When Field Names Differ

**Template Response**:

```text
It appears some expected fields may not be present or may have different names in your Salesforce configuration. 

Let me check your available fields and adapt the analysis accordingly...

[Execute discovery queries]

I've found these available fields for the analysis:
- [List available fields]

Here's the adapted analysis using your available data...
```

## Progressive Fallback Strategy

### Level 1: Configuration Value Substitution

- Replace default configuration values with discovered values
- Maintain original query logic
- Document changes made

### Level 2: Field Adaptation

- Substitute missing fields with available equivalents
- Adjust calculation logic as needed
- Explain adaptations to customer

### Level 3: Logic Reconstruction

- Rebuild analysis using available data structures
- Simplify if complex fields unavailable
- Focus on achievable insights

## Customer Communication Best Practices

### Transparency

- Always explain when adaptations are made
- Show both original and adapted queries when helpful
- Acknowledge limitations of adapted analysis

### Education

- Explain why differences occur (custom Salesforce configurations)
- Reference the [Configuration Reference](../configuration-reference.md) for standard values
- Suggest data standardization if relevant
- Provide guidance on improving data structure

### Value Delivery

- Prioritize getting customer actionable insights
- Adapt gracefully without dwelling on limitations
- Focus on what can be analyzed with available data

## Implementation Checklist

### For Every Query Execution

1. ✅ **Execute Original**: Try default configuration values first
1. ✅ **Detect Zero Results**: Check if meaningful data returned
1. ✅ **Discover Values**: Run field discovery queries if needed
1. ✅ **Adapt Query**: Rebuild with customer-specific values
1. ✅ **Communicate Changes**: Explain adaptations made
1. ✅ **Deliver Insights**: Provide business intelligence regardless

### For Complex Multi-Stage Analysis

1. ✅ **Validate Early**: Check field/value availability upfront
1. ✅ **Progressive Discovery**: Build understanding incrementally
1. ✅ **Adaptive Logic**: Modify business rules to fit available data
1. ✅ **Document Assumptions**: Clearly state what was adapted

## Error Handling Patterns

### When Configuration Values Fail

```sql
-- Instead of hard-coded values, use dynamic discovery
WITH available_statuses AS (
    SELECT DISTINCT status FROM lakehouse.object_lead 
    WHERE grax__deleted IS NULL AND status IS NOT NULL
)
-- Adapt logic based on what's actually available
```

### When Fields Missing

```sql
-- Check field availability first
SELECT COUNT(*) as record_count,
       COUNT(expected_field) as field_populated
FROM lakehouse.object_table
WHERE grax__deleted IS NULL
```

### When Business Logic Doesn't Apply

- Simplify analysis to use available data
- Explain business context limitations
- Provide alternative insights where possible

## Configuration Updates

### When Customer Differences Are Identified

1. **Document Discoveries**: Record what values the customer actually uses
1. **Consider Updates**: Evaluate if discoveries should update the [Configuration Reference](../configuration-reference.md)
1. **Share Learnings**: Contribute common variations back to the central configuration
1. **Maintain Flexibility**: Keep fallback patterns robust for edge cases

This fallback strategy ensures the knowledge base provides value to any customer while gracefully handling organizational differences in Salesforce implementation by leveraging the centralized [Configuration Reference](../configuration-reference.md).
