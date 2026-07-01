# PHP-Scoper Action

Automatically scope PHP project dependencies using [humbug/php-scoper](https://github.com/humbug/php-scoper).

Scoping rewrites all class names, function names, and namespaced code in your dependencies with a unique prefix. This is essential for WordPress plugins, Laravel packages, and any PHP library that bundles third-party dependencies alongside other code that may use different versions of the same libraries.

## Prerequisites

Before using this action, your project needs two things:

- **`composer.json` with autoload configuration** — php-scoper needs to know which files belong to your project so it can distinguish your code from third-party dependencies. The autoload section in your `composer.json` tells it where to look.

- **`scoper.inc.php`** — this configuration file defines the prefix to apply (e.g. `"MyPluginPrefix"`) and any exclusions. See the [php-scoper documentation](https://github.com/humbug/php-scoper) for how to write this file.

## Inputs

| Input | Default | Description |
|---|---|---|
| `php-version` | `8.2` | PHP version to use (e.g. `8.1`, `8.3`) |
| `scoper-version` | `~0.18.0` | php-scoper version to install (any Composer version constraint, e.g. `0.18.7`, `~0.18.0`, `dev-main`) |
| `scoper-config` | `./scoper.inc.php` | Path to your scoper configuration file |
| `working-directory` | `.` | Project directory where `composer.json` lives and where all relative paths resolve from |
| `output-dir` | `scoped-build` | Directory to write the scoped files into |
| `composer-install` | `true` | Whether to run `composer install` before scoping (`true` or `false`) |
| `composer-args` | `--no-dev --prefer-dist --optimize-autoloader` | Extra flags to pass to `composer install` |

### Path resolution

`scoper-config` and `output-dir` can be given as:

- **Relative paths** — resolved from `working-directory`. For example, with `working-directory: packages/my-package`, a `scoper-config` of `./scoper.inc.php` reads from `packages/my-package/scoper.inc.php`.
- **Absolute paths** — used as-is, ignoring `working-directory`. For example, `output-dir: /tmp/scoped-build` always writes to `/tmp/scoped-build`.

## Outputs

| Output | Description |
|---|---|
| `output-path` | Absolute path to the directory containing scoped files |
| `scoped-files` | Number of `.php` files in the scoped build |

You reference these in subsequent steps using `${{ steps.scoper.outputs.output-path }}` and `${{ steps.scoper.outputs.scoped-files }}` (see examples below).

## Usage

### Quick start

```yaml
- uses: wptechnix/php-scoper-action@v1
  id: scoper
```

### Full configuration

```yaml
- uses: wptechnix/php-scoper-action@v1
  id: scoper
  with:
    php-version: '8.2'
    scoper-version: '0.18.7'
    scoper-config: './scoper.inc.php'
    output-dir: 'build'
    composer-install: true
    composer-args: '--no-dev --prefer-dist --optimize-autoloader'
```

```yaml
- name: Use scoped build
  run: |
    echo "Output path: ${{ steps.scoper.outputs.output-path }}"
    echo "PHP files: ${{ steps.scoper.outputs.scoped-files }}"
```

### Working from a subdirectory

If your PHP project is not at the repository root but in a subdirectory, set `working-directory` so the action knows where to find `composer.json` and `scoper.inc.php`:

```yaml
- uses: wptechnix/php-scoper-action@v1
  with:
    working-directory: 'packages/my-package'
    output-dir: 'scoped-build'
```

Your `composer.json` must be located inside `working-directory` — that is where `composer install` runs. Because `working-directory` is set to `packages/my-package`, the action resolves all relative paths from there:

| Input | Value | Actual path on disk |
|---|---|---|
| `scoper-config` (default) | `./scoper.inc.php` | `packages/my-package/scoper.inc.php` |
| `output-dir` | `scoped-build` | `packages/my-package/scoped-build` |
| `composer install` runs in | — | `packages/my-package/` |

### Skipping composer install

You can skip `composer install` if your `vendor/` directory already exists — for example, when it was restored from cache or installed in an earlier step:

```yaml
- uses: wptechnix/php-scoper-action@v1
  with:
    composer-install: false
```

> **Important:** If you ran `composer install` with dev dependencies in an earlier step (e.g., for PHPStan), those dev packages will be included in the scoped build. The easiest way to avoid this is to let the action run `composer install` (the default) — it uses `--no-dev` to ensure only production dependencies are installed and scoped.

### Complete workflow example

```yaml
name: Build

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: wptechnix/php-scoper-action@v1
        id: scoper
        with:
          php-version: '8.2'
          scoper-version: '0.18.7'

      - name: Upload scoped build
        uses: actions/upload-artifact@v4
        with:
          name: scoped-build
          path: ${{ steps.scoper.outputs.output-path }}
```

## How it works

1. **PHP setup** — `shivammathur/setup-php` installs the PHP version you requested, along with Composer and required extensions.

2. **Validation** — checks that your `working-directory` exists and your `scoper.inc.php` file is present before running any installs.

3. **Composer install** — `ramsey/composer-install` runs `composer install` inside your project with your custom arguments. The `vendor/` directory is automatically cached between runs.

4. **Scoper install** — php-scoper is installed via Composer into a temporary directory — never added to your project's own `vendor/` or `composer.json`.

5. **Scoping** — `php-scoper add-prefix` reads your configuration, rewrites all matching PHP files with the configured prefix, and writes the result to the output directory.

6. **Outputs** — the absolute path and file count are written to `$GITHUB_OUTPUT` and available via `${{ steps.scoper.outputs.* }}`.
