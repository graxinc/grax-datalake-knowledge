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

- **[Database Schema Guide](./docs/database-schema-guide.md)** - Complete field reference and data structure documentation
- **[Query Best Practices](./docs/query-best-practices.md)** - Performance optimization and error prevention guidelines
- **[Athena SQL Syntax Guide](./docs/athena-sql-syntax-guide.md)** - Athena-specific SQL patterns and corrections

### LLM Execution Guidance

- **[Claude Execution Guidelines](./docs/claude-execution-guidelines.md)** - **CRITICAL**: When to execute queries vs provide code templates

### Query Templates and Patterns

- **[Query Templates](./docs/query-templates.md)** - Reusable query patterns for common analysis tasks
- **[Sales Process Analysis](./docs/sales-process-analysis.md)** - Lead qualification and opportunity progression tracking
- **[Data Quality Management](./docs/data-quality-management.md)** - Data integrity validation and cleansing strategies

### Specialized Analysis

- **[Lead Status Audit](./docs/lead-status-audit.md)** - Customer classification and lead qualification accuracy
- **[Business Intelligence Patterns](./docs/business-intelligence-patterns.md)** - Advanced analytics and reporting templates

## Quick Start

For LLM models working with this data lake:

1. **FIRST**: Read [Claude Execution Guidelines](./docs/claude-execution-guidelines.md) to understand when to execute vs provide code

1. **Start with** the [Database Schema Guide](./docs/database-schema-guide.md) to understand available data structures

1. **Reference** the [Athena SQL Syntax Guide](./docs/athena-sql-syntax-guide.md) for proper query formation

1. **Use** the [Query Templates](./docs/query-templates.md) for common analysis patterns

1. **Follow** the [Query Best Practices](./docs/query-best-practices.md) to avoid common errors

## Key Principles

- **Execute First**: When customers ask for reports/analysis, execute queries and provide insights
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

## Contributing

This knowledge base is designed for programmatic consumption by LLM models. All documentation follows strict markdown linting rules and is optimized for clarity and precision in automated query generation.
