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

## Claude Integration and Behavioral Standards

### Decision Tree Integration

Claude must follow the explicit decision tree patterns established in this repository

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
1. **Reference Existing Patterns**: Link to established templates rather than creating new examples
1. **Update Documentation**: When encountering new patterns, update appropriate sections
1. **Maintain Consistency**: Ensure all responses align with established knowledge base patterns

### Documentation Priority Framework

**When creating or updating content**:

- **Single Source of Truth**: Maintain authoritative information in one location
- **Reference Integration**: Link to existing patterns rather than duplicating content
- **Cross-Reference Consistency**: Ensure all related documents remain synchronized
- **Scalable Structure**: Support growth while maintaining logical organization

## Markdown Linting Requirements

### Zero-Tolerance Compliance

**EVERY file must pass ALL linting rules** as defined in `.markdownlint-cli2.yaml`. No exceptions.

### Mandatory Pre-Commit Validation

Before ANY documentation commits, you MUST:

1. **Review .markdownlint-cli2.yaml**: Examine the complete linting configuration file to understand all active rules and formatting requirements
1. **Validate Against Every Rule**: Check content line-by-line against each rule specified in `.markdownlint-cli2.yaml`
1. **Test with Linter**: If possible, run the markdownlint-cli2 tool against your content before committing
1. **Fix All Violations**: Address every linting error - partial compliance is not acceptable

### Critical Linting Configuration Reference

**All formatting and structural requirements are defined in `.markdownlint-cli2.yaml`**. This file specifies:

- File structure requirements (MD047, MD041, MD001, MD024)
- Formatting standards for emphasis, lists, and code blocks
- Content standards for spacing and HTML usage
- Heading hierarchy and link formatting rules

**Claude must reference `.markdownlint-cli2.yaml` directly** rather than memorizing or hardcoding any specific rule details, as the configuration file is the authoritative source for all linting requirements.

### Validation Process

**Before Every Commit**:

1. **Open `.markdownlint-cli2.yaml`**: Review the current linting configuration
1. **Rule-by-Rule Check**: Validate content against each active rule in the configuration
1. **Fix All Issues**: Address every violation found during validation
1. **Verify Compliance**: Ensure 100% compliance before committing

**Consequence of Non-Compliance**: Failed linting = Failed PR = Wasted effort requiring fixes and recommit.

## Exception Handling and Query Error Recovery

### Systematic Error Resolution Process

When Claude encounters query exceptions or errors:

1. **Immediate Diagnosis**: Identify the specific error type and root cause
1. **Customer Adaptation**: Use `Customer Fallback Instructions` patterns established in this repository
1. **Documentation Updates**: Update relevant documentation to prevent similar errors
1. **Pull Request Integration**: Add improvements to the active branch using the format `Documentation-Improvements-YYYY-MM-DD`

### Common Exception Categories

### Exception Response Protocol

**When any error occurs**:

1. **Acknowledge the Issue**: Confirm understanding of the specific error
1. **Implement Immediate Fix**: Provide working solution for the customer
1. **Update Documentation**: Enhance relevant sections to prevent recurrence
1. **Test Prevention**: Validate that documentation improvements prevent the error
1. **Document Learning**: Record patterns for future prevention

## Linting Error Response Protocol

### When Linting Errors Occur

**If user reports linting errors, Claude must**:

1. **Reference .markdownlint-cli2.yaml**: Review the linting configuration file to understand the specific rule violations
1. **Fix All Errors**: Address every reported error immediately in the affected documents
1. **Update Prevention Guidance**: If needed, enhance this CONTRIBUTING.md with better reference to linting configuration
1. **Validate Fixes**: Ensure all errors are resolved by checking against `.markdownlint-cli2.yaml`
1. **Document Learning**: Record patterns for future prevention

### Common Error Prevention Strategy

**Instead of memorizing specific rules**, always:

1. **Consult `.markdownlint-cli2.yaml`**: Reference the authoritative configuration file
1. **Follow Existing Examples**: Use patterns from successfully linting files in the repository
1. **Validate Before Committing**: Check content against the linting configuration
1. **Fix Systematically**: Address all violations, not just the ones that are easy to spot

## Configuration Reference Integration

### Mandatory Configuration Usage

**Never hardcode business values** in documentation or queries. All business-specific values must reference follow configuration pattern established in this repository.

### Configuration Updates

When encountering customer-specific values different from defaults:

1. **Document discoveries** in analysis comments
1. **Consider updates** to Configuration Reference if common variations
1. **Test all templates** work with new configuration values
1. **Maintain backward compatibility** with existing implementations

## Enhanced Testing and Validation Requirements

### Athena Query Testing Standards

**Before committing any SQL examples**, ensure all database and environment settings follow from patterns established in this repository

- Database parameter from Configuration Reference
- Workgroup parameter from Configuration Reference
- All queries follow query patterns and guidelines established in this repository
- Configuration values referenced from patterns established in this repository
- Query tested with customer fallback patterns from patterns established in this repository

### Integration Testing

- New documentation integrates with existing patterns
- No duplicate or conflicting guidance
- Directory structure maintained correctly
- Appropriate directory README.md updated to reference new content
- Cross-references use proper relative paths

## Branch and Pull Request Management Protocol

### Active Branch Usage

**For ongoing improvements**: Use existing active branch with current date format when available

**When to create new branch**: Only when no current active branch exists or for major structural changes

### Pull Request Integration

### Branch Naming Standards

**Documentation Updates**: Use format `Documentation-Improvements-YYYY-MM-DD`

**Feature Development**: Use descriptive names like `feature/new-analysis-pattern`

**Bug Fixes**: Use format `fix/error-description`

## Directory Structure Standards

### Logical Organization

**Scalable Directory Structure**: The repository uses a logical organization system where each major topic area has its own directory. Each directory contains a README.md file that explains the directory's purpose and provides links to all documentation within that area.

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

## Quality Assurance and Success Metrics

### Documentation Quality Standards

**Zero Defect Goals**:

- 100% markdown linting compliance per `.markdownlint-cli2.yaml`
- Zero hardcoded business values (all referenced from Configuration Reference)
- 100% cross-reference accuracy using relative paths
- Professional HTML artifact delivery rate: 100% for customer-facing reports

## Content Quality Standards

### Professional Writing

- Clear, concise business language appropriate for enterprise customers
- Actionable insights and recommendations
- Proper technical terminology
- Complete context and background where needed

### Visual Presentation

- Consistent formatting following `.markdownlint-cli2.yaml` standards
- Proper code block syntax highlighting as shown in existing repository files
- Clear table structures for reference data (see Configuration Reference)
- Appropriate heading hierarchy for navigation

## Testing and Validation

### Pre-Commit Checklist

- Document passes all markdown linting rules defined in `.markdownlint-cli2.yaml`
- All configuration values reference centralized source
- Code examples follow patterns established in this repository
- Links and cross-references work correctly
- Professional presentation standards follow patterns established in this repository
- No duplicate content - all examples reference authoritative sources
- Directory README files updated if adding new documents

### Enhanced Linting Prevention Strategy

**Reference `.markdownlint-cli2.yaml` directly** for all specific formatting requirements rather than memorizing rule details. The linting configuration file is the authoritative source for:

- File structure and content requirements
- Formatting standards for all markdown elements
- Spacing and organizational rules
- Code block and link formatting specifications

### Continuous Improvement Protocol

**When ANY linting error is reported**:

1. **Consult .markdownlint-cli2.yaml**: Review the linting configuration to understand the violated rule
1. **Fix All Errors**: Address every violation in affected documents
1. **Update Process**: Improve this CONTRIBUTING.md if better guidance on using the linting file is needed
1. **Validate Resolution**: Ensure all fixes comply with `.markdownlint-cli2.yaml` requirements
1. **Document Learning**: Update procedures to prevent similar issues

## Success Metrics

### Quality Indicators

- Zero markdown linting violations per `.markdownlint-cli2.yaml` standards
- Clear, actionable business intelligence delivery
- Scalable documentation structure with proper directory organization
- No duplicate content across repository
- Comprehensive directory README files providing clear navigation

This contributing guide ensures all documentation meets enterprise-grade standards while remaining accessible and actionable for both technical and business users. Following these guidelines maintains the knowledge base's effectiveness as a comprehensive resource for GRAX Data Lake analytics that grows and improves continuously.
