# GitHub Reusable Workflows

Centralized GitHub Actions workflows for WordPress plugins and themes. Consuming repos call these via `uses:` instead of duplicating CI logic.

## Versions

Callers pin a major version tag, never `main`:

```yaml
uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v2
```

- `v2`, `v1`… move to the latest release of that major version, so callers get fixes without editing anything.
- `v2.0.0`, `v2.1.0`… are fixed and never move.
- A change that requires callers to edit their workflow (an input removed or renamed, a different zip) is a new major version. Anything else is a minor or patch release of the current one.

| Version | Changes |
| ------- | ------- |
| `v2`    | Composer and npm are detected from `composer.json` and `package.json`; `has_composer` and `has_package_json` have no effect (a warning asks to remove them). `npm run build` is skipped when `package.json` has no `build` script. |
| `v1`    | Composer and npm are enabled with `has_composer` and `has_package_json`. |

### Releasing a new version of the workflows

After merging into `main` (example for `v2.1.0`):

```bash
git switch main && git pull
git tag -a v2.1.0 -m "v2.1.0: what changed"
git tag -fa v2 -m "v2: latest v2.x release"
git push origin v2.1.0 && git push --force origin v2
```

For a new major version, create `v3.0.0` and `v3` instead and leave `v2` where it is.

---

### `wp-release.yml`

Builds a production zip and creates a GitHub Release with it attached. Works for **plugins and themes**. Triggered by a `v*` tag on the calling repo.

Installs Composer dependencies without `--dev` when the repo has a `composer.json`, installs npm dependencies and runs `npm run build` when it has a `package.json` (the build is skipped if there is no `build` script), then zips the result excluding all development files. The file is named `package-name-1.2.0.zip`; a leading `v` on the tag is dropped. The archive contains a single top-level directory named after the repository, without the version, which WordPress needs in order to install and update the package in place. A Composer release requires `composer.lock`; the job fails if it is missing. The workflow only publishes when the caller ref is a tag.

Theme `screenshot.png` at the repo root is included. The `screenshots/` directory (plugin directory assets) is excluded. `src/` is **not** excluded by default (many plugins ship PHP there via PSR-4). If `src/` is only front-end source (SCSS/JS) and the build outputs to e.g. `dist/`, pass it via `exclude`.

**Inputs:**

| Input              | Type    | Default | Description                                                     |
| ------------------ | ------- | ------- | --------------------------------------------------------------- |
| `has_composer`     | boolean | `false` | Deprecated, no effect since v2: Composer is detected from `composer.json` |
| `has_package_json` | boolean | `false` | Deprecated, no effect since v2: npm is detected from `package.json` |
| `php_version`      | string  | `"8.4"` | PHP version to use for Composer                                 |
| `node_version`     | string  | `"22"`  | Node.js version to use for npm                                  |
| `exclude`          | string  | `""`    | Extra rsync exclude patterns (one per line or comma-separated). Added to the default list. |

**Always excluded from the zip:**

`.git/` `.github/` `.vscode/` `.DS_Store` `tests/` `bin/` `tools/` `vendor/bin/` `vendor/composer/autoload_aliases.php` `node_modules/` `screenshots/` `*.md` `*.yml` `*.yaml` `.gitignore` `.env` `composer.json` `composer.lock` `package.json` `package-lock.json` `phpstan.neon` `phpunit.xml` `.php-cs-fixer.php` `.php-cs-fixer.cache` `vite.config.mjs`

**Usage — PHP plugin with no assets:**

```yaml
name: Build release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v2
    secrets: inherit
```

**Usage — theme with Vite assets (`src/` is front-end only):**

```yaml
name: Build release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v2
    with:
      exclude: |
        src/
    secrets: inherit
```

**Usage — PHP plugin with Composer and built assets:**

```yaml
name: Build release

on:
  push:
    tags:
      - "v*"

permissions:
  contents: write

jobs:
  release:
    uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v2
    with:
      exclude: |
        resources/
    secrets: inherit
```

## Required secrets

| Secret                      | Used by         | Description                                                              |
| --------------------------- | --------------- | ------------------------------------------------------------------------ |
| `LCF_COMPOSER_GITHUB_TOKEN` | `wp-release.yml` | PAT or fine-grained token with read access to private Composer repos |

Add `LCF_COMPOSER_GITHUB_TOKEN` as an Actions repository secret on each repo that calls this workflow and has a `composer.json`. The caller passes it with `secrets: inherit`.
