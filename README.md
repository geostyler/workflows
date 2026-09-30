# workflows

Reusable [GitHub Actions workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) for
the repositories in the [geostyler](https://github.com/geostyler) organization.

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
must be referenced with `uses:` from a caller workflow in the consuming repository.

| Workflow | Replaces | Description |
| --- | --- | --- |
| [`commitlint.yml`](.github/workflows/commitlint.yml) | `commitlint.yml` | Lints commit messages with [wagoid/commitlint-github-action](https://github.com/wagoid/commitlint-github-action). Configurable `config-file` input, since repos use `.commitlintrc.js`, `.commitlintrc.cjs` or `.commitlintrc.mjs`. |
| [`notify-discord.yml`](.github/workflows/notify-discord.yml) | `notify-discord.yml` | Sends a release notification to Discord. Requires the `discord-webhook` secret and accepts a customizable `message`. |
| [`release.yml`](.github/workflows/release.yml) | `release.yml` | Builds and releases the package with [semantic-release](https://semantic-release.gitbook.io/). Supports both `npm`/`bun` (`package-manager` input) and either running `npx semantic-release` directly or via `cycjimmy/semantic-release-action` (`release-method` input). Outputs the released `tag_name` so it can be chained with `notify-discord.yml`. |
| [`stale.yml`](.github/workflows/stale.yml) | `stale.yml` | Marks and closes inactive issues/PRs with [actions/stale](https://github.com/actions/stale). All thresholds and labels are configurable inputs. |
| [`on-pull-request.yml`](.github/workflows/on-pull-request.yml) | `on-pullrequest.yml` (and `on-push-main.yml`) | Installs dependencies, lints, type-checks, tests and builds the project. Supports `npm`/`bun` with smart per-package-manager command defaults (just switch `package-manager` to `bun` and everything else adapts), a configurable Node.js version matrix, overridable commands per step (or `skip` to omit a step) and optional Coveralls reporting. Can be called from both `pull_request` and `push` triggers. |

### Example: commitlint

```yaml
# .github/workflows/commitlint.yml in a consuming repository
name: Lint Commit Messages
on: [pull_request, push]

jobs:
  commitlint:
    uses: geostyler/workflows/.github/workflows/commitlint.yml@main
    with:
      config-file: .commitlintrc.cjs
```

### Example: notify-discord

```yaml
# .github/workflows/notify-discord.yml in a consuming repository
name: Discord notification
on:
  release:
    types: [published]

jobs:
  discord-notification:
    uses: geostyler/workflows/.github/workflows/notify-discord.yml@main
    with:
      message: '${{ github.event.repository.name }} [${{ github.event.release.tag_name }}](${{ github.event.release.html_url }}) has been released. 🚀'
    secrets:
      discord-webhook: ${{ secrets.DISCORD_WEBHOOK }}
```

### Example: release

`release.yml` exposes the released `tag_name` and `release_url` as outputs, so a caller workflow can chain it with the
Discord notification, matching the pattern already used in some of the geostyler repositories:

```yaml
# .github/workflows/release.yml in a consuming repository
name: Release
on:
  workflow_dispatch:
  push:
    branches: [next]

jobs:
  release:
    uses: geostyler/workflows/.github/workflows/release.yml@main
    with:
      package-manager: bun
      release-method: action
    secrets:
      github-token: ${{ secrets.GH_RELEASE_TOKEN }}
      npm-token: ${{ secrets.NPM_TOKEN }}

  notify-discord:
    needs: release
    if: needs.release.outputs.tag_name != ''
    uses: geostyler/workflows/.github/workflows/notify-discord.yml@main
    with:
      message: '${{ github.repository }} [${{ needs.release.outputs.tag_name }}](${{ needs.release.outputs.release_url }}) has been released. 🚀'
    secrets:
      discord-webhook: ${{ secrets.DISCORD_WEBHOOK }}
```

### Example: stale

`stale.yml` only reacts to `workflow_call`, so the calling repository still needs a small wrapper workflow with
the actual `schedule` trigger:

```yaml
# .github/workflows/stale.yml in a consuming repository
name: Close inactive issues
on:
  schedule:
    - cron: "30 1 * * *"

jobs:
  close-issues:
    uses: geostyler/workflows/.github/workflows/stale.yml@main
```

### Example: on-pull-request

```yaml
# .github/workflows/on-pull-request.yml in a consuming repository
name: Test build
on: pull_request

jobs:
  build:
    uses: geostyler/workflows/.github/workflows/on-pull-request.yml@main
    with:
      node-versions: '["22", "24"]'
      package-manager: bun
```

Switching `package-manager` alone is enough: `install-command`, `lint-command`, `test-command` and
`build-command` all fall back to smart per-package-manager defaults (e.g. `bun run lint` instead of
`npm run lint`) when left unset. Override any of them individually if a repository needs a different command,
or set one to `skip` to omit that step entirely.

## Versioning

Consuming workflows should pin to a specific tag or commit SHA of this repository (e.g.
`geostyler/workflows/.github/workflows/release.yml@v1`) instead of `@main` once this repository starts
tagging releases, to avoid unexpected breaking changes.
