---
title: Frappe Ref — Frappe Syntax: Reports: Dashboard (Impertio)
category: frappe
tags: ['frappe', 'impertio', 'syntax', 'reports', 'reference', 'dashboard']
source: Impertio
---

# Number Cards, Dashboard Charts & Dashboards

## Number Cards

Number Cards display a single aggregated value on the workspace or module page.

### Three Source Types

#### 1. Document Type Number Card

Runs an aggregate function on a DocType. Configuration:

| Field | Required | Description |
|-------|----------|-------------|
| `document_type` | Yes | DocType to aggregate |
| `function` | Yes | Count, Sum, Average, Minimum, Maximum |
| `aggregate_function_based_on` | For Sum/Avg/Min/Max | Numeric field to aggregate |
| `filters_json` | No | JSON array of filters |
| `parent_document_type` | For child tables | Parent DocType name |

Example — Count of open Sales Orders:
```json
{
    "document_type": "Sales Order",
    "function": "Count",
    "filters_json": "[['Sales Order', 'docstatus', '=', 1], ['Sales Order', 'status', '!=', 'Completed']]"
}
```

Example — Sum of outstanding amount:
```json
{
    "document_type": "Sales Invoice",
    "function": "Sum",
    "aggregate_function_based_on": "outstanding_amount",
    "filters_json": "[['Sales Invoice', 'docstatus', '=', 1]]"
}
```

#### 2. Report Number Card

Pulls a value from a specific column of a report. Configuration:

| Field | Required | Description |
|-------|----------|-------------|
| `report_name` | Yes | Name of the report |
| `report_field` | Yes | Column fieldname to extract |
| `function` | Yes | Sum, Average, etc. applied to column |
| `filters_json` | No | Report filter values |

Example — Total revenue from a report:
```json
{
    "report_name": "Sales Analytics",
    "report_field": "total_amount",
    "function": "Sum",
    "filters_json": "{\"company\": \"My Company\"}"
}
```

#### 3. Custom Method Number Card

Calls a whitelisted Python method that returns a number. Configuration:

| Field | Required | Description |
|-------|----------|-------------|
| `method` | Yes | Dotted path to whitelisted function |

Python method signature:

```python
# In myapp/api.py
import frappe

@frappe.whitelist()
def get_active_user_count(filters=None):
    """Number Card calls this method. MUST return a numeric value."""
    return frappe.db.count("User", {"enabled": 1})
```

```python
@frappe.whitelist()
def get_monthly_revenue(filters=None):
    """Accepts optional filters from the Number Card."""
    result = frappe.db.sql("""
        SELECT SUM(grand_total)
        FROM `tabSales Invoice`
        WHERE docstatus = 1
        AND MONTH(posting_date) = MONTH(CURDATE())
        AND YEAR(posting_date) = YEAR(CURDATE())
    """)
    return result[0][0] or 0
```

### Number Card Formatting

| Property | Values | Description |
|----------|--------|-------------|
| `color` | Hex color or None | Card accent color |
| `show_percentage_stats` | 0 or 1 | Show change vs previous period |
| `stats_time_interval` | Daily, Weekly, Monthly, Yearly | Comparison period |

### Number Card Permissions

- Document Type cards: user MUST have read access to the DocType
- Report cards: user MUST have access to the report
- Custom Method cards: user MUST have read access to any DocType (basic desk access)

---

## Dashboard Charts

Dashboard Charts display visual charts on workspaces and module pages.

### Three Chart Sources

#### 1. Report Chart

Uses data from a Script Report or Query Report:

| Field | Required | Description |
|-------|----------|-------------|
| `chart_type` | Yes | "Report" |
| `report_name` | Yes | Name of source report |
| `x_field` | Yes | Column for x-axis |
| `y_axis` | Yes | Column(s) for y-axis |
| `type` | Yes | Bar, Line, Pie, Donut, Percentage |
| `filters_json` | No | Report filter values |
| `is_public` | No | Visible to all users |
| `timeseries` | No | Enable time-based x-axis |
| `timespan` | If timeseries | Last Year, Last Quarter, etc. |
| `time_interval` | If timeseries | Daily, Weekly, Monthly, Quarterly, Yearly |

#### 2. Group By Chart

Auto-aggregates a DocType field:

| Field | Required | Description |
|-------|----------|-------------|
| `chart_type` | Yes | "Group By" |
| `document_type` | Yes | DocType to query |
| `group_by_type` | Yes | Count, Sum, Average |
| `group_by_based_on` | Yes | Field to group by |
| `aggregate_function_based_on` | For Sum/Avg | Numeric field |
| `number_of_groups` | No | Limit groups shown (rest = "Other") |
| `type` | Yes | Bar, Line, Pie, Donut, Percentage |
| `filters_json` | No | Filter conditions |

Example — Sales Orders by status:
```json
{
    "chart_type": "Group By",
    "document_type": "Sales Order",
    "group_by_type": "Count",
    "group_by_based_on": "status",
    "type": "Pie",
    "filters_json": "[['Sales Order', 'docstatus', '=', 1]]"
}
```

#### 3. Custom Chart Source

Uses a Python function registered in `hooks.py`:

In `hooks.py`:
```python
dashboard_chart_source = [
    "myapp.chart_sources.monthly_revenue.get_data"
]
```

Chart source function:
```python
# myapp/chart_sources/monthly_revenue.py
import frappe
from frappe.utils import getdate, add_months

@frappe.whitelist()
def get_data(chart_name=None, chart=None, no_cache=None,
             filters=None, from_date=None, to_date=None,
             timespan=None, time_interval=None, heatmap_year=None):
    """
    MUST return dict with 'labels' and 'datasets' keys.
    """
    data = frappe.db.sql("""
        SELECT
            DATE_FORMAT(posting_date, '%%Y-%%m') as month,
            SUM(grand_total) as total
        FROM `tabSales Invoice`
        WHERE docstatus = 1
        GROUP BY month
        ORDER BY month
    """, as_dict=True)

    return {
        "labels": [row.month for row in data],
        "datasets": [
            {"name": "Revenue", "values": [row.total for row in data]}
        ]
    }
```

### Chart Data Format

All Dashboard Charts consume this format:

```python
{
    "labels": ["Jan", "Feb", "Mar", "Apr"],
    "datasets": [
        {
            "name": "Dataset 1",
            "values": [100, 200, 150, 300]
        },
        {
            "name": "Dataset 2",
            "values": [80, 150, 120, 250]
        }
    ]
}
```

For Heatmap charts:
```python
{
    "labels": [],
    "dataPoints": {
        1704067200: 5,   # Unix timestamps as keys
        1704153600: 3,
        1704240000: 8
    }
}
```

### Chart Types

| Type | Best For |
|------|----------|
| `Bar` | Comparing categories or time periods |
| `Line` | Showing trends over time |
| `Pie` | Distribution of a single metric |
| `Donut` | Same as pie, with center space |
| `Percentage` | Stacked percentage comparison |
| `Heatmap` | Activity density over calendar year |

### Custom Chart Options

For advanced configuration, use `custom_options` (JSON string):

```json
{
    "colors": ["#5e64ff", "#ffa00a", "#29cd42"],
    "barOptions": {
        "stacked": 1,
        "spaceRatio": 0.5
    },
    "lineOptions": {
        "regionFill": 1,
        "dotSize": 4
    },
    "axisOptions": {
        "xIsSeries": 1,
        "shortenYAxisNumbers": 1
    },
    "tooltipOptions": {
        "formatTooltipX": "d => d",
        "formatTooltipY": "d => d + ' units'"
    }
}
```

---

## Dashboard Configuration

Dashboards group multiple charts and number cards on a single page.

### Dashboard Document Fields

| Field | Type | Description |
|-------|------|-------------|
| `module` | Link | Module this dashboard belongs to |
| `is_default` | Check | Show by default for the module |
| `charts` | Table | Dashboard Chart child table |
| `cards` | Table | Number Card child table |

### Adding Charts to Dashboard

Each row in the Dashboard Chart child table:

```json
{
    "chart": "Monthly Revenue",    // Dashboard Chart name
    "width": "Full"                // "Full" or "Half"
}
```

### Adding Number Cards to Dashboard

Each row in the Number Card child table:

```json
{
    "card": "Active Users Count"   // Number Card name
}
```

### Workspace Integration

Charts and Number Cards can also be added to Workspaces (v14+):
- Add a "Chart" or "Number Card" block to the workspace
- Select the Dashboard Chart or Number Card document
- Configure layout position and width

### Dashboard Chart Cache

- Dashboard Charts are cached using `cache_source` decorator
- Cache key: `"chart-data:{chart_name}"`
- Cache clears on chart document update
- Force refresh with `no_cache=True` parameter

---

## Critical Rules

- **ALWAYS** return a numeric value from Number Card custom methods — returning `None` shows "0" but returning a string crashes
- **ALWAYS** ensure Dashboard Chart source functions return `{"labels": [...], "datasets": [...]}` format — any other format causes blank charts
- **NEVER** create Number Cards without setting proper permissions — unauthorized users see errors instead of data
- **ALWAYS** use `@frappe.whitelist()` on custom methods for both Number Cards and Dashboard Chart sources
- **ALWAYS** match `datasets[].values` length to `labels` length in custom chart sources
- **NEVER** rely on Dashboard Chart cache for real-time data — use `no_cache=True` or set appropriate cache TTL

---
*Source: github.com/Impertio-Studio/Frappe_Claude_Skill_Package/skills/source/syntax/frappe-syntax-reports/references/dashboard.md*
