# KubePattern Pattern authoring guide for AI assistants

**For people.** Give this file to an AI chatbot (upload it, or paste it as the first message), then describe the problem: the smell to detect, or the correct behaviour whose violation you want to find. The assistant answers with a KubePattern `Pattern` ready to `kubectl apply`, explains what it flags, and lists the Kubernetes and CRD versions it assumed, so an architect can check that the result matches the current APIs. If the engine rejects the Pattern, paste the linter error back into the chat: the assistant fixes it.

**Applies to.** The Pattern DSL `kubepattern.dev/v1` as implemented by the KubePattern engine on the `dev` branch (commit `1341079`, October 2026). Element anchors `[@]`, defaults, transforms, quoted keys, `kinds`, `kind: "*"`, `category` and `fromTarget` need that engine.

Everything below is addressed to the assistant.

---

## 1. Your task

You receive one of three requests:

| Request | What you produce |
|---|---|
| **A. A smell.** "Find the SecretStores nobody uses." | A Pattern that flags **every** resource affected by the smell, and nothing else. |
| **B. A correct behaviour.** "Every CNPG Cluster must have a ScheduledBackup." | First derive the anti-pattern, i.e. its violation: "a Cluster without a ScheduledBackup". Then do as in A. |
| **C. A Pattern and a linter error.** | The corrected Pattern, with the cause of the error in one sentence (see section 8). |

The result must be a valid Pattern that the user can apply to the cluster as it is. You cannot run the linter or the cluster yourself: write YAML that you have checked against the rules of this guide.

**What you may assume.** You know Kubernetes, and you know the CRDs the problem involves: their kinds, groups, served versions, plural names and the fields that hold references. When this is not true, **say so** and do not invent anything. Examples: you have never seen the CRD; you are unsure whether a field is `spec.storeRef` or `spec.secretStoreRef`; the project renamed fields between releases. Then you have two options:
- ask the user for the schema (`kubectl explain <plural>.spec --recursive --api-version=<group>/<version>`, `kubectl get crd <plural>.<group> -o yaml`) or for a sample object (`kubectl get <plural> -A -o yaml`);
- or write the Pattern anyway and mark every unverified path in the *Assumptions and versions* section.

A wrong path never makes the engine fail: it silently changes the result (section 6, rule 1). An honest "unverified" is worth more than a confident guess.

## 2. Output contract

Answer in this order, every time:

1. **Interpretation.** One sentence: "A `<Kind>` is defective when …". Then the family (section 4.2).
2. **The Pattern(s).** One YAML code block. When the problem needs several Patterns (section 6, rule 7), separate them with `---` in the same block.
3. **What it flags and what it does not.** Possible false positives and false negatives, and any part of the problem the DSL cannot express (section 7). Say that it is not covered; never approximate silently.
4. **How to try it.** The commands of section 9, adapted to the Pattern.
5. **Assumptions and versions.** Always present, even when you are sure:

   | Component | Version assumed | API used | Source |
   |---|---|---|---|
   | Kubernetes | e.g. ≥ 1.19; fields checked against the release you know best | `networking.k8s.io/v1` Ingress, `v1` Service | Kubernetes API reference / your training data |
   | `<operator or CRD project>` | the release whose CRDs you used | `<group>/<version>`, plural | docs, schema provided by the user, or training data |
   | KubePattern | engine `dev` (`1341079`) | `kubepattern.dev/v1` | this guide |

   Then add:
   - **Paths.** List every field path used, each marked *verified* (in a schema or object the user gave you) or *from knowledge* (from your training data).
   - **Knowledge cutoff.** State yours, and say that newer Kubernetes or CRD releases may have moved, renamed or deprecated the fields. For example: "a newer `v1beta2` may exist; check `kubectl api-resources`."

## 3. How the engine evaluates a Pattern

The engine runs as a CronJob. At each run it lints every Pattern installed in the cluster, then evaluates each valid one:

1. **Targets.** The resources of the target kind(s) that pass the target filters.
2. **Dependency candidates.** For each dependency: the resources of its kind(s) that pass its filters.
3. **Relationships.** For each target, the engine evaluates the relationships against the dependency candidates. A relationship **holds** when **at least one** candidate satisfies it.
4. **Smell.** If the relationship block evaluates to **true**, the target gets one `Smell` (`<pattern-name>-<target-uid>`), whatever the number of failing elements. With no relationships, every target that passes the filters gets a Smell.
5. **Prune.** A Smell whose condition no longer holds is deleted at the next run.

The relationship block combines three lists, all of which must pass:

- `matchAll`: every relationship holds;
- `matchAny`: at least one holds (ignored when empty);
- `matchNone`: none holds.

> **Relationships describe the defect, not the healthy state.** "A SecretStore is fine when an ExternalSecret references it" becomes "a SecretStore is defective when **no** ExternalSecret references it": `matchNone` with a relationship that recognises a reference.

## 4. From the problem to the Pattern

### 4.1 Steps
1. **Restate** the problem as "a `<Kind>` is defective when …". For request B, negate the correct behaviour first.
2. **Classify** it in a family (4.2). The family gives the shape.
3. **Choose the target.** It is the resource that receives the Smell, the one someone must fix or delete:
   - in *unused* problems, the resource that nobody references;
   - in *dangling* problems, the resource that holds the broken reference.
4. **List every way the relation can be expressed.** This step decides the quality of the Pattern. Cover:
   - **every referrer kind**, e.g. Certificate *and* CertificateRequest;
   - **every field** that can hold the reference: single fields, array entries, nested templates (`spec.template.spec…`), label selectors, string-encoded references (`name@version`, URLs, query strings);
   - **implicit API defaults**: an omitted `namespace` means the referrer's namespace; an omitted `kind` means the API's default kind; an omitted `group` means the default group.

   In an *unused* Pattern, a forgotten reference path produces false positives.
5. **Scope it.**
   - Namespaces: is the reference namespaced, cross-namespace, or cluster-scoped?
   - Exclude system namespaces, built-in objects and controller-generated objects, using target filters.
   - Narrow the dependencies with dependency filters.
6. **Write it** with the DSL (section 5), then **self-check** it against section 6 and section 8.

### 4.2 Families
These are the families of the [KubePattern registry](https://github.com/kubepattern/registry), with the shape each one takes and the registry Pattern to imitate (section 10).

| Family | Defect | Shape | Registry reference |
|---|---|---|---|
| `unused` | A definition exists, but nothing refers to it. | Target = the definition. Dependencies = the referrers. `matchNone`: one relationship per way of referring to it. | `eso-secretstore-not-used`, `certmanager-clusterissuer-not-used`, `flux-ocirepository-not-used` |
| `dangling` | A resource points at something that does not exist. | Target = the referrer, filtered on `EXISTS` of the reference (and `[@]` for arrays). Dependency = the referenced kind. `matchNone`. | `krateo-rolebinding-subject-sa-missing`, `krateo-navmenuitem-page-missing` |
| `companion` | A resource lacks the sibling that should accompany it. | Target = the resource. Dependency = the sibling. `matchNone`. | `cnpg-cluster-without-scheduledbackup` |
| `misconfig` | A relation exists that should not. | `matchAll` with the forbidden relation. | `cnpg-pooler-name-clashes-with-cluster` |
| `single` | A single resource lacks a field, or has a wrong value. | Target filters only; no dependencies and no relationships. | `missing-liveness-probe` |

Ownership problems use `owns` / `ownedBy` (`argocd-applicationset-generates-nothing`: an ApplicationSet that owns no Application). Selection problems use `selects` / `selectedBy` (e.g. a Service that selects no Pod).

**Request B, typical inversions:**

| Correct behaviour | Anti-pattern Pattern |
|---|---|
| "Every X must have a Y" | companion: target X, `matchNone` Y |
| "Every X must be used" | unused: target X, `matchNone` on each referrer |
| "References from X must resolve" | dangling: target X, filter `EXISTS` on the reference, `matchNone` on the referenced kind |
| "X must never relate to Y in this way" | misconfig: target X, `matchAll` with the relation |
| "Every container must set field F" | single: target filter `matchNone` `EXISTS` on `…containers[@].F` |

## 5. DSL reference

### 5.1 Skeleton
```yaml
apiVersion: kubepattern.dev/v1          # always this value
kind: Pattern                           # always Pattern
metadata:
  name: eso-secretstore-not-used        # lowercase letters, digits, '-' and '.'; Patterns are cluster-scoped
  labels:                               # registry metadata: no effect on detection
    kubepattern.dev/platform: external-secrets            # the registry folder (kubernetes, argocd, ...)
    kubepattern.dev/family: unused                        # unused | dangling | companion | misconfig | single
  annotations:
    kubepattern.dev/summary: "One English sentence: what the Pattern flags and why it matters."
spec:
  displayName: SecretStore Not Used     # required
  category: Security                    # required, free text; the registry uses Security, Reliability, Architecture, Cost, Misconfiguration
  severity: MEDIUM                      # required: LOW | MEDIUM | HIGH | CRITICAL
  message: "SecretStore {{target.metadata.namespace}}/{{target.metadata.name}} is not used."   # required
  reference: "https://..."              # optional: documentation of the smell or the API
  for: 1h                               # optional: minimum duration before the Smell becomes Active (Pending until then)
  target: {...}                         # required
  dependencies: [...]                   # optional: required by every relationship
  relationships: {matchAll: [...], matchAny: [...], matchNone: [...]}    # optional
```

**Message placeholders.** Only these are replaced; any other `{{…}}` stays as literal text:
- `{{target.metadata.name}}`, `{{target.metadata.namespace}}`, `{{target.metadata.uid}}`;
- `{{target.kind}}`, `{{target.apiVersion}}`: use `{{target.kind}}` when the target has several kinds;
- `{{smell.since}}`, `{{smell.phase}}`.

For a cluster-scoped target, do not print the namespace.

**`for`.** A Go duration (`30m`, `1h`, `24h`). Use it when the defect can be a transient state: a GitOps sync that creates the referrer after the referenced object, or a template rotation.

### 5.2 Resource types (target and dependencies)
A target and each dependency select resource types in exactly **one** of these forms:

| Form | Example | Notes |
|---|---|---|
| one kind | `{kind: Secret, apiVersion: v1, plural: secrets}` | `kind`, `apiVersion` and `plural` are all required. `plural` is the exact REST resource name (as in `kubectl api-resources`); a wrong plural makes the Pattern skipped at run time. |
| all kinds of a group version | `{kind: "*", apiVersion: widgets.templates.krateo.io/v1beta1}` | No `plural`. The kinds are found by discovery at run time. |
| a discovery category | `{category: fluxcd-appliers}` | Every kind whose API categories include it. No `kind`, `apiVersion` or `plural`. |
| a list | `kinds: [{kind: Markdown, apiVersion: ..., plural: markdowns}, {kind: "*", apiVersion: ...}, {category: ...}]` | The union of its entries. |
| derived from the targets (dependencies only) | `fromTarget: {apiVersionPath: status.apiVersion, kindPath: status.kind, pluralPath: status.resource}` | The kinds named by fields of the target candidates. `kindPath` is required, plus exactly one of `apiVersionPath` / `apiVersion` (fixed). `[@]` reads one kind per element; `[*]` is rejected. |

Each dependency also needs a unique `id`, which the relationships name in `with`. Target and dependencies may have `filters` (5.4).

Further rules:
- **Version.** Use a version the API server serves; prefer the storage/preferred version. If two Patterns read the same kind, use the same version in both.
- **Missing CRD.** An unknown group version (missing CRD) makes the Pattern **skipped**, not failed. A category that matches nothing gives no candidates.

### 5.3 Paths
A path addresses fields of one resource; its segments are separated by dots.

| Syntax | Meaning |
|---|---|
| `spec.secretStoreRef.name` | a nested field |
| `kind`, `apiVersion`, `metadata.namespace` | top-level fields work too |
| `spec.data[*].name` | every element of an array. Values from different elements are **not** correlated. |
| `spec.data[@].name` | the element **bound** by the evaluation (element scope, 5.7) |
| `spec.data[0].name` | one element |
| `metadata.labels['app.kubernetes.io/name']` | a map key containing dots or slashes (`["..."]` also works) |

A path that reaches nothing yields **no value**: a criterion with no value on one side fails, `EXISTS` is false, and `IS_EMPTY` is true. Paths are checked for syntax only, never against the CRD schema.

### 5.4 Filters
`filters: {matchAll: [...], matchAny: [...], matchNone: [...]}` on the target or on a dependency. Each condition reads **one** resource: `{path, operator, values}`.

| Operator | `values` | True when |
|---|---|---|
| `EXISTS` | none | the path yields at least one value |
| `IS_EMPTY` | none | the path is missing, or is an empty string, list or map |
| `EQUALS` | one or more strings | any value of the path equals any listed value. Booleans and numbers compare by their string form: `values: ["true"]`, `values: ["3"]`. |
| `GREATER_THAN`, `GREATER_OR_EQUAL`, `LESS_THAN`, `LESS_OR_EQUAL` | one integer | some value is an integer that satisfies the comparison. Quantities like `500m` or `1Gi` are not numbers. |
| `ARRAY_SIZE_EQUALS`, `ARRAY_SIZE_GREATER_THAN`, `ARRAY_SIZE_GREATER_OR_EQUAL`, `ARRAY_SIZE_LESS_THAN`, `ARRAY_SIZE_LESS_OR_EQUAL` | one integer | the path points at a list whose size satisfies the comparison. A missing list is false. |

`valuesFrom: <path>` replaces `values` (with `EQUALS` only): it compares with another field of the same resource, under the same element binding. Example: the menu entry whose `items[@].id` equals `spec.widgetData.resourceRefId`.

`matchNone` filters exclude: `{path: metadata.namespace, operator: EQUALS, values: [kube-system]}` drops a namespace; `{path: metadata.name, operator: EQUALS, values: [default]}` drops a built-in object.

### 5.5 Relationships
Each entry: `{with: <dependency id>, type: <type>, ...}`.

| Type | Holds when (with at least one dependency candidate) | Fields |
|---|---|---|
| `custom` | all `criteria` hold | `criteria` (at least one) |
| `owns` | the target owns the dependency, transitively through `ownerReferences` | no criteria |
| `ownedBy` | the dependency owns the target, transitively | no criteria |
| `selects` | the target's label selector at `selectorPath` (default `spec.selector`) matches the dependency's labels, and the optional `criteria` hold | `selectorPath`, `emptySelector`, `criteria` |
| `selectedBy` | the dependency's label selector at `selectorPath` matches the target's labels, and the optional `criteria` hold | same |

Selectors:
- **Shapes.** A selector may be a `metav1.LabelSelector` (`matchLabels`, `matchExpressions`) or a plain label map, as in `Service.spec.selector`. A missing or invalid selector selects nothing.
- **Namespaces are not implied.** Add a namespace criterion when the selection is namespaced.
- **Empty selectors.** A selector without requirements selects what `emptySelector` says (`all` or `none`). When `emptySelector` is unset:
  - a LabelSelector with empty `matchLabels`/`matchExpressions`, or `{}` in a field whose name ends in `Selector` (`podSelector: {}`), selects **everything**;
  - `{}` in any other field selects **nothing** (an empty Service selector).

  Set `emptySelector: all` where the API differs, e.g. a PodDisruptionBudget's `spec.selector: {}`, which selects every Pod in its namespace.

### 5.6 Criteria
A criterion compares the target side with the dependency side. The only operator is `EQUALS`, which holds when the two **value sets intersect**. All values are compared as strings.

| Field | Meaning |
|---|---|
| `targetPath`, `dependencyPath` | the path read on each side |
| `targetValue`, `dependencyValue` | a literal instead of a path, on one side only. E.g. `{targetValue: pages, dependencyPath: "spec.resourcesRefs.items[@].resource"}`: the bound entry must have `resource: pages`. |
| `targetDefault`, `dependencyDefault` | `{path: ...}` or `{value: ...}`, used when the path yields nothing: an omitted namespace defaults to the referrer's `metadata.namespace`, an omitted kind to `SecretStore`. Not allowed with a `[*]` path; the default path cannot contain `[*]`. |
| `targetTransform`, `dependencyTransform` | exactly one of: `{apiGroup: true}` (`cert-manager.io/v1` → `cert-manager.io`, `v1` → empty); `{split: {separator: "@", index: 0}}` (negative index counts from the end); `{regex: "name=([^&]+)"}` (exactly one capture group; no match drops the value); `{lowercase: true}`. Applied after the default. |
| `operator` | `EQUALS` (required) |

**A complete reference** usually needs one criterion each for:
- the name;
- the kind, when the referring field can name several kinds;
- the group, compared with `apiVersion` through `apiGroup`;
- the namespace, when both sides are namespaced.

### 5.7 Element scope `[@]`
All the paths of one resource that share the prefix up to a `[@]` refer to the **same** array element. The engine enumerates the elements and evaluates one binding at a time.

- **Dependency side.** A dependency matches when **one of its elements** satisfies all the criteria of the relationship. With `[*]`, `spec.data[*].sourceRef.storeRef.name` and `spec.data[*].sourceRef.storeRef.kind` could come from two different entries. With `[@]` they come from the same entry.
- **Target side.** The target filters and all relationships share the target's bindings. The target is defective when the relationships hold for **at least one** element that passes the target filters. Under `matchNone` this reads "**some** reference resolves to nothing". A target whose anchored array is missing or empty is not examined at all.

The linter enforces these rules:
1. **No `[@]` in dependency filters**, because dependencies are filtered before any binding. `[@]` must never follow a `[*]` in the same path.
2. **One nested chain per side.** The anchors of one side must form **a single nested chain**:
   - `spec.rules[@]` with `spec.rules[@].http.paths[@]` is fine;
   - `spec.data[@]` with `spec.dataFrom[@]` is rejected.

   The target side covers the target filters and every relationship. The dependency side is checked per relationship, so two relationships may anchor different arrays of the same dependency (`data[@]` in one, `dataFrom[@]` in another).
3. **Every relationship reads the element.** When the target side has anchors, **every relationship must read the innermost anchored element**, through a target path or a target default. A relationship that ignores it, such as one on another field of the target or `owns`/`ownedBy`, goes in its own Pattern.

### 5.8 `emitOnEmpty` (rare)
`target.emitOnEmpty: true` writes one synthetic Smell when **no** resource passes the target filters ("there is no X in the cluster"). If targets exist, they are evaluated normally: without relationships, each of them gets a Smell. For a pure absence check, add a relationship that is false for every existing target. Prefer a companion Pattern when one fits.

## 6. Semantic rules and silent failures

The linter checks syntax and structure, not meaning. Check these yourself:

1. **A wrong path fails silently.** A mistyped or outdated path yields no value:
   - inside `matchNone`, every target is flagged (false positives everywhere);
   - inside `matchAll`, no target is ever flagged.

   Use only paths you are sure of, and report the others (section 2).
2. **Namespaces are never implied.**
   - **Namespaced reference.** For a reference between namespaced resources, add `{targetPath: metadata.namespace, dependencyPath: metadata.namespace}`. When the reference can name another namespace, compare with the reference's namespace field and default it to the referrer's own namespace: `dependencyPath: spec.sourceRef.namespace, dependencyDefault: {path: metadata.namespace}`.
   - **Cluster-scoped side.** **Never** put a namespace criterion on a cluster-scoped side (ClusterIssuer, ClusterSecretStore, ClusterRole, …). It has no namespace value, so the criterion never holds, and under `matchNone` every target is flagged.
3. **Name alone is not a reference.** If the referring field can name several kinds, also compare the kind, with the API's default for an omitted kind (`dependencyDefault: {value: SecretStore}`). Otherwise an object of another kind with the same name hides the smell.
4. **`[*]` or `[@]`.**
   - Use `[@]` whenever **two or more** fields of the same array element must match together: name with kind, or name with namespace.
   - `[*]` is fine for a single field (`spec.pipeline[*].functionRef.name`).
   - In target filters, `matchNone EXISTS containers[*].livenessProbe` means "**no** container has a probe", while `matchNone EXISTS containers[@].livenessProbe` means "**some** container lacks one". Choose the one the problem asks for.
5. **"At least one", never "all" or "how many".**
   - A relationship holds with **any** dependency candidate.
   - A target with `[@]` is defective when **some** element is.
   - The DSL cannot say "all elements", "all dependencies", or "referenced at least N times".
6. **Filter what is not a real candidate.** Remove from the targets the objects the problem does not cover:
   - system namespaces (`vela-system`, `krateo-system`, `kube-system`);
   - built-in objects (the `default` AppProject);
   - objects without the reference (`EXISTS` filter in dangling Patterns).

   Remove from the dependencies what does not count as a use: `cert-manager` CertificateRequests owned by a Certificate are history, not uses (`metadata.ownerReferences IS_EMPTY`).
7. **Split when a rule forbids combining.** Independent arrays on the target side, or a relationship that does not read the anchored element, need **separate Patterns**. Example: Ingress `spec.rules[@].http.paths[@].backend` and `spec.defaultBackend` (section 11).
8. **`status` fields are controller-written.** They can lag or be missing; `status.x EXISTS` in the target filters keeps only reconciled objects.
9. **The CRD of Pattern accepts any spec.** `kubectl apply` never rejects a malformed Pattern: only the engine's linter does, at the next run, and it logs the error and skips that Pattern (section 8).
10. **Run-time skips.** A Pattern is skipped, with a warning in the engine log, when one of its kinds cannot be listed:
    - a missing CRD;
    - a wrong `plural`;
    - a version that is not served;
    - missing RBAC (the default chart grants `get/list/watch` on everything).

## 7. What the DSL cannot express

When the problem needs one of these, write the closest **correct** Pattern for the part that can be expressed, and state the gap in "What it flags and what it does not":
- **Chains.** A relation between two dependencies (A → B → C): relationships always compare the target with one dependency. Workaround: change the target, or split the problem into several Patterns.
- **Counting and universal quantifiers.** "Referenced by more than one", "all Pods ready", "every entry valid".
- **Arithmetic and units.** Comparing two fields numerically, or parsing quantities (`500m`, `1Gi`) and dates. Time is covered only by `spec.for`.
- **String matching in filters.** No `CONTAINS`, prefix or regex in filters. Transforms exist only inside relationship criteria. The criteria operators `CONTAINS` and `LABEL_SELECTOR` are rejected: use transforms and `selects` / `selectedBy`.
- **Data outside the Kubernetes API.** Secret contents interpreted, external systems, metrics, reachability.

## 8. Linter errors and how to fix them

The engine logs `skipping invalid pattern … error="malformed pattern: <message>"`. The linter stops at the **first** error, so after a fix another error may appear. In the messages, `<ctx>` is a location such as `spec.target`, `spec.dependencies[1]` or `spec.relationships.matchNone[0].criteria[2]`. Count indexes from 0.

| Message (after `malformed pattern:`) | Cause and fix |
|---|---|
| `yaml input is empty` | The Pattern has no content. |
| `pattern definition is not valid yaml: … cannot unmarshal !!X into Y` | A field has the wrong type. Typical cases: `values: true` or `values: "x"` instead of a list (`values: ["true"]`); `dependencies`, `criteria` or a `matchNone` written as a map instead of a list; a `{...}` placeholder left in the YAML. Broken YAML syntax (indentation, an unquoted `:` or `{{`) is rejected earlier by `kubectl apply`: quote messages and paths that contain `[`, `{`, `:` or `'`. |
| `apiVersion is empty` / `'X' is not a valid apiVersion…` | Set `apiVersion: kubepattern.dev/v1`. |
| `kind is empty` / `kind must be 'PatternAsCode' or 'Pattern'…` | Set `kind: Pattern`. |
| `metadata.name is empty` / `metadata.name contains invalid characters…` | Use lowercase letters, digits, `-` and `.` only. |
| `spec.displayName is empty`, `spec.category is empty`, `spec.message is empty` | Add the field. |
| `spec.severity is empty` / `spec.severity must be one of [LOW, MEDIUM, HIGH, CRITICAL]…` | Use one of the four values, in upper case. |
| `spec.for 'X' is not a valid duration…` / `…must not be negative` | Use a Go duration: `30m`, `1h`, `24h` (no `d`; write `72h`). |
| `<ctx>.kind is empty` | The target or dependency has no type: add `kind`+`apiVersion`+`plural`, `category`, `kinds` or (dependencies only) `fromTarget`. Inside `kinds[i]`: the entry has no `kind`. |
| `<ctx> must use only one of kind, category or kinds` | Keep one form. |
| `<ctx>.category cannot be combined with kind, apiVersion or plural` | A category entry stands alone. |
| `<ctx>.apiVersion is empty` | Add `apiVersion: <group>/<version>` (`v1` for the core group). |
| `<ctx>.pluralName is empty` | The YAML field is **`plural`**: add the REST resource name, e.g. `plural: secretstores`. |
| `<ctx>.plural must be empty when kind is '*'` | Remove `plural` from the wildcard entry. |
| `spec.dependencies[i].id is empty` / `…id 'X' is not unique` | Give each dependency a distinct `id`. |
| `….fromTarget cannot be combined with kind, apiVersion, plural, category or kinds` | Keep only `fromTarget`. |
| `….fromTarget needs kindPath and exactly one of apiVersionPath or apiVersion` | Set `kindPath`, and either `apiVersionPath` or a fixed `apiVersion`. |
| `….fromTarget: '[*]' is not allowed in path…` | Use `[@]`. |
| `….fromTarget: the paths anchor two independent arrays…` | Read apiVersion, kind and plural from the same array element. |
| `….filters.<group>[j].path is empty` | Add `path`. |
| `…: invalid path 'P': <reason>` | Path syntax error: `empty path`, `empty segment`, `path ends with '.'`, `unterminated '['`, `unterminated quoted key`, `invalid bracket '[x]' (expected [*], [@], an index or a quoted key)`, `'[…]' must follow a field`, `missing '.' before position N`. Fix the syntax (5.3); quote keys with dots or slashes: `labels['app.kubernetes.io/name']`. |
| `…: '[@]' is not allowed in path 'P' (only target filters and relationship paths bind array elements)` | `[@]` inside a dependency filter. Use `[*]` there, or move the condition into a relationship criterion (`dependencyPath` with `[@]`, `dependencyValue` for a constant). |
| `…: '[*]' cannot precede '[@]' in path 'P'` | Write `a[@].b[@]` (or `a[@].b[*]`), never `a[*].b[@]`. |
| `….operator is empty` / `….operator 'X' is not valid` | Filter operators are listed in 5.4. Upper case, with underscores. |
| `….values should be empty for operator EXISTS` (or `IS_EMPTY`) | Remove `values`. |
| `….values is empty for operator X` | Add `values` (a list of strings: `["true"]`, `["3"]`). |
| `….valuesFrom is supported only with operator EQUALS` / `…must set either values or valuesFrom, not both` | Use `valuesFrom` alone, with `EQUALS`. |
| `spec.relationships.<g>[i].with is empty` / `…with 'X' does not match any dependency id` | `with` must equal the `id` of a declared dependency. |
| `….type is empty` / `….type 'X' is not valid…` | Use `custom`, `owns`, `ownedBy`, `selects` or `selectedBy`. |
| `… of type 'custom' must have at least one criteria` | Add criteria, or use another type. |
| `… of type 'owns' must not declare criteria` (or `ownedBy`) | Remove the criteria. To restrict the owned objects, filter the dependency. |
| `….selectorPath and emptySelector are only valid for types selects and selectedBy` | Remove them, or change the type. |
| `….emptySelector 'X' is not valid…` | Use `all` or `none`. |
| `….criteria[k].targetPath is empty` (or `dependencyPath`) | Each side needs a path or a literal (`targetValue` / `dependencyValue`). |
| `… must set either targetPath or targetValue, not both` | Keep one. |
| `….targetDefault needs a targetPath` | A default applies to a path, not to a literal. |
| `… compares two literals` | One side must be a path. |
| `….targetDefault cannot be used with '[*]'…` | Change the path to `[@]`, so that each element gets its default. |
| `….path 'P' cannot contain '[*]'…` (in a default) | Use `[@]`, or a path without arrays. |
| `….operator 'CONTAINS' is not implemented yet…` (or `LABEL_SELECTOR`) | Use `EQUALS` with a transform (`regex`, `split`), or a `selects` / `selectedBy` relationship. |
| `….criteria[k].operator is empty` / `…not valid. Expected: EQUALS` | Set `operator: EQUALS`. |
| `….split.separator is empty` | Set a separator. |
| `….regex is not a valid regular expression…` / `…must have exactly one capture group, found N` | Fix the regex (Go RE2 syntax, no look-around); use `(?:…)` for the other groups. |
| `… must set exactly one of apiGroup, split, regex or lowercase` | One transform per side. |
| `spec: the target paths anchor two independent arrays, 'A' and 'B'…` | The target side anchors two arrays that are not nested. Use `[*]` for one of them, or split into two Patterns. |
| `spec.relationships.<g>[i]: the dependency paths anchor two independent arrays…` | Within one relationship, anchor one array of the dependency. Use one relationship per array, with the same `with`. |
| `spec.relationships.<g>[i] does not read the target element 'A'…` | That relationship ignores the anchored target element. Move it to its own Pattern, or make it read the element. |

**Run-time messages** (not linter errors, but the Pattern does not produce Smells):

| Log message | Meaning |
|---|---|
| `skipping pattern due to resource fetch failure (e.g. missing RBAC)` | A kind cannot be listed. Check `apiVersion`, `plural`, that the CRD is installed, and the engine's RBAC. |
| `no target candidates found, skipping pattern` | Nothing passes the target filters. This can be correct, or a wrong kind/version or filter path. |
| `derived kind is not served, no instances` | A `fromTarget` kind does not exist in the cluster. |
| `invalid label selector` | A selector at `selectorPath` is not a valid selector. |

## 9. Trying a Pattern

Write these commands for the user, adapted to the Pattern. They assume the Helm release `kubepattern` in `kubepattern-system`.
```bash
kubectl apply -f pattern.yaml
# run the analysis now instead of waiting for the schedule
kubectl create job kp-try-1 --from=cronjob/kubepattern-cronjob -n kubepattern-system
kubectl wait --for=condition=complete job/kp-try-1 -n kubepattern-system --timeout=5m
# linter and run-time problems of this Pattern
kubectl logs job/kp-try-1 -n kubepattern-system | grep '<pattern-name>'
# Smells produced by this Pattern
kubectl get smells -A -l kubepattern.dev/pattern-uid=$(kubectl get pattern <pattern-name> -o jsonpath='{.metadata.uid}')
```

Suggest a quick test as well: one object that must be flagged and one that must not, e.g. a referrer that uses the omitted-namespace default.

## 10. Annotated examples from the registry

These Patterns are in the registry and were evaluated on real clusters. Imitate their shape. The `metadata` blocks are trimmed where they repeat the same labels.

### 10.1 single: filters only (`kubernetes/missing-liveness-probe`)
```yaml
apiVersion: kubepattern.dev/v1
kind: Pattern
metadata:
  name: missing-liveness-probe
  labels: {kubepattern.dev/platform: kubernetes, kubepattern.dev/family: single}
  annotations:
    kubepattern.dev/summary: "A Deployment whose containers define no liveness probe, so Kubernetes cannot restart the application when it hangs."
spec:
  displayName: "Missing Liveness Probe"
  category: "Reliability"
  severity: "HIGH"
  message: "Deployment {{target.metadata.namespace}}/{{target.metadata.name}} defines no livenessProbe: Kubernetes cannot tell when to restart the application if it hangs."
  target:
    kind: Deployment
    apiVersion: apps/v1
    plural: deployments
    filters:
      matchNone:
        - path: "spec.template.spec.containers[*].livenessProbe"
          operator: EXISTS
```
- **No relationships**, so every target that passes the filters is flagged.
- With `[*]` it flags a Deployment where **no** container has a probe. To flag a Deployment where **some** container lacks one, write `containers[@].livenessProbe`.

### 10.2 unused, cluster-scoped target (`cert-manager/certmanager-clusterissuer-not-used`)
```yaml
spec:
  displayName: ClusterIssuer Not Used
  category: Security
  severity: LOW
  message: "ClusterIssuer {{target.metadata.name}} is not referenced by any Certificate or CertificateRequest; it still holds signing material."
  reference: "https://cert-manager.io/docs/usage/certificate/"
  target:
    kind: ClusterIssuer
    apiVersion: cert-manager.io/v1
    plural: clusterissuers
  dependencies:
    - id: certificate
      kind: Certificate
      apiVersion: cert-manager.io/v1
      plural: certificates
      filters:
        matchAll:
          - {path: "spec.issuerRef.kind", operator: EQUALS, values: ["ClusterIssuer"]}
    - id: request
      kind: CertificateRequest
      apiVersion: cert-manager.io/v1
      plural: certificaterequests
      filters:
        matchAll:
          - {path: "spec.issuerRef.kind", operator: EQUALS, values: ["ClusterIssuer"]}
          - {path: "metadata.ownerReferences", operator: IS_EMPTY}
  relationships:
    matchNone:
      - with: certificate
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.issuerRef.name", operator: EQUALS}
          - targetPath: "apiVersion"
            targetTransform: {apiGroup: true}
            dependencyPath: "spec.issuerRef.group"
            dependencyDefault: {value: "cert-manager.io"}
            operator: EQUALS
      - with: request
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.issuerRef.name", operator: EQUALS}
          - targetPath: "apiVersion"
            targetTransform: {apiGroup: true}
            dependencyPath: "spec.issuerRef.group"
            dependencyDefault: {value: "cert-manager.io"}
            operator: EQUALS
```
- **No namespace criterion**: the target is cluster-scoped.
- **Kind.** The dependency filters keep only references of kind `ClusterIssuer`; a filter suffices because `issuerRef` is a single field.
- **Group.** The group is compared through `apiGroup` on the target's `apiVersion`; an omitted `issuerRef.group` defaults to `cert-manager.io`.
- **History is not a use.** CertificateRequests owned by a Certificate are excluded.

### 10.3 unused, several reference paths, `[@]` and selectors (`external-secrets/eso-secretstore-not-used`)
```yaml
spec:
  displayName: SecretStore Not Used
  category: Security
  severity: MEDIUM
  message: "SecretStore {{target.metadata.namespace}}/{{target.metadata.name}} is not referenced by any ExternalSecret or PushSecret in its namespace, but keeps credentials to an external provider."
  reference: "https://external-secrets.io/latest/api/secretstore/"
  target:
    kind: SecretStore
    apiVersion: external-secrets.io/v1
    plural: secretstores
  dependencies:
    - {id: externalsecret, kind: ExternalSecret, apiVersion: external-secrets.io/v1, plural: externalsecrets}
    - {id: push, kind: PushSecret, apiVersion: external-secrets.io/v1alpha1, plural: pushsecrets}
  relationships:
    matchNone:
      - with: externalsecret
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.secretStoreRef.name", operator: EQUALS}
          - {targetPath: "kind", dependencyPath: "spec.secretStoreRef.kind", dependencyDefault: {value: SecretStore}, operator: EQUALS}
          - {targetPath: "metadata.namespace", dependencyPath: "metadata.namespace", operator: EQUALS}
      - with: externalsecret
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.data[@].sourceRef.storeRef.name", operator: EQUALS}
          - {targetPath: "kind", dependencyPath: "spec.data[@].sourceRef.storeRef.kind", dependencyDefault: {value: SecretStore}, operator: EQUALS}
          - {targetPath: "metadata.namespace", dependencyPath: "metadata.namespace", operator: EQUALS}
      - with: externalsecret
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.dataFrom[@].sourceRef.storeRef.name", operator: EQUALS}
          - {targetPath: "kind", dependencyPath: "spec.dataFrom[@].sourceRef.storeRef.kind", dependencyDefault: {value: SecretStore}, operator: EQUALS}
          - {targetPath: "metadata.namespace", dependencyPath: "metadata.namespace", operator: EQUALS}
      - with: push
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.secretStoreRefs[@].name", operator: EQUALS}
          - {targetPath: "kind", dependencyPath: "spec.secretStoreRefs[@].kind", dependencyDefault: {value: SecretStore}, operator: EQUALS}
          - {targetPath: "metadata.namespace", dependencyPath: "metadata.namespace", operator: EQUALS}
      - with: push
        type: selectedBy
        selectorPath: "spec.secretStoreRefs[@].labelSelector"
        criteria:
          - {targetPath: "kind", dependencyPath: "spec.secretStoreRefs[@].kind", dependencyDefault: {value: SecretStore}, operator: EQUALS}
          - {targetPath: "metadata.namespace", dependencyPath: "metadata.namespace", operator: EQUALS}
```
- **One relationship per field** that can hold a reference. `data[@]` and `dataFrom[@]` are in separate relationships, because anchors on independent arrays may not share one.
- **`[@]`** correlates each entry's name with **its own** kind. An omitted kind is `SecretStore`, the ESO default.
- **`selectedBy`** covers the stores that a PushSecret selects by label instead of by name.

### 10.4 unused, category and cross-namespace default (`flux/flux-ocirepository-not-used`)
```yaml
spec:
  displayName: OCIRepository Not Consumed
  category: Cost
  severity: LOW
  message: "OCIRepository {{target.metadata.namespace}}/{{target.metadata.name}} is polled by source-controller but no HelmRelease or Kustomization consumes it."
  target:
    kind: OCIRepository
    apiVersion: source.toolkit.fluxcd.io/v1
    plural: ocirepositories
  dependencies:
    - id: applier
      category: fluxcd-appliers
  relationships:
    matchNone:
      - with: applier
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.chartRef.name", operator: EQUALS}
          - {targetPath: "kind", dependencyPath: "spec.chartRef.kind", operator: EQUALS}
          - targetPath: "metadata.namespace"
            dependencyPath: "spec.chartRef.namespace"
            dependencyDefault: {path: "metadata.namespace"}
            operator: EQUALS
      - with: applier
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.sourceRef.name", operator: EQUALS}
          - {targetPath: "kind", dependencyPath: "spec.sourceRef.kind", operator: EQUALS}
          - targetPath: "metadata.namespace"
            dependencyPath: "spec.sourceRef.namespace"
            dependencyDefault: {path: "metadata.namespace"}
            operator: EQUALS
```
- **Category.** The referrers are found by API category, so an applier kind added in a later Flux release is covered.
- **Cross-namespace references.** A reference may name another namespace; when it omits it, it means the referrer's own namespace (`dependencyDefault: {path: metadata.namespace}`).

### 10.5 dangling, element by element on the target (`krateo/krateo-rolebinding-subject-sa-missing`)
```yaml
spec:
  displayName: RoleBinding to Missing ServiceAccount
  category: Security
  severity: MEDIUM
  message: "RoleBinding {{target.metadata.namespace}}/{{target.metadata.name}} grants a Role to a ServiceAccount that does not exist (e.g. an agent that was never deployed): the grant is dormant and is picked up by any future ServiceAccount with that name."
  reference: "https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-subjects"
  target:
    kind: RoleBinding
    apiVersion: rbac.authorization.k8s.io/v1
    plural: rolebindings
    filters:
      matchAll:
        - {path: "subjects[@].kind", operator: EQUALS, values: [ServiceAccount]}
  dependencies:
    - {id: serviceaccount, kind: ServiceAccount, apiVersion: v1, plural: serviceaccounts}
  relationships:
    matchNone:
      - with: serviceaccount
        type: custom
        criteria:
          - {targetPath: "subjects[@].name", dependencyPath: metadata.name, operator: EQUALS}
          - {targetPath: "subjects[@].namespace", targetDefault: {path: metadata.namespace}, dependencyPath: metadata.namespace, operator: EQUALS}
```
- **Per-subject check.** The filter keeps the ServiceAccount subjects, and the relationship checks each of them. The binding is flagged when **some** ServiceAccount subject does not exist.
- **Default namespace.** A subject without a namespace defaults to the binding's own.

### 10.6 dangling, the entry chosen by another field (`krateo/krateo-navmenuitem-page-missing`)
```yaml
  target:
    kind: NavMenuItem
    apiVersion: widgets.templates.krateo.io/v1beta1
    plural: navmenuitems
    filters:
      matchAll:
        - {path: "spec.resourcesRefs.items[@].id", operator: EQUALS, valuesFrom: spec.widgetData.resourceRefId}
  dependencies:
    - {id: page, kind: Page, apiVersion: widgets.templates.krateo.io/v1beta1, plural: pages}
  relationships:
    matchNone:
      - with: page
        type: custom
        criteria:
          - {targetPath: "spec.resourcesRefs.items[@].name", dependencyPath: metadata.name, operator: EQUALS}
          - {targetPath: "spec.resourcesRefs.items[@].namespace", dependencyPath: metadata.namespace, operator: EQUALS}
```
- `valuesFrom` binds only the entry that the item actually opens; the other entries are not examined.

### 10.7 companion and misconfig (`cnpg`)
```yaml
# companion: a Cluster without a ScheduledBackup in its namespace
  target: {kind: Cluster, apiVersion: postgresql.cnpg.io/v1, plural: clusters}
  dependencies:
    - {id: schedule, kind: ScheduledBackup, apiVersion: postgresql.cnpg.io/v1, plural: scheduledbackups}
  relationships:
    matchNone:
      - with: schedule
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "spec.cluster.name", operator: EQUALS}
          - {targetPath: "metadata.namespace", dependencyPath: "metadata.namespace", operator: EQUALS}
```
```yaml
# misconfig: a Pooler with the same name as a Cluster in its namespace (forbidden by the API contract)
  target: {kind: Pooler, apiVersion: postgresql.cnpg.io/v1, plural: poolers}
  dependencies:
    - {id: cluster, kind: Cluster, apiVersion: postgresql.cnpg.io/v1, plural: clusters}
  relationships:
    matchAll:
      - with: cluster
        type: custom
        criteria:
          - {targetPath: "metadata.name", dependencyPath: "metadata.name", operator: EQUALS}
          - {targetPath: "metadata.namespace", dependencyPath: "metadata.namespace", operator: EQUALS}
```

### 10.8 ownership (`argocd/argocd-applicationset-generates-nothing`)
```yaml
  target: {kind: ApplicationSet, apiVersion: argoproj.io/v1alpha1, plural: applicationsets}
  dependencies:
    - {id: application, kind: Application, apiVersion: argoproj.io/v1alpha1, plural: applications}
  relationships:
    matchNone:
      - with: application
        type: owns
```

### 10.9 Less common constructs (fragments)
```yaml
# fromTarget: the dependency kind is read from the target (krateo-compositiondefinition-unused)
  target:
    kind: CompositionDefinition
    apiVersion: core.krateo.io/v1alpha1
    plural: compositiondefinitions
    filters:
      matchAll:
        - {path: status.kind, operator: EXISTS}          # only reconciled definitions
  dependencies:
    - id: composition
      fromTarget: {apiVersionPath: status.apiVersion, kindPath: status.kind, pluralPath: status.resource}
  relationships:
    matchNone:
      - with: composition
        type: custom
        criteria:
          - {targetPath: status.kind, dependencyPath: kind, operator: EQUALS}
          - {targetPath: status.apiVersion, dependencyPath: apiVersion, operator: EQUALS}
```
```yaml
# kind "*" and a regex on a string-encoded reference (krateo-restaction-not-used)
  dependencies:
    - {id: widget, kind: "*", apiVersion: widgets.templates.krateo.io/v1beta1}
    - {id: caller, kind: RESTAction, apiVersion: templates.krateo.io/v1, plural: restactions}
  relationships:
    matchNone:
      - with: widget
        type: custom
        criteria:
          - {targetPath: metadata.name, dependencyPath: spec.apiRef.name, operator: EQUALS}
          - {targetPath: metadata.namespace, dependencyPath: spec.apiRef.namespace, operator: EQUALS}
      - with: caller
        type: custom
        criteria:
          - targetPath: metadata.name
            dependencyPath: "spec.api[@].path"
            dependencyTransform: {regex: "resource=restactions&name=([^&]+)"}
            operator: EQUALS
          - targetPath: metadata.namespace
            dependencyPath: "spec.api[@].path"
            dependencyTransform: {regex: "resource=restactions&name=[^&]+&namespace=([^&]+)"}
            operator: EQUALS
```
```yaml
# several target kinds, quoted label key, {{target.kind}} (krateo-widget-manager-missing)
  message: "{{target.kind}} {{target.metadata.namespace}}/{{target.metadata.name}} is labelled managed-by a RESTAction or Table that does not exist in its namespace: nothing keeps it up to date."
  target:
    kinds:
      - {kind: Markdown, apiVersion: widgets.templates.krateo.io/v1beta1, plural: markdowns}
      - {kind: YamlViewer, apiVersion: widgets.templates.krateo.io/v1beta1, plural: yamlviewers}
      - {kind: RESTAction, apiVersion: templates.krateo.io/v1, plural: restactions}
    filters:
      matchAll:
        - {path: "metadata.labels['managed-by']", operator: EXISTS}
```
```yaml
# a literal on the dependency's bound entry (page-not-referenced)
          - {targetValue: pages, dependencyPath: "spec.resourcesRefs.items[@].resource", operator: EQUALS}
```
```yaml
# split: a component type pinned to a revision, "name@v1" (kubevela-componentdefinition-not-used)
          - targetPath: "metadata.name"
            dependencyPath: "spec.components[*].type"
            dependencyTransform: {split: {separator: "@", index: 0}}
            operator: EQUALS
```

## 11. A complete answer

> **User:** Find the Ingresses that send traffic to a Service that does not exist.

**Interpretation.** An `Ingress` is defective when one of its backends names a Service that does not exist in the Ingress's namespace. Family: `dangling`.

```yaml
apiVersion: kubepattern.dev/v1
kind: Pattern
metadata:
  name: ingress-backend-service-missing
  labels:
    kubepattern.dev/platform: kubernetes
    kubepattern.dev/family: dangling
  annotations:
    kubepattern.dev/summary: "An Ingress whose rule routes a path to a Service that does not exist in its namespace, so requests to that path fail."
spec:
  displayName: Ingress Backend Service Missing
  category: Reliability
  severity: HIGH
  message: "Ingress {{target.metadata.namespace}}/{{target.metadata.name}} routes a path to a Service that does not exist in its namespace: requests to that path fail."
  reference: "https://kubernetes.io/docs/concepts/services-networking/ingress/"
  for: 10m
  target:
    kind: Ingress
    apiVersion: networking.k8s.io/v1
    plural: ingresses
    filters:
      matchAll:
        - {path: "spec.rules[@].http.paths[@].backend.service.name", operator: EXISTS}
  dependencies:
    - {id: service, kind: Service, apiVersion: v1, plural: services}
  relationships:
    matchNone:
      - with: service
        type: custom
        criteria:
          - {targetPath: "spec.rules[@].http.paths[@].backend.service.name", dependencyPath: metadata.name, operator: EQUALS}
          - {targetPath: metadata.namespace, dependencyPath: metadata.namespace, operator: EQUALS}
---
apiVersion: kubepattern.dev/v1
kind: Pattern
metadata:
  name: ingress-default-backend-service-missing
  labels:
    kubepattern.dev/platform: kubernetes
    kubepattern.dev/family: dangling
  annotations:
    kubepattern.dev/summary: "An Ingress whose default backend is a Service that does not exist in its namespace, so unmatched requests fail."
spec:
  displayName: Ingress Default Backend Service Missing
  category: Reliability
  severity: MEDIUM
  message: "Ingress {{target.metadata.namespace}}/{{target.metadata.name}} has a default backend Service that does not exist in its namespace."
  reference: "https://kubernetes.io/docs/concepts/services-networking/ingress/#default-backend"
  for: 10m
  target:
    kind: Ingress
    apiVersion: networking.k8s.io/v1
    plural: ingresses
    filters:
      matchAll:
        - {path: "spec.defaultBackend.service.name", operator: EXISTS}
  dependencies:
    - {id: service, kind: Service, apiVersion: v1, plural: services}
  relationships:
    matchNone:
      - with: service
        type: custom
        criteria:
          - {targetPath: "spec.defaultBackend.service.name", dependencyPath: metadata.name, operator: EQUALS}
          - {targetPath: metadata.namespace, dependencyPath: metadata.namespace, operator: EQUALS}
```

**What it flags and what it does not.**
- **Flagged:** every Ingress with at least one rule path whose `backend.service.name` matches no Service in the Ingress's namespace. An Ingress backend can only name a Service in its own namespace.
- **Two Patterns.** The default backend needs its own Pattern: the first Pattern examines the target element by element, and a relationship that does not read `paths[@]` is not allowed in it.
- **Not flagged:** resource backends (`backend.resource`), and a Service that exists but does not expose the port the Ingress asks for. That would be a separate Pattern: a `matchAll` on the Service's name plus a `matchNone` on name and port.
- `for: 10m` hides the window in which a GitOps sync creates the Ingress before its Service.

**How to try it.** The commands of section 9 with `<pattern-name>` = `ingress-backend-service-missing`. A quick test: an Ingress that points to `missing-svc` must be flagged; one that points to an existing Service must not.

**Assumptions and versions.**

| Component | Version assumed | API used | Source |
|---|---|---|---|
| Kubernetes | ≥ 1.19 (Ingress `networking.k8s.io/v1` is GA there), up to `<latest release I know>` | `networking.k8s.io/v1` Ingress, `v1` Service | Kubernetes API reference, training data |
| KubePattern | engine `dev` (`1341079`) | `kubepattern.dev/v1` | this guide |

- **Paths:** `spec.rules[].http.paths[].backend.service.name`, `spec.defaultBackend.service.name`, `metadata.name`, `metadata.namespace`. All *from knowledge*; none was checked against your cluster.
- **Knowledge cutoff:** `<my cutoff>`. The fields above have been stable since `networking.k8s.io/v1`; check with `kubectl explain ingress.spec.rules.http.paths.backend`.
