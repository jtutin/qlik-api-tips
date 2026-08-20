# Qlik Engine API field guide

## 1. Mental model: stable IDs and temporary handles

QIX is a stateful JSON-RPC API. Methods run against a handle:

- handle `-1` is the global Engine interface;
- `OpenDoc` returns an app handle;
- `GetMeasure`, `GetDimension`, `GetVariableByName`, and `GetObject` return object handles;
- those handles expire with the Engine session.

Persist `qId` or `qGenericId` in source control. Never persist `qHandle`, and never assume an app will always have handle `1`.

Use a unique JSON-RPC `id` for each request so responses can be correlated reliably.

## 2. Open the app for the job you are doing

Open with `qNoData: true` when you only need:

- app and object metadata;
- master-item definitions;
- variables;
- sheet structure;
- object properties.

This avoids loading the associative model and is usually the fastest way to do definition work.

Open with `qNoData: false` when you need:

- evaluated layouts or hypercube pages;
- fields and loaded table associations;
- expression behavior against actual data;
- reload validation.

Treat those as separate passes. Metadata extraction should not accidentally become a row-level data export.

## 3. Discover first, inspect selectively

Use `GetAllInfos` on the app handle as a lightweight catalogue of persistent `qId` and `qType` values. Filter that result locally and open only the object that matters:

```text
GetAllInfos
  -> GetMeasure(qId)   -> GetProperties
  -> GetDimension(qId) -> GetProperties
  -> GetObject(qId)    -> GetProperties or GetChildInfos
```

Qlik also supports session objects based on `qMeasureListDef`, `qDimensionListDef`, and similar list definitions. They are useful shortcuts, but do not make an empty list a blocker. A real-world app review found list objects returning no master items while `GetAllInfos` and the direct getter methods still worked correctly.

For a sheet, `GetChildInfos` gives a useful shallow inventory of visual objects. Use `GetFullPropertyTree` only when a nested container genuinely requires recursive inspection.

## 4. Create governed master items

### Master measures

Create a measure with `Doc.CreateMeasure`. For a normal single-expression master measure:

- set `qInfo.qType` to `measure`;
- set `qMeasure.qGrouping` to `N`;
- put the expression in `qMeasure.qDef`;
- set `qMeasure.qNumFormat` at creation time;
- use `qMetaDef` for title, description, and tags.

Qlik may assign a different ID if the supplied `qId` is missing or already used. Always record the returned `qInfo.qId`/`qReturn.qGenericId`, not the requested value.

Before creation, call `CheckExpression` with strict field validation where possible. A successful check has an empty `qErrorMsg`, `qBadFieldNames`, and `qDangerousFieldNames`.

### Master dimensions

Create a dimension with `Doc.CreateDimension`:

- set `qInfo.qType` to `dimension`;
- use `qDim.qFieldDefs` for one or more fields/expressions;
- use matching `qDim.qFieldLabels` entries;
- use `qGrouping: N` for a single dimension, `H` for drill-down, or `C` for cyclic grouping;
- put governance metadata in `qMetaDef`.

The master-dimension definition does not carry the full visual column presentation. For example, a date column's `qNumberPresentations` belongs in the visualization's `NxDimension.qDef`. Govern that presentation in the reusable object template that creates the table or chart.

### Variables

Use `Doc.CreateVariableEx`; the older `CreateVariable` method is deprecated.

Important constraints from Qlik:

- variable names are case-sensitive and must be unique;
- `qDefinition` contains the value or expression text;
- `qComment` is useful governance metadata;
- a variable created in the load script must be changed through the script, not with `GenericVariable.SetProperties`;
- in a published app, persistent variable creation may be restricted; use a transient session variable when appropriate.

### An idempotent create-or-update pattern

Do not blindly rerun `Create*` calls. A repeatable deployment should:

1. discover existing objects;
2. match by governed persistent ID, with name/title as a diagnostic fallback;
3. compare the current properties with the desired definition;
4. create when absent;
5. patch or set properties when present;
6. record the returned persistent ID;
7. re-read and verify.

This prevents duplicate master items and makes drift visible.

## 5. Reuse master items in visualizations

In a hypercube definition, reference governed items by ID:

```json
{
  "qLibraryId": "MASTER_DIMENSION_ID"
}
```

or:

```json
{
  "qLibraryId": "MASTER_MEASURE_ID"
}
```

This centralizes expressions, labels, and number formats. Changing the master item then updates every linked visualization.

Use an inline `qDef` only when a dimension or measure is intentionally local to that visualization. For inline measures, put `qNumFormat` in the same definition so the object is complete at creation time.

## 6. Build a straight table in one request

There are two creation paths:

- `Doc.CreateObject` creates a generic object at app level;
- `GenericObject.CreateChild` creates a visual object inside a sheet or other parent.

For a table that should appear on a sheet:

1. resolve the sheet with `GetObject`;
2. call `CreateChild` on the returned sheet handle;
3. send the complete table `qProp` once.

The request can include all of the following together:

- table type and persistent ID;
- title, subtitle, footnote, and title visibility;
- master or inline dimensions;
- master or inline measures;
- number formats;
- null and zero suppression;
- dimension sort criteria;
- inter-column sort priority;
- display column order and widths;
- cell foreground/background color expressions;
- the initial hypercube data page.

See [`06-create-straight-table-in-one-call.json`](../examples/06-create-straight-table-in-one-call.json).

Two details matter:

1. `qInterColumnSortOrder` controls sort priority, while `qColumnOrder`/the native table's `columnOrder` controls display position. They are not the same concern.
2. `qInitialDataFetch` describes the initial page requested in the layout. Its `qWidth` must cover all dimension and measure columns, and the cell count should remain within Engine limits. Fetch additional pages later rather than making an enormous initial request.

Native table presentation properties have changed across Qlik clients. The example includes both Engine hypercube ordering and the native straight-table display properties documented by Qlik. Verify the object in the exact Qlik version you deploy to.

### Persistent versus transient tables

Use `CreateChild` or `CreateObject` for a persistent object. Use `CreateSessionObject` for a temporary hypercube used by automation, testing, previews, or data extraction. A session object disappears when the session ends and should not be saved as app content.

## 7. Change existing objects safely

### Prefer `ApplyPatches` for narrow changes

A patch has:

- `qOp`: `add`, `remove`, or `replace`;
- `qPath`: a JSON path such as `/qMetaDef/description`;
- `qValue`: a **JSON-encoded string**.

That last point causes many errors. To replace a property with the string `New text`, `qValue` must contain `"New text"` including its JSON quotes.

Use the method signature for the target interface:

- `GenericMeasure.ApplyPatches` and `GenericDimension.ApplyPatches` take `qPatches`;
- `GenericObject.ApplyPatches` can additionally use `qSoftPatch` for session-only visual changes.

A soft patch appears in layout behavior but not in persistent properties and is cleared when the session ends.

### Use `SetProperties` deliberately

Use `SetProperties` when a complete native definition must change. The safe sequence is:

```text
GetProperties
  -> modify the complete returned qProp
  -> SetProperties
  -> GetProperties
  -> compare intended and actual values
```

Do not reconstruct an existing object from a partial memory of its schema. Omitting native properties during a full replacement can remove them.

The object type and persistent identifier cannot be changed through property updates.

## 8. Validate before mutating

Useful preflight methods include:

- `CheckExpression` for chart and master-measure expressions;
- `CheckNumberOrExpression` where a property may accept either;
- `CheckScriptSyntax` before replacing a load script;
- `GetTablesAndKeys` after opening with data to compare the actual loaded model with the intended script.

A fast deployment pipeline validates the whole desired batch first. Only after every definition passes should it begin creating or updating persistent objects.

## 9. Verify, then save once

After each mutation, re-read the affected object with `GetProperties` and compare the values that matter:

- expression and grouping;
- labels and dynamic label expressions;
- number format;
- color settings;
- metadata and tags;
- master-item library IDs;
- sort order and calculation conditions.

For normal master-item and object edits, call `Doc.SaveObjects` after the verified batch. This saves modified objects without saving data-model data.

Use `Doc.DoSave` only when a full app save, including data-model data, is intended—for example after a reload workflow.

Avoid saving after every tiny request. A better loop is:

```text
change -> verify -> change -> verify -> SaveObjects
```

## 10. Source-control structure

A useful repository represents governed app intent, not every byte of a QVF:

```text
source/
  measures.json       desired executable definitions
  dimensions.json
  variables.json
  load-script.qvs

inventories/
  measures.json       selected verified deployed metadata
  dimensions.json
  sheets.json
  important-objects.json
```

Good candidates for source control:

- persistent IDs and types;
- exact expressions and field definitions;
- master-item references;
- number formats, colors, sort rules, labels, and calculation conditions;
- selected sheet membership and object hierarchy;
- stable loaded table/field/key metadata when needed for validation.

Keep out of source control:

- session handles and session object IDs;
- authentication material;
- current selections and runtime caches;
- evaluated `qDataPages`;
- sensitive row-level values or identifiers;
- full recursive property exports with no maintenance purpose.

## 11. Practical troubleshooting

| Symptom | Likely cause | Action |
| --- | --- | --- |
| Empty master-item list | Session-list definition/client behavior | Use `GetAllInfos`, then direct getters |
| `Invalid handle` | Handle copied from another session or request chain | Resolve the app/object again in the current session |
| Patch stores extra quotes or fails | `qValue` not JSON-encoded correctly | Encode the replacement value once as JSON |
| UI table column order is wrong | Sort order confused with display order, or client-native property differs | Set both concerns explicitly and inspect a known-good native table |
| Duplicate master items | Non-idempotent deployment | Discover and compare before `Create*` |
| Variable cannot be updated | Script-defined or published-app restriction | Change the load script or use a session variable |
| Object looks correct until reconnect | Soft patch or unsaved change | Use a persistent patch and call `SaveObjects` after verification |
| Metadata request is slow | App opened with data unnecessarily | Reopen with `qNoData: true` |

## Official reference map

- [`Doc`: open, discover, validate, create, and save](https://qlik.dev/apis/json-rpc/qix/doc/)
- [`GenericObject`: child objects, properties, layouts, patches](https://qlik.dev/apis/json-rpc/qix/genericobject/)
- [`GenericMeasure`](https://qlik.dev/apis/json-rpc/qix/genericmeasure/)
- [`GenericDimension`](https://qlik.dev/apis/json-rpc/qix/genericdimension/)
- [`GenericVariable`](https://qlik.dev/apis/json-rpc/qix/genericvariable/)
- [Straight-table properties](https://help.qlik.com/en-US/sense-developer/May2026/Subsystems/APIs/Content/Sense_ClientAPIs/CapabilityAPIs/VisualizationAPI/table-properties.htm)
- [Table formatting and color examples](https://help.qlik.com/en-US/sense-developer/May2026/Subsystems/Mashups/Content/Sense_Mashups/Create/Visualizations/Table/create-table.htm)
- [Creating a persistent hypercube as a sheet child](https://help.qlik.com/en-US/sense-developer/May2026/csh/engineapi/create-hypercube)
