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

### Critical Linting Rules

**File Structure Requirements**:

- Files must end with single newline character (MD047)
- No duplicate headings at any level (MD024)
- First line must be top-level heading (MD041)
- Proper heading hierarchy - increment only by one level (MD001)

**Formatting Standards**:

- Emphasis with asterisks: `*italic*` not `_italic_`
- Strong emphasis with asterisks: `**bold**` not `__bold__`
- Unordered lists with dashes: `- item` not `* item`
- Code blocks must be fenced with backticks
- ATX headings only: `## Heading` not underlined format
- Horizontal rules: `---------` (exactly 9 dashes)

**Content Standards**:

- No trailing spaces at line endings (MD009)
- No multiple consecutive blank lines
- Headings surrounded by blank lines (MD022)
- Lists surrounded by blank lines (MD032)
- Fenced code blocks surrounded by blank lines (MD031)
- No HTML tags except `<img>` elements
- All links must be valid and properly formatted
- Fenced code blocks must specify language (MD040)

### Specific Formatting Requirements

**CRITICAL: Always add blank lines around headings and lists**:

```markdown
## Heading Above

Content here.

### Subheading Below

**List Requirements**:

- Always add blank line before list
- Keep list items properly formatted
- Always add blank line after list

Next paragraph starts here.
```

**Code Block Formatting Requirements**:

```text
<!-- WRONG: Missing language specification and blank lines -->
Content before code block.

```
SELECT * FROM table;
```

Content immediately after.

<!-- CORRECT: Language specified and surrounded by blank lines -->
Content before code block.

```sql
SELECT * FROM table;
```

Content after code block.

<!-- For directory structures, use 'text' -->
Directory structure example:

```text
docs/
├── file1.md
└── file2.md
```

Next section continues here.
```

**Trailing Spaces Elimination**:

- Never leave trailing spaces at end of lines
- Use editor settings to show/remove trailing whitespace
- Particularly check after bold/italic formatting and lists

### Validation Process

**Before Every Commit**:

1. Review content against each `.markdownlint-cli2.yaml` rule
1. Ensure proper heading hierarchy and no duplicates
1. **ADD BLANK LINES** around all headings and lists
1. **ADD BLANK LINES** around all fenced code blocks
1. **SPECIFY LANGUAGE** for all fenced code blocks
1. **REMOVE TRAILING SPACES** from all lines
1. Verify file ends with single newline
1. Check all links and references work correctly
1. Validate formatting follows exact standards

**Consequence of Non-Compliance**: Failed linting = Failed PR = Wasted effort requiring fixes and recommit.

## Linting Error Response Protocol

### When Linting Errors Occur

**If user reports linting errors, Claude must**:

1. **Fix all reported errors immediately** in the affected documents
1. **Update this CONTRIBUTING.md file** to add specific prevention guidance for the error types encountered
1. **Add examples** showing correct vs incorrect formatting for future reference
1. **Test the fixes** to ensure errors are resolved
1. **Document the learning** to prevent similar issues

### Common Error Prevention Examples

**MD022 - Headings must be surrounded by blank lines**:

```markdown
<!-- WRONG -->
Some content here.
### Heading Without Space
More content immediately after.

<!-- CORRECT -->
Some content here.

### Heading With Proper Spacing

More content with blank lines above and below heading.
```

**MD031 - Fenced code blocks must be surrounded by blank lines**:

```text
<!-- WRONG -->
Content before code block.

```
SELECT * FROM table;
```

Content immediately after.

<!-- CORRECT -->
Content before code block.

```sql
SELECT * FROM table;
```

Content after code block with proper spacing.
```

**MD032 - Lists must be surrounded by blank lines**:

```markdown
<!-- WRONG -->
Content before list.
- List item one
- List item two
Next paragraph without space.

<!-- CORRECT -->
Content before list.

- List item one
- List item two

Next paragraph with proper spacing.
```

**MD040 - Code blocks must specify language**:

```text
<!-- WRONG -->
Example query:

```
SELECT * FROM table;
```

Use this pattern for all queries.

<!-- CORRECT -->
Example query:

```sql
SELECT * FROM table;
```

Use this pattern for all queries.
```

**MD009 - No trailing spaces**:

```markdown
<!-- WRONG (trailing space after 'here') -->
Content here •
Next line.

<!-- CORRECT -->
Content here
Next line.
```

## Configuration Reference Integration

### Mandatory Configuration Usage

**Never hardcode business values** in documentation or queries. All business-specific values must reference the centralized [Configuration Reference](./docs/core-reference/configuration-reference.md).

**Examples**:

```markdown
<!-- DON'T: Hardcoded values -->
WHERE status IN ('Open', 'Working', 'Qualified')

<!-- DO: Reference configuration -->
-- Lead status values from Configuration Reference
WHERE status IN ('MQL', 'SAL', 'SQL')  
```

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

### Mandatory Patterns

**Latest Records Pattern** (always required):

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

**Deleted Record Filtering** (always required):

```sql
WHERE grax__deleted IS NULL
```

**Date Boundaries** (required for performance):

```sql
WHERE createddate_ts >= DATE_ADD('month', -24, CURRENT_DATE)
```

### Configuration Value Usage

All SQL examples must use configuration references, not hardcoded values:

```sql
-- Configuration Reference values for lead status
WHERE status IN (/* MQL, SAL, SQL from Configuration Reference */)

-- Configuration Reference values for opportunity stages  
WHERE stagename IN (/* Closed Won, Closed Lost from Configuration Reference */)
```

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

- Consistent formatting using markdown standards
- Proper code block syntax highlighting
- Clear table structures for reference data
- Appropriate heading hierarchy for navigation

### Business Value Focus

- Emphasize how GRAX enables customers to "Adapt Faster"
- Connect technical analysis to business outcomes
- Provide strategic recommendations alongside data insights
- Highlight unique value of complete historical data

## Testing and Validation

### Pre-Commit Checklist

- [ ] Document passes all markdown linting rules
- [ ] **Blank lines added around ALL headings and lists**
- [ ] **Blank lines added around ALL fenced code blocks**
- [ ] **Language specified for ALL code blocks**
- [ ] **No trailing spaces anywhere in document**
- [ ] All configuration values reference centralized source
- [ ] Code examples follow mandatory SQL patterns
- [ ] Links and cross-references work correctly
- [ ] Professional presentation standards met
- [ ] File ends with single newline character

### Integration Testing

- [ ] New documentation integrates with existing patterns
- [ ] No duplicate or conflicting guidance
- [ ] Directory structure maintained correctly
- [ ] README.md updated to reference new content

### Continuous Improvement Protocol

**When ANY linting error is reported**:

1. **Immediate Response**: Fix all errors in affected documents
1. **Prevention Update**: Add specific guidance to this CONTRIBUTING.md
1. **Example Addition**: Include correct/incorrect examples for the error type
1. **Documentation Enhancement**: Strengthen prevention guidelines
1. **Knowledge Transfer**: Update training materials and procedures

## Success Metrics

### Quality Indicators

- Zero markdown linting violations
- Consistent use of configuration references
- Professional HTML artifacts for all customer reports
- Clear, actionable business intelligence delivery
- Scalable documentation structure

### Customer Success

- Queries execute successfully with customer data
- Reports provide meaningful business insights
- Professional presentation enhances GRAX brand
- Documentation enables faster customer adaptation
- Error resolution maintains productivity

This contributing guide ensures all documentation meets enterprise-grade standards while remaining accessible and actionable for both technical and business users. Following these guidelines maintains the knowledge base's effectiveness as a comprehensive resource for GRAX Data Lake analytics.
