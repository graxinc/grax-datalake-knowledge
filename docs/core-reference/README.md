# Core Reference

This directory contains essential reference documentation that forms the foundation of the GRAX Data Lake Knowledge Base. These documents provide the fundamental configuration, database schema, and execution guidelines that all other documentation builds upon.

## Directory Purpose

The Core Reference directory serves as the authoritative source for foundational information required for successful interaction with the GRAX Data Lake. This includes configuration standards, database structure documentation, and critical execution guidelines that ensure consistent and effective use of the data lake infrastructure.

## Documents in This Directory

### Configuration and Standards

- **[Configuration Reference](configuration-reference.md)** - **START HERE**: Centralized configuration for all business-specific values, segmentation thresholds, and validation rules. This document serves as the single source of truth for all configurable values used throughout the knowledge base.

- **[Database Schema Guide](database-schema-guide.md)** - Complete field reference and data structure documentation for all Salesforce objects in the data lake, including field definitions, data types, and relationships.

### Execution Guidelines

- **[Claude Execution Guidelines](claude-execution-guidelines.md)** - **CRITICAL**: Comprehensive guidance on when to execute queries versus providing code templates, including customer intent analysis and professional reporting requirements.

## Key Principles

### Configuration-First Approach

All documentation and queries in this repository reference the [Configuration Reference](configuration-reference.md) rather than embedding hardcoded business values. This approach ensures:

- Adaptability across different customer Salesforce implementations
- Single point of maintenance for business logic changes
- Consistent application of business rules throughout the knowledge base

### Database Structure Understanding

The [Database Schema Guide](database-schema-guide.md) provides comprehensive understanding of:

- Salesforce object structure in the data lake environment
- Field definitions and data type specifications
- Relationship patterns between objects
- Universal system fields and their usage patterns

### Execution Best Practices

The [Claude Execution Guidelines](claude-execution-guidelines.md) ensure:

- Appropriate decision-making between execution and template provision
- Professional HTML artifact creation for customer-facing deliverables
- Consistent brand standards and presentation quality
- Error recovery and adaptation procedures

## Integration with Repository

### Cross-Reference Architecture

Core Reference documents are extensively referenced throughout the repository:

- **Query Guidance**: References Configuration Reference for business values and Database Schema Guide for field definitions
- **Analysis Patterns**: Uses Configuration Reference for segmentation rules and execution guidelines for report formatting
- **Advanced Topics**: Integrates with execution guidelines for professional reporting standards
- **Troubleshooting**: References configuration patterns for error resolution and adaptation procedures

### Dependency Relationships

- **Configuration Reference** → Referenced by all other documentation requiring business values
- **Database Schema Guide** → Referenced by all query-related documentation
- **Claude Execution Guidelines** → Referenced by all analysis and reporting documentation

## Usage Guidelines

### For New Contributors

1. **Start with Configuration Reference**: Understand the centralized approach to business values
1. **Review Database Schema Guide**: Familiarize yourself with data lake structure and field definitions
1. **Study Execution Guidelines**: Learn when to execute queries versus provide templates

### For Documentation Updates

- Always reference Core Reference documents rather than duplicating information
- Update Configuration Reference when adding new business logic or values
- Ensure new documentation integrates with existing core reference patterns
- Maintain cross-reference integrity when updating core documents

### For Query Development

- Use Configuration Reference values instead of hardcoding business logic
- Reference Database Schema Guide for accurate field usage and data types
- Follow Execution Guidelines for appropriate response patterns

## Maintenance and Updates

### Configuration Management

The [Configuration Reference](configuration-reference.md) requires updates when:

- New business values or thresholds are identified
- Customer implementations reveal common variations
- Segmentation logic needs refinement
- Validation rules require adjustment

### Schema Documentation

The [Database Schema Guide](database-schema-guide.md) requires updates when:

- New Salesforce objects are added to the data lake
- Field definitions change or new fields are introduced
- Relationship patterns are discovered or modified
- Data type specifications need clarification

### Execution Guidelines Updates

The [Claude Execution Guidelines](claude-execution-guidelines.md) require updates when:

- New execution patterns are established
- Professional reporting standards evolve
- Error recovery procedures are refined
- Customer interaction patterns change

## Success Metrics

### Reference Effectiveness

- Consistent use of Configuration Reference values across all repository documentation
- Accurate database schema references in all query-related content
- Appropriate execution decisions based on established guidelines

### Documentation Quality

- Zero hardcoded business values in repository documentation
- Complete cross-reference integrity maintained across all documents
- Professional presentation standards consistently applied

### Customer Success

- Successful query execution using centralized configuration patterns
- Accurate database interactions based on schema documentation
- Professional deliverables meeting execution guidelines standards

This Core Reference directory provides the essential foundation that enables the entire GRAX Data Lake Knowledge Base to function as a cohesive, adaptable, and professional resource for customers implementing data lake analytics solutions.
