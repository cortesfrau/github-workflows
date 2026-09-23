# GitHub Reusable Workflows

Centralized GitHub Actions workflows for the Blimburn Seeds plugin ecosystem. Consuming repos call these via `uses:` instead of duplicating CI logic.

**Usage:**

```yaml
jobs:
  tests:
    uses: cortesfrau/github-workflows/.github/workflows/plugin-tests.yml@main
    secrets: inherit
```

---

### `plugin-release.yml`

Builds a production zip and creates a GitHub Release with it attached. Triggered by a `v*` tag on the calling repo.

Installs dependencies without `--dev`, runs the asset build if applicable, then zips the result excluding all development files. The file is named `plugin-name-1.2.0.zip`; a leading `v` on the tag is dropped. The archive contains a single top-level directory named after the repository, without the version, which WordPress needs in order to install and update the plugin. A Composer release requires `composer.lock`; the job fails if it is missing. The workflow only publishes when the caller ref is a tag.

**Inputs:**

| Input              | Type    | Default | Description                                                     |
| ------------------ | ------- | ------- | --------------------------------------------------------------- |
| `has_composer`     | boolean | `false` | Install Composer dependencies from `composer.lock` (`--no-dev`) before zipping |
| `has_package_json` | boolean | `false` | Install npm dependencies and run `npm run build` before zipping |
| `php_version`      | string  | `"8.4"` | PHP version to use for Composer                                 |
| `node_version`     | string  | `"22"`  | Node.js version to use for npm                                  |
| `exclude`          | string  | `""`    | Extra rsync exclude patterns (one per line or comma-separated). Added to the default list. |

**Always excluded from the zip:**

`.git/` `.github/` `.vscode/` `tests/` `bin/` `tools/` `vendor/bin/` `vendor/composer/autoload_aliases.php` `node_modules/` `screenshots/` `*.md` `*.yml` `*.yaml` `.gitignore` `.env` `composer.json` `composer.lock` `package.json` `package-lock.json` `phpstan.neon` `phpunit.xml` `.php-cs-fixer.php` `.php-cs-fixer.cache` `vite.config.mjs`

**Usage — PHP plugin with no assets:**

```yaml
name: Build plugin dist

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    uses: blimburn/github-workflows/.github/workflows/plugin-release.yml@main
    with:
      has_composer: true
      has_package_json: false
    secrets: inherit
```

**Usage — PHP plugin with built assets:**

```yaml
name: Build plugin dist

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  tests:
    uses: blimburn/github-workflows/.github/workflows/plugin-tests.yml@main
    secrets: inherit

  release:
    needs: tests
    uses: blimburn/github-workflows/.github/workflows/plugin-release.yml@main
    with:
      has_composer: true
      has_package_json: true
      exclude: |
        resources/
    secrets: inherit
```

## Required secrets

| Secret                 | Used by                              | Description                                                     |
| ---------------------- | ------------------------------------ | --------------------------------------------------------------- |
| `LCF_COMPOSER_GITHUB_TOKEN` | `plugin-tests.yml`, `plugin-release.yml` | PAT or fine-grained token with read access to private Blimburn repos |

Add `LCF_COMPOSER_GITHUB_TOKEN` as an Actions repository secret on each plugin repo that calls these workflows. The caller passes it with `secrets: inherit`.
