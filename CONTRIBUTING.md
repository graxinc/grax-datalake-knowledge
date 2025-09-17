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

## Exception Handling and Query Error Recovery

### Systematic Error Resolution Process

When Claude encounters query exceptions or errors:

1. **Immediate Diagnosis**: Identify the specific error type and root cause
2. **Customer Adaptation**: Use [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md) to adapt queries for customer-specific configurations
3. **Documentation Updates**: Update relevant documentation to prevent similar errors
4. **Pull Request Integration**: Add improvements to the active branch using the format `Documentation-Improvements-YYYY-MM-DD`

### Common Exception Categories

**Query Execution Errors**:

- Column resolution failures → Use field discovery patterns from [Database Schema Guide](./docs/core-reference/database-schema-guide.md)
- Configuration mismatches → Apply [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md)
- Performance timeouts → Follow optimization patterns in [Query Best Practices](./docs/query-guidance/query-best-practices.md)

**Linting Violations**:

- MD022 (heading spacing) → Reference proper formatting in existing docs
- MD031 (code block spacing) → Follow patterns from [Query Templates](./docs/query-guidance/query-templates.md)
- MD040 (language specification) → Use `sql`, `bash`, or `text` as appropriate

**Configuration Mismatches**:

- Lead status value differences → Implement discovery patterns from [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md)
- Opportunity stage variations → Apply adaptive query reconstruction
- Account classification differences → Use dynamic value discovery

### Exception Response Protocol

**When any error occurs**:

1. **Acknowledge the Issue**: Confirm understanding of the specific error
2. **Implement Immediate Fix**: Provide working solution for the customer
3. **Update Documentation**: Enhance relevant sections to prevent recurrence
4. **Test Prevention**: Validate that documentation improvements prevent the error
5. **Document Learning**: Record patterns for future prevention

## Claude Integration and Behavioral Standards

### Mandatory Response Patterns

**When Claude encounters customer requests**:

1. **Execute First Philosophy**: Always attempt to execute queries and provide insights rather than just providing code templates
2. **HTML Artifact Requirement**: ALL analysis results must be delivered as professional HTML artifacts following [Reporting Brand Standards](./docs/advanced-topics/reporting-brand-standards.md)
3. **Configuration Reference Priority**: Always use values from [Configuration Reference](./docs/core-reference/configuration-reference.md) before attempting queries
4. **Graceful Adaptation**: When queries fail, implement [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md) automatically

### Decision Tree Integration

Claude must follow the explicit decision tree in [Claude Execution Guidelines](./docs/core-reference/claude-execution-guidelines.md):

- Execute queries for "Build/Show/Analyze" requests
- Provide code templates only for "How to/Template/Syntax" requests
- Always deliver professional HTML reports with complete GRAX branding
- Include interactive visualizations unless explicitly requested otherwise

### Behavioral Compliance Standards

**Required Behaviors**:

- **Project Knowledge First**: Search project knowledge before providing any response
- **Configuration Integration**: Reference centralized configuration instead of hardcoding values
- **Professional Presentation**: Deliver enterprise-grade HTML artifacts for all customer-facing reports
- **Error Recovery**: Implement automatic fallback patterns when standard queries fail
- **Documentation Updates**: Contribute improvements when encountering new patterns or errors

## Project Knowledge Integration

### Mandatory Knowledge Search Protocol

Before providing any response, Claude must:

1. **Search Project Knowledge**: Use project_knowledge_search to find relevant existing documentation
2. **Reference Existing Patterns**: Link to established templates rather than creating new examples
3. **Update Documentation**: When encountering new patterns, update appropriate sections
4. **Maintain Consistency**: Ensure all responses align with established knowledge base patterns

### Knowledge Base Navigation

**Primary Reference Sources** (check in order):

1. [Configuration Reference](./docs/core-reference/configuration-reference.md) - All business values
2. [Claude Execution Guidelines](./docs/core-reference/claude-execution-guidelines.md) - Behavioral decisions
3. [Query Templates](./docs/query-guidance/query-templates.md) - SQL patterns
4. [Reporting Brand Standards](./docs/advanced-topics/reporting-brand-standards.md) - Presentation requirements
5. [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md) - Error recovery patterns

### Documentation Priority Framework

**When creating or updating content**:

- **Single Source of Truth**: Maintain authoritative information in one location
- **Reference Integration**: Link to existing patterns rather than duplicating content
- **Cross-Reference Consistency**: Ensure all related documents remain synchronized
- **Scalable Structure**: Support growth while maintaining logical organization

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
2. Ensure proper heading hierarchy and no duplicates
3. Add blank lines around all headings and lists
4. Add blank lines around all fenced code blocks
5. Specify language for all fenced code blocks
6. Remove trailing spaces from all lines
7. Verify file ends with single newline
8. Check all links and references work correctly
9. Validate formatting follows exact standards

**Consequence of Non-Compliance**: Failed linting = Failed PR = Wasted effort requiring fixes and recommit.

## Linting Error Response Protocol

### When Linting Errors Occur

**If user reports linting errors, Claude must**:

1. **Fix all reported errors immediately** in the affected documents
2. **Update this CONTRIBUTING.md file** to add specific prevention guidance for the error types encountered
3. **Reference existing documentation** rather than embedding problematic code samples
4. **Test the fixes** to ensure errors are resolved
5. **Document the learning** to prevent similar issues

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
2. **Consider updates** to Configuration Reference if common variations
3. **Test all templates** work with new configuration values
4. **Maintain backward compatibility** with existing implementations

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

## Enhanced Testing and Validation Requirements

### Athena Query Testing Standards

**Before committing any SQL examples**:

- [ ] Database parameter: `lakehouse`
- [ ] Workgroup parameter: `lakehouse`
- [ ] All queries include `grax__deleted IS NULL` filtering
- [ ] Latest records pattern applied using `grax__idseq`
- [ ] Configuration values referenced from [Configuration Reference](./docs/core-reference/configuration-reference.md)
- [ ] Query tested with customer fallback patterns from [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md)

### HTML Artifact Validation

**Before delivering reports to customers**:

- [ ] Complete GRAX branding implementation per [Reporting Brand Standards](./docs/advanced-topics/reporting-brand-standards.md)
- [ ] Interactive elements included where appropriate
- [ ] Mobile responsiveness verified
- [ ] Professional polish meeting enterprise standards
- [ ] Executive summary with actionable insights included
- [ ] Charts and visualizations using brand colors
- [ ] Proper semantic HTML5 structure

### Integration Testing

- [ ] New documentation integrates with existing patterns
- [ ] No duplicate or conflicting guidance
- [ ] Directory structure maintained correctly
- [ ] Appropriate directory README.md updated to reference new content
- [ ] Cross-references use proper relative paths

## Branch and Pull Request Management Protocol

### Active Branch Usage

**For ongoing improvements**: Use existing active branch with current date format when available

**When to create new branch**: Only when no current active branch exists or for major structural changes

### Pull Request Integration

**When updating documentation due to exceptions**:

1. **Use Active PR**: Add improvements to existing active pull request when available
2. **Comprehensive Updates**: Include all related documentation improvements, not just the specific error fix
3. **Cross-Reference Updates**: Ensure all related documents maintain consistency
4. **Testing Verification**: Validate that fixes prevent the original error

### Branch Naming Standards

**Documentation Updates**: Use format `Documentation-Improvements-YYYY-MM-DD`

**Feature Development**: Use descriptive names like `feature/new-analysis-pattern`

**Bug Fixes**: Use format `fix/error-description`

## Directory Structure Standards

### Logical Organization

**Scalable Directory Structure**: The repository uses a logical organization system where each major topic area has its own directory. Each directory contains a README.md file that explains the directory's purpose and provides links to all documentation within that area.

**Current Major Areas** (subject to expansion as repository grows):

- `getting-started/`: Access via [Getting Started README](./docs/getting-started/README.md)
- `core-reference/`: Access via [Core Reference README](./docs/core-reference/README.md)
- `query-guidance/`: Access via [Query Guidance README](./docs/query-guidance/README.md)
- `analysis-patterns/`: Access via [Analysis Patterns README](./docs/analysis-patterns/README.md)
- `advanced-topics/`: Access via [Advanced Topics README](./docs/advanced-topics/README.md)
- `troubleshooting/`: Access via [Troubleshooting README](./docs/troubleshooting/README.md)

**Directory Expansion Principles**:

- Each new topic area receives its own directory with descriptive naming
- Every directory must contain a README.md explaining its purpose and scope
- Directory README files must link to all child documentation
- New directories follow the same organizational patterns as existing ones
- Directory structure supports unlimited growth while maintaining logical organization

### Cross-Reference Requirements

- Use relative paths for all internal links
- Respect directory structure in navigation
- Link to related documentation appropriately
- Update README.md files when adding new documents to directories
- Maintain consistency in directory README structure across the repository

## Content Standards

### Avoid Duplication

**Single Source of Truth**: Each piece of information should exist in exactly one authoritative location within the repository. All other references should link to that authoritative source rather than duplicating content.

**Reference-Based Approach**: Rather than embedding examples or samples in multiple documents, create comprehensive examples in their appropriate domain-specific locations and reference them from other documents.

**Documentation Evolution**: The repository continuously improves and expands. All documentation should be written with the assumption that content will grow, change, and be refined over time.

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
2. **Execute discovery queries** to find customer-specific values
3. **Adapt the analysis** using customer's actual configuration
4. **Communicate changes** made and why
5. **Follow** [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md)

### Documentation Improvement Process

When encountering exceptions or query errors:

1. **Understand the root cause** and what documentation caused the issue
2. **Use existing branch** with current date format or create new one
3. **Update affected documents** with recommended improvements
4. **Test the fixes** prevent the error
5. **Update pull request** with comprehensive description of changes

## Quality Assurance and Success Metrics

### Documentation Quality Standards

**Zero Defect Goals**:

- 100% markdown linting compliance per `.markdownlint-cli2.yaml`
- Zero hardcoded business values (all referenced from Configuration Reference)
- 100% cross-reference accuracy using relative paths
- Professional HTML artifact delivery rate: 100% for customer-facing reports

### Claude Performance Standards

**Behavioral Compliance**:

- Execute queries first for analysis requests (not code templates)
- Professional HTML artifacts for all reports
- Graceful adaptation when customer configurations differ
- Comprehensive error recovery using established fallback patterns

### Continuous Improvement Protocol

**When any error occurs**:

1. **Immediate Fix**: Resolve the specific issue
2. **Pattern Analysis**: Identify if this represents a broader documentation gap
3. **Prevention Enhancement**: Strengthen documentation to prevent similar issues
4. **Knowledge Integration**: Ensure fixes align with existing knowledge patterns

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
- [ ] No duplicate content - all examples reference authoritative sources
- [ ] Directory README files updated if adding new documents

### Continuous Improvement Protocol

**When ANY linting error is reported**:

1. **Immediate Response**: Fix all errors in affected documents
2. **Prevention Update**: Add specific guidance to this CONTRIBUTING.md by referencing existing documentation
3. **Reference Integration**: Point to existing files rather than embedding problematic samples
4. **Documentation Enhancement**: Strengthen prevention guidelines through better cross-references
5. **Knowledge Transfer**: Update training materials and procedures

## Success Metrics

### Quality Indicators

- Zero markdown linting violations per `.markdownlint-cli2.yaml` standards
- Consistent use of configuration references per [Configuration Reference](./docs/core-reference/configuration-reference.md)
- Professional HTML artifacts following [HTML Report Template](./docs/advanced-topics/html-report-template.md)
- Clear, actionable business intelligence delivery
- Scalable documentation structure with proper directory organization
- No duplicate content across repository
- Comprehensive directory README files providing clear navigation

### Customer Success

- Queries execute successfully with customer data using patterns from [Query Templates](./docs/query-guidance/query-templates.md)
- Reports provide meaningful business insights following [Business Intelligence Patterns](./docs/analysis-patterns/business-intelligence-patterns.md)
- Professional presentation enhances GRAX brand per [Reporting Brand Standards](./docs/advanced-topics/reporting-brand-standards.md)
- Documentation enables faster customer adaptation via [Customer Fallback Instructions](./docs/troubleshooting/customer-fallback-instructions.md)
- Error resolution maintains productivity

This contributing guide ensures all documentation meets enterprise-grade standards while remaining accessible and actionable for both technical and business users. Following these guidelines maintains the knowledge base's effectiveness as a comprehensive resource for GRAX Data Lake analytics that grows and improves continuously.
