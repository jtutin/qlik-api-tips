# Visualization template governance

Use governed visualization templates as stable deployment contracts. Client-specific chart properties belong in the template; deployment metadata belongs in a companion manifest.

## Standard metadata tags

Use these tags on governed master visualizations where applicable:

- `MASTER_VISUALIZATION`
- visualization type, for example `BARCHART`, `LINECHART`, or `TABLE`
- `DOMAIN:<DOMAIN>`
- `MEASURE:<MASTER_MEASURE_ID>`
- `DIMENSION:<FIELD_OR_MASTER_DIMENSION_ID>`
- `TEMPLATE:<TEMPLATE_ID>`

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

For `MASTER_TABLE_V1`, the column collection is variable. Each dimension column supplies a field, display label, and component ID. Each measure column supplies a persistent master-measure ID and component ID. Validate master-measure IDs before creating the table and keep all column-order and column-width arrays synchronized with the final column count.

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

### MASTER_TABLE_V1

Validated with both dimension-only tables and mixed dimension/master-measure tables.

Standard presentation:

```text
all columns = explicitly left aligned
header wrapping = off
cell wrapping = off
column widths = automatic (-1)
horizontal scrolling = on
main title = 18px
subtitle = 15px
border = #d9d9d9
shadow = 0px 1px 2px 0px
```

The table template deliberately keeps the dimension and measure arrays variable. Repeat the governed dimension/measure blocks as required and update `qInterColumnSortOrder`, `qColumnOrder`, `columnOrder`, and `columnWidths` to match the total number of columns.

Master measures are referenced through `qLibraryId`; their governed labels and number formats should be inherited rather than duplicated locally. A missing or incorrect persistent master-measure ID can cause that column to fail even when the table schema itself is valid, so validate all referenced IDs before `CreateObject`.

## Post-create validation

After `CreateObject`, verify the persistent definition with `GetProperties`:

```text
qInfo.qType = masterobject
visualization = expected template visualization type
qHyperCubeDef.qDimensions = expected dimensions
qHyperCubeDef.qMeasures[*].qLibraryId = expected master measures
```

When the app is opened with data, use `GetLayout` to verify runtime evaluation:

```text
qDimensionInfo populated
qMeasureInfo populated when measures are present
qDataPages populated
no hypercube error present
```

For line charts using `MASTER_LINECHART_V1`, also verify the Y-axis contract:

```text
measureAxis.autoMinMax = false
measureAxis.minMax = min
measureAxis.min = 0
```

For tables using `MASTER_TABLE_V1`, also verify:

```text
all requested columns are present
all master measure IDs resolve
measure formatting is inherited
textAlign.auto = false
textAlign.align = left
multiline.wrapTextInHeaders = false
multiline.wrapTextInCells = false
```

Call `SaveObjects` only after the verified batch.

## Versioning

A native-chart capture is required only when establishing a new visualization type or when re-validating compatibility after a material Qlik client upgrade. Increment the template ID when a change alters the governed visual contract rather than merely correcting metadata.
