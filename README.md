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
| `.github/workflows/release.yaml` | Releases the chart. It is a thin wrapper around `krateo-platformops/.github` `release-oci.yaml@main`, which packages and pushes, and, in `krateo-blueprints`, `krateo-blueprints/charts` `publish-chart.yaml`, which adds the chart to the organisation's helm index. A seeded repository never receives scaffold updates, so packaging fixes land in release-oci and reach every repository from there. The publisher is pinned to a release, not a branch, because it receives an org-wide write token. |
| `.github/dependabot.yml` | Proposes each new publisher release, and other action upgrades, as a pull request. |
| `.helmignore` | The chart sits at the repository root, so without it the package would carry `.git/`, `.github/` and the other repository files. It does not exclude `compositiondefinition.yaml`, which ships inside the package. |
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
2. packages and pushes the chart at that version, with its `compositiondefinition.yaml` inside,
3. creates a GitHub release with `compositiondefinition.yaml` attached, and
4. in `krateo-blueprints`, adds the chart to the organisation's helm index (see below).

This happens once per version. A merge that leaves the version unchanged releases nothing, and the
pull request's check warns about it beforehand. To release again, bump the version in `Chart.yaml`.

**Page sets release by tag.** A page set's `Chart.yaml` carries the `CHART_VERSION` placeholder, so a
merge releases nothing. Push a version tag (`0.1.0`, or a lower-case pre-release such as `0.1.0-rc.1`)
on a commit that is on `main`:

```sh
git tag 0.1.0 origin/main && git push origin 0.1.0
```

The release stamps the tag into `Chart.yaml` and into `compositiondefinition.yaml`, both the copy in
the package and the one attached to the GitHub release. The copy on `main` keeps the placeholder.
A page set also needs `annotations: krateo.io/kind: page-set` in `Chart.yaml` (see below).

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

## The helm index and the Marketplace

In `krateo-blueprints`, every released chart is also added to the organisation's helm index, which
the portal's Marketplace installs from. The rules are krateo-blueprints/charts
[PUBLISHING.md](https://github.com/krateo-blueprints/charts/blob/main/PUBLISHING.md). A repository
this scaffold seeded in any other account, or on a self-hosted SCM, releases to its own registry and
skips everything in this section.

**Which channel.** The content decides, by the publisher's rule:

| The chart has | Channel | On the Marketplace |
| --- | --- | --- |
| `compositiondefinition.yaml` and `values.schema.json` | `blueprints` | yes, as a card |
| the same, and `annotations: krateo.io/kind: page-set` in `Chart.yaml` | `pages` | no |
| no `compositiondefinition.yaml` | `operators` | no |

**What `Chart.yaml` must carry.** The release refuses a chart without these, on the pull request
already, so a merge never releases a chart the index then turns away:

- `sources:` listing `https://github.com/<owner>/<repo>`, this repository;
- `annotations: krateo.io/source-repo: <owner>/<repo>`. It names the chart's owner in the index,
  and the portal gives a live-index chart a Marketplace card only when it carries one;
- for `blueprints` and `pages`: an `https://` `icon`, and `keywords:`, which the Marketplace shows
  as tags.

A chart with the `CHART_VERSION` placeholder and a registration file must also carry
`krateo.io/kind: page-set`. Otherwise it would be listed on the Marketplace as a blueprint.

**How it gets there.** Before anything is tagged or pushed, the release checks that the org secret
`CHARTS_PUBLISH_TOKEN` can write to `krateo-blueprints/charts`. A missing or dead token fails the
release there. After the push, the release pulls the archive back from GHCR, the same bytes, with
the stamped `compositiondefinition.yaml` inside, and hands it to `publish-chart.yaml`. That workflow
stores it, merges it into the channel's index, and succeeds only once the served index lists it
and an anonymous `helm pull` installs it. A failure anywhere in this section fails the release.

A version published before release-oci shipped the registration file inside the package has an
archive without one. That version is never replaced, so it cannot be listed. Release a new version.

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
