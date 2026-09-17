# blueprint-builder

A starting point for a **Krateo blueprint** — a Helm chart that core-provider turns into a CRD, so
a claim of that Kind provisions everything the chart declares.

Scaffold a new repository from this one (the Blueprint Builder does it for you via the publish
form's *Scaffold from* field), rename the chart, replace the example resource, tag a release.

## What you must change

| | |
|---|---|
| `chart/Chart.yaml` `name:` | **becomes the CRD Kind** — `blueprint-builder` → `BlueprintBuilder`, plural `blueprintbuilders` |
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
   kubectl create namespace blueprint-builder-system
   kubectl apply -f https://github.com/<owner>/<repo>/releases/download/<tag>/compositiondefinition.yaml
   ```

   Apply the copy from the **release**, not from the branch — the branch still carries the
   `CHART_VERSION` placeholder, and registering that gets you a chart version that does not exist.

4. **Install** — create a claim of the generated Kind, from the portal or with `kubectl`.

## Rendering locally

`helm template` refuses the `CHART_VERSION` placeholder, so pass a version:

```bash
helm template example chart --namespace blueprint-builder-system --version 0.1.0
```

## Publishing a blueprint others can install

The release workflow packages every chart here and pushes it to
`oci://ghcr.io/<owner>/charts/<chart>:<tag>` on a semver tag, and attaches the stamped
`CompositionDefinition` to the GitHub release.

**The repository must be PUBLIC for anyone else to install the result.** A GHCR package inherits the
visibility of the repository that published it, and there is no API to change it afterwards — both
`PATCH /orgs/{org}/packages/container/{pkg}/visibility` and `PATCH .../{pkg}` return 404. Published
from a private repo, the chart pushes fine and then fails at install with `unauthorized` for everyone
outside the org. The release workflow checks this and writes a warning into the job summary rather
than letting it be discovered by the person it fails for.

The loop this closes: author a blueprint -> publish it -> the tag builds a public OCI chart ->
register the `CompositionDefinition` -> it appears in the marketplace -> someone else installs it.

## `.krateoignore`

When this repo SEEDS another (a builder publish sets `source.url`), git-provider reads
`.krateoignore` from the root and skips what it lists. `chart/` and `README.md` are excluded: a
seeded repo receives its own composed chart, and copying the example in beside it would make the
release workflow publish two charts and ship the example ConfigMap to whoever installs the result.

GitHub's own "Use this template" ignores that file, so the hand-authored path still gets the complete
example.
