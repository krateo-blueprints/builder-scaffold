# builder-scaffold

The seed for every repository a Krateo builder publishes to. The Blueprint Composer and the Portal
Builder both point at it (`AUTOPILOT_BLUEPRINT_BUILDER_TEMPLATE` and `AUTOPILOT_PAGE_BUILDER_TEMPLATE`
in the frontend chart). On a first publish, the builder creates the destination repository and copies
this one into it whole. The chart itself arrives afterwards, as the builder's pull request.

It holds no chart, no CompositionDefinition and no `.krateoignore`. The builder copies it onto the
pull request's branch (`builder/<name>`) before it commits the person's files there, and it never
overwrites a file that already exists — so a chart or a registration file in the scaffold would shadow
the one the person actually built. `main` receives none of it until the pull request is merged.

## What a seeded repository gets

| File | Why |
| --- | --- |
| `.github/workflows/release.yaml` | Releases the chart. It is a thin wrapper around `krateo-platformops/.github` `release-oci.yaml@main` and `krateo-blueprints/charts` `publish-chart.yaml@main`, because a seeded repository never receives scaffold updates. Packaging and index fixes land in those workflows and reach every repository from there. |
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

**Blueprints release on merge.** A blueprint's `Chart.yaml` carries a literal version, for example
`0.1.0` or a lower-case pre-release such as `0.2.0-rc.1`, and the composer shows its API version as
`v0-1-0` (or `v0-2-0-rc-1`). When the pull request is merged, the workflow:

1. tags that commit with the version,
2. packages and pushes the chart at that version,
3. creates a GitHub release with `compositiondefinition.yaml` attached, and
4. adds the chart to the Marketplace's blueprints index (see below).

This happens once per version. A merge that leaves the version unchanged releases nothing, and the
pull request's check warns about it beforehand. To release again, bump the version in `Chart.yaml`.

**Page sets release by tag.** A page set's `Chart.yaml` carries the `CHART_VERSION` placeholder, so a
merge releases nothing. Push a version tag (`0.1.0`, or a lower-case pre-release such as `0.1.0-rc.1`)
on a commit that is on `main`:

```sh
git tag 0.1.0 origin/main && git push origin 0.1.0
```

The release stamps the tag into `Chart.yaml` and into the `compositiondefinition.yaml` attached to the
GitHub release. The copy on `main` keeps the placeholder.

A tag on a literal-version chart must match `Chart.yaml`. Anything else is refused, because the
package would then serve a different API version from the one the composer showed. Upper case and
build metadata (`0.1.0-RC1`, `0.1.0+build`) are refused in a tag and in `Chart.yaml` alike: the
composer refuses them too, because Kubernetes refuses them in the API version.

Every pull request to `main` runs the same checks and releases nothing.

A manual run (**Run workflow**) releases `main` by the same rules as a merge. On any other branch or
tag it is refused: it would tag and publish a commit nobody reviewed, and the merge would then find
its version already released.

If the chart push fails after the tag has been cut, re-run the failed jobs of that same run. A new
run finds the tag and treats the version as already released.

## The Marketplace

The portal's Marketplace lists the blueprints index at
`https://krateo-blueprints.github.io/charts/blueprints/index.yaml`. After the push, the release
packages the chart once more for that index and hands it to `krateo-blueprints/charts`
`publish-chart.yaml@main`, which stores the package in that repository's releases and merges its
entry into the index. The portal reads the live index beside its curated catalog, so the blueprint
gets a card, and an Install that pre-fills it, without a new catalog release.

That package differs from the OCI one in two ways only: it carries `compositiondefinition.yaml`
(stamped with the version), because the index takes only full blueprints, and its `Chart.yaml`
carries `krateo.io/source-repo: <owner>/<repo>`. The index treats that annotation as the name's
owner, and the portal lists a live-index chart only when it carries one. Every other annotation,
`krateo.io/category` included, is the one committed in `Chart.yaml`; none is added when absent, so
a chart with no category shows none.

The chart is not added, and a warning in the run says why, when:

- no token may write to `krateo-blueprints/charts`. The release uses the org secret
  `CHARTS_PUBLISH_TOKEN`, or `RELEASE_FEED_TOKEN` when that is not set, and tries it read-only
  first: a token that is missing (any repository outside the `krateo-blueprints` org), refused, or
  without push there is a warning, not a red run,
- the chart is a page set (`CHART_VERSION`): the Marketplace lists blueprints,
- `values.schema.json`, `compositiondefinition.yaml` or an `https://` `icon` in `Chart.yaml` is
  missing, which the index refuses, or
- the index already has a chart of that name from anywhere else. Rename the chart to list it.

None of these fails the release. The chart is already pushed and registrable from the portal. A
refusal the read-only check cannot foresee (a protected `gh-pages`, a name taken between the check
and the merge) still fails the `index` job: it calls `publish-chart.yaml` as a reusable workflow,
and GitHub does not allow `continue-on-error` on such a job. Re-run it once the cause is fixed.

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
