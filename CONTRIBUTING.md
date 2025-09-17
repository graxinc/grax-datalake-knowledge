# Contributing to GRAX Data Lake Knowledge Base

This document provides comprehensive guidelines for creating, updating, and maintaining documentation in the GRAX Data Lake Knowledge Base. All contributors, including AI assistants like Claude, must follow these standards to ensure consistency, quality, and professional presentation.

## Repository Standards

### Branch and Pull Request Naming

**Branch Naming Convention**: `Documentation-Improvements-YYYY-MM-DD`

- Use current date in format: `2025-09-17`
- Always create branches for documentation changes
- Never commit directly to main branch

**Pull Request Naming**: `Documentation Improvements - YYYY-MM-DD`

- Match branch name exactly
- Include comprehensive description of changes
- Reference any issues being addressed

### Repository Configuration

**Target Repository**: `graxinc/grax-datalake-knowledge`

**Required Settings**:

- Database: `lakehouse`  
- Workgroup: `lakehouse`
- Catalog: `lakehouse`
- Never use `information_schema` (users lack access)

## Markdown Linting Requirements

### Zero-Tolerance Compliance

**EVERY file must pass ALL linting rules** as defined in `.markdownlint-cli2.yaml`. No exceptions.

### Critical Linting Rules Reference

**File Structure Requirements**: Refer to `.markdownlint-cli2.yaml` for complete specifications including:

- Files must end with single newline character (MD047)
- No duplicate headings at any level (MD024)
- First line must be top-level heading (MD041)
- Proper heading hierarchy - increment only by one level (MD001)

**Formatting Standards**: All formatting requirements are defined in `.markdownlint-cli2.yaml`:

- Emphasis with asterisks (not underscores)
- Strong emphasis with asterisks (not underscores)
- Unordered lists with dashes (not asterisks)
- Code blocks must be fenced with backticks
- ATX headings only (no underlined format)
- Horizontal rules with exactly 9 dashes

**Content Standards**: Complete requirements available in `.markdownlint-cli2.yaml`:

- No trailing spaces at line endings (MD009)
- No multiple consecutive blank lines
- Headings surrounded by blank lines (MD022)
- Lists surrounded by blank lines (MD032)
- Fenced code blocks surrounded by blank lines (MD031)
- No HTML tags except `<img>` elements
- All links must be valid and properly formatted
- Fenced code blocks must specify language (MD040)

### Validation Process

**Before Every Commit**:

1. Review content against each `.markdownlint-cli2.yaml` rule
1. Ensure proper heading hierarchy and no duplicates
1. Add blank lines around all headings and lists
1. Add blank lines around all fenced code blocks
1. Specify language for all fenced code blocks
1. Remove trailing spaces from all lines
1. Verify file ends with single newline
1. Check all links and references work correctly
1. Validate formatting follows exact standards

**Consequence of Non-Compliance**: Failed linting = Failed PR = Wasted effort requiring fixes and recommit.

## Linting Error Response Protocol

### When Linting Errors Occur

**If user reports linting errors, Claude must**:

1. **Fix all reported errors immediately** in the affected documents
1. **Update this CONTRIBUTING.md file** to add specific prevention guidance for the error types encountered
1. **Reference existing documentation** rather than embedding problematic code samples
1. **Test the fixes** to ensure errors are resolved
1. **Document the learning** to prevent similar issues

### Common Error Prevention Guidelines

**MD022 - Headings Spacing**: Ensure all headings have blank lines both above and below. See existing documents in this repository for proper examples of heading formatting.

**MD031 - Code Block Spacing**: All fenced code blocks require blank lines both before and after the code block. Reference [Query Templates](./docs/query-guidance/query-templates.md) for proper code block formatting examples.

**MD032 - Lists Spacing**: All lists must have blank lines before and after the entire list. See [Configuration Reference](./docs/core-reference/configuration-reference.md) for examples of proper list formatting.

**MD040 - Code Block Language**: All fenced code blocks must specify a language. For SQL queries, use `sql`. For directory structures, use `text`. For shell commands, use `bash`. Reference existing files in the repository for proper language usage.

**MD009 - Trailing Spaces**: Never leave trailing spaces at line endings. Configure your editor to show and remove trailing whitespace automatically.

## Configuration Reference Integration

### Mandatory Configuration Usage

**Never hardcode business values** in documentation or queries. All business-specific values must reference the centralized [Configuration Reference](./docs/core-reference/configuration-reference.md).

**Examples Available In**: [Configuration Reference](./docs/core-reference/configuration-reference.md) contains all proper patterns for referencing configuration values instead of hardcoding business logic.

### Configuration Updates

When encountering customer-specific values different from defaults:

1. **Document discoveries** in analysis comments
1. **Consider updates** to Configuration Reference if common variations
1. **Test all templates** work with new configuration values
1. **Maintain backward compatibility** with existing implementations

## Professional Reporting Standards

### HTML Artifact Requirements

**For ALL customer-facing reports and analyses**:

- Must follow [Reporting Brand Standards](./docs/advanced-topics/reporting-brand-standards.md)
- Must use [HTML Report Template](./docs/advanced-topics/html-report-template.md)
- Required branding: GRAX Primary Purple (#552C98), Secondary Blue (#5F6FE6)
- Professional typography using Work Sans font family
- Interactive elements where appropriate
- Executive summary with actionable insights

### When to Create HTML vs Code Templates

**Execute Queries and Create HTML When**:

- "Build/Show/Analyze/Report" language in requests
- Customer wants insights, trends, or business intelligence
- Request implies deliverable report or presentation
- Analysis requires data visualization

**Provide Code Templates When**:

- Customer asks "how to" or requests syntax
- Request explicitly asks for templates or examples
- Educational/learning context
- Customer wants to run queries themselves

## Directory Structure Standards

### Logical Organization

**Six-Directory Structure**:

- `getting-started/`: New user onboarding and basic concepts
- `core-reference/`: Essential configuration, database schema, execution guidelines  
- `query-guidance/`: SQL patterns, best practices, syntax guides
- `analysis-patterns/`: Business intelligence templates, specialized analysis
- `advanced-topics/`: Professional reporting standards, enterprise customization
- `troubleshooting/`: Error handling, debugging, fallback procedures

### Cross-Reference Requirements

- Use relative paths for all internal links
- Respect directory structure in navigation
- Link to related documentation appropriately
- Update README.md when adding new documents

## SQL Query Standards

### Mandatory Patterns Reference

**All SQL query patterns and examples are available in**: [Query Templates](./docs/query-guidance/query-templates.md)

**Required Patterns Documentation**: [Query Best Practices](./docs/query-guidance/query-best-practices.md) contains comprehensive guidance on:

- Latest Records Pattern (always required)
- Deleted Record Filtering (always required) 
- Date Boundaries (required for performance)

### Configuration Value Usage

**All examples of proper configuration value usage are documented in**: [Configuration Reference](./docs/core-reference/configuration-reference.md)

**Query syntax patterns are available in**: [Athena SQL Syntax Guide](./docs/query-guidance/athena-sql-syntax-guide.md)

## Error Handling and Adaptation

### Customer Configuration Differences

When queries fail due to customer Salesforce differences:

1. **Identify the issue**: Compare expected vs actual configuration values
1. **Execute discovery queries** to find customer-specific values
1. **Adapt the analysis** using customer's actual configuration
1. **Communicate changes** made and why
1. **Follow** [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md)

### Documentation Improvement Process

When encountering exceptions or query errors:

1. **Understand the root cause** and what documentation caused the issue
1. **Use existing branch** `Documentation-Improvements-2025-09-17` or create new one
1. **Update affected documents** with recommended improvements
1. **Test the fixes** prevent the error
1. **Update pull request** with comprehensive description of changes

## Content Quality Standards

### Professional Writing

- Clear, concise business language appropriate for enterprise customers
- Actionable insights and recommendations
- Proper technical terminology
- Complete context and background where needed

### Visual Presentation

- Consistent formatting using markdown standards per `.markdownlint-cli2.yaml`
- Proper code block syntax highlighting as shown in existing repository files
- Clear table structures for reference data (see Configuration Reference)
- Appropriate heading hierarchy for navigation

### Business Value Focus

- Emphasize how GRAX enables customers to "Adapt Faster"
- Connect technical analysis to business outcomes
- Provide strategic recommendations alongside data insights
- Highlight unique value of complete historical data

## Testing and Validation

### Pre-Commit Checklist

- [ ] Document passes all markdown linting rules defined in `.markdownlint-cli2.yaml`
- [ ] Blank lines added around ALL headings and lists
- [ ] Blank lines added around ALL fenced code blocks
- [ ] Language specified for ALL code blocks
- [ ] No trailing spaces anywhere in document
- [ ] All configuration values reference centralized source
- [ ] Code examples follow patterns shown in [Query Templates](./docs/query-guidance/query-templates.md)
- [ ] Links and cross-references work correctly
- [ ] Professional presentation standards per [Reporting Brand Standards](./docs/advanced-topics/reporting-brand-standards.md)
- [ ] File ends with single newline character

### Integration Testing

- [ ] New documentation integrates with existing patterns
- [ ] No duplicate or conflicting guidance
- [ ] Directory structure maintained correctly
- [ ] README.md updated to reference new content

### Continuous Improvement Protocol

**When ANY linting error is reported**:

1. **Immediate Response**: Fix all errors in affected documents
1. **Prevention Update**: Add specific guidance to this CONTRIBUTING.md by referencing existing documentation
1. **Reference Integration**: Point to existing files rather than embedding problematic samples
1. **Documentation Enhancement**: Strengthen prevention guidelines through better cross-references
1. **Knowledge Transfer**: Update training materials and procedures

## Success Metrics

### Quality Indicators

- Zero markdown linting violations per `.markdownlint-cli2.yaml` standards
- Consistent use of configuration references per [Configuration Reference](./docs/core-reference/configuration-reference.md)
- Professional HTML artifacts following [HTML Report Template](./docs/advanced-topics/html-report-template.md)
- Clear, actionable business intelligence delivery
- Scalable documentation structure

### Customer Success

- Queries execute successfully with customer data using patterns from [Query Templates](./docs/query-guidance/query-templates.md)
- Reports provide meaningful business insights following [Business Intelligence Patterns](./docs/analysis-patterns/business-intelligence-patterns.md)
- Professional presentation enhances GRAX brand per [Reporting Brand Standards](./docs/advanced-topics/reporting-brand-standards.md)
- Documentation enables faster customer adaptation via [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md)
- Error resolution maintains productivity

This contributing guide ensures all documentation meets enterprise-grade standards while remaining accessible and actionable for both technical and business users. Following these guidelines maintains the knowledge base's effectiveness as a comprehensive resource for GRAX Data Lake analytics.
