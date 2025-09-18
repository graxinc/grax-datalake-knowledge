# Data Quality Framework

## Overview

The GRAX Data Lake Data Quality Framework provides comprehensive tools for analyzing, scoring, and remediating data quality issues across all Salesforce objects stored in the data lakehouse. This framework ensures customers can maintain high-quality data standards while adapting to their specific business requirements.

## Framework Components

The data quality framework consists of three core analytical components for each Salesforce object:

### Analysis Phase

**Comprehensive Assessment**: Systematic evaluation of data completeness, accuracy, consistency, and integrity patterns across all critical fields and business processes.

**Multi-Dimensional Coverage**:

- **Completeness Analysis**: Missing value patterns and field population rates
- **Accuracy Analysis**: Data format validation and business rule compliance  
- **Consistency Analysis**: Cross-field relationships and logical dependencies
- **Integrity Analysis**: Reference integrity and historical data patterns

### Scoring Phase

**Quantitative Assessment**: Standardized scoring methodology that produces comparable quality metrics across different objects and time periods.

**Scoring Dimensions**:

- **Field-Level Scores**: Individual field quality metrics (0-100 scale)
- **Record-Level Scores**: Overall record quality assessment
- **Object-Level Scores**: Aggregate quality metrics for entire objects
- **Trend Scores**: Quality improvement or degradation over time

### Remediation Phase

**Actionable Solutions**: Specific recommendations and automated fixes for identified data quality issues.

**Remediation Categories**:

- **Immediate Fixes**: Automated corrections for common data quality issues
- **Process Improvements**: Workflow enhancements to prevent future issues
- **Training Recommendations**: User education to improve data entry practices
- **System Enhancements**: Configuration changes to enforce data quality

## Directory Structure

Each Salesforce object follows a standardized directory structure within `docs/data-quality/`:

```text
docs/data-quality/
├── README.md (this file)
├── lead/
│   ├── analysis.md
│   ├── scoring.md
│   └── remediation.md
├── opportunity/
│   ├── analysis.md
│   ├── scoring.md
│   └── remediation.md
└── account/
    ├── analysis.md
    ├── scoring.md
    └── remediation.md
```

### Directory Naming Convention

**Object Directory Names**: Use lowercase Salesforce API names (e.g., `lead`, `opportunity`, `account`, `contact`, `case`)

**Corresponding Data Lake Tables**: Each directory maps to its respective lakehouse table:

- **lead** → `lakehouse.object_lead`
- **opportunity** → `lakehouse.object_opportunity`  
- **account** → `lakehouse.object_account`
- **contact** → `lakehouse.object_contact`
- **case** → `lakehouse.object_case`

## Implementation Guidelines

### Configuration Integration

All data quality analyses must reference centralized configuration values from [Configuration Reference](../core-reference/configuration-reference.md) rather than embedding hardcoded values.

**Required References**:

- Business process values (lead status, opportunity stages)
- Segmentation thresholds (employee counts, revenue ranges)
- Quality benchmarks and acceptable ranges
- Field validation rules and formats

### Quality Standards

**Professional Presentation**: All data quality reports must follow [Reporting Brand Standards](../advanced-topics/reporting-brand-standards.md) and use the [HTML Report Template](../advanced-topics/html-report-template.md).

**Athena Integration**: All queries must follow established patterns from [Query Best Practices](../query-guidance/query-best-practices.md) and [Athena SQL Syntax Guide](../query-guidance/athena-sql-syntax-guide.md).

### Framework Expansion

**Adding New Objects**: To extend the framework to additional Salesforce objects:

1. **Create Directory**: Create `docs/data-quality/[object_api_name]/`
1. **Standard Documents**: Create `analysis.md`, `scoring.md`, and `remediation.md`
1. **Update Configuration**: Add object-specific configuration values to [Configuration Reference](../core-reference/configuration-reference.md)
1. **Update README**: Document the new object in the main repository [README.md](../../README.md)

**Consistency Requirements**: All object implementations must follow identical structure and methodology to ensure framework consistency and customer adaptability.

## Integration with Existing Documentation

### Cross-Reference Architecture

**Core References**:

- **[Configuration Reference](../core-reference/configuration-reference.md)**: Source of all business-specific values
- **[Database Schema Guide](../core-reference/database-schema-guide.md)**: Field definitions and data types
- **[Query Templates](../query-guidance/query-templates.md)**: Standardized query patterns

**Analysis References**:

- **[Business Intelligence Patterns](../analysis-patterns/business-intelligence-patterns.md)**: Advanced analytical frameworks
- **[Sales Process Analysis](../analysis-patterns/sales-process-analysis.md)**: Lead and opportunity progression patterns

**Quality Assurance**:

- **[Query Best Practices](../query-guidance/query-best-practices.md)**: Performance and accuracy guidelines
- **[Athena SQL Syntax Guide](../query-guidance/athena-sql-syntax-guide.md)**: Platform-specific SQL patterns

## Success Metrics

### Framework Effectiveness

**Customer Adoption**:

- Reduction in data quality-related support tickets
- Increased customer self-service capability
- Faster time-to-insight for data quality assessments

**Quality Improvement**:

- Measurable data quality score improvements over time
- Reduced manual data remediation effort
- Enhanced business intelligence accuracy and reliability

**Scalability**:

- Framework extension to additional Salesforce objects
- Adaptability across different customer implementations
- Integration with customer-specific business processes

The Data Quality Framework serves as a comprehensive foundation for maintaining and improving data quality standards across the entire GRAX Data Lake ecosystem while providing customers with actionable insights and automated solutions.
