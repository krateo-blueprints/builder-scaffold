# builder-scaffold

The seed for every repository a Krateo builder publishes to. The Blueprint Composer and the Portal
Builder both point at it (`AUTOPILOT_BLUEPRINT_BUILDER_TEMPLATE` and `AUTOPILOT_PAGE_BUILDER_TEMPLATE`
in the frontend chart). On a first publish, the builder creates the destination repository and copies
this one into it whole. The chart itself arrives afterwards, as the builder's pull request.

It holds no chart, no CompositionDefinition and no `.krateoignore`. Anything seeded here would land on
the destination's `main` before the pull request does, and the builder never overwrites a file that
already exists. So a chart or a registration file in the scaffold would shadow the one the person
actually built.

## What a seeded repository gets

| File | Why |
| --- | --- |
| `.github/workflows/release.yaml` | Releases the chart. It is a thin wrapper around `krateo-platformops/.github` `release-oci.yaml@main`, because a seeded repository never receives scaffold updates. Packaging fixes land in the org workflow and reach every repository from there. |
| `.helmignore` | The chart sits at the repository root, so without it the package would carry `.git/`, `.github/` and the other repository files. |
| `.gitignore` | Keeps build output (`charts/`, `dist/`, `*.tgz`) out of git. |

This README is not copied. The destination repository is created with its own README, and the copy
skips files that already exist.

## One chart, at the root

A builder repository holds exactly one chart, and it sits at the root: `Chart.yaml`, `values.yaml`,
`values.schema.json`, `templates/`. Its registration file sits next to it as
`compositiondefinition.yaml`. The release refuses a second chart. It also refuses a registration file
that names another chart, points at another package, or pins another version.

The package is published as `oci://ghcr.io/<owner>/charts/<chart name>`.

## How a release happens

**Blueprints release on merge.** A blueprint's `Chart.yaml` carries a literal version (for example
`0.1.0`), and the composer shows its API version as `v0.1.0`. When the pull request is merged, the
workflow:

1. tags that commit with the version,
2. packages and pushes the chart at that version, and
3. creates a GitHub release with `compositiondefinition.yaml` attached.

This happens once per version. A merge that leaves the version unchanged releases nothing, and the
pull request's check warns about it beforehand. To release again, bump the version in `Chart.yaml`.

**Page sets release by tag.** A page set's `Chart.yaml` carries the `CHART_VERSION` placeholder, so a
merge releases nothing. Push a semver tag on a commit that is on `main`:

```sh
git tag 0.1.0 origin/main && git push origin 0.1.0
```

The release stamps the tag into `Chart.yaml` and into the `compositiondefinition.yaml` attached to the
GitHub release. The copy on `main` keeps the placeholder.

A tag on a literal-version chart must match `Chart.yaml`. Anything else is refused, because the
package would then serve a different API version from the one the composer showed.

Every pull request to `main` runs the same checks and releases nothing.

If the chart push fails after the tag has been cut, re-run the failed jobs of that same run. A new
run finds the tag and treats the version as already released.

## Registering the chart

A released chart is not installable until a CompositionDefinition registers it. A person does that
from the portal. Open the builder's deliverables list, then the chart's row, then **Register**. This
opens the install form (`/marketplace/<chart name>/install`) pre-filled with the package URL and
version from the publish. Review the values and submit.

Autopilot does not register charts. It hands the person to this step.

The package gets its repository's visibility. For a private repository the package is private too,
and registering it needs `chart.credentials`, which the install form exposes.

## This repository

This repository runs its own `release.yaml`. With no `Chart.yaml` at the root, the workflow reports
"no Chart.yaml at the root yet" and releases nothing. A freshly seeded repository takes the same path
before its first pull request merges.

A change here reaches only repositories seeded after it.
