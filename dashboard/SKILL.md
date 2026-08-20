---
name: dashboard
author: liurui
description: Generate, repair, or review Guance Dashboard JSON from real metrics CSV files, tag metadata, and resource-catalog or custom-object CSV/JSON data.
---

# Guance Dashboard Generation Skill

Generate Guance Dashboard JSON from real metrics CSV files. Standard resource dashboards also use resource-catalog or custom-object data for instance-property tables. When the user provides a manually edited Dashboard JSON, treat it as feedback and carry its verified corrections into unfinished charts.

This skill must produce an operations-usable dashboard, not a thin demo. Telemetry fields must come from the user-provided metrics CSV, resource properties must come from the supplied object data, and first-party documentation may confirm semantics without inventing fields. Every final DQL must pass `dqlcheck`.

## Reference Routing

- Read [Dashboard JSON Contract](references/dashboard-json-contract.md) when generating or repairing JSON fields, units, layout, or style.
- Read [Query Semantics and Readability](references/query-semantics.md) when classifying counters, gauges, pre-aggregated fields, overview queries, or trend grouping.
- Read [Resource Object Tables](references/resource-object-tables.md) when object data is present or an instance-property table and value mappings are required.
- Read [Official Field Research](references/official-field-research.md) and consult first-party sources when units, status, mode, billing, capacity, or boolean semantics are unclear.

## Workflow

```text
Input: csv/{{type}}*.csv plus tag metadata
       resource-object CSV/JSON for a standard resource dashboard
       optional manually edited Dashboard JSON
        ->
   [Check real inputs] <- refuse generation when metrics CSV is missing
        ->               <- request object export when a standard resource table is required
   [Parse metrics, tags, and object fields]
        ->
   [Research official units and enum semantics]
        ->
   [Apply operations baseline]
        ->
     [Generate JSON]
        ->
   [Autofix style] <- required before DQL validation
        ->
   [Validate DQL] <- validate one query at a time
        ->
Output: output/dashboard/{{type}}/{{type}}.json
```

## Mandatory Rules

### Step 1: CSV File Check

Before generation, the skill must verify that at least one matching CSV file exists:

- `csv/{{type}}*.csv`
- `csv/{{type}}.csv`
- `csv/{{type}}/*.csv`

If no CSV file exists, refuse generation.

Do not:

- Invent metrics from memory or from the internet.
- Use sample metrics as a substitute for the user's CSV.
- Generate a dashboard without CSV input.
- Omit unit configuration from chart outputs.

### Step 1A: Resource Object Check

A standard resource dashboard must build its instance-property table from resource-catalog or custom-object CSV/JSON data. The object input must provide:

- the object measurement or class
- at least one real object record rather than only a field template
- queryable fields with real types and values, with top-level fields distinguished from nested `message` paths
- readable account or instance names and a stable instance identifier, or the product's equivalent fields

When object input is missing:

1. do not create or modify an instance-property table
2. do not assemble a fake resource table from metric tags
3. stop standard resource-dashboard generation and request an object export
4. continue with a telemetry-only dashboard only after the user explicitly accepts the missing resource table and document that exception

A review-only task that does not modify an existing object table may proceed without object input. Any new object field, unit, or enum mapping still requires a real object sample.

### Step 2: Parse CSV Fields

Read these fields from the CSV with either Chinese or English column names:

- `指标名` or `metric_name`
- `字段类型` or `data_type`
- `单位` or `unit`
- `操作` or `tag_key` or `Tag`

The parsed tag field is the source of truth for variable naming.

Also classify:

- raw counters, raw gauges, and cloud-side pre-aggregated values
- related `_average`, `_max`, and `_min` metric families
- additive quantities, ratios, utilization, latency, capacity, and status fields
- readable account and instance names, stable instance IDs, and node or shard dimensions

For object data, capture the class, top-level fields, nested `message` paths, sample values, field types, unit metadata, and low-cardinality enum candidates. A nested field is not a top-level `CO::` field, but a sample-backed `@json-path` may be used when the path passes `dqlcheck`; prefer an equivalent top-level field when one exists.

### Step 3: Derive Variable Code From CSV

The variable `code` must come from the actual CSV tag keys. Do not rename tags by datasource convention.

Prefer readable filters and stable identity when the actual fields exist:

1. a readable account field such as `account_name`
2. a readable instance field such as `instance_name`
3. a stable instance field such as `instance_id`
4. otherwise use the real `instanceId`, `host`, or first parsed tag without renaming it
5. fall back to `host` only when no tag exists

Example logic:

```python
def get_variable_codes(csv_row):
    tag_text = csv_row.get("操作") or csv_row.get("tag_key") or csv_row.get("Tag") or ""
    tags = [t.strip() for t in tag_text.split(",") if t.strip()]

    preferred = [
        key for key in ["account_name", "instance_name", "instance_id", "instanceId", "host"]
        if key in tags
    ]

    return preferred or tags[:1] or ["host"]
```

Consistency is mandatory:

- validate every entry in `main.vars` rather than assuming `main.vars[0]` is the only variable
- every DQL variable reference must resolve to an actual variable code
- every filter `name` and `value` must match its actual variable and `#{code}`
- `groupBy` must match the DQL `BY` display dimensions
- filter dimensions do not all need to appear in `BY`; readable names belong in legends while stable IDs may remain filters or object grouping keys

### Step 4: Operations Coverage Baseline

The dashboard must be useful for troubleshooting, not just overview display.

Minimum requirements for a complete standard resource dashboard:

- at least 1 instance-level `table` chart; for a standard resource dashboard this must query real object data
- at least 1 row of `singlestat` KPI cards, usually 4 to 8 cards
- at least 6 `sequence` trend charts

An explicitly accepted telemetry-only exception may omit the object-backed instance table, but the delivery notes must record that limitation.

If CSV coverage exists, do not silently skip important dimensions unless they are explicitly low-value and you say so.

When matching CSV files expose user, schema, table, replication, queue, tool, or operation dimensions, add the corresponding high-cardinality operational views. Drive this from the real measurements and tags instead of hard-coding one product's measurement names into the generic skill.

Do not:

- ship only overview cards
- drop user/schema/table views when the CSV supports them
- force static capacity, architecture, status, or creation-time fields into telemetry summary cards; put them in the resource-object table

## Style Autofix

Autofix is required after JSON generation and before DQL validation.

### Required Autofix Actions

1. Set every `dashboardExtend.groupUnfoldStatus` entry to `true`.
2. Put the overview group first.
3. Put list-style groups immediately after overview.
4. Remove `dashboardExtend.groupColor`.
5. Set `main.groups[].extend.bgColor` from a restrained but distinguishable operations palette.
6. Remove `main.groups[].extend.colorKey`.
7. For overview `singlestat` charts, force `extend.settings.valueColor` from a varied palette.
8. For overview `singlestat` charts, force `extend.settings.bgColor` to a transparent background derived from `valueColor`.
9. For overview `singlestat` charts, force `extend.settings.borderColor = "#E5E7EB"`.
10. For a grouped `sequence` query that returns multiple series, clear the query color and `settings.colors` so the UI palette can distinguish those series.
11. Write back the autofixed JSON and use that result for final validation.

Reference pseudocode:

```python
def to_alpha_bg(hex_color, alpha=0.12):
    h = hex_color.lstrip("#")
    r, g, b = int(h[0:2], 16), int(h[2:4], 16), int(h[4:6], 16)
    return f"rgba({r}, {g}, {b}, {alpha})"

def autofix_dashboard_style(dashboard):
    groups = [g["name"] for g in dashboard["main"]["groups"]]
    dashboard["dashboardExtend"]["groupUnfoldStatus"] = {name: True for name in groups}

    list_groups = [g for g in groups if "list" in g.lower() and g.lower() != "overview"]
    other_groups = [g for g in groups if g not in (["overview"] + list_groups)]
    ordered = (["overview"] if "overview" in groups else []) + list_groups + other_groups

    group_palette = ["#3B82F6", "#94A3B8", "#22C55E", "#F59E0B", "#06B6D4", "#EF4444", "#8B5CF6"]
    multi = ["#06B6D4", "#10B981", "#EAB308", "#F97316", "#EF4444", "#8B5CF6", "#EC4899"]

    dashboard["dashboardExtend"].pop("groupColor", None)

    order_index = {name: idx for idx, name in enumerate(ordered)}
    dashboard["main"]["groups"].sort(key=lambda group: order_index[group["name"]])
    dashboard["main"]["charts"].sort(
        key=lambda chart: order_index.get(
            chart.get("group", {}).get("name", ""), len(order_index)
        )
    )

    for idx, group in enumerate(dashboard["main"]["groups"]):
        ext = group.setdefault("extend", {})
        ext["bgColor"] = to_alpha_bg(group_palette[idx % len(group_palette)], 0.10)
        ext.pop("colorKey", None)

    card_idx = 0
    for chart in dashboard["main"]["charts"]:
        settings = chart.setdefault("extend", {}).setdefault("settings", {})
        group_name = chart.get("group", {}).get("name", "")
        if chart.get("type") == "singlestat" and group_name.lower() == "overview":
            settings["valueColor"] = multi[card_idx % len(multi)]
            settings["bgColor"] = to_alpha_bg(settings["valueColor"], 0.14)
            settings["borderColor"] = "#E5E7EB"
            card_idx += 1
        if chart.get("type") == "sequence":
            for item in chart.get("queries", []):
                if item.get("query", {}).get("groupBy"):
                    item["color"] = ""
                    settings["colors"] = []

    return dashboard
```

## Core JSON Rules

### Base Structure

The generated JSON must follow this shape:

```json
{
  "title": "MySQL Monitoring",
  "dashboardType": "CUSTOM",
  "dashboardExtend": {
    "groupUnfoldStatus": {}
  },
  "dashboardMapping": [],
  "dashboardOwnerType": "node",
  "dashboardBindSet": [],
  "thumbnail": "",
  "summary": "Description",
  "main": {
    "vars": [],
    "charts": [],
    "groups": []
  }
}
```

Do not put `chartGroupPos` inside `main`.

### Variable Definition

Use the real tag key in the variable definition and downstream queries.

Example:

```json
{
  "name": "Host",
  "code": "host",
  "type": "QUERY",
  "definition": {
    "value": "SHOW_TAG_VALUE(from=['mysql'],keyin=['host'])[10m]"
  },
  "multiple": true,
  "includeStar": true
}
```

### Groups

Groups are required under `main.groups`.

Rules:

- overview first
- list groups second when present
- trend and detail groups after that
- every group should have `extend.bgColor`

## Units

Every chart must configure units.

If the CSV or object schema does not provide a unit, research the current service's first-party API, metric, or SDK documentation. Use `["custom", ""]` and mark the result `UNVERIFIED` only when the unit still cannot be confirmed. Otherwise map it to a standard pair.

Recommended mappings:

| CSV unit | units |
|---|---|
| `-` | `["custom", ""]` |
| `percent` | `["percent", "percent"]` |
| `%` | `["percent", "percent"]` |
| `B` | `["digital", "B"]` |
| `KB` | `["digital", "KB"]` |
| `MB` | `["digital", "MB"]` |
| `GB` | `["digital", "GB"]` |
| `B/S` | `["traffic", "B/S"]` |
| `KB/S` | `["traffic", "KB/S"]` |
| `MB/S` | `["traffic", "MB/S"]` |
| `ms` | `["time", "ms"]` |
| `s` | `["time", "s"]` |
| `iops` | `["throughput", "iops"]` |
| `ops` | `["throughput", "ops"]` |
| `reqps` | `["throughput", "reqps"]` |

Only use `custom` when the unit is actually empty or remains unrecognized after research. Distinguish bytes from bits, ratios from percentages, cumulative values from rates, and instance totals from node or shard values.

For percentage-like metrics, confirm the raw value domain as well as the display symbol for the current product, API version, and collector. A `%` unit, a `percent` field name, or observed values below `1` cannot independently distinguish `0..1` ratios from `0..100` percentage points. Apply scaling and standard percent units only after first-party documentation, API examples, a verified formula, or same-time console comparison confirms the domain; otherwise preserve the raw value with an `UNVERIFIED` custom unit. Keep Dashboard and Monitor thresholds on the same confirmed raw scale.

## Chart Rules

### Singlestat Rules

Use `singlestat` for overview KPIs.

Required behavior:

- when the default `*` filter can return several resources, reduce the grouped source series into one value with explicit business meaning
- use `series_sum("M::... BY <stable_id>")` for additive quantities such as total connections, throughput, errors, or queue length
- use a direct `M::measurement:(avg(field) AS alias) { filters }` without `BY` for non-additive ratios, utilization, hit rate, load, or average latency
- do not implement a non-additive average as `avg("M::... BY <stable_id>")`; averaging a grouped intermediate is not the same query semantics as the intended filtered-scope average
- use `series_max(...)` only for an explicitly labeled worst-resource KPI
- prefer an object count for resource quantity rather than summing telemetry
- keep the inner DQL grouped by a stable resource ID only when an outer series reducer such as `series_sum` or `series_max` actually requires per-resource input
- use `fill = null`
- use `funcList = []`
- use `fieldFunc = "last"` for outer series reductions and `fieldFunc = "avg"` for direct non-additive averages
- do not use rollup syntax here
- do not mechanically force every card to use either `series_sum` or no outer reducer
- keep `queryFuncs` aligned with the selected outer reducer; direct averages use an empty `queryFuncs`
- include units
- include outer query metadata fields such as `name`, `type`, `unit`, `color`, and `qtype`

Example validation commands:

```bash
./dql/bin/dqlcheck -q "series_sum(\"M::\`service\`:(last(\`connections_average\`) AS \`Connections\`) { \`instance_id\` = '#{instance_id}' } BY \`instance_id\`\")"
./dql/bin/dqlcheck -q "M::\`service\`:(avg(\`cpu_average\`) AS \`CPU Utilization\`) { \`instance_id\` = '#{instance_id}' }"
```

### Sequence Rules

Use `sequence` for trends.

Required behavior:

- classify the metric before choosing a query; raw counters, raw gauges, and cloud-side pre-aggregated fields are not interchangeable
- raw counters should prefer rate or compatible rollup logic
- raw gauges should usually use `AVG(field)`
- cloud-side pre-aggregated `*_average` fields should use `fill(last(field), linear)` rather than a second `AVG()`
- use `fill = "linear"`
- use `currentChartType = "area"`
- use `chartType = "areaLine"`
- use `funcList = []`
- use `queryFuncs = []`
- use `groupByTime = ""`
- use `fieldFunc = "last"` for rate-like counters and cloud-side pre-aggregated fields queried with `last`
- use `fieldFunc = "avg"` only for raw gauges whose DQL actually uses `AVG(field)`
- use `=` in filters, not regex operators
- include units

Do not add extra Grafana-style rendering flags that Guance does not need.

Choose the query from the metric's aggregation semantics:

| Metric kind | Preferred trend DQL | `fieldFunc` |
|---|---|---|
| Raw counter | `rate(field)` or compatible rollup logic | `last` |
| Raw gauge | `AVG(field)` | `avg` |
| Cloud-side `*_average` | `fill(last(field), linear)` | `last` |
| Cloud-side `*_max` / `*_min` | Use `fill(last(...), linear)` only when peak or floor analysis is explicitly needed | `last` |

Do not apply a second `AVG()` to already aggregated `*_average` fields. Default to the `_average` member of an `_average/_max/_min` family, keep one metric field per `queries[]` item, prefer readable instance names in ordinary multi-instance legends, and leave grouped-query colors empty so the UI can distinguish the returned series.

### Table Rules

Use `table` charts for instance lists and high-cardinality operational views.

At least one object-backed instance-level table is mandatory for a complete standard resource dashboard. Omit it only under the explicitly accepted telemetry-only exception from Step 1A.

When the CSV supports richer dimensions, prefer:

- user-level tables
- schema-level tables
- table-level tables
- queue, tool, or operation top-N tables for runtime dashboards

For a standard resource dashboard, use a real custom-object query for the instance table:

```dql
CO::service_object:(last(`account_name`), last(`instance_name`), last(`region_id`), last(`status`)) { `account_name` = '#{account_name}' and `instance_id` = '#{instance_id}' } BY `instance_id`
```

Use `namespace: "custom_object"`, the real object measurement/class as `dataSource`, and a stable resource ID in `groupBy`. Query top-level fields found in the object sample, plus validated `@json-path` fields only when no equivalent top-level field exists. Fixed array indexes do not expand a complete child-resource list. Keep stable IDs in filters and grouping without exposing them as columns when readable names suffice. Use verified `alias`/`fieldMapping` and `valMappings`; preserve unknown raw values only for operationally important fields and omit optional opaque fields rather than guessing.

## Layout Rules

### Overview Cards

Preferred dimensions:

- height `h = 6`
- use full 24-column width

Recommended widths:

| cards per row | width |
|---|---|
| 8 | 3 |
| 6 | 4 |
| 4 | 6 |
| 3 | 8 |

If there are 6 cards in a row, use width `4` so the row spans the full dashboard width.

### Sequence Layout

Prefer stable layouts:

- 3 charts per row: `w = 8`, `h = 10`
- 4 charts per row: `w = 6`, `h = 10`
- 2 charts per row: `w = 12`, `h = 10`
- 1 chart per row: `w = 24`, `h = 10`

Default to 3 charts per row unless the group divides cleanly into 4.

### Table Layout

Tables should usually be full-width:

- `w = 24`
- `h = 10`

## DQL Validation

Every final chart DQL must pass `dqlcheck` one by one.

Validation steps:

1. extract every final DQL from dashboard charts
2. validate each query individually
3. minimally repair failed queries and retry
4. keep only validated DQL in the final dashboard
5. report the total, passed, and failed query counts in the delivery notes

Commands:

```bash
./dql/bin/dqlcheck -q '<DQL>'
./dql/bin/dqlcheck --file /tmp/query.dql
```

Pass criteria:

- all `sequence` DQL passes
- all `singlestat` DQL passes
- all `table` DQL passes

If a query still fails after repair attempts, say so explicitly and do not present it as a valid final result.

## Validation Checklist

- CSV file exists before generation.
- a standard resource dashboard has a real object class and at least one real object record.
- The dashboard JSON is structurally valid.
- `main.groups` exists.
- `main` does not contain `chartGroupPos`.
- style autofix has been applied before DQL validation.
- all groups are unfolded in `dashboardExtend.groupUnfoldStatus`.
- overview is the first group.
- list groups follow overview when present.
- variable code comes from the CSV tag field.
- every DQL variable reference resolves across all `main.vars` entries.
- every `filters.name` and `filters.value` matches an actual variable, and `groupBy` matches the DQL `BY` display dimensions.
- every chart has units configured.
- every multi-resource `singlestat` uses a reducer that matches metric semantics; additive cards may use `series_sum`, while ratios and latency use direct `avg(field)` without `BY` or an outer quoted-query average.
- every `singlestat` uses `fill = null`.
- every `sequence` uses valid fill and chart-type settings.
- a complete standard resource dashboard has at least 1 object-backed instance table; an accepted telemetry-only exception is documented instead.
- a standard resource instance table uses `CO::`, `custom_object`, the real object class, and a stable resource ID.
- internal IDs may remain in filters and `BY` without appearing in the visible table; presentation metadata contains only intentional columns.
- at least 1 overview KPI row exists.
- at least 6 trend charts exist.
- all final DQL has passed `dqlcheck`.
- delivery notes report the DQL validation totals and result.

## Related Skills

- `dql/SKILL.md`
- `unit/SKILL.md`
