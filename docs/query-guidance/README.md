# Query Guidance

This directory contains comprehensive guidance for building effective SQL queries against the GRAX Data Lake using Amazon Athena.

## Query Development Workflow

1. **Start with Configuration**: Review [Configuration Reference](../core-reference/configuration-reference.md) for business-specific values
1. **Choose Templates**: Select appropriate patterns from [Query Templates](query-templates.md)
1. **Apply Best Practices**: Follow performance and error prevention guidelines
1. **Use Proper Syntax**: Reference Athena-specific SQL patterns and corrections

## Documents in This Section

### Core Query Resources

- **[Query Templates](query-templates.md)** - Reusable query patterns for common analysis tasks
- **[Query Best Practices](query-best-practices.md)** - Performance optimization and error prevention guidelines
- **[Athena SQL Syntax Guide](athena-sql-syntax-guide.md)** - Athena-specific SQL patterns and corrections

## Key Principles for All Queries

### Mandatory Patterns

**Latest Records**: Always use for current state analysis

```sql
WITH latest_records AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_[table]
    WHERE grax__deleted IS NULL
)
SELECT * FROM latest_records WHERE rn = 1
```

**Deleted Record Filtering**: Always exclude soft-deleted records

```sql
WHERE grax__deleted IS NULL
```

**Date Boundaries**: Always include for performance

```sql
WHERE createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
```

### Configuration Integration

**Never hardcode values** - Always reference [Configuration Reference](../core-reference/configuration-reference.md):

```sql
-- DON'T: Hardcoded values
WHERE status IN ('Open', 'Working', 'Qualified')

-- DO: Reference configuration
-- Lead status values from Configuration Reference
WHERE status IN ('MQL', 'SAL', 'SQL')  
```

## Common Query Patterns

### Basic Analysis Pattern

```sql
-- Standard analysis template
WITH latest_[object] AS (
    -- Get current state
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY id ORDER BY grax__idseq DESC) as rn
    FROM lakehouse.object_[table]
    WHERE grax__deleted IS NULL
        AND createddate_ts >= DATE_ADD('month', -12, CURRENT_DATE)
),
enriched_data AS (
    -- Add business logic and segmentation
    SELECT 
        *,
        CASE 
            WHEN field = 'value1' THEN 'Category A'
            WHEN field = 'value2' THEN 'Category B'
            ELSE 'Other'
        END as segment
    FROM latest_[object]
    WHERE rn = 1
)
SELECT 
    segment,
    COUNT(*) as record_count,
    -- Add relevant metrics
FROM enriched_data
GROUP BY segment
ORDER BY record_count DESC
```

### Cross-Object Analysis Pattern

```sql
-- Multi-object relationship analysis
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
ORDER BY total_pipeline DESC
LIMIT 20
```

## Performance Guidelines

### Query Optimization Priorities

1. **Date Filtering**: Leverage partitioning with date bounds
1. **Deleted Record Filtering**: Reduce scan volume early
1. **Latest Record Patterns**: Ensure current state accuracy
1. **Appropriate Limiting**: Use LIMIT for testing and top-N analysis
1. **Index-Friendly Operations**: Structure queries for optimal performance

### Testing and Development

1. **Start Small**: Use LIMIT 10 for initial development
1. **Add Date Bounds**: Include reasonable time ranges
1. **Validate Results**: Check for expected data patterns
1. **Remove Limits**: Only after validation for production queries
1. **Monitor Performance**: Watch for timeout issues

## Error Prevention

### Common Mistakes to Avoid

- **Missing deleted record filtering**: Always include `grax__deleted IS NULL`
- **Forgetting latest record patterns**: Historical data requires version control
- **Hardcoding configuration values**: Reference the Configuration Reference document
- **Missing date bounds**: Performance issues with large scans
- **Incorrect field types**: Use proper suffix patterns (_f, _ts, _d, etc.)

### Debugging Approach

1. **Start with Basic Query**: Verify data exists
1. **Add Filters Incrementally**: Build complexity step by step
1. **Check Configuration Values**: Ensure picklist values match
1. **Validate Relationships**: Confirm foreign key references
1. **Test with Small Limits**: Prevent runaway queries during development

## Integration with Other Documentation

Query guidance integrates with:

- **[Database Schema Guide](../core-reference/database-schema-guide.md)** - Field definitions and relationships
- **[Configuration Reference](../core-reference/configuration-reference.md)** - Business values and validation rules
- **[Business Intelligence Patterns](../analysis-patterns/business-intelligence-patterns.md)** - Advanced analytics templates
- **[Customer Fallback Instructions](../troubleshooting/customer-fallback-instructions.md)** - Error recovery and adaptation
- **[Claude Execution Guidelines](../core-reference/claude-execution-guidelines.md)** - When to execute vs. provide templates

Use this section as your primary resource for all query development activities in the GRAX Data Lake environment.
