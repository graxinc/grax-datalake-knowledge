# GRAX HTML Report Template

## Overview

This document provides a complete, production-ready HTML template that Claude and other LLMs must use when creating GRAX reports. This template implements all brand standards from [Reporting and Brand Standards](./reporting-brand-standards.md) and ensures consistent, professional output.

**MANDATORY USAGE**: This template MUST be used for ALL GRAX reports unless customer explicitly requests a different format.

## Complete HTML Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GRAX [Report Type] | [Date/Period]</title>
    <link href="https://fonts.googleapis.com/css2?family=Work+Sans:wght@400;600;700&family=Inter:wght@400;500;600&display=swap" rel="stylesheet">
    <style>
        /* GRAX Brand Colors */
        :root {
            --primary-purple: #552C98;
            --secondary-blue: #5F6FE6;
            --grax-black: #222222;
            --teal-1: #57ADC6;
            --teal-2: #61CAEA;
            --light-gray: #ECEDEB;
            --alert-pink: #F75AA6;
            --warning-orange: #F7685A;
            --accent-purple: #C459E5;
            --white: #FFFFFF;
        }

        /* Reset and Base Styles */
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Inter', system-ui, -apple-system, sans-serif;
            line-height: 1.6;
            color: var(--grax-black);
            background: linear-gradient(135deg, #f8f9ff 0%, #ffffff 100%);
            min-height: 100vh;
        }

        /* Header Styles */
        .grax-header {
            background: var(--white);
            padding: 2rem 0;
            border-bottom: 3px solid var(--primary-purple);
            box-shadow: 0 2px 20px rgba(85, 44, 152, 0.1);
        }

        .header-content {
            max-width: 1200px;
            margin: 0 auto;
            padding: 0 2rem;
            display: flex;
            align-items: center;
            justify-content: space-between;
            flex-wrap: wrap;
            gap: 1rem;
        }

        .grax-logo {
            height: 50px;
            filter: drop-shadow(0 2px 8px rgba(85, 44, 152, 0.2));
        }

        .report-title {
            font-family: 'Work Sans', sans-serif;
            font-weight: 700;
            font-size: 2.5rem;
            color: var(--primary-purple);
            text-align: center;
            flex-grow: 1;
            margin: 0 2rem;
        }

        .report-date {
            font-family: 'Work Sans', sans-serif;
            font-weight: 600;
            color: var(--grax-black);
            font-size: 1.1rem;
        }

        /* Main Content Container */
        .grax-main {
            max-width: 1200px;
            margin: 0 auto;
            padding: 3rem 2rem;
        }

        /* Executive Summary */
        .executive-summary {
            background: linear-gradient(135deg, var(--primary-purple) 0%, var(--secondary-blue) 100%);
            color: var(--white);
            padding: 3rem;
            border-radius: 16px;
            margin-bottom: 3rem;
            box-shadow: 0 8px 32px rgba(85, 44, 152, 0.3);
        }

        .executive-summary h2 {
            font-family: 'Work Sans', sans-serif;
            font-weight: 700;
            font-size: 2rem;
            margin-bottom: 1.5rem;
        }

        .executive-summary p {
            font-size: 1.2rem;
            line-height: 1.8;
            opacity: 0.95;
        }

        /* Key Metrics Grid */
        .key-metrics {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 2rem;
            margin: 3rem 0;
        }

        .metric-card {
            background: var(--white);
            padding: 2rem;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(85, 44, 152, 0.1);
            border-top: 4px solid var(--primary-purple);
            transition: transform 0.3s ease, box-shadow 0.3s ease;
        }

        .metric-card:hover {
            transform: translateY(-4px);
            box-shadow: 0 8px 32px rgba(85, 44, 152, 0.2);
        }

        .metric-value {
            font-family: 'Work Sans', sans-serif;
            font-weight: 700;
            font-size: 2.5rem;
            color: var(--primary-purple);
            margin-bottom: 0.5rem;
        }

        .metric-label {
            font-weight: 600;
            color: var(--grax-black);
            font-size: 1.1rem;
            margin-bottom: 0.5rem;
        }

        .metric-change {
            font-size: 0.9rem;
            font-weight: 600;
        }

        .metric-change.positive {
            color: var(--teal-1);
        }

        .metric-change.negative {
            color: var(--warning-orange);
        }

        .metric-change.neutral {
            color: var(--grax-black);
        }

        /* Section Headers */
        h1 {
            font-family: 'Work Sans', sans-serif;
            font-weight: 700;
            font-size: 2.5rem;
            color: var(--primary-purple);
            margin: 2rem 0 1rem 0;
            text-align: center;
        }

        h2 {
            font-family: 'Work Sans', sans-serif;
            font-weight: 700;
            font-size: 2rem;
            color: var(--grax-black);
            margin: 3rem 0 1.5rem 0;
            padding-bottom: 0.5rem;
            border-bottom: 2px solid var(--light-gray);
        }

        h3 {
            font-family: 'Work Sans', sans-serif;
            font-weight: 600;
            font-size: 1.5rem;
            color: var(--grax-black);
            margin: 2rem 0 1rem 0;
        }

        /* Content Sections */
        .content-section {
            background: var(--white);
            padding: 2.5rem;
            margin: 2rem 0;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(85, 44, 152, 0.08);
        }

        /* Data Tables */
        .data-table {
            width: 100%;
            border-collapse: collapse;
            margin: 1.5rem 0;
            background: var(--white);
            border-radius: 8px;
            overflow: hidden;
            box-shadow: 0 4px 20px rgba(85, 44, 152, 0.1);
        }

        .data-table th {
            background: var(--primary-purple);
            color: var(--white);
            padding: 1rem;
            text-align: left;
            font-family: 'Work Sans', sans-serif;
            font-weight: 600;
        }

        .data-table td {
            padding: 1rem;
            border-bottom: 1px solid var(--light-gray);
            transition: background-color 0.2s ease;
        }

        .data-table tbody tr:hover {
            background: rgba(85, 44, 152, 0.05);
        }

        .data-table tbody tr:nth-child(even) {
            background: rgba(236, 237, 235, 0.3);
        }

        /* Charts and Visualizations */
        .chart-container {
            background: var(--white);
            padding: 2rem;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(85, 44, 152, 0.1);
            margin: 2rem 0;
        }

        .chart-title {
            font-family: 'Work Sans', sans-serif;
            font-weight: 600;
            font-size: 1.3rem;
            color: var(--grax-black);
            margin-bottom: 1rem;
            text-align: center;
        }

        /* Simple Bar Chart Styles */
        .bar-chart {
            display: flex;
            align-items: end;
            height: 300px;
            gap: 1rem;
            padding: 1rem 0;
            margin: 1rem 0;
        }

        .bar {
            flex: 1;
            background: linear-gradient(to top, var(--primary-purple), var(--secondary-blue));
            border-radius: 4px 4px 0 0;
            position: relative;
            min-height: 20px;
            transition: all 0.3s ease;
        }

        .bar:hover {
            filter: brightness(1.1);
        }

        .bar-label {
            position: absolute;
            bottom: -25px;
            left: 50%;
            transform: translateX(-50%);
            font-size: 0.9rem;
            font-weight: 600;
            color: var(--grax-black);
            text-align: center;
            width: 100%;
        }

        .bar-value {
            position: absolute;
            top: -25px;
            left: 50%;
            transform: translateX(-50%);
            font-size: 0.9rem;
            font-weight: 600;
            color: var(--grax-black);
            text-align: center;
            width: 100%;
        }

        /* Callout Boxes */
        .callout {
            padding: 1.5rem;
            border-radius: 8px;
            margin: 1.5rem 0;
            border-left: 4px solid;
        }

        .callout.insight {
            background: rgba(87, 173, 198, 0.1);
            border-color: var(--teal-1);
        }

        .callout.warning {
            background: rgba(247, 104, 90, 0.1);
            border-color: var(--warning-orange);
        }

        .callout.success {
            background: rgba(87, 173, 198, 0.1);
            border-color: var(--teal-1);
        }

        /* Recommendations */
        .recommendations {
            background: linear-gradient(135deg, rgba(85, 44, 152, 0.05), rgba(95, 111, 230, 0.05));
            padding: 2rem;
            border-radius: 12px;
            margin: 2rem 0;
            border: 1px solid rgba(85, 44, 152, 0.1);
        }

        .recommendations h3 {
            color: var(--primary-purple);
            margin-bottom: 1rem;
        }

        .recommendation-list {
            list-style: none;
            padding: 0;
        }

        .recommendation-list li {
            padding: 1rem;
            margin: 0.5rem 0;
            background: var(--white);
            border-radius: 8px;
            border-left: 4px solid var(--secondary-blue);
            box-shadow: 0 2px 8px rgba(85, 44, 152, 0.1);
        }

        /* Footer */
        .grax-footer {
            background: var(--grax-black);
            color: var(--white);
            padding: 1.5rem 0;
            text-align: center;
            margin-top: 4rem;
        }

        .footer-content {
            max-width: 1200px;
            margin: 0 auto;
            padding: 0 2rem;
            font-family: 'Work Sans', sans-serif;
            font-size: 0.9rem;
        }

        /* Responsive Design */
        @media (max-width: 768px) {
            .header-content {
                flex-direction: column;
                text-align: center;
            }

            .report-title {
                font-size: 2rem;
                margin: 1rem 0;
            }

            .grax-main {
                padding: 2rem 1rem;
            }

            .key-metrics {
                grid-template-columns: 1fr;
            }

            .executive-summary {
                padding: 2rem;
            }

            .content-section {
                padding: 1.5rem;
            }
        }

        @media (max-width: 480px) {
            .report-title {
                font-size: 1.5rem;
            }

            .metric-value {
                font-size: 2rem;
            }

            .data-table {
                font-size: 0.9rem;
            }

            .data-table th,
            .data-table td {
                padding: 0.5rem;
            }
        }

        /* Animation and Transitions */
        .fade-in {
            animation: fadeIn 0.8s ease-in;
        }

        @keyframes fadeIn {
            from { opacity: 0; transform: translateY(20px); }
            to { opacity: 1; transform: translateY(0); }
        }

        /* Interactive Elements */
        .interactive {
            cursor: pointer;
            transition: all 0.3s ease;
        }

        .interactive:hover {
            transform: translateY(-2px);
        }

        /* Status Indicators */
        .status-indicator {
            display: inline-block;
            width: 12px;
            height: 12px;
            border-radius: 50%;
            margin-right: 0.5rem;
        }

        .status-good { background: var(--teal-1); }
        .status-warning { background: var(--warning-orange); }
        .status-critical { background: var(--alert-pink); }
        .status-neutral { background: var(--light-gray); }
    </style>
</head>
<body>
    <header class="grax-header">
        <div class="header-content">
            <img src="https://www.grax.com/wp-content/uploads/2025/07/GRAX-Logo-K-Horizontal.png" alt="GRAX Logo" class="grax-logo">
            <h1 class="report-title">[REPORT TITLE]</h1>
            <div class="report-date">[DATE RANGE]</div>
        </div>
    </header>

    <main class="grax-main">
        <!-- Executive Summary Section -->
        <section class="executive-summary fade-in">
            <h2>Executive Summary</h2>
            <p>This analysis demonstrates how GRAX enables your organization to <strong>Adapt Faster</strong> by transforming complete data history into strategic intelligence for [business outcome].</p>
        </section>

        <!-- Key Metrics Grid -->
        <section class="key-metrics fade-in">
            <!-- Example Metric Cards - Replace with actual data -->
            <div class="metric-card interactive">
                <div class="metric-value">$450K</div>
                <div class="metric-label">Monthly Sales Velocity</div>
                <div class="metric-change positive">↗ +15% from last month</div>
            </div>
            
            <div class="metric-card interactive">
                <div class="metric-value">40.4%</div>
                <div class="metric-label">Win Rate</div>
                <div class="metric-change positive">↗ +2.1% improvement</div>
            </div>
            
            <div class="metric-card interactive">
                <div class="metric-value">270</div>
                <div class="metric-label">Avg Sales Cycle (Days)</div>
                <div class="metric-change neutral">→ Stable</div>
            </div>
        </section>

        <!-- Content Sections -->
        <section class="content-section fade-in">
            <h2>[Section Title]</h2>
            
            <!-- Example Table -->
            <table class="data-table">
                <thead>
                    <tr>
                        <th>Segment</th>
                        <th>Pipeline Value</th>
                        <th>Win Rate</th>
                        <th>Status</th>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td>Enterprise</td>
                        <td>$8.9M</td>
                        <td>40.4%</td>
                        <td><span class="status-indicator status-good"></span>Excellent</td>
                    </tr>
                    <!-- Add more rows as needed -->
                </tbody>
            </table>
        </section>

        <!-- Chart Section -->
        <section class="chart-container fade-in">
            <h3 class="chart-title">[Chart Title]</h3>
            <div class="bar-chart">
                <!-- Example bars - Replace with actual data visualization -->
                <div class="bar" style="height: 80%;">
                    <div class="bar-value">$450K</div>
                    <div class="bar-label">Q1</div>
                </div>
                <div class="bar" style="height: 100%;">
                    <div class="bar-value">$525K</div>
                    <div class="bar-label">Q2</div>
                </div>
                <!-- Add more bars as needed -->
            </div>
        </section>

        <!-- Insights and Callouts -->
        <div class="callout insight">
            <h3>🎯 Key Insight</h3>
            <p>[Strategic insight enabled by complete historical data]</p>
        </div>

        <div class="callout warning">
            <h3>⚠️ Action Required</h3>
            <p>[Critical issue or opportunity requiring attention]</p>
        </div>

        <!-- Recommendations Section -->
        <section class="recommendations fade-in">
            <h3>Strategic Recommendations</h3>
            <ul class="recommendation-list">
                <li><strong>Immediate Action:</strong> [Specific recommendation with timeline]</li>
                <li><strong>Strategic Initiative:</strong> [Long-term recommendation]</li>
                <li><strong>Performance Target:</strong> [Measurable goal with deadline]</li>
            </ul>
        </section>

        <!-- Historical Intelligence Value -->
        <section class="content-section fade-in">
            <h2>Competitive Advantage Through Historical Intelligence</h2>
            <p><strong>GRAX Unique Insights:</strong> This comprehensive analysis is only possible through complete data retention. Organizations with limited historical data cannot achieve this level of strategic intelligence.</p>
            
            <p><strong>Adapt Faster Capability:</strong> Your complete data history enables rapid response to market changes and data-driven optimization that competitors cannot match.</p>
        </section>
    </main>

    <footer class="grax-footer">
        <div class="footer-content">
            CONFIDENTIAL | [MM/DD/YYYY HH:MM] | Produced from GRAX Analytics Platform
        </div>
    </footer>

    <script>
        // Add interactive behaviors
        document.addEventListener('DOMContentLoaded', function() {
            // Animate metric cards on scroll
            const observer = new IntersectionObserver((entries) => {
                entries.forEach(entry => {
                    if (entry.isIntersecting) {
                        entry.target.style.opacity = '1';
                        entry.target.style.transform = 'translateY(0)';
                    }
                });
            });

            document.querySelectorAll('.metric-card').forEach(card => {
                observer.observe(card);
            });

            // Add click handlers for interactive elements
            document.querySelectorAll('.interactive').forEach(element => {
                element.addEventListener('click', function() {
                    // Add any interactive functionality here
                    console.log('Interactive element clicked');
                });
            });
        });
    </script>
</body>
</html>
```

## Usage Instructions for Claude

### 1. Template Customization

Replace the following placeholders with actual data:

- `[REPORT TITLE]` - Specific report name (e.g., "Sales Velocity Analysis")
- `[DATE RANGE]` - Report period (e.g., "September 2025" or "Q3 2025")
- `[Section Title]` - Actual section headers
- `[Chart Title]` - Descriptive chart titles
- `[MM/DD/YYYY HH:MM]` - Current timestamp

### 2. Data Integration

- Replace example metric cards with actual calculated values
- Update table data with real query results
- Modify chart data to reflect actual metrics
- Customize insights and recommendations based on analysis

### 3. Chart Implementation

For more complex visualizations, you can:

- Use the provided CSS bar chart for simple comparisons
- Add SVG elements for custom charts
- Include Chart.js or similar library for advanced visualizations
- Ensure all charts use GRAX brand colors

### 4. Responsive Behavior

The template includes:

- Mobile-responsive design
- Flexible grid layouts
- Scalable typography
- Touch-friendly interactive elements

### 5. Brand Compliance Checklist

Before delivering, verify:

- [ ] GRAX logo is properly displayed
- [ ] All brand colors are used correctly
- [ ] Typography follows hierarchy standards
- [ ] Header and footer elements are complete
- [ ] Interactive elements function properly
- [ ] Content includes "Adapt Faster" messaging
- [ ] Historical intelligence value is highlighted

### 6. Accessibility Features

The template includes:

- Semantic HTML structure
- Proper heading hierarchy
- Alt text for images
- High contrast color ratios
- Keyboard navigation support

## Configuration Integration

This template uses values from [Configuration Reference](./docs/core-reference/configuration-reference.md) for:

- Brand colors and styling
- Typography specifications
- Logo placement requirements
- Footer attribution standards

Organizations can customize these elements by updating the Configuration Reference document rather than modifying individual reports.

## Quality Assurance

Every report using this template should:

1. **Load quickly** with optimized images and CSS
1. **Display consistently** across different browsers and devices
1. **Maintain accessibility** standards for all users
1. **Reflect brand standards** accurately and completely
1. **Provide value** through clear insights and recommendations
1. **Enable action** with specific, implementable suggestions

This template ensures that all GRAX reports maintain professional consistency while effectively communicating the unique value of complete historical data intelligence through polished, interactive HTML presentations.
