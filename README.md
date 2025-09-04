# Data Lake Knowledge Base

Comprehensive knowledge resources for LLM models to understand and query Salesforce data through Amazon Athena.

## Overview

This repository contains structured documentation that enables large language models to effectively understand, navigate, and query Salesforce data stored in a data lake using Amazon Athena. The knowledge base is optimized for LLM consumption and provides comprehensive coverage of data structures, query patterns, and best practices.

## Documentation Structure

### Core Reference Materials

- **[Database Schema Guide](./docs/database-schema-guide.md)** - Complete field reference and data structure documentation
- **[Query Best Practices](./docs/query-best-practices.md)** - Performance optimization and error prevention guidelines
- **[Athena SQL Syntax Guide](./docs/athena-sql-syntax-guide.md)** - Athena-specific SQL patterns and corrections

### Query Templates and Patterns

- **[Query Templates](./docs/query-templates.md)** - Reusable query patterns for common analysis tasks
- **[Sales Process Analysis](./docs/sales-process-analysis.md)** - Lead qualification and opportunity progression tracking
- **[Data Quality Management](./docs/data-quality-management.md)** - Data integrity validation and cleansing strategies

### Specialized Analysis

- **[Lead Status Audit](./docs/lead-status-audit.md)** - Customer classification and lead qualification accuracy
- **[Business Intelligence Patterns](./docs/business-intelligence-patterns.md)** - Advanced analytics and reporting templates

## Quick Start

For LLM models working with this data lake:

1. **Start with** the [Database Schema Guide](./docs/database-schema-guide.md) to understand available data structures

1. **Reference** the [Athena SQL Syntax Guide](./docs/athena-sql-syntax-guide.md) for proper query formation

1. **Use** the [Query Templates](./docs/query-templates.md) for common analysis patterns

1. **Follow** the [Query Best Practices](./docs/query-best-practices.md) to avoid common errors

## Key Principles

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
