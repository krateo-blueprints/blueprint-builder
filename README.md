# blueprint-page-template

A starting point for a **Krateo blueprint** — a Helm chart that core-provider turns into a CRD, so
a claim of that Kind provisions everything the chart declares.

Scaffold a new repository from this one (the Blueprint Builder does it for you via the publish
form's *Scaffold from* field), rename the chart, replace the example resource, tag a release.

## What you must change

| | |
|---|---|
| `chart/Chart.yaml` `name:` | **becomes the CRD Kind** — `my-blueprint` → `MyBlueprint`, plural `myblueprints` |
| `chart/values.schema.json` | **is** the generated CRD's `spec`. No schema, no CRD, not installable |
| `chart/values.yaml` | defaults, kept in step with the schema |
| `chart/templates/` | what the blueprint actually provisions — delete the example |
| `compositiondefinition.yaml` | the `name`, `namespace` and chart `url` |

Leave `version: CHART_VERSION` alone in **both** `Chart.yaml` and `compositiondefinition.yaml`:
the release workflow substitutes it from the git tag.

## The schema is the API

`values.schema.json` is not documentation. core-provider reads it and the JSON schema *becomes* the
generated CRD's `spec` — so a value with no schema entry cannot be set on a claim at all, and a
chart with no schema never becomes a CRD. Write it first.

Two consequences worth knowing before you hit them:

- **The chart version is the API version.** `1.2.0` is served as `composition.krateo.io/v1-2-0`, and
  each served version carries its own `composition-dynamic-controller` Deployment — 50m CPU and
  128Mi of requests. Version deliberately; do not bump per edit.
- **A non-empty `default` at any level** of the schema is a known CRD-generation hazard. The portal's
  blueprint preview lints for it, and the Krateo docs cover the rule.

## From repo to installed blueprint

1. **Publish** — commit the chart. The Blueprint Builder does this through a `BuilderPublish` claim,
   which creates the repository if it does not exist (existing ones are adopted, not re-created)
   and opens a change request.
2. **Release** — tag `X.Y.Z`. `.github/workflows/release-tag.yaml` discovers every `Chart.yaml` in
   the repo, substitutes `CHART_VERSION`, and pushes each chart to
   `oci://ghcr.io/<owner>/charts/<chart name>:<tag>`. It also stamps `compositiondefinition.yaml`
   with the tag and **attaches it to the GitHub release**.
3. **Register** — apply the stamped `CompositionDefinition` from the release:

   ```bash
   kubectl create namespace my-blueprint-system
   kubectl apply -f https://github.com/<owner>/<repo>/releases/download/<tag>/compositiondefinition.yaml
   ```

   Apply the copy from the **release**, not from the branch — the branch still carries the
   `CHART_VERSION` placeholder, and registering that gets you a chart version that does not exist.

4. **Install** — create a claim of the generated Kind, from the portal or with `kubectl`.

## Rendering locally

`helm template` refuses the `CHART_VERSION` placeholder, so pass a version:

```bash
helm template example chart --namespace my-blueprint-system --version 0.1.0
```
