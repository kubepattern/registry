# `Widget Not Referenced`

**ID**: `krateo-widget-not-referenced` | **Type**: `CUSTOM` | **Severity**: `WARNING` | **Category**: `krateo-ui`
**Docs**: [Krateo Widgets](https://docs.krateo.io/key-concepts/kcp/frontend-widget-api-reference) | **Source**: [GitHub](https://www.google.com/search?q=https://github.com/GabrieleGroppo/kubepattern-registry/tree/main/doc/widget-not-referenced-pattern.json)

-----

## Problem

UI Widgets in the Krateo Control Plane that are **not referenced** by any container widget (e.g., Panel, Row, Column, Page) represent orphaned resources. These resources consume cluster capacity and clutter the configuration without providing any visible value to the end-user.

**Example**: A developer creates a **Table** to display deployment metrics but forgets to wire it into the dashboard Page. The Table resource remains in the cluster, consuming memory and processing cycles, but users never see it.

-----

## Solution

Every Widget must be referenced in at least one container widget's `resourcesRefs.items[]` array to establish visibility in the UI hierarchy. This ensures the Resource Graph remains clean and all configured UI components serve an active purpose.

**Key Concepts**: Resource Graph Integrity, Dead Configuration Elimination, Widget Lifecycle Management

-----

## Implementation

This pattern applies to **all** Krateo widgets, but is illustrated here using a `Table` and a `Panel`.

```yaml
# ✅ CORRECT: Referenced Widget
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Table
metadata:
  name: deployment-metrics-table
  namespace: production
spec:
  widgetData:
    columns:
      - title: Deployment
        valueKey: name
      - title: Replicas
        valueKey: replicas
    pageSize: 10

---
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Panel
metadata:
  name: metrics-dashboard
  namespace: production
spec:
  widgetData:
    items:
      - resourceRefId: deployment-metrics-table
        size: 24
  resourcesRefs:
    items:
      - id: deployment-metrics-table      # ✅ Establishes reference
        apiVersion: widgets.templates.krateo.io/v1beta1
        name: deployment-metrics-table
        namespace: production
        resource: tables
        verb: GET
```

**Anti-pattern** (avoid):

```yaml
# ❌ WRONG: Orphaned Widget
apiVersion: widgets.templates.krateo.io/v1beta1
kind: Table
metadata:
  name: unused-metrics
  namespace: production
spec:
  widgetData:
    columns:
      - title: Field
        valueKey: field
# No Panel, Row, Column, or Page references this Table
# Resource exists but provides zero value
```

-----

## Detection

**Violates if**:

  - The Widget resource has **zero inbound REFERENCES\_KRATEO relationships** from any container widget.
  - `minRelationshipPoints: 1` is not satisfied.
  - The Widget exists in a non-system namespace but is never wired into the UI graph.

**Applicable Widgets**:
This rule applies to the full list of Krateo UI components:

  * `BarChart`
  * `Button`
  * `Column`
  * `DataGrid`
  * `Filters`
  * `FlowChart`
  * `Form`
  * `LineChart`
  * `Markdown`
  * `Page`
  * `Panel`
  * `Paragraph`
  * `PieChart`
  * `Row`
  * `Table`
  * `TabList`
  * `YamlViewer`

-----

## Trade-offs

⚠️ **Development Phase**: During active development, widgets may be temporarily unreferenced before UI wiring is complete (acceptable as a transient state).

⚠️ **Template Libraries**: Organizations maintaining widget templates for reuse (e.g., standard Tables or Forms) may keep intentionally unreferenced resources in a library namespace.

✅ **Benefits**:

  - **Performance**: Reduced cluster resource consumption.
  - **Clarity**: Cleaner configuration with no "dead code."
  - **Maintainability**: Faster debugging; if it exists, it is used.
  - **Accuracy**: Documentation of active UI components matches reality.