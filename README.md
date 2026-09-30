# workflows

Reusable [GitHub Actions workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) and
shareable issue templates for the repositories in the [geostyler](https://github.com/geostyler) organization.

This repository was created by comparing the workflows used in
[geostyler/geostyler](https://github.com/geostyler/geostyler),
[geostyler/geostyler-style](https://github.com/geostyler/geostyler-style),
[geostyler/geostyler-openlayers-parser](https://github.com/geostyler/geostyler-openlayers-parser),
[geostyler/geostyler-sld-parser](https://github.com/geostyler/geostyler-sld-parser),
[geostyler/geostyler-qgis-parser](https://github.com/geostyler/geostyler-qgis-parser) and
[geostyler/geostyler-cli](https://github.com/geostyler/geostyler-cli), and extracting what they have in common
into reusable workflows that can be called (`workflow_call`) from every repository instead of duplicating the
same YAML everywhere.

## Available reusable workflows

All workflows live in [`.github/workflows`](.github/workflows) and are triggered via `workflow_call`, so they
must be referenced with `uses:` from a caller workflow in the consuming repository, e.g.:

```yaml
# .github/workflows/commitlint.yml in a consuming repository
name: Lint Commit Messages
on: [pull_request, push]

jobs:
  commitlint:
    uses: geostyler/workflows/.github/workflows/reusable-commitlint.yml@main
    with:
      config-file: .commitlintrc.cjs
```

| Workflow | Replaces | Description |
| --- | --- | --- |
| [`reusable-commitlint.yml`](.github/workflows/reusable-commitlint.yml) | `commitlint.yml` | Lints commit messages with [wagoid/commitlint-github-action](https://github.com/wagoid/commitlint-github-action). Configurable `config-file` input, since repos use `.commitlintrc.js`, `.commitlintrc.cjs` or `.commitlintrc.mjs`. |
| [`reusable-notify-discord.yml`](.github/workflows/reusable-notify-discord.yml) | `notify-discord.yml` | Sends a release notification to Discord. Requires the `discord-webhook` secret and accepts a customizable `message`. |
| [`reusable-release.yml`](.github/workflows/reusable-release.yml) | `release.yml` | Builds and releases the package with [semantic-release](https://semantic-release.gitbook.io/). Supports both `npm`/`bun` (`package-manager` input) and either running `npx semantic-release` directly or via `cycjimmy/semantic-release-action` (`release-method` input). Outputs the released `tag_name` so it can be chained with `reusable-notify-discord.yml`. |
| [`reusable-stale.yml`](.github/workflows/reusable-stale.yml) | `stale.yml` | Marks and closes inactive issues/PRs with [actions/stale](https://github.com/actions/stale). All thresholds and labels are configurable inputs. |
| [`reusable-on-pull-request.yml`](.github/workflows/reusable-on-pull-request.yml) | `on-pullrequest.yml` (and `on-push-main.yml`) | Installs dependencies, lints, type-checks, tests and builds the project. Supports `npm`/`bun`, a configurable Node.js version matrix, custom commands per step and optional Coveralls reporting. Can be called from both `pull_request` and `push` triggers. |
| [`reusable-docs-deploy.yml`](.github/workflows/reusable-docs-deploy.yml) | `on-publish.yml` / docs job of `on-push-main.yml` | Extra shared workflow found while comparing the repositories: builds the documentation/styleguide and deploys it to the `gh-pages` branch with [JamesIves/github-pages-deploy-action](https://github.com/JamesIves/github-pages-deploy-action). |

### Example: full release pipeline

Since `reusable-release.yml` exposes the released `tag_name` as an output, a caller workflow can chain it with
the Discord notification, matching the pattern already used in some of the geostyler repositories:

```yaml
name: Release
on:
  workflow_dispatch:
  push:
    branches: [next]

jobs:
  release:
    uses: geostyler/workflows/.github/workflows/reusable-release.yml@main
    with:
      package-manager: bun
      release-method: action
    secrets:
      github-token: ${{ secrets.GH_RELEASE_TOKEN }}
      npm-token: ${{ secrets.NPM_TOKEN }}

  notify-discord:
    needs: release
    if: needs.release.outputs.tag_name != ''
    uses: geostyler/workflows/.github/workflows/reusable-notify-discord.yml@main
    secrets:
      discord-webhook: ${{ secrets.DISCORD_WEBHOOK }}
```

### Example: scheduled stale bot

`reusable-stale.yml` only reacts to `workflow_call`, so the calling repository still needs a small wrapper
workflow with the actual `schedule` trigger:

```yaml
name: Close inactive issues
on:
  schedule:
    - cron: "30 1 * * *"

jobs:
  close-issues:
    uses: geostyler/workflows/.github/workflows/reusable-stale.yml@main
```

## Shareable issue templates

GitHub does not support referencing issue templates from another repository directly (the only built-in reuse
mechanism is the organization-wide [`.github` health-file repository](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file-for-your-organization),
which only applies to repositories that don't define their own templates). To still avoid re-writing the same
templates in every repository, ready-to-copy templates are provided in [`templates/ISSUE_TEMPLATE`](templates/ISSUE_TEMPLATE):

- `bug_report.md`
- `feature_request.md`
- `question.md`

Copy the ones you need into `.github/ISSUE_TEMPLATE/` of the target repository and adjust labels/assignees as
required.

## Versioning

Consuming workflows should pin to a specific tag or commit SHA of this repository (e.g.
`geostyler/workflows/.github/workflows/reusable-release.yml@v1`) instead of `@main` once this repository starts
tagging releases, to avoid unexpected breaking changes.
