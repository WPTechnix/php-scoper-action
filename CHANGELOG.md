# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-07-01

### Added
- Composite GitHub Action for scoping PHP dependencies with humbug/php-scoper
- PHP version selection via `php-version` input (default: 8.2)
- php-scoper version selection via `scoper-version` input (default: ~0.18.0)
- Working directory support via `working-directory` input — relative paths resolve from here; absolute paths also accepted
- Configurable scoper config path via `scoper-config` input (default: ./scoper.inc.php)
- Configurable output directory via `output-dir` input (default: scoped-build)
- Optional `composer install` step with configurable arguments (`composer-install`, `composer-args`)
- Early validation of working directory and scoper config before any install
- Outputs for scoped build path (`output-path`) and PHP file count (`scoped-files`)
- Two test fixtures: basic (standalone) and full (with external dependency symfony/polyfill-php80)
- Integration test workflow covering PHP 8.1, 8.2, and 8.3
- Commit linting workflow using commitlint

### Fixed
- Test fixture config keys use kebab-case to match php-scoper 0.18 configuration format
