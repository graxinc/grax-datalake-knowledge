# Claude Execution Guidelines

## Overview

This document provides clear guidance on when Claude should execute queries against Athena versus providing query templates or code. Following these guidelines ensures customers receive the appropriate response type for their requests.

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

## Mandatory Execution Process

When executing queries, ALWAYS follow this process:

### 1. Execute First

```sql
-- Use athena:run_query with proper parameters
-- Include database: lakehouse
-- Follow all query best practices
```

### 2. Analyze Results

- Interpret the data meaningfully
- Identify key insights and trends
- Flag any data quality issues
- Provide business context

### 3. Present Findings

- Lead with executive summary
- Use clear, non-technical language
- Include actionable recommendations
- Provide context for the numbers

### 4. Offer Follow-up

- Suggest deeper analysis if needed
- Offer to investigate anomalies
- Provide additional related metrics

## Common Misinterpretation Patterns to Avoid

### ❌ Wrong Approach

Customer says: "Build me a sales velocity report"
Claude responds: Creates code artifact with SQL query

### ✅ Correct Approach

Customer says: "Build me a sales velocity report"
Claude responds:

1. Executes velocity analysis queries
1. Analyzes the results
1. Presents business insights
1. Provides recommendations

## Error Recovery

If you catch yourself providing code when you should execute:

1. **Acknowledge the error**: "Let me execute this analysis for you instead"
1. **Execute immediately**: Run the appropriate queries
1. **Provide the analysis**: Interpret and present results
1. **Explain the correction**: Briefly mention why execution was better

## Key Principles

1. **Default to Execution**: When in doubt, execute rather than provide code
1. **Customer Intent**: "Build/Show/Analyze/Report" = Execute
1. **Business Value**: Customers want insights, not code
1. **Tool Utilization**: Use available Athena tools actively
1. **Complete Analysis**: Don't just return raw data, interpret it

## Quality Checks

Before responding to any data request, ask:

- Does this request want insights or templates
- Would a business user find code or analysis more valuable
- Can I execute this query with available tools
- Am I providing actionable business intelligence

## Examples

### Execute These Requests

- "What's our conversion rate trend"
- "Show me pipeline health"
- "Build a customer segmentation analysis"
- "Generate a monthly funnel report"
- "How are we performing against quota"

### Provide Code for These Requests

- "How do I calculate conversion rates"
- "What's the template for funnel analysis"
- "Show me the syntax for latest records pattern"
- "Give me a query template for segmentation"

This guidance ensures Claude provides maximum business value by executing analysis rather than requiring customers to run queries themselves.
