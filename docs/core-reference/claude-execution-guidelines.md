# Claude Execution Guidelines

**CRITICAL FOR LLM MODELS**: This document defines when Claude should execute queries versus providing code templates when working with GRAX Data Lake analytics.

## Executive Decision Framework

### Execute Queries When

**Customer asks for reports, insights, or analysis** - Execute immediately to provide value:

- "Generate a sales pipeline report"
- "Show me lead conversion rates"
- "What are our top performing industries?"
- "Create a quarterly business review"
- "Analyze our sales velocity trends"

### Provide Code When

**Customer asks for templates, examples, or wants to modify queries** - Provide reusable code:

- "Show me how to query opportunity data"
- "Give me a template for lead analysis"
- "I want to customize this query"
- "Help me understand the schema"
- "Provide a starting point for..."

## Execution Priority Matrix

| Request Type | Action | Rationale |
|-------------|--------|----------|
| **"Generate/Create/Show me [analysis]"** | EXECUTE | Customer wants insights |
| **"Analyze/Report on [topic]"** | EXECUTE | Customer needs business intelligence |
| **"What are our [metrics]?"** | EXECUTE | Direct question requiring data |
| **"How to query/Template for [analysis]"** | CODE | Customer wants reusable patterns |
| **"Help me build/customize [query]"** | CODE | Customer wants to modify |
| **"Example of [pattern]"** | CODE | Educational/reference purpose |

## Execution Best Practices

### When Executing Queries

1. **Always start with execution** - Provide immediate value
1. **Use latest records patterns** - Ensure current state analysis
1. **Apply proper filtering** - Include `grax__deleted IS NULL`
1. **Reference configuration** - Use [Configuration Reference](configuration-reference.md) values
1. **Add date bounds** - Include reasonable time ranges for performance
1. **Provide context** - Explain what the results mean

### Query Execution Template

```sql
-- Always structure executed queries like this:
WITH latest_records AS (
    -- Get current state of records
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_[TABLE]
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
)
SELECT 
    -- Business-relevant fields
    field1,
    field2,
    COUNT(*) as record_count
FROM latest_records
WHERE rn = 1
    -- Add business logic filters from Configuration Reference
    AND status IN ('value1', 'value2')  
GROUP BY field1, field2
ORDER BY record_count DESC
LIMIT 20  -- Always limit for initial execution
```

## Professional Reporting Requirements

### When Customer Requests Reports

**MANDATORY**: All GRAX reports must be delivered as professional HTML artifacts following [Reporting Brand Standards](../advanced-topics/reporting-brand-standards.md).

1. **Execute Query First** - Get the data and insights
1. **Create HTML Artifact** - Use [HTML Report Template](../advanced-topics/html-report-template.md)
1. **Include Visualizations** - Charts, graphs, and interactive elements
1. **Apply GRAX Branding** - Purple color scheme, Work Sans fonts
1. **Provide Business Context** - Explain insights and recommendations

### Report Delivery Pattern

```markdown
## [Analysis Name] - Executive Summary

**Key Insights**: [1-2 sentence business impact]

**Critical Metrics**: [3-5 key findings]

**Strategic Recommendations**: [Actionable next steps]

[Execute Athena query to gather data]

[Create HTML artifact with visualizations and branding]
```

## Error Handling During Execution

### Common Execution Issues

| Error Type | Immediate Action | Follow-up |
|------------|------------------|----------|
| **Column Not Found** | Check [Database Schema Guide](database-schema-guide.md) | Provide corrected query |
| **No Results** | Use [Customer Fallback Instructions](../troubleshooting/customer-fallback-instructions.md) | Adapt configuration |
| **Performance Timeout** | Add date filters and LIMIT clause | Optimize query |
| **Configuration Mismatch** | Reference customer-specific values | Update [Configuration Reference](configuration-reference.md) |

### Error Recovery Process

1. **Immediate Correction** - Fix the query and re-execute
1. **Explain the Issue** - Tell customer what went wrong
1. **Provide Working Solution** - Show corrected approach
1. **Update Documentation** - Improve guidance to prevent recurrence

## Customer Communication Patterns

### After Successful Execution

```markdown
I've analyzed your [data type] and found [key insight]. Here are the key findings:

[Present results with business context]

**Business Impact**: [Explain what this means for their organization]

**Recommendations**: [Specific actions they can take]

Would you like me to:
- Drill down into any specific area?
- Create additional analysis?
- Generate a comprehensive report with visualizations?
```

### When Providing Code Templates

```markdown
Here's a template for [analysis type] that you can customize:

[Provide well-commented code]

**To customize this query**:
1. Update the date range on line [X]
1. Modify the status filters on line [Y] using values from our Configuration Reference
1. Adjust the GROUP BY fields based on your specific needs

**Configuration Note**: This template uses values from our [Configuration Reference](../core-reference/configuration-reference.md). Update these to match your Salesforce setup.
```

## Integration with Documentation Structure

This execution approach integrates with:

- **[Configuration Reference](configuration-reference.md)** - Business values for executed queries
- **[Database Schema Guide](database-schema-guide.md)** - Field validation during execution
- **[Query Templates](../query-guidance/query-templates.md)** - Reusable patterns for code delivery
- **[Query Best Practices](../query-guidance/query-best-practices.md)** - Performance optimization during execution
- **[Reporting Brand Standards](../advanced-topics/reporting-brand-standards.md)** - Professional report delivery
- **[Customer Fallback Instructions](../troubleshooting/customer-fallback-instructions.md)** - Error recovery during execution

## Quality Assurance Checklist

### Before Executing Any Query

- [ ] Query includes `grax__deleted IS NULL`
- [ ] Latest records pattern is applied
- [ ] Date bounds are included for performance
- [ ] Configuration values are referenced, not hardcoded
- [ ] LIMIT clause prevents runaway queries
- [ ] Business context will be provided with results

### After Query Execution

- [ ] Results are explained in business terms
- [ ] Key insights are highlighted
- [ ] Recommendations are provided when appropriate
- [ ] HTML artifact created for reports (when requested)
- [ ] Follow-up questions are offered

### For Code Templates

- [ ] Code is well-commented and educational
- [ ] Customization instructions are clear
- [ ] Configuration Reference is linked
- [ ] Examples show proper patterns
- [ ] Integration with other docs is noted

Following these guidelines ensures customers receive maximum value from their GRAX Data Lake while maintaining professional standards and proper technical implementation.
