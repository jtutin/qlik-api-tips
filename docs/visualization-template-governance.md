# Visualization template governance

Use governed visualization templates as stable deployment contracts. Client-specific chart properties belong in the template; deployment metadata belongs in a companion manifest.

## Standard metadata tags

Use these tags on governed master visualizations where applicable:

- `MASTER_VISUALIZATION`
- visualization type, for example `BARCHART` or `LINECHART`
- `DOMAIN:<DOMAIN>`
- `MEASURE:<MASTER_MEASURE_ID>`
- `DIMENSION:<FIELD_OR_MASTER_DIMENSION_ID>`
- `TEMPLATE:<TEMPLATE_ID>`

Example:

```text
MASTER_VISUALIZATION
LINECHART
DOMAIN:ED
MEASURE:MASTER_MEASURE_AVERAGE_WAIT_MINUTES
DIMENSION:EVENT_DATE
TEMPLATE:LINECHART_V1
```

These tags make persistent objects self-describing and support automated dependency discovery, impact analysis, and template-version auditing.

## Parameter standard

For governed single-dimension, single-measure bar and line charts, require:

```text
OBJECT_ID
TITLE
DESCRIPTION
DIMENSION_FIELD
DIMENSION_LABEL
MASTER_MEASURE_ID
```

Allow these optional deployment values:

```text
SUBTITLE
FOOTNOTE
DOMAIN
TAGS
LEGEND_SHOW
```

`MASTER_LINECHART_V1` additionally allows `LINE_THICKNESS` and `DATA_POINT_SIZE` when an intentional override is required. All other visual properties should remain governed by the template unless there is an explicit reason to override them.

## Current governed templates

### MASTER_BARCHART_V1

Validated against native bar-chart client version `1.39.8`.

Standard presentation:

```text
bar width = 0.8
scrollbar = none
main title = 18px
subtitle = 15px
legend = false by default
```

### MASTER_LINECHART_V1

Validated against native line-chart client version `1.47.1`.

Standard presentation:

```text
line thickness = 3
data point size = 10
scrollbar = none
null mode = gap
main title = 18px
subtitle = 15px
legend = false by default
Y-axis minimum = 0
Y-axis maximum = automatic
```

The fixed zero baseline is represented by:

```text
measureAxis.autoMinMax = false
measureAxis.minMax = min
measureAxis.min = 0
```

The stored `measureAxis.max` value is not treated as a fixed maximum when `minMax = min`.

## Post-create validation

After `CreateObject`, verify the persistent definition with `GetProperties`:

```text
qInfo.qType = masterobject
visualization = expected template visualization type
qHyperCubeDef.qDimensions[0] = expected dimension
qHyperCubeDef.qMeasures[0].qLibraryId = expected master measure
```

When the app is opened with data, use `GetLayout` to verify runtime evaluation:

```text
qDimensionInfo populated
qMeasureInfo populated
qDataPages populated
no hypercube error present
```

For line charts using `MASTER_LINECHART_V1`, also verify the Y-axis contract:

```text
measureAxis.autoMinMax = false
measureAxis.minMax = min
measureAxis.min = 0
```

Call `SaveObjects` only after the verified batch.

## Versioning

A native-chart capture is required only when establishing a new visualization type or when re-validating compatibility after a material Qlik client upgrade. Increment the template ID when a change alters the governed visual contract rather than merely correcting metadata.
