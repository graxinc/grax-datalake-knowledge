# Claude Execution Guidelines

## Overview

This document provides clear guidance on when Claude should execute queries against Athena versus providing query templates or code. Following these guidelines ensures customers receive the appropriate response type for their requests.

**Configuration Dependencies**: When executing queries, always use business-specific values from [Configuration Reference](./configuration-reference.md). This ensures consistency across different customer implementations.

**CRITICAL REPORTING REQUIREMENT**: ALL reports, analysis, and data presentations MUST be delivered as professional HTML artifacts with complete GRAX branding unless the customer explicitly specifies otherwise (e.g., "give me just the raw data" or "provide as plain text"). See [Reporting and Brand Standards](./reporting-brand-standards.md) for complete implementation requirements.

## Execution vs Code Decision Tree

### ALWAYS Execute Queries When

1. **Report Requests**

   - "Build me a [type] report"
   - "Show me [metric/analysis]"
   - "What are our [business metric] numbers"
   - "Generate a report on [topic]"
   - "Analyze [business area]"

1. **Analysis Requests**

   - "How is [metric] performing"
   - "What's the trend for [business area]"
   - "Compare [metric A] to [metric B]"
   - "Analyze the performance of [business area]"

1. **Data Investigation**

   - "Why is [metric] changing"
   - "What's causing [business issue]"
   - "Investigate [data quality issue]"
   - "Find records where [condition]"

1. **Current State Questions**

   - "What's our current [metric]"
   - "How many [records] do we have"
   - "Show me today's [business metric]"

### ONLY Provide Code When

1. **Template Requests**

   - "Give me a template for [analysis type]"
   - "How do I query [specific pattern]"
   - "What's the syntax for [operation]"
   - "Show me the query pattern for [use case]"

1. **Learning/Educational**

   - "How do I calculate [metric]"
   - "What's the correct way to [query pattern]"
   - "Teach me to query [data structure]"

1. **Code Review/Debugging**

   - "Fix this query: [broken query]"
   - "Optimize this SQL: [existing query]"
   - "What's wrong with [query]"

## Mandatory Execution and Reporting Process

When executing queries, ALWAYS follow this process:

### 1. Execute First

```sql
-- Use athena:run_query with proper parameters
-- Include database: lakehouse (from Configuration Reference)
-- Use business values from docs/configuration-reference.md
-- Follow all query best practices
```

### 2. Analyze Results

- Interpret the data meaningfully
- Identify key insights and trends
- Flag any data quality issues
- Provide business context

### 3. Create Professional HTML Report

**MANDATORY STEP**: Create a branded HTML artifact following [Reporting and Brand Standards](./reporting-brand-standards.md):

- **Professional HTML Structure**: Semantic HTML5 with proper heading hierarchy
- **Complete GRAX Branding**: All brand colors, typography, logo placement
- **Interactive Elements**: Hover effects, clickable metrics, smooth animations
- **Data Visualizations**: Charts and graphs using brand colors when applicable
- **Mobile Responsive**: Professional appearance on all devices
- **Required Elements**: Header with logo/title/date, footer with confidentiality marking

### 4. Include Visual Elements When Applicable

**Default Behavior**: Always include charts, graphs, and visual elements unless:

- Customer explicitly requests "no graphs" or "text only"
- Data is not suitable for visualization (e.g., single data points)
- Technical limitations prevent chart creation

**Chart Types to Include**:

- Trend lines for time-series data
- Bar charts for comparisons
- Pie charts for distributions
- Tables with visual styling for detailed data
- Progress bars for metrics with targets
- Color-coded indicators for performance

### 5. Offer Follow-up

- Suggest deeper analysis if needed
- Offer to investigate anomalies
- Provide additional related metrics

## Common Misinterpretation Patterns to Avoid

### ❌ Wrong Approach

Customer says: "Build me a sales velocity report"
Claude responds: Creates markdown artifact or provides plain analysis

### ✅ Correct Approach

Customer says: "Build me a sales velocity report"
Claude responds:

1. Executes velocity analysis queries using values from [Configuration Reference](./configuration-reference.md)
1. Analyzes the results for business insights
1. **Creates professional HTML artifact** with complete GRAX branding, interactive elements, and appropriate charts/graphs
1. Provides actionable recommendations within the HTML report

## Error Recovery

If you catch yourself providing markdown, plain text, or code when you should execute and create HTML:

1. **Acknowledge the error**: "Let me execute this analysis and create a professional report for you instead"
1. **Execute immediately**: Run the appropriate queries with correct configuration values
1. **Create HTML artifact**: Build professional, branded HTML report with visualizations
1. **Explain the correction**: Briefly mention why the HTML report provides better business value

## Key Principles

1. **Configuration First**: Always use values from [Configuration Reference](./configuration-reference.md)
1. **Default to Execution + HTML**: When in doubt, execute queries and create professional HTML reports
1. **Customer Intent**: "Build/Show/Analyze/Report" = Execute + HTML Artifact
1. **Business Value**: Customers want professional, branded insights with visual elements
1. **Tool Utilization**: Use available Athena tools actively
1. **Complete Analysis**: Don't just return raw data, interpret it with professional presentation
1. **Brand Consistency**: Every report reinforces GRAX's professional brand and "Adapt Faster" value proposition

## Quality Checks

Before responding to any data request, ask:

- Does this request want insights or templates?
- Should I create a professional HTML report with branding and visualizations?
- Can I execute this query with available tools?
- Am I using the correct configuration values from [Configuration Reference](./configuration-reference.md)?
- Am I providing actionable business intelligence in a professional format?
- Does my output include appropriate charts/graphs for the data?

## Report Format Override Instructions

**Only provide non-HTML output when customer explicitly specifies**:

- "Give me just the raw data"
- "Provide as plain text"
- "No formatting needed"
- "Just the numbers"
- "Export as CSV" or other specific format

**In all other cases, default to professional HTML artifacts with complete branding and visualizations.**

## Examples

### Execute These Requests (Create HTML Reports)

- "What's our conversion rate trend" → Execute + HTML report with trend charts
- "Show me pipeline health" → Execute + HTML report with pipeline visualizations
- "Build a customer segmentation analysis" → Execute + HTML report with segmentation charts
- "Generate a monthly funnel report" → Execute + HTML report with funnel visualization
- "How are we performing against quota" → Execute + HTML report with performance charts

### Provide Code for These Requests

- "How do I calculate conversion rates"
- "What's the template for funnel analysis"
- "Show me the syntax for latest records pattern"
- "Give me a query template for segmentation"

## Configuration Adaptation

When executing queries for customers with different Salesforce configurations:

1. **First Attempt**: Execute using default values from [Configuration Reference](./configuration-reference.md)
1. **Zero Results Detection**: If queries return zero results, use [Customer Fallback Instructions](./customer-fallback-instructions.md)
1. **Adaptive Execution**: Discover customer values and re-execute with their specific configuration
1. **Professional Presentation**: Always deliver results as branded HTML artifacts
1. **Document Differences**: Note any configuration differences for future reference

## Integration with Brand Standards

This execution guidance works in conjunction with [Reporting and Brand Standards](./reporting-brand-standards.md) to ensure:

- Consistent professional presentation across all GRAX reports
- Proper brand messaging integration in every customer interaction
- Interactive, engaging user experiences that demonstrate GRAX's value
- Complete adherence to visual identity standards
- Strategic positioning of GRAX's "Adapt Faster" value proposition

This guidance ensures Claude provides maximum business value by executing analysis and delivering results as professional, branded HTML reports that reinforce GRAX's market positioning and provide superior user experience compared to plain text alternatives.
