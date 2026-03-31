# Autoconfig UX: From `wrangler deploy` to Automatic PRs

A user-experience focused walkthrough of Wrangler's automatic configuration — what the user sees at every step, and how Workers Builds creates pull requests for you when you connect a repo from the dashboard.

> Based on analysis of [`cloudflare/workers-sdk`](https://github.com/cloudflare/workers-sdk) and the [Cloudflare Workers docs](https://developers.cloudflare.com/workers/).

---

## Table of Contents

- [The Two Worlds](#the-two-worlds)
- [World 1: Local CLI Experience](#world-1-local-cli-experience)
  - [Scenario A: `wrangler deploy` on an unconfigured project](#scenario-a-wrangler-deploy-on-an-unconfigured-project)
  - [Scenario B: `wrangler setup`](#scenario-b-wrangler-setup)
  - [Scenario C: Already configured](#scenario-c-already-configured)
  - [Dry-run mode](#dry-run-mode)
  - [Modifying detected settings](#modifying-detected-settings)
- [World 2: Workers Builds (Dashboard + Git)](#world-2-workers-builds-dashboard--git)
  - [The Full Journey: Connecting a Repo](#the-full-journey-connecting-a-repo)
  - [What Workers Builds Does Behind the Scenes](#what-workers-builds-does-behind-the-scenes)
  - [The Configuration PR](#the-configuration-pr)
  - [The Name Conflict PR](#the-name-conflict-pr)
  - [After Merging the PR](#after-merging-the-pr)
- [How Workers Builds Reads Wrangler's Output](#how-workers-builds-reads-wranglers-output)
  - [The Output File Mechanism](#the-output-file-mechanism)
  - [The `autoconfig` Output Entry](#the-autoconfig-output-entry)
  - [The `deploy` Output Entry](#the-deploy-output-entry)
  - [Environment Variables Workers Builds Sets](#environment-variables-workers-builds-sets)
- [The Double-Build Problem](#the-double-build-problem)
- [UX Differences: Local vs. CI](#ux-differences-local-vs-ci)
- [Supported Frameworks](#supported-frameworks)
- [Files Created and Modified](#files-created-and-modified)
- [Troubleshooting](#troubleshooting)

---

## The Two Worlds

Autoconfig has two distinct user experiences:

| | **Local (CLI)** | **Workers Builds (Dashboard)** |
|---|---|---|
| Trigger | `wrangler deploy` or `wrangler setup` | Connecting a repo via the Cloudflare dashboard |
| Interactive? | Yes — prompts for confirmation | No — runs non-interactively |
| Config committed? | Changes stay local until you commit | A PR is opened automatically |
| Deploy happens? | Yes (with `wrangler deploy`) | Yes, plus a preview deployment on the PR |
| User action needed? | Review terminal output | Review and merge the PR |

---

## World 1: Local CLI Experience

### Scenario A: `wrangler deploy` on an unconfigured project

You have an Astro project with no `wrangler.toml` / `wrangler.json` / `wrangler.jsonc`. You run:

```bash
npx wrangler deploy
```

Here's what you see, step by step:

#### 1. Detection

Wrangler scans your project and prints what it found:

```
Detected Project Settings:
 - Worker Name: my-astro-app
 - Framework: Astro
 - Build Command: npx astro build
 - Output Directory: dist
```

#### 2. Confirmation

```
Do you want to modify these settings? (y/N)
```

If you press Enter (default: No), it proceeds with the detected settings.

#### 3. Operations Summary

Wrangler shows exactly what it will do:

```
📦 Install packages:
 - wrangler (devDependency)

📝 Update package.json scripts:
 - "deploy": "npx astro build && wrangler deploy"
 - "preview": "npx astro build && wrangler dev"
 - "cf-typegen": "wrangler types"

📄 Create wrangler.jsonc:
  {
    "$schema": "node_modules/wrangler/config-schema.json",
    "name": "my-astro-app",
    "main": "dist/_worker.js/index.js",
    "compatibility_date": "2026-03-31",
    "compatibility_flags": ["nodejs_compat"],
    "assets": {
      "binding": "ASSETS",
      "directory": "dist"
    },
    "observability": {
      "enabled": true
    }
  }

🛠️  Configuring project for Astro
```

#### 4. Final Confirmation

```
Proceed with setup? (y/N)
```

#### 5. Execution

If confirmed, Wrangler:
- Installs `wrangler` as a dev dependency
- Runs the framework's `configure()` method (e.g. `astro add cloudflare` for Astro)
- Updates `package.json` with the new scripts
- Creates `wrangler.jsonc`
- Updates `.gitignore`
- Runs the build command
- Deploys to Cloudflare

#### 6. Deploy Output

```
🎉 Your project is now setup to deploy to Cloudflare
You can now deploy with npm run deploy
```

The deploy then proceeds as normal and your Worker is live.

---

### Scenario B: `wrangler setup`

Same flow as above, but **without the deploy step**. Useful for configuring first and deploying later:

```bash
npx wrangler setup
```

The `--yes` flag skips all prompts:

```bash
npx wrangler setup --yes
```

The `--build` flag also runs the build command:

```bash
npx wrangler setup --yes --build
```

---

### Scenario C: Already configured

If a `wrangler.jsonc` (or `.toml` / `.json`) already exists:

```bash
npx wrangler setup
```

```
🎉 Your project is already setup to deploy to Cloudflare
You can now deploy with npm run deploy
```

No changes are made. Autoconfig exits immediately.

---

### Dry-run mode

See what would happen without touching any files:

```bash
npx wrangler setup --dry-run
```

Shows the full operations summary, then:

```
✋  Autoconfig process run in dry-run mode, existing now.
```

---

### Modifying detected settings

If you answer "yes" to "Do you want to modify these settings?", you get interactive prompts:

```
What do you want to name your Worker? (my-astro-app)
> my-custom-name

What framework is your application using?
> Astro
  Angular
  Next.js
  Nuxt
  React Router
  SvelteKit
  Static
  ...

What directory contains your applications' output/asset files? (dist)
> dist

What is your application's build command? (npx astro build)
> npm run build
```

Then the updated settings are displayed:

```
Updated Project Settings:
 - Worker Name: my-custom-name
 - Framework: Astro
 - Build Command: npm run build
 - Output Directory: dist
```

---

## World 2: Workers Builds (Dashboard + Git)

### The Full Journey: Connecting a Repo

Here's what happens when you import a repository from the Cloudflare dashboard that has **no Wrangler config file**:

#### Step 1: Import from Dashboard

1. Go to **Workers & Pages** in the Cloudflare dashboard
2. Click **Create application** > **Import a repository**
3. Select your GitHub/GitLab account and repository
4. Configure build settings:
   - **Build command**: (optional, e.g. `npm run build`)
   - **Deploy command**: `npx wrangler deploy` (default)
   - **Root directory**: (optional, for monorepos)
5. Click **Save and Deploy**

#### Step 2: Workers Builds Triggers the First Build

Workers Builds clones your repo and runs the deploy command (`npx wrangler deploy`). Since there's no Wrangler config file, autoconfig kicks in **non-interactively**.

Under the hood, Workers Builds sets environment variables before invoking Wrangler:

```
CI=true
WORKERS_CI=1
WRANGLER_OUTPUT_FILE_DIRECTORY=/path/to/output/
WRANGLER_CI_OVERRIDE_NAME=my-worker
WRANGLER_CI_MATCH_TAG=abc123
WORKERS_CI_BRANCH=main
WORKERS_CI_COMMIT_SHA=def456
WORKERS_CI_BUILD_UUID=build-789
```

#### Step 3: Autoconfig Runs Non-Interactively

Because `CI=true`, all prompts are auto-answered:
- "Do you want to modify these settings?" → **No** (fallback)
- "Proceed with setup?" → **Yes** (fallback)
- Framework disambiguation with multiple matches → **Throws an error** (can't prompt)

Autoconfig:
1. Detects the framework
2. Creates `wrangler.jsonc` in the build environment
3. Updates `package.json` scripts
4. Installs adapters
5. Runs the build
6. Writes structured output to the output file

#### Step 4: The Deploy Proceeds

After autoconfig, Wrangler re-reads the newly created config and deploys. The deploy also writes structured output.

#### Step 5: Workers Builds Reads the Output

Workers Builds reads the ND-JSON output file that Wrangler wrote. It finds:
- An `autoconfig` entry with the full `AutoConfigSummary` — framework ID, wrangler config, scripts, build/deploy/version commands
- A `deploy` entry with the worker name, version ID, and targets

#### Step 6: Workers Builds Creates the PR

Using the autoconfig summary data, Workers Builds creates a **pull request** against your repository with all the configuration changes.

---

### What Workers Builds Does Behind the Scenes

```
    User clicks "Save and Deploy" in Dashboard
                    |
                    v
    Workers Builds clones the repo
                    |
                    v
    Sets CI env vars (WRANGLER_OUTPUT_FILE_DIRECTORY, etc.)
                    |
                    v
    Runs: npx wrangler deploy
                    |
                    v
    Wrangler finds no config file
                    |
                    v
    Autoconfig runs non-interactively
      |-- Detects framework via @netlify/build-info
      |-- Creates wrangler.jsonc (in build env)
      |-- Updates package.json scripts
      |-- Installs adapters & wrangler
      |-- Runs build command
      |-- Writes autoconfig summary to output file
                    |
                    v
    Wrangler re-reads config, deploys the Worker
      |-- Writes deploy result to output file
                    |
                    v
    Workers Builds reads the output file
                    |
                    v
    Workers Builds creates a PR with:
      |-- wrangler.jsonc
      |-- package.json (updated scripts + deps)
      |-- package-lock.json / yarn.lock / pnpm-lock.yaml
      |-- Framework config files (e.g. astro.config.mjs)
      |-- .gitignore updates
      |-- .assetsignore (if needed)
                    |
                    v
    PR includes a preview deployment link
                    |
                    v
    User reviews, tests preview, merges PR
                    |
                    v
    Future builds skip autoconfig (config already exists)
```

---

### The Configuration PR

When Workers Builds creates a configuration PR, it includes:

#### PR Contents

| File | What Changed |
|---|---|
| `wrangler.jsonc` | New file — Worker config with name, compatibility date, assets, observability |
| `package.json` | New `deploy`, `preview`, `cf-typegen` scripts + `wrangler` devDependency |
| `package-lock.json` (or equivalent) | Updated lock file with new dependencies |
| Framework config (varies) | e.g. `astro.config.mjs` updated by `astro add cloudflare` |
| Framework adapter (varies) | e.g. `@astrojs/cloudflare` added as dependency |
| `.gitignore` | `.wrangler` and `.dev.vars*` entries added |
| `.assetsignore` | `_worker.js` and `_routes.json` excluded (if applicable) |

#### PR Description

The PR description includes:
- **Detected settings** — Framework name, build command, deploy command, version command
- **Preview link** — A working preview URL generated from the autoconfig deployment
- **Next steps** — Links to docs for adding bindings, custom domains, etc.

> **Important**: When you merge the PR, Workers Builds will also update your build and deploy commands in the dashboard if they don't match the detected settings. This ensures future deployments succeed.

#### Only Created When Deploy Command is `npx wrangler deploy`

A configuration PR is **only created** when your deploy command is `npx wrangler deploy` (the default). If you have a custom deploy command, autoconfig still runs and configures your project during the build, but no PR is created — the changes only exist in the build environment.

---

### The Name Conflict PR

Separately from the configuration PR, Workers Builds can also create a **name conflict PR**. This happens when:

- You rename your Worker in the dashboard but not in your config file
- You connect a repository that was previously used with a different Worker
- The `name` field in `wrangler.jsonc` doesn't match the connected Worker

#### How It Works (in the code)

In `packages/wrangler/src/deploy/index.ts`, during deploy:

```typescript
const ciOverrideName = getCIOverrideName();
// WRANGLER_CI_OVERRIDE_NAME is set by Workers Builds

if (ciOverrideName !== undefined && ciOverrideName !== name) {
    logger.warn(
        `Failed to match Worker name. Your config file is using the Worker name "${name}",` +
        ` but the CI system expected "${ciOverrideName}".` +
        ` Overriding using the CI provided Worker name.` +
        ` Workers Builds connected builds will attempt to open a pull request` +
        ` to resolve this config name mismatch.`
    );
    name = ciOverrideName;
    workerNameOverridden = true;
}
```

The deploy proceeds with the override name, and the output entry includes `worker_name_overridden: true`. Workers Builds sees this flag and creates a PR to update the `name` field in the config file.

---

### After Merging the PR

Once the configuration PR is merged:

1. **Future pushes** trigger Workers Builds as normal
2. Wrangler finds the `wrangler.jsonc` file → `configured: true`
3. **Autoconfig is skipped entirely** — no double-build, no detection, no configuration
4. The build just runs the build command and deploys

---

## How Workers Builds Reads Wrangler's Output

### The Output File Mechanism

Wrangler has a structured output system (`packages/wrangler/src/output.ts`) that writes ND-JSON (newline-delimited JSON) to a file. Workers Builds uses this to communicate with Wrangler without parsing stdout.

The output file location is controlled by environment variables:

| Variable | Purpose |
|---|---|
| `WRANGLER_OUTPUT_FILE_PATH` | Exact path to the output file |
| `WRANGLER_OUTPUT_FILE_DIRECTORY` | Directory where Wrangler generates a timestamped output file |

Workers Builds sets `WRANGLER_OUTPUT_FILE_DIRECTORY` before running Wrangler. Each line in the output file is a JSON object with a `type` field and a `timestamp`.

### The `autoconfig` Output Entry

Written by both `wrangler setup` and `wrangler deploy` after autoconfig completes:

```json
{
  "type": "autoconfig",
  "version": 1,
  "command": "deploy",
  "summary": {
    "scripts": {
      "deploy": "npx astro build && wrangler deploy",
      "preview": "npx astro build && wrangler dev",
      "cf-typegen": "wrangler types"
    },
    "wranglerInstall": true,
    "wranglerConfig": {
      "$schema": "node_modules/wrangler/config-schema.json",
      "name": "my-astro-app",
      "compatibility_date": "2026-03-31",
      "compatibility_flags": ["nodejs_compat"],
      "main": "dist/_worker.js/index.js",
      "assets": { "binding": "ASSETS", "directory": "dist" },
      "observability": { "enabled": true }
    },
    "outputDir": "dist",
    "frameworkId": "astro",
    "buildCommand": "npx astro build",
    "deployCommand": "npx wrangler deploy",
    "versionCommand": "npx wrangler versions upload"
  },
  "timestamp": "2026-03-31T12:00:00.000Z"
}
```

Workers Builds uses this summary to:
- Construct the PR with the correct file contents
- Update the build/deploy commands in the dashboard settings
- Generate the PR description with detected settings

### The `deploy` Output Entry

Written after a successful deploy:

```json
{
  "type": "deploy",
  "version": 1,
  "worker_name": "my-astro-app",
  "worker_tag": "abc123def456",
  "version_id": "version-guid-here",
  "targets": ["https://my-astro-app.username.workers.dev"],
  "worker_name_overridden": false,
  "wrangler_environment": undefined,
  "timestamp": "2026-03-31T12:00:05.000Z"
}
```

If `worker_name_overridden` is `true`, Workers Builds knows to create a name conflict PR.

### Environment Variables Workers Builds Sets

| Variable | Value | Used By |
|---|---|---|
| `CI` | `true` | Disables interactive prompts |
| `WORKERS_CI` | `1` | Identifies Workers Builds specifically |
| `WRANGLER_OUTPUT_FILE_DIRECTORY` | Build-specific path | Tells Wrangler where to write structured output |
| `WRANGLER_CI_OVERRIDE_NAME` | The Worker name from the dashboard | Overrides the `name` field in config |
| `WRANGLER_CI_MATCH_TAG` | The Worker's unique tag | Verifies the config matches the right Worker |
| `WORKERS_CI_BRANCH` | Branch name from push event | Available for build customization |
| `WORKERS_CI_COMMIT_SHA` | SHA1 of the commit | Available for build customization |
| `WORKERS_CI_BUILD_UUID` | UUID of the current build | Available for build customization |

---

## The Double-Build Problem

When a configuration PR hasn't been merged yet, **every build runs autoconfig**, which means:

1. **First build**: Autoconfig detects framework, builds the project to generate config
2. **Second build**: The actual deploy re-reads the freshly created config and builds again

This means your project gets built **twice** per deployment. Merging the configuration PR eliminates this because the config already exists, so autoconfig is skipped.

> This is the main reason the PR description encourages you to merge it — faster deployments and version-controlled settings.

---

## UX Differences: Local vs. CI

| Behavior | Local (`wrangler deploy`) | Workers Builds (CI) |
|---|---|---|
| **Framework detection** | Same `@netlify/build-info` | Same `@netlify/build-info` |
| **Multiple frameworks found** | Prompts user to choose | Throws `MultipleFrameworksCIError` |
| **"Modify settings?" prompt** | Shown (default: No) | Skipped (fallback: No) |
| **"Proceed with setup?" prompt** | Shown (default: Yes) | Skipped (fallback: Yes) |
| **Pages project detection** | Prompts about `functions/` dir | Falls back to `false` |
| **Worker name** | From `package.json` or dir name | `WRANGLER_CI_OVERRIDE_NAME` takes precedence |
| **Config changes** | Applied to local filesystem | Applied in build env, then PR'd to repo |
| **Wrangler installed** | Yes, as devDependency | Yes, in the build environment |
| **Build runs** | Yes (unless `--build false`) | Yes, always |
| **Deploy happens** | Yes (with `wrangler deploy`) | Yes |
| **PR created** | No | Yes (configuration PR and/or name conflict PR) |

---

## Supported Frameworks

| Framework | Adapter/Tool | Min Version | What Autoconfig Does |
|---|---|---|---|
| **Next.js** | `@opennextjs/cloudflare` | 14.2.35 | Runs `@opennextjs/cloudflare migrate`. Configures R2 caching if available. |
| **Astro** | `@astrojs/cloudflare` | 4.0.0 | Runs `astro add cloudflare` |
| **SvelteKit** | `@sveltejs/adapter-cloudflare` | 2.20.3 | Runs `sv add sveltekit-adapter` |
| **Nuxt** | Built-in Cloudflare preset | 3.21.0 | Configures Nitro cloudflare preset |
| **React Router** | Cloudflare Vite plugin | 7.0.0 | Configures Cloudflare preset |
| **Solid Start** | Built-in Cloudflare preset | 1.0.0 | Configures Nitro cloudflare preset |
| **TanStack Start** | Cloudflare Vite plugin | 1.132.0 | Configures Nitro cloudflare preset |
| **Angular** | — | 19.0.0 | Configures SSR for Workers |
| **Analog** | Built-in Cloudflare preset | 2.0.0 | Configures Nitro cloudflare preset |
| **Vite** | `@cloudflare/vite-plugin` | 6.0.0 | Installs Cloudflare Vite plugin |
| **Vike** | — | 0.0.0 | Configures for Cloudflare |
| **Waku** | — | 1.0.0-alpha.4 | Configures for Cloudflare |
| **Qwik** | — | 1.1.0 | Sets platform config |
| **Static sites** | None | — | Any directory with an `index.html` |

---

## Files Created and Modified

### `wrangler.jsonc`

The core config file. Example for an Astro project:

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-astro-app",
  "main": "dist/_worker.js/index.js",
  "compatibility_date": "2026-03-31",
  "compatibility_flags": ["nodejs_compat"],
  "assets": {
    "binding": "ASSETS",
    "directory": "dist"
  },
  "observability": {
    "enabled": true
  }
}
```

The exact config varies per framework. The `nodejs_compat` flag is always added.

### `package.json`

New scripts are added:

```json
{
  "scripts": {
    "deploy": "npx astro build && wrangler deploy",
    "preview": "npx astro build && wrangler dev",
    "cf-typegen": "wrangler types"
  }
}
```

- `deploy` = build + deploy
- `preview` = build + local dev server
- `cf-typegen` = generate TypeScript types for bindings (only if the project uses TypeScript and has server-side code)

`wrangler` is also added as a `devDependency`.

### `.gitignore`

```
# wrangler files
.wrangler
.dev.vars*
!.dev.vars.example
```

### `.assetsignore`

For frameworks that generate worker files in the output directory:

```
_worker.js
_routes.json
```

This prevents uploading Wrangler internals as static assets.

---

## Troubleshooting

### Multiple frameworks detected

**Locally**: Wrangler prompts you to choose.

**In Workers Builds**: The build fails with:

```
Wrangler was unable to automatically configure your project to work with
Cloudflare, since multiple frameworks were found: Next.js, Vite.

To fix this issue either:
- check your project's configuration to make sure that the target framework
  is the only configured one and try again
- run `wrangler setup` locally to get an interactive user experience where
  you can specify what framework you want to target
```

**Fix**: Set the **root directory** in Workers Builds settings to point to the specific project within your monorepo.

### Framework not detected

Ensure your `package.json` includes the framework as a dependency. Autoconfig uses `@netlify/build-info` which analyzes `package.json` dependencies and config files.

### Configuration already exists

If `wrangler.jsonc` / `wrangler.json` / `wrangler.toml` already exists, autoconfig returns `{ configured: true }` immediately and does nothing. To re-run autoconfig, delete the config file first.

### Worker name mismatch in CI

If the `name` in your config doesn't match what Workers Builds expects, you'll see:

```
Failed to match Worker name. Your config file is using the Worker name "old-name",
but the CI system expected "new-name". Overriding using the CI provided Worker name.
Workers Builds connected builds will attempt to open a pull request to resolve
this config name mismatch.
```

The deploy proceeds with the CI-provided name, and a name conflict PR is created.

### Workspaces / Monorepos

Support for workspaces is limited. Wrangler analyzes the directory where you run the command, but doesn't resolve dependencies installed at the workspace root. If your framework is only in the root `package.json`, detection may fail.

**Fix**: Set the root directory to the specific project within the workspace, or ensure the framework dependency is in the project's own `package.json`.
