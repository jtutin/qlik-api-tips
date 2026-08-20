# Qlik Engine API tips

Practical, copy/paste-oriented notes for developing Qlik apps through the Qlik Engine JSON API (QIX).

The focus is repeatable app engineering:

- discover and inspect persistent app objects;
- create and maintain master measures, dimensions, and variables;
- create complete reusable master visualizations in one request;
- make small, verifiable changes safely;
- keep a useful, sanitised representation of app metadata in Git.

## Start here

Read the [Engine API field guide](docs/engine-api-field-guide.md), then adapt the requests in [`examples/`](examples/).

The shortest safe development loop is:

```text
OpenDoc
  -> GetAllInfos
  -> resolve the persistent object you need
  -> GetProperties
  -> validate the intended expression/definition
  -> Create*, ApplyPatches, or SetProperties
  -> GetProperties again and verify
  -> SaveObjects once after the verified batch
```

## Most reusable lessons

1. **Store persistent IDs, never handles.** A `qId`/`qGenericId` can be governed in source control. A `qHandle` belongs only to the current Engine session.
2. **Inspect metadata without loading data.** Open with `qNoData: true` for object discovery and property work. Reopen with data only when the associative model or evaluated results are required.
3. **Use direct getters as the dependable fallback.** If a measure-list or dimension-list session object is empty or incomplete, use `GetAllInfos`, then `GetMeasure`, `GetDimension`, or `GetObject` and `GetProperties`.
4. **Prefer small patches.** `ApplyPatches` is faster and safer for a narrow change. Use `SetProperties` only when intentionally replacing a complete native property definition.
5. **Build reusable visuals at app level.** A complete master visualization can be created in one `CreateObject` request with its master-item references, hypercube, titles, formatting, colors, and metadata.
6. **Verify before saving.** Re-read the object after every mutation. Use `SaveObjects` after a verified batch; reserve `DoSave` for a full app/data save.
7. **Keep evaluated data out of Git.** Store stable definitions and selected metadata, not `qDataPages`, selections, credentials, cookies, certificates, or sensitive row-level values.

## Example requests

| File | Purpose |
| --- | --- |
| [`01-open-app-metadata.json`](examples/01-open-app-metadata.json) | Open an app without data for fast metadata work |
| [`02-discover-persistent-objects.json`](examples/02-discover-persistent-objects.json) | Catalogue persistent object IDs and types |
| [`03-create-master-measure.json`](examples/03-create-master-measure.json) | Create a governed master measure with formatting |
| [`04-create-master-dimension.json`](examples/04-create-master-dimension.json) | Create a governed master dimension |
| [`05-create-variable.json`](examples/05-create-variable.json) | Create a persistent variable with `CreateVariableEx` |
| [`06-create-master-visualization-in-one-call.json`](examples/06-create-master-visualization-in-one-call.json) | Create a complete reusable app-level master visualization |
| [`07-patch-measure-description.json`](examples/07-patch-measure-description.json) | Make a targeted metadata change |
| [`08-save-objects.json`](examples/08-save-objects.json) | Persist a verified batch of object changes |
| [`09-resolve-sheet.json`](examples/09-resolve-sheet.json) | Resolve a persistent sheet ID to its current-session handle |
| [`10-check-expression.json`](examples/10-check-expression.json) | Validate an expression before creating or updating objects |
| [`11-get-measure.json`](examples/11-get-measure.json) | Resolve a master measure to its current-session handle |
| [`12-get-properties.json`](examples/12-get-properties.json) | Read native properties for verification or a safe full update |
| [`13-create-linked-master-visualization-instance.json`](examples/13-create-linked-master-visualization-instance.json) | Create a sheet instance linked to a master visualization |

The numeric handles in the examples are illustrative. Replace them with handles returned in the current session. Also replace every `YOUR_*` value.

## Official Qlik references

- [QIX `Doc` methods](https://qlik.dev/apis/json-rpc/qix/doc/)
- [QIX `GenericObject` methods](https://qlik.dev/apis/json-rpc/qix/genericobject/)
- [QIX `GenericMeasure` methods](https://qlik.dev/apis/json-rpc/qix/genericmeasure/)
- [QIX `GenericDimension` methods](https://qlik.dev/apis/json-rpc/qix/genericdimension/)
- [QIX `GenericVariable` methods](https://qlik.dev/apis/json-rpc/qix/genericvariable/)
- [Qlik `CreateObject` reference](https://help.qlik.com/en-US/sense-developer/May2026/Subsystems/EngineJSONAPI/Content/service-doc-createobject.htm)
- [Qlik bar-chart properties](https://help.qlik.com/en-US/sense-developer/May2026/Subsystems/APIs/Content/Sense_ClientAPIs/QlikVisual/qlik-visual-barchart-properties.htm)
- [Qlik master-visualization guidance](https://help.qlik.com/en-US/sense/May2026/Subsystems/Hub/Content/Sense_Hub/Visualizations/create-master-visualization.htm)
- [Qlik generic-object publishing](https://help.qlik.com/en-US/sense-developer/May2026/Subsystems/EngineJSONAPI/Content/service-genericobject-publish.htm)
- [Qlik .NET SDK master-object workflow](https://help.qlik.com/en-US/sense-developer/May2026/Subsystems/NetSDKAPI/Content/Sense_NetSDKAPI/HowTos/Net-Sdk-How-To-Library.htm)

## Scope and version note

These examples target the Engine JSON API and use Qlik's May 2026 developer documentation. Native visualization properties can vary by Qlik release and client. When exact UI behavior matters, create one known-good visualization in the target environment, read it with `GetProperties`, and use that native property shape as the local master-object template.
