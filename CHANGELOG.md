# Changelog

All notable changes to the workflows in this repository. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and versions follow [Semantic Versioning](https://semver.org/) as described in the [versioning rule](README.md#versioning-rule).

Callers pin the major tag (`@v4`), which always points to the latest release listed here for that major version.

## [Unreleased]

---

## [4.0.0] - 2026-11-05

### Added

- **Breaking:** the release is titled with the version (`1.2.0`, without the `v`) and its body is that version's section of `CHANGELOG.md`. The job fails before building if `CHANGELOG.md` is missing, has no section for the version or the section is empty. Accepted headings: `## [1.2.0] - 2026-05-20`, `## [1.2.0] — 2026-05-20` and `## 1.2.0`.
- Package validation: the job fails if `composer.json` configures Strauss (`extra.strauss`) and `vendor-prefixed/autoload.php` (or the configured `target_directory`) is not in the package.
- Warnings for `.claude/`, `plans/`, `tools/strauss.phar` and `vendor/composer/autoload_aliases.php` found in the package.

### Changed

- `templates/.distignore`:
  - excludes `/languages/*.po` and `/languages/*.pot`, the translation sources: WordPress only loads the compiled `.mo`, `.l10n.php` and `.json` files. Delete both lines if translations are edited on the site (e.g. with Loco Translate, which needs the `.pot`);
  - excludes `/.claude/` and `/plans/`;
  - excludes every Markdown file in the root (`/*.md`: README, CHANGELOG, AGENTS, CLAUDE…) instead of a fixed list, and keeps `LICENSE*` with a `+ /LICENSE*` rule;
  - Strauss lines (`/tools/`, `/vendor/composer/autoload_aliases.php`) are active instead of commented out. They are no-ops in repos that don't use Strauss.
- The job times out after 20 minutes, and the checkout does not keep the token in the repo's git config.
- The job summary also collapses `vendor-prefixed/`.

---

## [3.0.0] - 2026-11-04

### Changed

- **Breaking:** `wp-release.yml` requires a `.distignore` at the root of the calling repo and fails without one. It is the single source of truth for what is left out of the zip. A template for plugins and themes is in `templates/.distignore`.
- **Breaking:** the default exclude list is replaced by a short safety net (`.git/`, `.env`, `.env.*`, `/auth.json`, `node_modules/`, `.DS_Store`, `Thumbs.db`). Dev files such as `composer.json`, `phpunit.xml` or `vite.config.*` are excluded only if `.distignore` lists them.
- **Breaking:** `/tools/`, `/vendor/composer/autoload_aliases.php` (Strauss) and `/screenshots/` are no longer excluded by default. The template lists them commented out; uncomment them when they apply.
- **Breaking:** the tagged commit must be on the repo's default branch. A tag pushed from another branch fails instead of publishing unmerged code.
- **Breaking:** a repo with `package.json` must commit `package-lock.json`.
- **Breaking:** when `node_version` is not passed, Node.js comes from the repo's `.nvmrc`, or defaults to `24` (was `22`, end of life in April 2027) if there is no `.nvmrc`. Commit a `.nvmrc` or pass `node_version` to pin another version.
- Tags containing `-` (e.g. `v1.2.0-beta.1`) publish a pre-release.
- Actions updated to their current major versions, which run on the Node 24 Actions runtime (`actions/checkout` v7, `actions/setup-node` v7, `softprops/action-gh-release` v3). All actions are pinned to commit SHAs and Dependabot keeps them up to date.
- The job summary lists the files in the zip and its size.

### Added

- Package validation before publishing: the job fails if there is no plugin file with a `Plugin Name:` header or `style.css` with a `Theme Name:` header, if the `Version:` header does not match the tag, or if `composer.json` defines `autoload` and `vendor/autoload.php` is missing.
- A warning for each standard dev file found in the package root, to catch an outdated `.distignore`.

### Removed

- **Breaking:** inputs `exclude`, `has_composer` and `has_package_json`. Callers that pass them fail. Move `exclude` patterns to `.distignore`, anchored with `/`.

### Fixed

- Exclude patterns no longer match inside `vendor/`. In v1 and v2, unanchored defaults (`tools/`, `bin/`, `tests/`, `*.md`) and caller patterns such as `src/` also removed those paths from every dependency, which could break PSR-4 autoloading and dropped the dependencies' `LICENSE.md` files.

---

## [2.0.0] - 2026-10-30

### Changed

- **Breaking:** Composer and npm are detected from `composer.json` and `package.json` instead of caller flags.
- `npm run build` is skipped when `package.json` has no `build` script.

### Deprecated

- Inputs `has_composer` and `has_package_json`. They have no effect and the run shows a warning asking to remove them.

---

## [1.0.0] - 2026-10-30

First tagged release.

### Added

- `wp-release.yml`: builds a WordPress-installable zip for plugins and themes from a `v*` tag and attaches it to a GitHub Release. The zip is named `package-name-1.2.0.zip` and contains a single `package-name/` directory.
- Composer (`--no-dev`, `composer.lock` required) and npm build steps, enabled with `has_composer` and `has_package_json`.
- Default exclude list for dev files, plus extra patterns through the `exclude` input.
- Private Composer repositories through the `LCF_COMPOSER_GITHUB_TOKEN` secret.

---

[Unreleased]: https://github.com/cortesfrau/github-workflows/compare/v4.0.0...HEAD
[4.0.0]: https://github.com/cortesfrau/github-workflows/compare/v3.0.0...v4.0.0
[3.0.0]: https://github.com/cortesfrau/github-workflows/compare/v2.0.0...v3.0.0
[2.0.0]: https://github.com/cortesfrau/github-workflows/compare/v1.0.0...v2.0.0
[1.0.0]: https://github.com/cortesfrau/github-workflows/releases/tag/v1.0.0
