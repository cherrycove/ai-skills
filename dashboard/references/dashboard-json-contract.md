# Dashboard JSON Contract

## Contents

- [Top-Level Structure](#1-top-level-structure)
- [Groups](#2-groups)
- [Variables](#3-variables)
- [Common Query Fields](#4-common-query-fields)
- [Units](#5-units)
- [Singlestat](#6-singlestat)
- [Sequence](#7-sequence)
- [Layout](#8-layout)
- [Final Checks](#9-final-checks)

## 1. Top-Level Structure

```json
{
  "title": "Service Monitoring",
  "dashboardType": "CUSTOM",
  "dashboardExtend": {"groupUnfoldStatus": {"Overview": true}},
  "dashboardMapping": [],
  "dashboardOwnerType": "node",
  "iconSet": {"url": "", "icon": ""},
  "dashboardBindSet": [],
  "thumbnail": "",
  "tagInfo": [],
  "summary": "",
  "main": {"vars": [], "charts": [], "groups": []}
}
```

- `main.groups` must exist.
- `main` must not contain `chartGroupPos`.
- Every chart must contain `uuid`, `chartGroupUUID`, `group`, `prevGroupName`, and `pos`.

## 2. Groups

- Put `Overview` first.
- Put list-style groups such as `Instance List` or `Host List` second.
- Order the remaining groups by the normal troubleshooting path.
- Set every group to `true` in `dashboardExtend.groupUnfoldStatus`.
- Remove `dashboardExtend.groupColor`.
- Use `rgba(...)` for `main.groups[].extend.bgColor`.
- Remove `main.groups[].extend.colorKey`.

## 3. Variables

Variable codes must use real tag keys:

```json
{
  "name": "Instance Name",
  "seq": 1,
  "datasource": "dataflux",
  "code": "instance_name",
  "type": "QUERY",
  "definition": {
    "tag": "",
    "field": "",
    "value": "SHOW_TAG_VALUE(from=['service'],keyin=['instance_name'])[10m]",
    "metric": "",
    "object": ""
  },
  "valueSort": "default",
  "hide": 0,
  "isHiddenAsterisk": 0,
  "multiple": true,
  "includeStar": true,
  "extend": {}
}
```

Multi-variable validation must inspect all of `main.vars`. DQL may filter on several variables, while `BY` and `groupBy` should contain only the dimensions that belong in the chart legend or table grouping.

## 4. Common Query Fields

Every outer `queries[]` item contains:

```json
{
  "name": "",
  "type": "sequence",
  "unit": "",
  "color": "",
  "qtype": "dql",
  "query": {},
  "datasource": "dataflux"
}
```

Every `query.filters[]` item contains `id`, `op`, `name`, `type`, `logic`, and `value`. Use `=` as the default operator unless the product semantics explicitly require another operator.

## 5. Units

Every chart must contain `extend.settings.units` with one entry for every query field.

| Input unit | Configuration |
|---|---|
| no unit / `-` | `["custom", ""]` |
| `%` / `percent` | `["percent", "percent"]` |
| `B`, `KB`, `MB`, `GB` | `["digital", "B/KB/MB/GB"]` |
| `B/S`, `KB/S`, `MB/S` | `["traffic", "B/S/KB/S/MB/S"]` |
| `ms`, `s` | `["time", "ms/s"]` |
| `iops` | `["throughput", "iops"]` |
| `ops` | `["throughput", "ops"]` |
| `reqps` | `["throughput", "reqps"]` |

Use `custom` only when the unit is genuinely empty or remains unknown after first-party research. Preserve exact symbol, dimension, and case.

For `%` metrics, also verify the raw value domain for the current source. Neither the `%` symbol, the metric name, nor values below `1` establish whether the data uses a `0..1` ratio or `0..100` percentage points. Configure conversion and standard percent units only after the domain is confirmed; otherwise preserve the raw value with an `UNVERIFIED` custom unit.

## 6. Singlestat

- For a default `*` filter, reduce grouped resource series into one value with explicit business meaning.
- Use `series_sum` over stable-ID-grouped input for additive totals, direct `avg(field)` without `BY` for non-additive ratios or average latency, and `series_max` only for an explicitly labeled worst-resource card.
- Do not use `avg("M::... BY stable_id")` for a filtered-scope average; use a stable resource ID in the inner `BY` only when an outer series reducer needs per-resource input.
- Prefer an object count for resource quantity.
- Do not mechanically apply or prohibit `series_sum` across every card.
- Set `fill: null`.
- Set `funcList: []`.
- Set `fieldFunc: "last"` for outer series reductions and `fieldFunc: "avg"` for direct non-additive averages.
- Do not use rollup syntax.
- Keep `queryFuncs` aligned with the selected outer reducer; direct averages use an empty `queryFuncs`.
- Configure `valueColor`, a derived transparent `bgColor`, and `borderColor: "#E5E7EB"`.
- Rotate overview cards through a varied palette rather than assigning one color to every card.

Suggested palette: `#06B6D4`, `#10B981`, `#EAB308`, `#F97316`, `#EF4444`, `#8B5CF6`, and `#EC4899`.

## 7. Sequence

- Set `fill: "linear"`.
- Set `currentChartType: "area"`.
- Set `chartType: "areaLine"`.
- Set `funcList: []`, `queryFuncs: []`, and `groupByTime: ""`.
- Do not add unsupported or unnecessary `showLine`, `openStack`, `stackType`, `xAxisShowType`, `timeInterval`, or `legendPostion` settings.
- When one grouped query returns multiple instance series, leave the query color and `settings.colors` empty so the UI can color each returned series.

## 8. Layout

Overview cards default to `h=6`:

| Per row | `w` | `x` |
|---|---|---|
| 8 | 3 | 0,3,6,9,12,15,18,21 |
| 6 | 4 | 0,4,8,12,16,20 |
| 4 | 6 | 0,6,12,18 |
| 3 | 8 | 0,8,16 |

Trend charts default to `h=10`:

| Per row | `w` | `x` |
|---|---|---|
| 4 | 6 | 0,6,12,18 |
| 3 | 8 | 0,8,16 |
| 2 | 12 | 0,12 |
| 1 | 24 | 0 |

Tables default to `w=24` and `h=10`. Chart positions must not overlap within a group.

## 9. Final Checks

- JSON parses and contains no unknown temporary fields.
- Group order and unfolded state agree.
- Every chart has complete unit settings.
- Every query has the required common fields.
- Every variable reference resolves, filters match variables, and `groupBy` matches DQL `BY`.
- Table aliases, field mappings, and value mappings reference actual query fields.
- Every multi-resource `singlestat` reducer matches metric semantics; additive totals may use `series_sum`, while ratios and latency use direct `avg(field)` without `BY` or an outer quoted-query average.
- High-cardinality charts avoid duplicate statistic lines, node-level noise, and same-color grouped series.
- Every DQL passes `dqlcheck` individually.
