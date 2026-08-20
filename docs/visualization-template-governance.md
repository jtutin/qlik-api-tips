# Visualization template governance

Use governed visualization templates as stable deployment contracts. Client-specific chart properties belong in the template; deployment metadata belongs in a companion manifest.

## Standard metadata tags

Use these tags on governed master visualizations where applicable:

- `MASTER_VISUALIZATION`
- visualization type, for example `BARCHART`
- `DOMAIN:<DOMAIN>`
- `MEASURE:<MASTER_MEASURE_ID>`
- `DIMENSION:<FIELD_OR_MASTER_DIMENSION_ID>`
- `TEMPLATE:<TEMPLATE_ID>`

Example:

```text
MASTER_VISUALIZATION
BARCHART
DOMAIN:ED
MEASURE:MASTER_MEASURE_AVERAGE_WAIT_MINUTES
DIMENSION:CATEGORY
TEMPLATE:BARCHART_V1
```

These tags make persistent objects self-describing and support automated dependency discovery, impact analysis, and template-version auditing.

## Parameter standard

For `MASTER_BARCHART_V1`, require:

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

All other visual properties should remain governed by the template unless there is an explicit reason to override them.

## Post-create validation

After `CreateObject`, verify the persistent definition with `GetProperties`:

```text
qInfo.qType = masterobject
visualization = barchart
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

Call `SaveObjects` only after the verified batch.

## Versioning

A native-chart capture is required only when establishing a new visualization type or when re-validating compatibility after a material Qlik client upgrade. Increment the template ID when a change alters the governed visual contract rather than merely correcting metadata.
