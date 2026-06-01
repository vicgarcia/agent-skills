---
name: charts
description: Generate publication-quality charts (bar, line, pie, scatter, radar, funnel, gauge, treemap, boxplot, heatmap, candlestick, sankey) as SVG or PNG files using charts-cli and ECharts JSON configs. Use when the user asks to visualize data, create a chart, plot metrics, or render any statistical or business graphic.
compatibility: Requires Node.js 18+. Install once with `npm install -g charts-cli`. Outputs .svg or .png files. No browser required.
---

# charts-cli

Generate charts from ECharts JSON configurations using a single CLI command. No browser, no GUI, no dependencies beyond Node.

```bash
charts render --config option.json -o chart.png -W 900 -H 500
echo '{"series":[...]}' | charts render -o chart.svg
```

---

## Installation

```bash
npm install -g charts-cli
```

Verify installation:
```bash
charts --version
```

Explore available chart types and their full ECharts schemas:
```bash
charts schema --list        # list all supported series and component types
charts schema bar           # show full ECharts option schema for bar charts
charts schema sankey        # show schema for sankey diagrams
```

---

## Core Workflow

1. Gather data from the codebase, API, database query, CSV, or user input
2. Transform into arrays matching ECharts series format
3. Write the ECharts JSON config (use a recipe below as starting point)
4. Run `charts render` to produce SVG or PNG
5. Open or display the output file

**From file:**
```bash
charts render --config option.json -o chart.png -W 900 -H 500
```

**From stdin:**
```bash
cat option.json | charts render -o chart.png -W 900 -H 500
echo '{"xAxis":{"type":"category","data":["A","B","C"]},"yAxis":{"type":"value"},"series":[{"type":"bar","data":[10,20,15]}]}' | charts render -o chart.svg
```

---

## CLI Reference

### `charts render`

| Flag | Default | Description |
|------|---------|-------------|
| `--config <file>` | stdin | Path to ECharts JSON option file |
| `-o, --output <file>` | stdout | Output path; `.svg` or `.png` extension determines format |
| `-W, --width <n>` | 800 | Canvas width in pixels |
| `-H, --height <n>` | 400 | Canvas height in pixels |
| `--theme <name\|path>` | — | Built-in: `dark`, `vintage`; or path to a `.json` theme file |
| `--format <type>` | auto | `svg` or `png` — overrides extension detection |

### `charts schema`

| Arg/Flag | Description |
|----------|-------------|
| `--list` | Print all supported series and component types |
| `<type>` | Print the full ECharts option schema for that type (e.g. `bar`, `sankey`, `gauge`) |

### Output format rules

- `.svg` output → vector, scales losslessly, best for web/docs/embedding
- `.png` output → raster, best for reports, screenshots, or sending to humans
- No `-o` flag → SVG written to stdout (pipe into a file or another command)

---

## Design Standards

Apply these standards to every chart for consistent, professional output.

### Background

Always set at the top level:
```json
"backgroundColor": "#ffffff"
```

Required for PNG readability — PNGs without it render as transparent in most viewers. Omit only when using `--theme dark` (the theme sets its own background).

### Color palette

Six-color standard sequence. Apply in order; cycle for more than 6 series.

| Index | Name | Hex |
|-------|------|-----|
| 0 | Indigo | `#5470c6` |
| 1 | Teal | `#91cc75` |
| 2 | Amber | `#fac858` |
| 3 | Red | `#ee6666` |
| 4 | Violet | `#73c0de` |
| 5 | Cyan | `#3ba272` |

Set at the top level of every config:
```json
"color": ["#5470c6","#91cc75","#fac858","#ee6666","#73c0de","#3ba272"]
```

### Title

```json
"title": {
  "text": "Descriptive Chart Title",
  "left": "center",
  "textStyle": { "fontSize": 16, "fontWeight": "bold" }
}
```

### Tooltip

Always include. Use `"trigger": "axis"` for axis-based charts, `"trigger": "item"` for pie/funnel/gauge/treemap.

```json
"tooltip": { "trigger": "axis" }
```

### Legend

Include when there are 2+ series. Place at bottom to avoid competing with the title:
```json
"legend": { "bottom": 0 }
```

### Axis styling (bar, line, scatter)

Standard axis style for clean, minimal look:
```json
"xAxis": {
  "type": "category",
  "axisTick": { "show": false },
  "axisLine": { "lineStyle": { "color": "#e0e0e0" } }
},
"yAxis": {
  "type": "value",
  "axisTick": { "show": false },
  "axisLine": { "show": false },
  "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
}
```

### Grid padding

Always use `containLabel: true` to prevent axis labels from being clipped:
```json
"grid": { "left": "5%", "right": "5%", "bottom": "10%", "containLabel": true }
```

---

## Chart Recipes

Complete, copy-paste-ready ECharts JSON configs. Replace the sample data with real values.

---

### Bar Chart

Best for: comparing discrete categories, rankings, quantities across groups.

**Single series (vertical):**
```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#91cc75","#fac858","#ee6666","#73c0de","#3ba272"],
  "title": { "text": "Monthly Revenue", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "axis" },
  "grid": { "left": "5%", "right": "5%", "bottom": "10%", "containLabel": true },
  "xAxis": {
    "type": "category",
    "data": ["Jan","Feb","Mar","Apr","May","Jun"],
    "axisTick": { "show": false },
    "axisLine": { "lineStyle": { "color": "#e0e0e0" } }
  },
  "yAxis": {
    "type": "value",
    "axisTick": { "show": false },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "series": [{
    "type": "bar",
    "data": [120, 200, 150, 80, 70, 110],
    "barMaxWidth": "50%",
    "itemStyle": { "borderRadius": [4, 4, 0, 0] },
    "label": { "show": true, "position": "top", "fontWeight": "bold" }
  }]
}
```

**Grouped bar (multi-series):**
```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#91cc75"],
  "title": { "text": "Q1 vs Q2 Sales by Region", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "axis" },
  "legend": { "bottom": 0 },
  "grid": { "left": "5%", "right": "5%", "bottom": "15%", "containLabel": true },
  "xAxis": {
    "type": "category",
    "data": ["North","South","East","West"],
    "axisTick": { "show": false },
    "axisLine": { "lineStyle": { "color": "#e0e0e0" } }
  },
  "yAxis": {
    "type": "value",
    "axisTick": { "show": false },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "series": [
    { "name": "Q1", "type": "bar", "data": [320, 180, 240, 150], "barMaxWidth": "40%", "itemStyle": { "borderRadius": [4,4,0,0] } },
    { "name": "Q2", "type": "bar", "data": [290, 210, 270, 190], "barMaxWidth": "40%", "itemStyle": { "borderRadius": [4,4,0,0] } }
  ]
}
```

**Stacked bar:**
Add `"stack": "total"` to each series. Use to show composition + totals simultaneously.
```json
"series": [
  { "name": "Product A", "type": "bar", "stack": "total", "data": [120, 132, 101, 134], "itemStyle": { "borderRadius": [0,0,0,0] } },
  { "name": "Product B", "type": "bar", "stack": "total", "data": [220, 182, 191, 234], "itemStyle": { "borderRadius": [4,4,0,0] } }
]
```

**Horizontal bar** (for long category names):
Swap axis types. Set `"itemStyle": { "borderRadius": [0,4,4,0] }` for right-side rounding.
```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6"],
  "title": { "text": "Top Languages by Usage", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "axis" },
  "grid": { "left": "5%", "right": "10%", "containLabel": true },
  "xAxis": {
    "type": "value",
    "axisTick": { "show": false },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "yAxis": {
    "type": "category",
    "data": ["Python","JavaScript","TypeScript","Go","Rust","Java","C++"],
    "axisTick": { "show": false },
    "axisLine": { "lineStyle": { "color": "#e0e0e0" } }
  },
  "series": [{
    "type": "bar",
    "data": [45, 38, 22, 18, 12, 10, 8],
    "barMaxWidth": "50%",
    "itemStyle": { "borderRadius": [0, 4, 4, 0] },
    "label": { "show": true, "position": "right", "fontWeight": "bold" }
  }]
}
```

---

### Line Chart

Best for: trends over time, continuous data, rate-of-change, multi-series comparison.

```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#91cc75","#fac858"],
  "title": { "text": "Weekly Active Users", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "axis" },
  "legend": { "bottom": 0 },
  "grid": { "left": "5%", "right": "5%", "bottom": "15%", "containLabel": true },
  "xAxis": {
    "type": "category",
    "data": ["Mon","Tue","Wed","Thu","Fri","Sat","Sun"],
    "boundaryGap": false,
    "axisTick": { "show": false },
    "axisLine": { "lineStyle": { "color": "#e0e0e0" } }
  },
  "yAxis": {
    "type": "value",
    "axisTick": { "show": false },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "series": [
    {
      "name": "Web",
      "type": "line",
      "smooth": true,
      "data": [820, 932, 901, 934, 1290, 1330, 1320],
      "symbol": "circle",
      "symbolSize": 8,
      "lineStyle": { "width": 2 },
      "itemStyle": { "borderWidth": 2, "borderColor": "#ffffff" }
    },
    {
      "name": "Mobile",
      "type": "line",
      "smooth": true,
      "data": [420, 580, 540, 610, 890, 920, 870],
      "symbol": "circle",
      "symbolSize": 8,
      "lineStyle": { "width": 2 },
      "itemStyle": { "borderWidth": 2, "borderColor": "#ffffff" }
    }
  ]
}
```

**Area line (single series only):**
Add `"areaStyle": { "opacity": 0.15 }` to the series. Only use for a single series — stacked areas are visually noisy.

**Step line** (for discrete state changes, e.g. config versions, on/off status):
Replace `"smooth": true` with `"step": "end"` and remove `"symbol"`.

---

### Pie / Donut Chart

Best for: part-to-whole proportions. Limit to 5–7 segments; more becomes unreadable.

**Donut (recommended):**
```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#91cc75","#fac858","#ee6666","#73c0de","#3ba272"],
  "title": { "text": "Traffic Sources", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "item", "formatter": "{b}: {c} ({d}%)" },
  "legend": { "bottom": 0 },
  "series": [{
    "type": "pie",
    "radius": ["40%", "70%"],
    "center": ["50%", "55%"],
    "data": [
      { "name": "Organic",  "value": 1048 },
      { "name": "Direct",   "value": 735 },
      { "name": "Referral", "value": 580 },
      { "name": "Social",   "value": 484 },
      { "name": "Email",    "value": 300 }
    ],
    "itemStyle": { "borderColor": "#ffffff", "borderWidth": 2 },
    "label": { "formatter": "{b}\n{d}%" }
  }]
}
```

**Plain pie:** Use `"radius": "70%"` instead of `["40%", "70%"]`.

**Rose chart** (variable radius encodes magnitude — use when both count and proportion matter):
Add `"roseType": "area"` to the series.

---

### Scatter Plot

Best for: correlation between two variables, distribution shape, outlier detection, cluster analysis.

```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#91cc75"],
  "title": { "text": "Latency vs Payload Size", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "item" },
  "legend": { "bottom": 0 },
  "grid": { "left": "5%", "right": "5%", "bottom": "15%", "containLabel": true },
  "xAxis": {
    "type": "value",
    "name": "Payload (KB)",
    "nameLocation": "middle",
    "nameGap": 30,
    "axisTick": { "show": false },
    "axisLine": { "lineStyle": { "color": "#e0e0e0" } },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "yAxis": {
    "type": "value",
    "name": "Latency (ms)",
    "nameLocation": "middle",
    "nameGap": 40,
    "axisTick": { "show": false },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "series": [
    {
      "name": "GET",
      "type": "scatter",
      "symbolSize": 10,
      "data": [[12,45],[18,62],[24,58],[30,80],[45,95],[60,110],[80,140]]
    },
    {
      "name": "POST",
      "type": "scatter",
      "symbolSize": 10,
      "data": [[15,70],[22,85],[35,95],[50,120],[70,155],[90,180],[110,210]]
    }
  ]
}
```

**Variable bubble size:** Set `"symbolSize"` to a function or encode size as the third value in each point using `"encode": { "x": 0, "y": 1, "symbolSize": 2 }`.

---

### Radar Chart

Best for: multi-dimensional performance profiles, comparing entities across shared attributes (5–8 dimensions ideal).

```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#91cc75"],
  "title": { "text": "Developer Skill Profiles", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "item" },
  "legend": { "bottom": 0 },
  "radar": {
    "indicator": [
      { "name": "Backend",  "max": 100 },
      { "name": "Frontend", "max": 100 },
      { "name": "DevOps",   "max": 100 },
      { "name": "Testing",  "max": 100 },
      { "name": "Design",   "max": 100 },
      { "name": "Infra",    "max": 100 }
    ],
    "splitLine": { "lineStyle": { "color": "#e0e0e0" } },
    "splitArea": { "areaStyle": { "color": ["#fafafa","#ffffff"] } }
  },
  "series": [{
    "type": "radar",
    "data": [
      {
        "name": "Alice",
        "value": [90, 60, 75, 85, 40, 70],
        "areaStyle": { "opacity": 0.2 }
      },
      {
        "name": "Bob",
        "value": [50, 85, 65, 70, 80, 55],
        "areaStyle": { "opacity": 0.2 }
      }
    ]
  }]
}
```

The `"indicator"` array length must match each `"value"` array length exactly.

---

### Funnel Chart

Best for: conversion rates, pipeline stages, sequential drop-off analysis (sales, onboarding, signups).

```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#73c0de","#91cc75","#fac858","#ee6666"],
  "title": { "text": "Checkout Conversion Funnel", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "item", "formatter": "{b}: {c}%" },
  "legend": { "bottom": 0 },
  "series": [{
    "type": "funnel",
    "left": "15%",
    "width": "70%",
    "min": 0,
    "max": 100,
    "minSize": "10%",
    "maxSize": "100%",
    "sort": "descending",
    "gap": 2,
    "label": { "show": true, "position": "inside", "color": "#ffffff", "fontWeight": "bold" },
    "itemStyle": { "borderColor": "#ffffff", "borderWidth": 1 },
    "data": [
      { "name": "Visitors",   "value": 100 },
      { "name": "Signups",    "value": 62 },
      { "name": "Activated",  "value": 38 },
      { "name": "Paid",       "value": 18 },
      { "name": "Retained",   "value": 9 }
    ]
  }]
}
```

Values are treated as relative widths — use percentages (100 → widest, descend from there) for a clean visual. Actual counts work too but set `"min"` and `"max"` to match the range.

---

### Gauge

Best for: single KPI readout with threshold zones (utilization, score, progress toward a target).

```json
{
  "backgroundColor": "#ffffff",
  "title": { "text": "CPU Utilization", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "series": [{
    "type": "gauge",
    "startAngle": 200,
    "endAngle": -20,
    "min": 0,
    "max": 100,
    "splitNumber": 5,
    "radius": "75%",
    "axisLine": {
      "lineStyle": {
        "width": 20,
        "color": [
          [0.4, "#91cc75"],
          [0.7, "#fac858"],
          [1.0, "#ee6666"]
        ]
      }
    },
    "pointer": { "itemStyle": { "color": "auto" } },
    "axisTick": { "distance": -28, "length": 8, "lineStyle": { "color": "#ffffff", "width": 2 } },
    "splitLine": { "distance": -35, "length": 14, "lineStyle": { "color": "#ffffff", "width": 4 } },
    "axisLabel": { "color": "inherit", "distance": 40, "fontSize": 12 },
    "detail": {
      "valueAnimation": true,
      "formatter": "{value}%",
      "color": "inherit",
      "fontSize": 24,
      "fontWeight": "bold",
      "offsetCenter": [0, "30%"]
    },
    "data": [{ "value": 72, "name": "Usage" }]
  }]
}
```

The `"color"` array on `"axisLine.lineStyle"` defines threshold zones as `[fraction, color]` pairs — each fraction is where that zone ends (0.0–1.0 of the full range). Adjust thresholds to match domain semantics (green = healthy, yellow = warning, red = critical).

Use `-W 500 -H 500` for gauges — they render best square.

---

### Heatmap

Best for: density across two categorical axes, activity calendars, correlation matrices, time-of-week patterns.

```json
{
  "backgroundColor": "#ffffff",
  "title": { "text": "Commit Activity by Hour & Day", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "item", "formatter": "{c} commits" },
  "grid": { "left": "10%", "right": "15%", "bottom": "15%", "containLabel": true },
  "xAxis": {
    "type": "category",
    "data": ["12am","3am","6am","9am","12pm","3pm","6pm","9pm"],
    "splitArea": { "show": true },
    "axisTick": { "show": false }
  },
  "yAxis": {
    "type": "category",
    "data": ["Sun","Mon","Tue","Wed","Thu","Fri","Sat"],
    "splitArea": { "show": true },
    "axisTick": { "show": false }
  },
  "visualMap": {
    "min": 0,
    "max": 20,
    "calculable": true,
    "orient": "horizontal",
    "left": "center",
    "bottom": "0%",
    "inRange": { "color": ["#ebedf0","#c6e48b","#7bc96f","#239a3b","#196127"] }
  },
  "series": [{
    "type": "heatmap",
    "data": [
      [0,0,3],[1,0,1],[2,0,0],[3,0,0],[4,0,2],[5,0,4],[6,0,8],[7,0,5],
      [0,1,2],[1,1,0],[2,1,5],[3,1,12],[4,1,18],[5,1,15],[6,1,10],[7,1,6],
      [0,2,1],[1,2,0],[2,2,7],[3,2,14],[4,2,19],[5,2,16],[6,2,11],[7,2,4],
      [0,3,0],[1,3,2],[2,3,8],[3,3,13],[4,3,17],[5,3,14],[6,3,9],[7,3,3],
      [0,4,1],[1,4,1],[2,4,6],[3,4,11],[4,4,16],[5,4,13],[6,4,7],[7,4,2],
      [0,5,4],[1,5,5],[2,5,3],[3,5,2],[4,5,4],[5,5,6],[6,5,5],[7,5,3],
      [0,6,6],[1,6,8],[2,6,4],[3,6,1],[4,6,2],[5,6,3],[6,6,6],[7,6,4]
    ],
    "itemStyle": { "borderWidth": 2, "borderColor": "#ffffff" },
    "emphasis": { "itemStyle": { "shadowBlur": 10 } }
  }]
}
```

Heatmap `"data"` format: `[xIndex, yIndex, value]`. Indices reference the position in the `xAxis.data` and `yAxis.data` arrays. The `"visualMap"` component maps values to colors — adjust `"min"` and `"max"` to match your data range.

---

### Treemap

Best for: hierarchical proportional breakdowns — disk usage, codebase size by module, budget allocation, org headcount.

```json
{
  "backgroundColor": "#ffffff",
  "title": { "text": "Codebase by Module", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "item", "formatter": "{b}: {c} lines" },
  "series": [{
    "type": "treemap",
    "width": "90%",
    "height": "80%",
    "top": "60",
    "label": { "show": true, "formatter": "{b}\n{c}" },
    "upperLabel": { "show": true, "height": 22, "color": "#ffffff" },
    "breadcrumb": { "show": false },
    "data": [
      {
        "name": "Frontend",
        "value": 8200,
        "children": [
          { "name": "components", "value": 4500 },
          { "name": "pages",      "value": 2200 },
          { "name": "utils",      "value": 1500 }
        ]
      },
      {
        "name": "Backend",
        "value": 6400,
        "children": [
          { "name": "api",     "value": 3200 },
          { "name": "models",  "value": 1800 },
          { "name": "workers", "value": 1400 }
        ]
      },
      {
        "name": "Infrastructure",
        "value": 2100,
        "children": [
          { "name": "terraform", "value": 1200 },
          { "name": "k8s",       "value": 900  }
        ]
      }
    ]
  }]
}
```

Leaf node `"value"` determines tile size. Parent `"value"` should equal the sum of children's values for accurate proportional rendering.

---

### Boxplot

Best for: statistical distribution, quartile analysis, spread comparison across categories, outlier detection.

```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6"],
  "title": { "text": "Response Time Distribution by Endpoint", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": {
    "trigger": "item",
    "formatter": "Min: {c0}<br>Q1: {c1}<br>Median: {c2}<br>Q3: {c3}<br>Max: {c4}"
  },
  "grid": { "left": "10%", "right": "10%", "bottom": "10%", "containLabel": true },
  "xAxis": {
    "type": "category",
    "data": ["/api/users","/api/orders","/api/search","/api/checkout"],
    "axisTick": { "show": false },
    "axisLine": { "lineStyle": { "color": "#e0e0e0" } }
  },
  "yAxis": {
    "type": "value",
    "name": "ms",
    "axisTick": { "show": false },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "series": [{
    "type": "boxplot",
    "data": [
      [10, 28, 45, 72, 120],
      [15, 35, 55, 90, 200],
      [5,  18, 30, 45, 80 ],
      [20, 45, 70, 110, 180]
    ]
  }]
}
```

Each data row format: `[min, Q1, median, Q3, max]`. Pre-compute these statistics from raw samples before writing the config.

---

### Candlestick

Best for: OHLC financial timeseries — stock prices, crypto, any asset with open/high/low/close per period.

```json
{
  "backgroundColor": "#ffffff",
  "title": { "text": "AAPL – Daily OHLC", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": {
    "trigger": "axis",
    "formatter": "Open: {c0}<br>Close: {c1}<br>Low: {c2}<br>High: {c3}"
  },
  "grid": { "left": "10%", "right": "5%", "bottom": "10%", "containLabel": true },
  "xAxis": {
    "type": "category",
    "data": ["Jan","Feb","Mar","Apr","May","Jun"],
    "axisTick": { "show": false },
    "axisLine": { "lineStyle": { "color": "#e0e0e0" } }
  },
  "yAxis": {
    "type": "value",
    "scale": true,
    "axisTick": { "show": false },
    "axisLine": { "show": false },
    "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } }
  },
  "series": [{
    "type": "candlestick",
    "data": [
      [184.2, 188.4, 182.1, 189.5],
      [188.4, 185.2, 183.0, 190.1],
      [185.2, 191.0, 184.5, 192.3],
      [191.0, 186.8, 185.5, 191.8],
      [186.8, 193.5, 186.0, 194.2],
      [193.5, 196.1, 192.0, 197.0]
    ],
    "itemStyle": {
      "color": "#91cc75",
      "color0": "#ee6666",
      "borderColor": "#91cc75",
      "borderColor0": "#ee6666"
    }
  }]
}
```

Each data row format: `[open, close, low, high]`. Use green (`#91cc75`) for up-candles (close ≥ open) and red (`#ee6666`) for down-candles — set in `"itemStyle.color"` (up) and `"itemStyle.color0"` (down). Use `"scale": true` on `yAxis` so the axis starts near the data range, not zero.

---

### Sankey Diagram

Best for: flow volumes between named nodes — user journeys, energy flows, resource allocation, budget flows.

```json
{
  "backgroundColor": "#ffffff",
  "title": { "text": "User Journey Flow", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "item", "formatter": "{b}: {c}" },
  "series": [{
    "type": "sankey",
    "left": "5%",
    "right": "20%",
    "top": "60",
    "bottom": "20",
    "nodeWidth": 20,
    "nodeGap": 12,
    "lineStyle": { "curveness": 0.5, "opacity": 0.4 },
    "label": { "position": "right" },
    "nodes": [
      { "name": "Homepage" },
      { "name": "Search" },
      { "name": "Product" },
      { "name": "Cart" },
      { "name": "Checkout" },
      { "name": "Purchase" },
      { "name": "Bounce" }
    ],
    "links": [
      { "source": "Homepage", "target": "Search",   "value": 5000 },
      { "source": "Homepage", "target": "Product",  "value": 3000 },
      { "source": "Homepage", "target": "Bounce",   "value": 2000 },
      { "source": "Search",   "target": "Product",  "value": 4000 },
      { "source": "Search",   "target": "Bounce",   "value": 1000 },
      { "source": "Product",  "target": "Cart",     "value": 3500 },
      { "source": "Product",  "target": "Bounce",   "value": 1500 },
      { "source": "Cart",     "target": "Checkout", "value": 2800 },
      { "source": "Cart",     "target": "Bounce",   "value": 700  },
      { "source": "Checkout", "target": "Purchase", "value": 2200 },
      { "source": "Checkout", "target": "Bounce",   "value": 600  }
    ]
  }]
}
```

`"nodes"` defines the node list; `"links"` defines directed flows by node name. Every name in `"links"` must appear in `"nodes"`. The layout is automatic — ECharts computes node positions. Use `-W 1000 -H 600` for sankey diagrams with many nodes.

---

## Themes

### Built-in themes

```bash
charts render --config option.json -o chart.png --theme dark
charts render --config option.json -o chart.png --theme vintage
```

When using `--theme dark`, omit `"backgroundColor": "#ffffff"` from the config — the theme sets its own.

### Custom theme file

```bash
charts render --config option.json -o chart.png --theme ./brand-theme.json
```

A theme file is an ECharts theme JSON object containing color overrides, font settings, and component defaults. Generate one from the [ECharts theme builder](https://echarts.apache.org/en/theme-builder.html) and save as `.json`.

---

## Mixed / Combo Charts

Combine series types on the same axes by declaring multiple series with different `"type"` values. Use a second Y-axis when units differ.

**Bar + line overlay (e.g. volume bars + price line):**
```json
{
  "backgroundColor": "#ffffff",
  "color": ["#5470c6","#ee6666"],
  "title": { "text": "Revenue and Growth Rate", "left": "center", "textStyle": { "fontSize": 16, "fontWeight": "bold" } },
  "tooltip": { "trigger": "axis" },
  "legend": { "bottom": 0 },
  "grid": { "left": "5%", "right": "8%", "bottom": "15%", "containLabel": true },
  "xAxis": { "type": "category", "data": ["Q1","Q2","Q3","Q4"], "axisTick": { "show": false }, "axisLine": { "lineStyle": { "color": "#e0e0e0" } } },
  "yAxis": [
    { "type": "value", "name": "Revenue ($K)", "axisTick": { "show": false }, "axisLine": { "show": false }, "splitLine": { "lineStyle": { "type": "dashed", "color": "#f0f0f0" } } },
    { "type": "value", "name": "Growth (%)",   "axisTick": { "show": false }, "axisLine": { "show": false }, "splitLine": { "show": false } }
  ],
  "series": [
    { "name": "Revenue",  "type": "bar",  "yAxisIndex": 0, "data": [420, 580, 510, 670], "barMaxWidth": "50%", "itemStyle": { "borderRadius": [4,4,0,0] } },
    { "name": "Growth %", "type": "line", "yAxisIndex": 1, "data": [null, 38, -12, 31],  "smooth": true, "symbol": "circle", "symbolSize": 8, "lineStyle": { "width": 2 } }
  ]
}
```

Use `null` in a data array to represent a missing data point — ECharts renders a gap in the line.

---

## Data Transformation Patterns

Prepare data from real sources before writing the config.

### From key-value object
```python
data = {"Jan": 120, "Feb": 200, "Mar": 150}
categories = list(data.keys())   # ["Jan", "Feb", "Mar"]
values     = list(data.values()) # [120, 200, 150]
```

### From list of records
```python
records = [{"month": "Jan", "sales": 120}, {"month": "Feb", "sales": 200}]
categories = [r["month"] for r in records]
values     = [r["sales"]  for r in records]
```

### Multi-series from records
```python
months   = sorted(set(r["month"]   for r in records))
products = sorted(set(r["product"] for r in records))

series = [
    {
        "name": product,
        "type": "bar",
        "data": [next((r["sales"] for r in records if r["month"] == m and r["product"] == product), 0) for m in months]
    }
    for product in products
]
```

### From pandas DataFrame
```python
import json

config = {
    "backgroundColor": "#ffffff",
    "xAxis": {"type": "category", "data": df["month"].tolist()},
    "yAxis": {"type": "value"},
    "series": [{"type": "bar", "data": df["revenue"].tolist()}]
}

with open("option.json", "w") as f:
    json.dump(config, f, indent=2)
```

### Boxplot statistics from raw samples
```python
import statistics

def boxplot_row(samples):
    s = sorted(samples)
    n = len(s)
    q1 = s[n // 4]
    q2 = s[n // 2]
    q3 = s[3 * n // 4]
    return [min(s), q1, q2, q3, max(s)]
```

### Sankey nodes from links
```python
all_names = set()
for link in links:
    all_names.add(link["source"])
    all_names.add(link["target"])
nodes = [{"name": n} for n in sorted(all_names)]
```

---

## Chart Type Selection Guide

| Goal | Best chart type |
|------|----------------|
| Compare values across categories | Bar (vertical) |
| Compare many categories with long names | Bar (horizontal) |
| Show composition within totals | Stacked bar |
| Show trend over time | Line |
| Highlight single-series trend with area | Line + area |
| Show part-to-whole proportions | Pie or Donut |
| Compare profiles across multiple dimensions | Radar |
| Show correlation between two numeric variables | Scatter |
| Analyze conversion or sequential drop-off | Funnel |
| Display a KPI with threshold zones | Gauge |
| Show hierarchical proportional breakdown | Treemap |
| Analyze distribution / outliers across groups | Boxplot |
| Display OHLC financial timeseries | Candlestick |
| Show density across two categorical axes | Heatmap |
| Visualize flow volumes between nodes | Sankey |
| Overlay bar and line on shared axis | Mixed (bar + line) |

---

## Output Size Guidelines

| Use case | Recommended flags |
|----------|------------------|
| Default / general purpose | `-W 800 -H 400` |
| Wide dashboard / timeline | `-W 1200 -H 400` |
| Square: radar, pie, gauge | `-W 600 -H 600` |
| Tall: funnel, treemap | `-W 700 -H 600` |
| Gauge (single KPI) | `-W 500 -H 500` |
| Sankey with many nodes | `-W 1000 -H 600` |
| Heatmap | `-W 900 -H 500` |
| Horizontal bar (many items) | `-W 800 -H 600` |

---

## Agent Instructions

### Step 1: Gather and verify data

Identify the data source (query result, API response, log file, CSV, in-memory state). Extract values and verify they are numeric — replace nulls with 0 or omit the point. Confirm array lengths are consistent across all series and axis definitions.

### Step 2: Choose chart type

Apply the selection guide above. When in doubt: bar for comparisons, line for trends, pie for proportions, scatter for correlation.

### Step 3: Write the config

Copy the matching recipe. Replace all placeholder data with real values. Set a descriptive `"title.text"`. Apply all design standards: `"backgroundColor"`, `"color"` palette, axis styling, `"tooltip"`, `"legend"` (if multi-series), `"grid"` with `"containLabel": true`.

### Step 4: Name files descriptively

Config file: `chart-<topic>.json` (e.g. `chart-revenue.json`, `chart-user-funnel.json`)
Output file: `chart-<topic>.png` or `chart-<topic>.svg`

### Step 5: Render

```bash
charts render --config chart-<topic>.json -o chart-<topic>.png -W 900 -H 500
```

Adjust `-W` and `-H` from the size guidelines above based on chart type.

### Step 6: Confirm output

Report the output file path to the user. In a UI context, display the image inline.

### Handle missing tool

If `charts` command is not found:
```bash
npm install -g charts-cli
```
Then re-run the render command.

---

## Validation Checklist

Before running `charts render`, verify:

- [ ] `"backgroundColor": "#ffffff"` is set at the top level (omit only with `--theme dark`)
- [ ] `"color"` palette array is present with the 6 standard hex values
- [ ] `"title.text"` is descriptive, not placeholder
- [ ] All series `"data"` arrays contain only numbers — no strings, no `null` (use `0` or omit)
- [ ] `xAxis.data` length matches every series `"data"` array length for axis-based charts
- [ ] Pie / funnel data uses `{ "name": "...", "value": N }` objects, not plain numbers
- [ ] Sankey `"links"` only reference names that exist in `"nodes"`
- [ ] Boxplot data rows have exactly 5 values: `[min, Q1, median, Q3, max]`
- [ ] Candlestick data rows have exactly 4 values: `[open, close, low, high]`
- [ ] Radar `"indicator"` array length matches each `"value"` array length in the series data
- [ ] Heatmap `"data"` uses `[xIndex, yIndex, value]` format with valid indices
- [ ] `"visualMap"` is present for heatmap with correct `"min"` and `"max"` for data range
- [ ] Multi-axis combo charts have `"yAxisIndex"` set on each series
- [ ] Output `-o` path has `.png` or `.svg` extension
