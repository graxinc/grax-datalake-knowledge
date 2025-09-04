# GRAX Data Lake Knowledge Base for LLM Usage

Comprehensive knowledge resources for LLM models to understand and query GRAX Data Lakehouse which contains Salesforce historical data.

## Getting Started

This document assumes you have GRAX Data Lake and GRAX Data Lakehouse deployed. 

- [What is GRAX?](https://documentation.grax.com/)
- [What is GRAX Data Lake?](https://www.grax.com/products/data-lake/)
- [What is GRAX Data Lakehouse?](https://www.grax.com/products/data-lakehouse/)
- [Deploying GRAX](https://documentation.grax.com/platform)
- [Backup & Replicate Salesforce data to AWS](https://documentation.grax.com/protect-data/backup)
- [Enable Data Lake](https://documentation.grax.com/reuse-data/data-lake#getting-started)
- [Enable Data Lakehouse](https://documentation.grax.com/reuse-data/data-lake/aws-data-lakehouse)
- [GRAX Documentation](https://documentation.grax.com/)

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
