# GitHub Reusable Workflows

Centralized GitHub Actions workflows for WordPress plugins and themes. Consuming repos call these via `uses:` instead of duplicating CI logic.

## Versions

Callers pin a major version tag, never `main`:

```yaml
uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v3
```

- `v3`, `v2`… move to the latest release of that major version, so callers get fixes without editing anything.
- `v3.0.0`, `v3.1.0`… are fixed and never move. Pin one of these (or a commit SHA) to opt out of updates.

What changed in each version is in [CHANGELOG.md](CHANGELOG.md).

## Workflows

### `wp-release.yml`

Builds a production zip and creates a GitHub Release with it attached. Works for **plugins and themes**. Triggered by a `v*` tag on the calling repo.

1. Fails unless the caller ref is a tag, the tagged commit is on the repo's default branch (usually `main`) and the repo has a `.distignore`. Tag from the default branch after merging; a tag pushed from another branch does not publish.
2. With `composer.json`: requires `composer.lock` and runs `composer install --no-dev`.
3. With `package.json`: requires `package-lock.json`, sets up Node.js (`node_version` input, else `.nvmrc`, else 24), runs `npm ci` and `npm run build` (skipped if there is no `build` script).
4. Copies the repo into the package folder, leaving out what `.distignore` and the safety net exclude.
5. Validates the package (see below).
6. Zips it as `package-name-1.2.0.zip` (a leading `v` on the tag is dropped). The archive contains a single top-level directory named after the repository, without the version, which WordPress needs to install and update the package in place.
7. Publishes a GitHub Release with the zip, marked as pre-release when the tag contains `-`. The job summary lists the files in the zip.

**Inputs:**

| Input          | Type   | Default | Description                                                                                                              |
| -------------- | ------ | ------- | ------------------------------------------------------------------------------------------------------------------------ |
| `php_version`  | string | `"8.4"` | PHP version to use for Composer                                                                                          |
| `node_version` | string | `""`    | Node.js version to use for npm. When empty, the version in the repo's `.nvmrc` is used, or `24` if there is no `.nvmrc`. |

#### What goes into the zip: `.distignore`

Every calling repo has a `.distignore` at its root listing everything that is **not** shipped. It is the single source of truth: reading it tells you exactly what the zip contains. The file name follows the WordPress convention also used by `wp dist-archive`.

Start from [`templates/.distignore`](templates/.distignore), the same template for plugins and themes, and uncomment or add what is specific to the project.

The syntax is rsync exclude patterns. **A leading `/` anchors the pattern to the repo root; without it, the pattern matches at any depth, including inside `vendor/`.** `src/` would remove `vendor/*/src/` and break every PSR-4 dependency, so use `/src/`. Leave a pattern unanchored only when you mean "everywhere" (e.g. `*.map`).

Criteria for the template: a pattern is active in the template only if no plugin or theme ever needs it at runtime and its name is a convention of the ecosystem (git, Composer, npm, WordPress, QA tools). Anything that depends on the project (`/src/`, Strauss's `/tools/`, `/screenshots/`) is left commented out or added per repo. The template never excludes `LICENSE*` files.

#### Safety net

Regardless of `.distignore`, the workflow always leaves out files that leak data or never belong to a WordPress package:

`.git/` `.env` `.env.*` `/auth.json` `node_modules/` `.DS_Store` `Thumbs.db`

This list stays short on purpose. Standard dev files (`composer.json`, `phpunit.xml`…) are not in it: they are the job of `.distignore`.

#### Validation

Before publishing, the job fails if:

- The package root has no PHP file with a `Plugin Name:` header and no `style.css` with a `Theme Name:` header.
- The `Version:` header of that file does not match the tag without its leading `v` (`v1.2.0` → `1.2.0`, `v1.2.0-beta.1` → `1.2.0-beta.1`).
- `composer.json` defines `autoload` but `vendor/autoload.php` is not in the package.

It also emits a warning for each standard dev file found in the package root (`composer.json`, `phpunit.xml*`, `vite.config.*`, `tests/`…), which usually means `.distignore` is missing a line or is older than the template.

**Usage:**

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
    uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v3
    secrets: inherit
```

With a different PHP or Node.js version (for Node.js, a `.nvmrc` in the repo is preferred):

```yaml
uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v3
with:
  php_version: "8.3"
  node_version: "22"
secrets: inherit
```

#### Migrating from v2

1. Copy `templates/.distignore` to the repo root.
2. Move what you passed in `exclude:` into `.distignore`, anchored with `/` (`src/` → `/src/`).
3. Strauss projects: uncomment `/tools/` and `/vendor/composer/autoload_aliases.php`. Repos with `screenshots/`: uncomment `/screenshots/`. v2 excluded these by default.
4. Remove `exclude`, `has_composer` and `has_package_json` from the caller. v3 does not accept them and the run fails if they are passed.
5. Make sure `package-lock.json` is committed if the repo has a `package.json`.
6. Check the Node.js version: it now comes from `.nvmrc` when the repo has one, or defaults to 24 (was 22). Prefer committing a `.nvmrc` over passing `node_version`, so CI builds with the same version as local development.
7. Tag releases from the default branch only. v3 refuses tags whose commit is not on it.
8. Change `@v2` to `@v3` and check the job summary of the first release.

## Required secrets

| Secret                      | Used by          | Description                                                          |
| --------------------------- | ---------------- | -------------------------------------------------------------------- |
| `LCF_COMPOSER_GITHUB_TOKEN` | `wp-release.yml` | PAT or fine-grained token with read access to private Composer repos |

Add `LCF_COMPOSER_GITHUB_TOKEN` as an Actions repository secret on each repo that calls this workflow and has a `composer.json`. The caller passes it with `secrets: inherit`.

## Maintaining this repository

### Versioning rule

A change that requires callers to edit their workflow (an input removed or renamed, a new required file) or changes the zip they get is a new major version. Anything else is a minor or patch release of the current one.

### Third-party actions

Actions used by the workflows are pinned to a commit SHA, with the version in a comment (`uses: actions/checkout@<sha> # v7.0.1`). A tag like `@v7` can be moved by its owner, or by anyone who compromises their account, and the code runs with `contents: write` and the repo secrets; a SHA cannot change. Dependabot (`.github/dependabot.yml`) checks weekly and opens a single PR updating SHA and comment together. Review the release notes before merging it, and never replace a SHA with a tag.

### Releasing a new version

Every pull request that changes a workflow adds its entry under `## [Unreleased]` in `CHANGELOG.md`.

When releasing (example for `v3.1.0`):

1. Rename `## [Unreleased]` to `## [3.1.0] - YYYY-MM-DD`, add a new empty `## [Unreleased]` above it and update the compare links at the bottom. Do it in the pull request when it is the release, or in a commit on `main` before tagging.
2. After merging, tag `main` and move the major tag:

```bash
git switch main && git pull
git tag -a v3.1.0 -m "v3.1.0: what changed"
git tag -fa v3 -m "v3: latest v3.x release"
git push origin v3.1.0 && git push --force origin v3
```

For a new major version, create `v4.0.0` and `v4` instead and leave `v3` where it is.
