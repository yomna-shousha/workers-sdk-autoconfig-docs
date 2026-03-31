# How Autoconfig Works in `cloudflare/workers-sdk`

A deep-dive into the autoconfig system inside Wrangler — how it detects your JS framework, configures your project for Cloudflare Workers, and what happens in CI.

> Based on analysis of the [`cloudflare/workers-sdk`](https://github.com/cloudflare/workers-sdk) repository at commit `260d0ad`.

---

## Table of Contents

- [What is Autoconfig?](#what-is-autoconfig)
- [Step-by-Step Flow](#step-by-step-flow)
  - [Step 1: Entry Point — Command Triggers](#step-1-entry-point--command-triggers)
  - [Step 2: Detection Phase — `getDetailsForAutoConfig()`](#step-2-detection-phase--getdetailsforautoconfig)
  - [Step 3: Framework Detection — `detectFramework()`](#step-3-framework-detection--detectframework)
  - [Step 4: Configuration Phase — `runAutoConfig()`](#step-4-configuration-phase--runautoconfig)
  - [Step 5: Framework-Specific Configuration](#step-5-framework-specific-configuration)
- [What Happens in CI](#what-happens-in-ci)
  - [The E2E Test Infrastructure](#the-e2e-test-infrastructure)
  - [CI-Specific Autoconfig Behavior](#ci-specific-autoconfig-behavior)
  - [The E2E Autoconfig Test](#the-e2e-autoconfig-test)
  - [Telemetry Events in CI](#telemetry-events-in-ci)
- [Summary Diagram](#summary-diagram)
- [Key Source Files](#key-source-files)

---

## What is Autoconfig?

Autoconfig is a system inside Wrangler that **automatically detects your JS framework, configures your project for Cloudflare Workers, and sets up all the necessary files** (`wrangler.jsonc`, `package.json` scripts, etc.) without the user needing to do any manual configuration.

It's triggered by two commands:

1. **`wrangler setup`** — the dedicated setup command
2. **`wrangler deploy`** — deploy also runs autoconfig automatically if the project isn't already configured (controlled by the `--experimental-autoconfig` / `--x-autoconfig` flag, which defaults to `true`)

---

## Step-by-Step Flow

### Step 1: Entry Point — Command Triggers

There are **two entry points** into autoconfig:

#### `wrangler setup` (`packages/wrangler/src/setup.ts`)

The dedicated setup command. It:

1. Calls `sendAutoConfigProcessStartedMetricsEvent()` with `command: "wrangler setup"`
2. Calls `getDetailsForAutoConfig()` to detect the project
3. If not already configured, calls `runAutoConfig()` with options derived from CLI args:
   - `--yes` / `-y` — skip all confirmation prompts
   - `--build` — run the build command after configuration
   - `--dry-run` — don't apply any filesystem changes
   - `--install-wrangler` — whether to install wrangler (default: true)
   - `--completion-message` — whether to display the completion message (default: true)
4. Writes output via `writeOutput()` for programmatic consumers
5. Sends `sendAutoConfigProcessEndedMetricsEvent()`

#### `wrangler deploy` (`packages/wrangler/src/deploy/index.ts`)

Before doing any actual deploy work, the deploy command checks if autoconfig should run:

```typescript
const shouldRunAutoConfig =
    args.experimentalAutoconfig &&
    !args.script &&
    !args.assets;
```

If `true`, it runs the same `getDetailsForAutoConfig()` -> `runAutoConfig()` pipeline. After autoconfig completes, it **re-reads the config** (`readConfig()`) and proceeds with the normal deploy.

---

### Step 2: Detection Phase — `getDetailsForAutoConfig()`

**File:** `packages/wrangler/src/autoconfig/details/index.ts`

This is the "detection" phase. It figures out everything about the project:

1. **Already configured?** — If a real `wrangler.toml`/`wrangler.json`/`wrangler.jsonc` config file exists (and it's not a Pages project via `pages_build_output_dir`), it returns `{ configured: true }` immediately. No further autoconfig needed.

2. **Framework detection** — Calls `detectFramework()` (see Step 3).

3. **Package.json parsing** — Reads the project's `package.json` if present.

4. **Already configured by framework?** — Calls `framework.isConfigured(projectPath)`. Each framework class can override this to check if the framework's own Cloudflare adapter/plugin is already set up.

5. **Output directory detection** — Uses the framework's reported `dist` directory, or falls back to a heuristic: finds the first child directory containing an `index.html` file.

6. **Worker name derivation** — Derives from (in order of precedence):
   - `WRANGLER_CI_OVERRIDE_NAME` environment variable
   - `package.json` `name` field
   - Directory basename
   
   The name is sanitized to valid worker name rules: lowercase, alphanumeric + dashes, max 63 chars.

7. **Build command detection** — Gets the framework's build command and prefixes it with the appropriate package manager runner (e.g. `npx astro build`).

Returns an `AutoConfigDetails` object with all this info.

---

### Step 3: Framework Detection — `detectFramework()`

**File:** `packages/wrangler/src/autoconfig/details/framework-detection.ts`

This is the core detection engine:

#### 3a. Use `@netlify/build-info`

Creates a `Project` instance from `@netlify/build-info` and calls `project.getBuildSettings()`. This library analyzes the project structure (package.json deps, config files, etc.) and returns an array of detected `Settings` objects with framework info, build commands, and output dirs.

#### 3b. Package Manager Detection

Maps the package manager detected by `@netlify/build-info` to Wrangler's internal types:

| Detected | Wrangler Type |
|---|---|
| `pnpm` | `PnpmPackageManager` |
| `yarn` | `YarnPackageManager` |
| `bun` | `BunPackageManager` |
| `npm` / default | `NpmPackageManager` |

#### 3c. Workspace Detection

Checks if the project is at the root of a monorepo workspace. If so, validates that the project path is actually one of the workspace packages. Throws a `UserError` if you run autoconfig from the workspace root without targeting a specific project.

#### 3d. Pages Project Check

Calls `isPagesProject()` which checks:

- Is `pages_build_output_dir` set in wrangler config?
- Does a cached `pages.json` exist in the cache folder?
- Does a `functions/` directory exist? (prompts the user if interactive, defaults to `false` in CI)

#### 3e. Framework Disambiguation

`maybeFindDetectedFramework()` handles when multiple frameworks are detected:

| Scenario | Behavior |
|---|---|
| 0 frameworks detected | Returns `undefined` (falls back to Static) |
| 1 framework | Use it |
| 2 frameworks, one is Vite | Pick the non-Vite one (Vite is usually just a build tool) |
| 2 frameworks: Waku + Hono | Pick Waku (Waku uses Hono internally) |
| Multiple known frameworks in CI | **Throws `MultipleFrameworksCIError`** (can't prompt the user) |
| Multiple known frameworks locally | Picks the first one (user can override interactively) |

---

### Step 4: Configuration Phase — `runAutoConfig()`

**File:** `packages/wrangler/src/autoconfig/run.ts`

This is where changes actually happen:

#### 4a. Assert Not Configured

Validates the project isn't already set up.

#### 4b. Display Detected Settings

Shows the user:
- Worker Name
- Framework
- Build Command
- Output Directory

#### 4c. Interactive Confirmation

Unless `--yes` or dry-run, asks the user if they want to modify settings. If yes, prompts for:
- Worker name
- Framework selection (dropdown of all 16+ known frameworks)
- Output directory
- Build command

#### 4d. Framework Validation

Checks if the detected framework is in the "supported" list.

**Currently supported:**
- Analog, Angular, Astro, Next.js, Nuxt, Qwik, React Router, Solid Start, SvelteKit, TanStack Start, Vite, Vike, Waku, Static

**Detected but NOT supported (errors):**
- Hono, Cloudflare Pages

#### 4e. Version Validation

For supported frameworks, validates the installed version is within the supported range (min/max versions). Warns if above the known max major version.

| Framework | Min Version | Max Known Major |
|---|---|---|
| Analog | 2.0.0 | 2 |
| Angular | 19.0.0 | 21 |
| Astro | 4.0.0 | 6 |
| Next.js | 14.2.35 | 16 |
| Nuxt | 3.21.0 | 4 |
| Qwik | 1.1.0 | 1 |
| React Router | 7.0.0 | 7 |
| Solid Start | 1.0.0 | 2 |
| SvelteKit | 2.20.3 | 2 |
| TanStack Start | 1.132.0 | 1 |
| Vite | 6.0.0 | 8 |
| Vike | 0.0.0 | 0 |
| Waku | 1.0.0-alpha.4 | 1 |

#### 4f. Dry-Run Framework Configure

Calls `framework.configure({ dryRun: true })` to get what config changes the framework would make without actually doing them.

#### 4g. Build Operations Summary

Computes and displays:
- **Packages to install:** wrangler as devDependency
- **`package.json` script updates:** `deploy`, `preview`, optionally `cf-typegen`
- **`wrangler.jsonc` content** to create/update
- **Framework-specific configuration** description

#### 4h. Confirmation

Asks "Proceed with setup?"

#### 4i. Apply Changes (if not dry-run)

1. **Install wrangler** — via `installWrangler(packageManager.type, isWorkspaceRoot)`
2. **Run `framework.configure()`** for real — each framework class implements this differently (e.g. installing adapters, creating config files)
3. **Update `package.json`** — Adds/updates `deploy`, `preview`, `cf-typegen` scripts
4. **Create/update `wrangler.jsonc`** — Writes the computed config with:
   - `$schema`
   - `name`
   - `compatibility_date`
   - `observability.enabled: true`
   - `nodejs_compat` compatibility flag
   - Framework-specific settings (like `main`, `assets`, etc.)
5. **Update `.gitignore`** — Adds wrangler internal files
6. **Update `.assetsignore`** — If output dir equals project path, adds wrangler files
7. **Run build** — Executes the build command if `runBuild` is true

#### 4j. Telemetry

Sends `autoconfig_configuration_completed` metrics event with success/failure.

---

### Step 5: Framework-Specific Configuration

**File:** `packages/wrangler/src/autoconfig/frameworks/framework-class.ts`

Each framework extends the `Framework` abstract class and implements `configure()`:

```
Framework (abstract)
  |-- Static          -- plain static files, no framework adapter
  |-- Astro           -- installs @astrojs/cloudflare adapter
  |-- NextJs          -- configures OpenNext for Cloudflare
  |-- Nuxt            -- sets Nitro cloudflare preset
  |-- SvelteKit       -- installs @sveltejs/adapter-cloudflare
  |-- ReactRouter     -- configures Cloudflare preset
  |-- Angular         -- configures SSR for Workers
  |-- Qwik            -- sets platform config
  |-- SolidStart      -- configures Nitro cloudflare preset
  |-- TanstackStart   -- configures Nitro cloudflare preset
  |-- Vite            -- installs @cloudflare/vite-plugin
  |-- Analog          -- configures Nitro cloudflare preset
  |-- Vike            -- configures for Cloudflare
  |-- Waku            -- configures for Cloudflare
  |-- Hono            -- detected but NOT supported (errors)
  |-- CloudflarePages -- detected but NOT supported (errors)
```

Each `configure()` returns a `ConfigurationResults` object with:

- **`wranglerConfig`** — framework-specific wrangler config overrides (or `null` if the framework manages its own config file, e.g. Nuxt via `nitro.toml`)
- **`packageJsonScriptsOverrides`** — custom `deploy`/`preview`/`typegen` scripts
- **`buildCommandOverride`** / **`deployCommandOverride`** / **`versionCommandOverride`** — override the standard commands

---

## What Happens in CI

### The E2E Test Infrastructure

**Workflow:** `.github/workflows/e2e-wrangler.yml`

1. **Matrix strategy** — Runs on macOS, Windows, and Ubuntu with 4 shards each (12 total jobs).

2. **Path filter** — Skips the entire workflow if only markdown files changed.

3. **Sharding** (`tools/e2e/runIndividualE2EFiles.ts`) — Uses **greedy bin-packing** to balance test files across shards by estimated duration. The autoconfig test (`e2e/autoconfig/setup.test.ts`) is one of the fastest at ~2 seconds estimated.

4. **Per-file caching** — Each e2e test file runs as an individual turbo task (via `WRANGLER_E2E_TEST_FILE` env var) so that turbo can cache at file-level granularity.

5. **Retries** — Failed tests are retried up to 4 times before the job fails.

6. **Vitest config** (`packages/wrangler/e2e/vitest.config.mts`) — 90-second test timeout, single worker thread, bail on first failure.

---

### CI-Specific Autoconfig Behavior

There are several CI-specific behaviors baked into autoconfig:

#### Worker Name Override

In CI (e.g. Workers Builds), the `WRANGLER_CI_OVERRIDE_NAME` env var overrides the worker name. If it doesn't match the config file name, Wrangler warns:

> "Failed to match Worker name. Your config file is using the Worker name 'X', but the CI system expected 'Y'. Overriding using the CI provided Worker name."

#### No Interactive Prompts

In CI (`isNonInteractiveOrCI()` returns `true`):

- Framework disambiguation **throws `MultipleFrameworksCIError`** instead of prompting
- Pages detection `functions/` directory check falls back to `false` instead of prompting
- All confirmation dialogs use their `fallbackValue`

#### `--yes` Flag

`wrangler setup --yes` skips all confirmations. Used in CI pipelines for non-interactive setup.

#### Auto-Trigger on Deploy

When CI runs `wrangler deploy` on an unconfigured project, autoconfig runs first, creates `wrangler.jsonc`, then re-reads config and proceeds with deploy. This is the "zero-config deploy" path.

---

### The E2E Autoconfig Test

**File:** `packages/wrangler/e2e/autoconfig/setup.test.ts`

Tests two scenarios:

1. **Hono app already configured for Cloudflare Workers:**
   - `wrangler setup --yes` prints "Your project is already setup to deploy to Cloudflare"
   - Exits with status 0

2. **Hono app NOT configured for Cloudflare:**
   - `wrangler setup --yes` errors with "The detected framework ('Hono') cannot be automatically configured."
   - Exits with non-zero status

---

### Telemetry Events in CI

The full telemetry chain fires these events (all correlated by a UUID `autoConfigId`):

| Order | Event | Key Fields |
|---|---|---|
| 1 | `autoconfig_process_started` | command, dryRun |
| 2 | `autoconfig_detection_started` | autoConfigId, command |
| 3 | `autoconfig_detection_completed` | framework, configured, success |
| 4 | `autoconfig_configuration_started` | framework, dryRun |
| 5 | `autoconfig_configuration_completed` | framework, success, dryRun |
| 6 | `autoconfig_process_ended` | command, success, dryRun |

---

## Summary Diagram

```
wrangler setup / wrangler deploy
       |
       v
sendAutoConfigProcessStartedMetricsEvent()
       |
       v
getDetailsForAutoConfig()
  |-- Already has wrangler config? --> return { configured: true }
  |-- detectFramework()
  |     |-- @netlify/build-info --> getBuildSettings()
  |     |-- Package manager detection
  |     |-- isPagesProject() check
  |     +-- Framework disambiguation (CI: error on ambiguity)
  |-- Parse package.json
  |-- Derive worker name (CI: WRANGLER_CI_OVERRIDE_NAME)
  +-- Detect output dir + build command
       |
       v
  configured? --yes--> "Already setup!" (done)
       |
      no
       |
       v
runAutoConfig()
  |-- Display detected settings
  |-- Confirm / modify settings (skipped with --yes / CI)
  |-- Validate framework is supported + version check
  |-- framework.configure({ dryRun: true })
  |-- Build operations summary
  |-- Confirm "Proceed with setup?"
  |-- Install wrangler (devDependency)
  |-- framework.configure({ dryRun: false })
  |-- Update package.json scripts
  |-- Create/update wrangler.jsonc
  |-- Update .gitignore
  |-- Run build command
  +-- Send telemetry
       |
       v
  (wrangler deploy only: re-read config, proceed with deploy)
```

---

## Key Source Files

| File | Purpose |
|---|---|
| `packages/wrangler/src/setup.ts` | `wrangler setup` command definition |
| `packages/wrangler/src/deploy/index.ts` | `wrangler deploy` command (autoconfig entry point) |
| `packages/wrangler/src/autoconfig/run.ts` | Core autoconfig orchestration logic |
| `packages/wrangler/src/autoconfig/types.ts` | TypeScript types for autoconfig |
| `packages/wrangler/src/autoconfig/details/index.ts` | Detection phase + user interaction |
| `packages/wrangler/src/autoconfig/details/framework-detection.ts` | Framework detection via `@netlify/build-info` |
| `packages/wrangler/src/autoconfig/frameworks/index.ts` | Framework registry + lookup functions |
| `packages/wrangler/src/autoconfig/frameworks/all-frameworks.ts` | All known frameworks + version constraints |
| `packages/wrangler/src/autoconfig/frameworks/framework-class.ts` | Abstract `Framework` base class |
| `packages/wrangler/src/autoconfig/git.ts` | `.gitignore` utilities |
| `packages/wrangler/src/autoconfig/errors.ts` | Autoconfig-specific error classes |
| `packages/wrangler/src/autoconfig/telemetry-utils.ts` | Telemetry event helpers |
| `.github/workflows/e2e-wrangler.yml` | CI workflow for e2e tests |
| `tools/e2e/runIndividualE2EFiles.ts` | E2E test runner with sharding |
| `packages/wrangler/e2e/autoconfig/setup.test.ts` | E2E tests for `wrangler setup` |
