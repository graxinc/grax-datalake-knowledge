# GRAX Data Lake Knowledge Base for LLM Usage

AI-ready knowledge resources that supercharge LLM models with complete historical Salesforce data from your GRAX Data Lakehouse. Built for Amazon Athena engine version 3, these resources work with any LLM to unlock the full value of your data—no limits, just intelligence-ready insights.

## Getting Started

This document assumes you have GRAX Data Lake and GRAX Data Lakehouse deployed.

- [What is GRAX?](https://documentation.grax.com/)
- [What is the GRAX Data Lake?](https://www.grax.com/products/data-lake/)
- [What is the GRAX Data Lakehouse?](https://www.grax.com/products/data-lakehouse/)
- [How do I enable the GRAX Data Lake?](https://documentation.grax.com/reuse-data/data-lake#getting-started)
- [How do I Enable the GRAX Data Lakehouse?](https://documentation.grax.com/reuse-data/data-lake/aws-data-lakehouse)
- [Where is GRAX Documentation?](https://documentation.grax.com/)
- [Where is the `GRAX Data Lake Knowledge Base for LLM Usage` Public Repository?](https://github.com/graxinc/grax-datalake-knowledge)
- [How do I get a free trial and demo?](mailto:sales@grax.com?Subject=Free%20Trial%20for%20GRAX%20Data%20Lakehouse%20)

## Overview

This repository contains structured documentation that enables large language models to effectively understand, navigate, and query Salesforce data stored in a data lake using Amazon Athena. The knowledge base is optimized for LLM consumption and provides comprehensive coverage of data structures, query patterns, and best practices.

## Documentation Structure

### Core Reference Materials

- **[Configuration Reference](./docs/configuration-reference.md)** - **START HERE**: Centralized configuration for all business-specific values
- **[Database Schema Guide](./docs/database-schema-guide.md)** - Complete field reference and data structure documentation
- **[Query Best Practices](./docs/query-best-practices.md)** - Performance optimization and error prevention guidelines
- **[Athena SQL Syntax Guide](./docs/athena-sql-syntax-guide.md)** - Athena-specific SQL patterns and corrections

### LLM Execution Guidance

- **[Claude Execution Guidelines](./docs/claude-execution-guidelines.md)** - **CRITICAL**: When to execute queries vs provide code templates

### Reporting and Brand Standards

- **[Reporting and Brand Standards](./docs/reporting-brand-standards.md)** - **ESSENTIAL**: Professional branding and HTML artifact requirements for all GRAX reports
- **[HTML Report Template](./docs/html-report-template.md)** - **MANDATORY**: Complete HTML template with CSS styling for all GRAX reports

### Multi-Customer Support

- **[Customer Fallback Instructions](./docs/customer-fallback-instructions.md)** - Adapting queries for different Salesforce implementations

### Query Templates and Patterns

- **[Query Templates](./docs/query-templates.md)** - Reusable query patterns for common analysis tasks
- **[Sales Process Analysis](./docs/sales-process-analysis.md)** - Lead qualification and opportunity progression tracking
- **[Data Quality Management](./docs/data-quality-management.md)** - Data integrity validation and cleansing strategies

### Specialized Analysis

- **[Lead Status Audit](./docs/lead-status-audit.md)** - Customer classification and lead qualification accuracy
- **[Business Intelligence Patterns](./docs/business-intelligence-patterns.md)** - Advanced analytics and reporting templates

## Quick Start

For LLM models working with this data lake:

1. **FIRST**: Read [Configuration Reference](./docs/configuration-reference.md) to understand all business-specific values

1. **CRITICAL**: Read [Claude Execution Guidelines](./docs/claude-execution-guidelines.md) to understand when to execute vs provide code

1. **ESSENTIAL**: Review [Reporting and Brand Standards](./docs/reporting-brand-standards.md) for professional HTML artifact requirements

1. **MANDATORY**: Use [HTML Report Template](./docs/html-report-template.md) for all GRAX reports with complete branding and visualizations

1. **Understand**: Review [Database Schema Guide](./docs/database-schema-guide.md) for available data structures

1. **Reference**: Use [Athena SQL Syntax Guide](./docs/athena-sql-syntax-guide.md) for proper query formation

1. **Apply**: Use [Query Templates](./docs/query-templates.md) for common analysis patterns

1. **Follow**: Apply [Query Best Practices](./docs/query-best-practices.md) to avoid common errors

1. **Adapt**: Use [Customer Fallback Instructions](./docs/customer-fallback-instructions.md) when queries return zero results

## Key Principles

- **Configuration First**: Always reference the Configuration Reference for business-specific values
- **Execute First**: When customers ask for reports/analysis, execute queries and provide insights
- **Professional Presentation**: ALL GRAX reports must be delivered as polished HTML artifacts following brand standards using the mandatory HTML template
- **Interactive Visualizations**: Include charts, graphs, and interactive elements in all reports unless explicitly requested otherwise
- **Graceful Adaptation**: Use fallback strategies when customer configurations differ from defaults
- All queries must filter deleted records using `grax__deleted IS NULL`
- Use latest records pattern for current state analysis
- Follow proper field naming conventions with type suffixes
- Apply sequential stage logic for sales funnel analysis
- Prioritize performance optimization and error prevention

## Database Environment

- **Engine**: Amazon Athena
- **Workgroup**: lakehouse
- **Catalog**: lakehouse
- **Database**: lakehouse
- **Data Source**: Salesforce historical data with complete change tracking

## Field Naming Conventions

- `_f` suffix: Numeric fields for calculations
- `_ts` suffix: Timestamp fields for date operations
- `_d` suffix: Date-only fields
- `_b` suffix: Boolean fields
- `_i` suffix: Integer fields

## Customization for Different Organizations

The knowledge base is built with GRAX's default Salesforce configuration but can be easily adapted:

1. **Review** the [Configuration Reference](./docs/configuration-reference.md) for all default values
1. **Modify** configuration values to match your organization's Salesforce setup
1. **Test** queries with your specific field values and stage names
1. **Use** the [Customer Fallback Instructions](./docs/customer-fallback-instructions.md) for automatic adaptation

## Contributing

This knowledge base is designed for programmatic consumption by LLM models. All documentation follows strict markdown linting rules and is optimized for clarity and precision in automated query generation.

## Support

We'd love to help you get the most out of the GRAX Data Lake Knowledge Base! If you need assistance, have questions, or run into any issues:

- **Get Help**: Email us at [help@grax.com](mailto:help@grax.com?Subject=grax-datalake-knowledge%20Help%20Needed) - we're here to help!
- **Documentation & Guides**: Visit our comprehensive documentation at [https://documentation.grax.com/support](https://documentation.grax.com/support)

Our team is committed to helping you succeed with your data lake implementation and we welcome all feedback to make this knowledge base even better.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
