# CSV-to-Report Automation Pipeline — MVP Specification

**Version:** 1.1  
**Date:** February 9, 2026  
**Target:** 2–3 Day MVP  
**Stack:** Python 3.11+ | pandas | Jinja2 | PyYAML  
**PDF Engine:** WeasyPrint (Docker/Linux) or HTML + Print-to-PDF (Windows fallback)  
**DOCX Engine:** python-docx (optional)  
**Deployment:** CLI tool; Docker container for cross-platform PDF support

-----

## 1. Problem Statement

Analysts manually perform a repetitive four-step workflow: ingest raw CSV data, normalize columns, compute metrics, and paste results into formatted reports. This process is error-prone, time-consuming, and not scalable. The goal is a drop-in automation pipeline where a user drops a CSV and receives a finished report, with the ability to configure new report types without writing code.

-----

## 2. Architecture Overview

### Pipeline Flow

```
CSV(s) → Ingest → Normalize → Global Filter (optional) → Metrics (with local filters) → Derived Metrics → Render
                                                                                                    ↑
                                                                                          report_config.yaml
```

**Core Principle:** The YAML config is the single source of truth for each report type. It declaratively maps CSV columns → normalization rules → filters → metric functions → derived metrics → template slots. No Python editing required to create a new report.

**Key changes from v1.0:** The architecture now includes a Global Filter stage (optional, applied after normalization to subset all data before metrics), per-metric local filters (to compute metrics on subsets without a separate config), and a Derived Metrics stage (to perform safe arithmetic on computed metrics in Python rather than leaking logic into templates).

-----

## 3. Project Structure

```
report-engine/
├── cli.py                      # CLI entry point
├── config/
│   └── reports/
│       ├── weekly_ops.yaml     # Example report config
│       └── monthly_summary.yaml
├── engine/
│   ├── __init__.py
│   ├── ingest.py               # CSV loading + validation
│   ├── normalize.py            # Column normalization registry
│   ├── filters.py              # Global + local filter engine  [NEW]
│   ├── metrics.py              # Metric computation registry
│   ├── derived.py              # Derived metric expressions    [NEW]
│   └── render.py               # Template rendering + output
├── templates/
│   ├── base.html               # Shared HTML layout
│   └── weekly_ops.html         # Jinja2 HTML template
├── static/
│   └── style.css               # Report stylesheet
├── output/                     # Generated reports
├── requirements.txt
├── Dockerfile                  # WeasyPrint-safe environment  [NEW]
└── README.md
```

-----

## 4. YAML Config Schema

Each report type is defined by a single YAML file. This is the primary interface for report authors.

### 4.1 Full Schema Example

```yaml
# config/reports/weekly_ops.yaml

report:
  name: "Weekly Operations Summary"
  version: "1.0"
  template: "weekly_ops.html"
  output_formats: [html, pdf]
  output_filename: "weekly_ops_{data_date}"    # REVISED: supports {data_date}

input:
  files:
    - alias: encounters
      description: "Daily encounter data export"
      required_columns:
        - encounter_date
        - sector
        - encounter_type
        - nationality
        - count
      optional_columns:
        - notes
        - officer_id

normalize:
  - column: encounter_date
    rule: parse_date
    params:
      format: "%m/%d/%Y"

  - column: sector
    rule: uppercase_strip

  - column: encounter_type
    rule: map_values
    params:
      mapping:
        "IA": "Inadmissible Alien"
        "EWI": "Entry Without Inspection"
      default: "Unknown"

  - column: count
    rule: to_numeric
    params: { fill_value: 0 }

# NEW in v1.1: Global filter applied to all data before metrics
global_filters:
  - { column: "sector", op: "==", value: "EL PASO" }

metrics:
  - name: total_encounters
    function: sum_column
    params:
      column: count

  - name: encounters_sector_a
    function: sum_column
    params:
      column: count
      # NEW: Per-metric local filter
      filters:
        - { column: "sector", op: "==", value: "TUCSON" }

  - name: encounters_by_sector
    function: group_sum
    params:
      group_by: sector
      value_column: count

  - name: top_nationalities
    function: group_sum_top_n
    params:
      group_by: nationality
      value_column: count
      n: 10

  - name: total_officers
    function: count_unique
    params:
      column: officer_id

# NEW in v1.1: Derived metrics computed from other metrics
derived_metrics:
  - name: encounters_per_officer
    expression: "total_encounters / total_officers"
    default: 0.0                    # Safe fallback for div-by-zero
  - name: sector_a_share_pct
    expression: "(encounters_sector_a / total_encounters) * 100"
    default: 0.0

metadata:
  report_title: "Weekly Operations Summary"
  classification: "UNCLASSIFIED // FOR OFFICIAL USE ONLY"
  prepared_by: "Automated Report Engine"
```

### 4.2 Output Filename Tokens

The `output_filename` field supports the following tokens, resolved at render time:

|Token            |Resolves To                                       |Example        |
|-----------------|--------------------------------------------------|---------------|
|`{date}`         |Current system date (YYYY-MM-DD)                  |2026-02-09     |
|`{timestamp}`    |Current system datetime                           |20260209_143022|
|`{data_date}`    |Max date found in the first date column in the CSV|2026-02-07     |
|`{data_date_min}`|Min date found in the first date column in the CSV|2026-02-01     |

**Rationale (v1.1):** If a user runs a report for last week’s data today, the filename should reflect the data’s date range, not today’s date. The `{data_date}` token pulls the max date from the first date-typed column encountered in the normalized DataFrame.

-----

## 5. Module Specifications

### 5.1 `engine/ingest.py` — CSV Ingestion

**Responsibilities:** Load one or more CSVs, validate against the config’s `required_columns`, return a dict of named DataFrames.

```python
def ingest(config: dict, input_dir: str) -> dict[str, pd.DataFrame]:
    """
    Load CSVs based on config['input']['files'].
    Returns: Dict mapping alias → DataFrame
    Raises: IngestError on missing columns, file not found, unparseable CSV
    """
```

**MVP Behavior:**

- Accept a directory (auto-match files by alias) or explicit `--input alias=path` CLI args.
- Validate required columns exist; warn on missing optional columns.
- Support CSV and TSV (auto-detect delimiter via `csv.Sniffer` or explicit config).
- Return raw DataFrames; normalization happens in the next stage.

-----

### 5.2 `engine/normalize.py` — Normalization Registry

**Responsibilities:** Apply column-level transformations declaratively. New rules registered as simple decorated functions.

```python
NORMALIZERS: dict[str, Callable] = {}

def register(name: str):
    def wrapper(fn):
        NORMALIZERS[name] = fn
        return fn
    return wrapper

@register("parse_date")
def parse_date(series, format=None):
    return pd.to_datetime(series, format=format)

@register("to_numeric")
def to_numeric(series, fill_value=0):
    return pd.to_numeric(series, errors='coerce').fillna(fill_value)

def apply_normalizations(df, rules):
    df = df.copy()
    for rule in rules:
        fn = NORMALIZERS[rule['rule']]
        df[rule['column']] = fn(df[rule['column']], **rule.get('params', {}))
    return df
```

**MVP Normalizers shipped with the engine:**

|Name              |Description                               |
|------------------|------------------------------------------|
|`parse_date`      |String → datetime with configurable format|
|`to_numeric`      |Coerce to number, fill NaN                |
|`uppercase_strip` |Upper-case + trim whitespace              |
|`lowercase_strip` |Lower-case + trim whitespace              |
|`map_values`      |Categorical value remapping via dict      |
|`strip_whitespace`|Trim only                                 |
|`fill_missing`    |Fill NaN with a static value              |

**Extensibility:** Users add new normalizers by writing a decorated function in `normalize.py` or in a `custom_normalizers.py` file that gets auto-imported.

-----

### 5.3 `engine/filters.py` — Filter Engine [NEW in v1.1]

**Responsibilities:** Apply global and per-metric row-level filters. This addresses the v1.0 gap where there was no way to subset data before or during metric computation.

```python
OPERATORS = {
    '==': lambda s, v: s == v,
    '!=': lambda s, v: s != v,
    '>':  lambda s, v: s > v,
    '>=': lambda s, v: s >= v,
    '<':  lambda s, v: s < v,
    '<=': lambda s, v: s <= v,
    'in': lambda s, v: s.isin(v),
    'not_in': lambda s, v: ~s.isin(v),
}

def apply_filters(df: pd.DataFrame, filters: list[dict]) -> pd.DataFrame:
    for f in filters:
        op_fn = OPERATORS[f['op']]
        df = df[op_fn(df[f['column']], f['value'])]
    return df

def apply_global_filters(df, config):
    gf = config.get('global_filters', [])
    if gf:
        return apply_filters(df, gf)
    return df
```

**Supported filter operators:**

|Operator       |Example                                                      |Description        |
|---------------|-------------------------------------------------------------|-------------------|
|`==`           |`{ column: "sector", op: "==", value: "EL PASO" }`           |Exact match        |
|`!=`           |`{ column: "sector", op: "!=", value: "OTHER" }`             |Not equal          |
|`>`            |`{ column: "count", op: ">", value: 100 }`                   |Greater than       |
|`>=`, `<=`, `<`|(analogous)                                                  |Numeric comparisons|
|`in`           |`{ column: "sector", op: "in", value: ["EL PASO","TUCSON"] }`|Value in list      |
|`not_in`       |`{ column: "type", op: "not_in", value: ["OTHER"] }`         |Value not in list  |

-----

### 5.4 `engine/metrics.py` — Metrics Registry

**Responsibilities:** Compute named metrics from normalized, filtered DataFrames. Each metric function returns a value injected into the template context. Per-metric local filters are applied before computation.

```python
def compute_metrics(df, metric_configs):
    results = {}
    for mc in metric_configs:
        fn = METRICS[mc['function']]
        params = mc.get('params', {}).copy()

        # Extract and apply local filters if present
        local_filters = params.pop('filters', None)
        df_target = apply_filters(df, local_filters) if local_filters else df

        results[mc['name']] = fn(df_target, **params)
    return results
```

**MVP Metric Functions:**

|Name               |Returns|Description                              |
|-------------------|-------|-----------------------------------------|
|`sum_column`       |`float`|Sum of a numeric column                  |
|`mean_column`      |`float`|Mean of a numeric column                 |
|`count_rows`       |`int`  |Total row count                          |
|`count_unique`     |`int`  |Distinct values in a column              |
|`group_sum`        |`dict` |Grouped sum (for tables/charts)          |
|`group_sum_top_n`  |`dict` |Top N grouped sums                       |
|`group_count`      |`dict` |Grouped row counts                       |
|`period_comparison`|`dict` |Current vs. previous period with % change|
|`percentile`       |`float`|Nth percentile of a column               |

-----

### 5.5 `engine/derived.py` — Derived Metrics [NEW in v1.1]

**Responsibilities:** Compute metrics that are arithmetic expressions of other already-computed metrics. This prevents logic from leaking into Jinja2 templates and handles edge cases like division by zero safely in Python.

```python
import ast
import operator

SAFE_OPS = {
    ast.Add: operator.add, ast.Sub: operator.sub,
    ast.Mult: operator.mul, ast.Div: operator.truediv,
}

def safe_eval(expression: str, variables: dict) -> float:
    """
    Evaluate a simple arithmetic expression using only +, -, *, /
    and named variables. No exec/eval; uses AST parsing.
    """
    tree = ast.parse(expression, mode='eval')
    return _eval_node(tree.body, variables)

def compute_derived(metrics: dict, derived_configs: list[dict]) -> dict:
    for dc in derived_configs:
        try:
            value = safe_eval(dc['expression'], metrics)
            if not isinstance(value, (int, float)) or value != value:  # NaN check
                value = dc.get('default', 0.0)
        except (ZeroDivisionError, KeyError, TypeError):
            value = dc.get('default', 0.0)
        metrics[dc['name']] = round(value, 4)
    return metrics
```

**Security note:** The `safe_eval` function uses Python’s `ast` module to parse expressions into an AST and only evaluates addition, subtraction, multiplication, and division. It does not use `eval()` or `exec()`. Only numeric literals and variable names present in the metrics dict are resolved.

-----

### 5.6 `engine/render.py` — Template Rendering + Output

**Responsibilities:** Inject computed metrics + metadata into a Jinja2 template, resolve filename tokens, produce output in the requested format(s).

```python
def render_report(config, metrics, df, template_dir, output_dir):
    env = Environment(loader=FileSystemLoader(template_dir))

    # Custom Jinja2 filters
    env.filters['format_number'] = lambda v: f'{v:,.0f}'
    env.filters['format_pct']    = lambda v: f'{v:+.1f}%'
    env.filters['format_date']   = lambda v: v.strftime('%B %d, %Y')

    context = {
        'metrics': metrics,
        'meta': config.get('metadata', {}),
        'generated_at': datetime.now().isoformat(),
        'report_name': config['report']['name'],
    }

    # Resolve filename tokens including {data_date}  [REVISED v1.1]
    base_name = resolve_filename(config, df)

    template = env.get_template(config['report']['template'])
    html_content = template.render(**context)

    for fmt in config['report']['output_formats']:
        if fmt == 'html':  write_html(html_content, output_dir, base_name)
        elif fmt == 'pdf': write_pdf(html_content, output_dir, base_name)
        elif fmt == 'docx': write_docx(config, metrics, context, ...)
```

-----

### 5.7 `cli.py` — Command-Line Interface

```
Usage:
  python cli.py --config config/reports/weekly_ops.yaml \
                --input encounters=data/encounters_2026-02-03.csv \
                --output output/

Flags:
  --config      Path to report YAML config (required)
  --input       Explicit alias=filepath mappings (repeatable)
  --data-dir    Directory to auto-match CSVs by alias name
  --output      Output directory (default: ./output)
  --formats     Override output formats (e.g. --formats pdf,html)
  --dry-run     Validate config + data without generating output
  --validate    Check config schema + column matches only  [NEW]
  --verbose     Print normalization/metric debug info
```

**Pipeline Execution Order:**

```
1.  Parse CLI args
2.  Load + validate YAML config (schema check)
3.  Ingest CSV(s) → dict of DataFrames
4.  Apply normalization rules → cleaned DataFrames
5.  Apply global filters (if defined)             [NEW]
6.  Compute metrics (with per-metric filters)     [REVISED]
7.  Compute derived metrics                       [NEW]
8.  Render template with metrics + metadata
9.  Write output file(s)
10. Print summary (files generated, row counts, warnings)
```

-----

## 6. PDF Engine Strategy [NEW in v1.1]

**Risk identified in review:** WeasyPrint depends on GTK+ libraries that are difficult to install on locked-down Windows machines without admin rights. This is a Day 2 blocker if not planned for.

**Tiered approach:**

|Plan           |Environment        |Approach                                               |Effort|
|---------------|-------------------|-------------------------------------------------------|------|
|A (preferred)  |Docker / Linux     |WeasyPrint in container. Dockerfile included in repo.  |Low   |
|B (MVP safe)   |Windows / no Docker|Generate HTML only. Users “Print to PDF” from Chrome.  |Zero  |
|C (fallback)   |Any                |wkhtmltopdf binary wrapper (statically linked, no GTK).|Low   |
|D (last resort)|Any                |fpdf2 or reportlab. Harder to style but pure Python.   |Medium|

**Recommendation:** Ship HTML as the guaranteed output format on Day 2. If WeasyPrint installs cleanly in <1 hour, enable PDF. If it fights you, abandon it and note PDF-via-Docker as a post-MVP item. Include a Dockerfile in the repo from Day 1 so the Docker path is always available.

**Minimal Dockerfile for WeasyPrint:**

```dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y \
    libpango-1.0-0 libpangocairo-1.0-0 libgdk-pixbuf2.0-0 \
    libffi-dev libcairo2 && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . /app
WORKDIR /app
ENTRYPOINT ["python", "cli.py"]
```

-----

## 7. Jinja2 Template Example

Templates use only values from the metrics dict and metadata. No raw data manipulation occurs in templates. Derived metrics are pre-computed in Python, so templates remain clean.

```html
{# templates/weekly_ops.html #}
<!DOCTYPE html>
<html><head><style>
  body { font-family: Arial; margin: 40px; color: #1a1a1a; }
  .banner { background: #003366; color: white; padding: 12px;
            text-align: center; font-size: 11px; }
  h1 { color: #003366; border-bottom: 2px solid #003366; }
  .metric-card { display: inline-block; background: #f5f7fa;
    border-left: 4px solid #003366; padding: 16px 24px; margin: 8px; }
  .metric-value { font-size: 28px; font-weight: bold; color: #003366; }
  .metric-label { font-size: 12px; color: #666; text-transform: uppercase; }
</style></head><body>

  <div class="banner">{{ meta.classification }}</div>
  <h1>{{ report_name }}</h1>

  {# Key Metric Cards - note encounters_per_officer is pre-computed #}
  <div class="metric-card">
    <div class="metric-value">{{ metrics.total_encounters | format_number }}</div>
    <div class="metric-label">Total Encounters</div>
  </div>
  <div class="metric-card">
    <div class="metric-value">{{ metrics.encounters_per_officer | format_number }}</div>
    <div class="metric-label">Per Officer</div>
  </div>

  {# Table: Encounters by Sector #}
  <table>
    {% for sector, count in metrics.encounters_by_sector.items() %}
      <tr><td>{{ sector }}</td><td>{{ count | format_number }}</td></tr>
    {% endfor %}
  </table>
</body></html>
```

-----

## 8. How Users Create a New Report (No Code Required)

1. **Create a new YAML config** in `config/reports/`. Copy an existing config and modify the input columns, normalization rules, metrics, and metadata.
1. **Create a matching Jinja2 template** in `templates/`. Copy an existing template and adjust the HTML layout, referencing metric names from the YAML.
1. **Run the CLI:** `python cli.py --config config/reports/new_report.yaml --input alias=data.csv`
1. **(Optional) Validate first:** `python cli.py --validate --config config/reports/new_report.yaml --input alias=data.csv`

No Python changes required. The `--validate` flag (new in v1.1) checks that the template exists, all YAML-referenced columns exist in the CSV, all normalizer/metric function names are valid, and all derived metric expressions reference existing metric names.

-----

## 9. Error Handling Strategy

|Error                                   |Behavior                                                                             |
|----------------------------------------|-------------------------------------------------------------------------------------|
|Missing required column in CSV          |Halt with clear message listing missing columns and the config file that expects them|
|Unknown normalizer name in YAML         |Halt with message + list of registered normalizers                                   |
|Unknown metric function in YAML         |Halt with message + list of registered metric functions                              |
|Invalid filter operator                 |Halt with message + list of supported operators (==, !=, >, <, >=, <=, in, not_in)   |
|Derived metric references unknown metric|Halt with message listing which metric name was not found                            |
|Derived metric division by zero         |Use default value from config; warn to stderr                                        |
|Template not found                      |Halt with message + list of available templates in templates/                        |
|NaN/null in metric computation          |Warn to stderr, substitute 0 or “N/A”, continue                                      |
|YAML syntax error                       |Halt with line number + parse error                                                  |
|Empty CSV / zero rows after filtering   |Warn, generate report with “No data” placeholders                                    |
|WeasyPrint failure (missing GTK)        |Fall back to HTML output, warn user, suggest Docker                                  |

All errors include the config file path, the pipeline stage that failed (ingest/normalize/filter/metrics/derived/render), and a suggested fix.

-----

## 10. Dependencies

```
# requirements.txt
pandas>=2.0
pyyaml>=6.0
jinja2>=3.1
weasyprint>=60.0       # HTML to PDF (Linux/Docker only)
python-docx>=1.0       # DOCX output (optional)
```

No paid software. No database. No web server. Pure Python CLI.

-----

## 11. MVP Milestones (Revised)

**Revision note:** Milestone plan adjusted per review feedback. Key changes: more time for normalization testing on Day 1, WeasyPrint gets a 1-hour time-box on Day 2, and config validation is now a Day 3 deliverable.

### Day 1: Ingest + Normalize + Filters

|Block|Deliverable                                            |Notes                                                                                          |
|-----|-------------------------------------------------------|-----------------------------------------------------------------------------------------------|
|AM   |Project scaffolding, `ingest.py`, YAML schema finalized|Validate column presence; support CSV/TSV auto-detect                                          |
|PM   |`normalize.py` with 7 normalizers + unit tests         |Test `parse_date` with malformed dates, `to_numeric` with mixed types. Do not skip these tests.|
|PM   |`filters.py` with all 8 operators + unit tests         |Test edge cases: empty result set, NaN comparisons                                             |

### Day 2: Metrics + Derived + Render

|Block          |Deliverable                                                        |Notes                                                                      |
|---------------|-------------------------------------------------------------------|---------------------------------------------------------------------------|
|AM             |`metrics.py` with 9 metric functions, `derived.py` with `safe_eval`|Unit test derived metrics: div-by-zero, missing metric ref, NaN propagation|
|PM (first hour)|WeasyPrint time-box: attempt PDF install                           |If it works in <1 hour, enable PDF. If not, ship HTML only and move on.    |
|PM             |`render.py` with HTML output, base template, one working report    |End-to-end: CSV in, HTML report out                                        |

### Day 3: Polish + Handoff

|Block|Deliverable                                                     |Notes                                                                  |
|-----|----------------------------------------------------------------|-----------------------------------------------------------------------|
|AM   |`cli.py` wired end-to-end, `--validate` flag, error handling    |Config validation: template exists, columns match, function names valid|
|AM   |Second report config (proves extensibility without code changes)|Different CSV schema, different metrics, different template            |
|PM   |Dockerfile, README, handoff documentation, team walkthrough     |Include example data + example report in repo for onboarding           |

-----

## 12. Post-MVP Roadmap

Explicitly out of scope for the MVP but documented for planning:

- **Web UI wrapper** — Flask/FastAPI front-end for CSV upload + report download.
- **Scheduled execution** — Cron job or Windows Task Scheduler integration.
- **Multi-CSV joins** — Config syntax for joining multiple input files before metric computation.
- **Chart generation** — Matplotlib/Plotly charts rendered as embedded images.
- **Custom filter plugins** — Auto-load user-defined normalizers/metrics from a `plugins/` directory.
- **Audit log** — JSON log per run (input hash, config hash, output hash, timestamp) for reproducibility.
- **CSV/Excel output** — Export computed metrics as structured CSV or XLSX for further analysis.

-----

## 13. Change Log: v1.0 → v1.1

|Area             |v1.0                                 |v1.1 (This Document)                                                                   |
|-----------------|-------------------------------------|---------------------------------------------------------------------------------------|
|Pipeline stages  |Ingest → Normalize → Metrics → Render|Ingest → Normalize → Global Filter → Metrics (local filters) → Derived Metrics → Render|
|Filtering        |Not supported                        |Global filters (YAML) + per-metric local filters                                       |
|Derived metrics  |Not supported; math in Jinja         |Dedicated stage with `safe_eval`, default values, div-by-zero handling                 |
|Filename tokens  |`{date}`, `{timestamp}` only         |Added `{data_date}`, `{data_date_min}` for data-driven filenames                       |
|PDF strategy     |WeasyPrint assumed available         |Tiered approach: WeasyPrint (Docker), HTML fallback (Windows), wkhtmltopdf (alt)       |
|Config validation|Not included                         |`--validate` CLI flag checks schema, columns, functions, templates                     |
|Milestones       |3-day plan                           |Revised with WeasyPrint 1-hour time-box, mandatory normalize tests, validation on Day 3|
|Project structure|5 engine modules                     |7 engine modules (added `filters.py`, `derived.py`) + Dockerfile                       |