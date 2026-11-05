# GitHub Reusable Workflows

Centralized GitHub Actions workflows for WordPress plugins and themes. Consuming repos call these via `uses:` instead of duplicating CI logic.

## Versions

Callers pin a major version tag, never `main`:

```yaml
uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v4
```

- `v4`, `v3`… move to the latest release of that major version, so callers get fixes without editing anything.
- `v4.0.0`, `v4.1.0`… are fixed and never move. Pin one of these (or a commit SHA) to opt out of updates.

What changed in each version is in [CHANGELOG.md](CHANGELOG.md).

## Workflows

### `wp-release.yml`

Builds a production zip and creates a GitHub Release with it attached. Works for **plugins and themes**. Triggered by a `v*` tag on the calling repo.

1. Fails unless the caller ref is a tag, the tagged commit is on the repo's default branch (usually `main`) and the repo has a `.distignore`. Tag from the default branch after merging; a tag pushed from another branch does not publish.
2. Takes the release notes from `CHANGELOG.md` (see [Release title and notes](#release-title-and-notes)) and fails if the version has no section.
3. With `composer.json`: requires `composer.lock` and runs `composer install --no-dev`. Strauss runs from the repo's `post-install-cmd` as usual and builds `vendor-prefixed/`.
4. With `package.json`: requires `package-lock.json`, sets up Node.js (`node_version` input, else `.nvmrc`, else 24), runs `npm ci` and `npm run build` (skipped if there is no `build` script).
5. Copies the repo into the package folder, leaving out what `.distignore` and the safety net exclude.
6. Validates the package (see below).
7. Zips it as `package-name-1.2.0.zip` (a leading `v` on the tag is dropped). The archive contains a single top-level directory named after the repository, without the version, which WordPress needs to install and update the package in place.
8. Publishes a GitHub Release titled with the version (`1.2.0`) and the changelog section as its body, with the zip attached, marked as pre-release when the tag contains `-` (`v1.2.0-beta.1`). The job summary lists the files in the zip.

**Inputs:**

| Input          | Type   | Default | Description                                                                                                              |
| -------------- | ------ | ------- | ------------------------------------------------------------------------------------------------------------------------ |
| `php_version`  | string | `"8.4"` | PHP version to use for Composer                                                                                          |
| `node_version` | string | `""`    | Node.js version to use for npm. When empty, the version in the repo's `.nvmrc` is used, or `24` if there is no `.nvmrc`. |

#### Release title and notes

The release is titled with the tag without its leading `v` (`v1.2.0` → `1.2.0`). Its body is that version's section of `CHANGELOG.md` in the repo root: everything between its `## ` heading and the next `## ` heading, without trailing `---` separators. These headings are recognised:

```markdown
## [1.2.0] - 2026-05-20
## [1.2.0] — 2026-05-20
## 1.2.0
```

The job fails before building anything if `CHANGELOG.md` is missing, has no section for the version or the section is empty. Write the section, with its date, in the same pull request that bumps the `Version:` header.

#### What goes into the zip: `.distignore`

Every calling repo has a `.distignore` at its root listing everything that is **not** shipped. It is the single source of truth: reading it tells you exactly what the zip contains. The file name follows the WordPress convention also used by `wp dist-archive`.

Start from [`templates/.distignore`](templates/.distignore), the same template for plugins and themes, and uncomment or add what is specific to the project. The template already handles Strauss: it excludes `/tools/` and `/vendor/composer/autoload_aliases.php` and keeps `vendor-prefixed/`.

The syntax is rsync exclude patterns. **A leading `/` anchors the pattern to the repo root; without it, the pattern matches at any depth, including inside `vendor/`.** `src/` would remove `vendor/*/src/` and break every PSR-4 dependency, so use `/src/`. Leave a pattern unanchored only when you mean "everywhere" (e.g. `*.map`). A line starting with `+ ` re-includes what a later rule would exclude; the first matching rule wins.

Criteria for the template: a pattern is active only if no plugin or theme ever needs it at runtime and its name is a convention of the ecosystem (git, Composer, npm, Strauss, WordPress, QA tools, translation sources, root Markdown docs). Anything that depends on the project (`/src/`, `/screenshots/`) is left commented out or added per repo. The template never excludes `LICENSE*` files or `readme.txt`.

#### Safety net

Regardless of `.distignore`, the workflow always leaves out files that leak data or never belong to a WordPress package:

`.git/` `.env` `.env.*` `/auth.json` `node_modules/` `.DS_Store` `Thumbs.db`

This list stays short on purpose. Standard dev files (`composer.json`, `phpunit.xml`…) are not in it: they are the job of `.distignore`.

#### Validation

Before publishing, the job fails if:

- The package root has no PHP file with a `Plugin Name:` header and no `style.css` with a `Theme Name:` header.
- The `Version:` header of that file does not match the tag without its leading `v` (`v1.2.0` → `1.2.0`, `v1.2.0-beta.1` → `1.2.0-beta.1`).
- `composer.json` defines `autoload` but `vendor/autoload.php` is not in the package.
- `composer.json` configures Strauss (`extra.strauss`) but `vendor-prefixed/autoload.php` (or the configured `target_directory`) is not in the package.

It also emits a warning for each standard dev file found in the package (`composer.json`, `phpunit.xml*`, `vite.config.*`, `tests/`, `tools/strauss.phar`…), which usually means `.distignore` is missing a line or is older than the template.

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
    uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v4
    secrets: inherit
```

With a different PHP or Node.js version (for Node.js, a `.nvmrc` in the repo is preferred):

```yaml
uses: cortesfrau/github-workflows/.github/workflows/wp-release.yml@v4
with:
  php_version: "8.3"
  node_version: "22"
secrets: inherit
```

#### Migrating to v4

v4 is required: `@main`, `@v2` and `@v3` get no fixes, and a repo that calls `@main` breaks whenever this repository changes. For each repo that calls `wp-release.yml`:

1. `CHANGELOG.md` in the repo root, with a section for the version you are about to tag (see [Release title and notes](#release-title-and-notes)). v4 fails without it.
2. `.distignore` in the repo root:
   - No `.distignore` yet (callers of `@main` or `@v2`): copy [`templates/.distignore`](templates/.distignore). Move anything passed in `exclude:` into it, anchored with `/` (`src/` → `/src/`). Uncomment `/screenshots/` if the repo has that folder.
   - Copied from the v3 template: update it to the current one. It now excludes `.po`/`.pot`, `/.claude/`, `/plans/` and every root `*.md`, and the Strauss lines are active.
3. Remove `exclude`, `has_composer` and `has_package_json` from the caller, if present. The run fails if they are passed.
4. If the repo has a `package.json`, commit `package-lock.json`. Node.js comes from `.nvmrc`, or defaults to 24: prefer committing a `.nvmrc` over passing `node_version`, so CI builds with the same version as local development.
5. Tag releases from the default branch only. Tags whose commit is not on it are refused.
6. Change the caller to `@v4` and check the job summary of the first release: the files in the zip, and the release title and notes.

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

When releasing (example for `v4.1.0`):

1. Rename `## [Unreleased]` to `## [4.1.0] - YYYY-MM-DD`, add a new empty `## [Unreleased]` above it and update the compare links at the bottom. Do it in the pull request when it is the release, or in a commit on `main` before tagging.
2. After merging, tag `main` and move the major tag:

```bash
git switch main && git pull
git tag -a v4.1.0 -m "v4.1.0: what changed"
git tag -fa v4 -m "v4: latest v4.x release"
git push origin v4.1.0 && git push --force origin v4
```

For a new major version, create `v5.0.0` and `v5` instead and leave `v4` where it is.
