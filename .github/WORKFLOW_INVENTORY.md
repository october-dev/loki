# GitHub Actions inventory

This records a proposed keep, replace or remove decision for every workflow and local action Loki inherited from Hermes Agent ([#1](https://github.com/october-dev/loki/issues/1), acceptance criterion 1).

- **Audited commit:** `810bc72390535f719b985e66a655eee16966c72e`
- **Nothing was executed.** Actions are disabled on this repository. Every fact below comes from reading the files, so runtime behavior is unconfirmed.
- **Nothing was changed.** This document edits no workflow and enables none. The decisions are proposals for maintainers to review.

## Summary

| | Keep | Replace | Remove | Total |
|---|---|---|---|---|
| Workflows | 36 | 14 | 2 | 52 |
| Local actions | 9 | 1 | 1 | 11 |

- **Keep:** validation or same-repo housekeeping, with no external publishing.
- **Replace:** the capability may be useful, but it targets or depends on a Nous-owned destination or needs an October-owned one first. It stays disabled until that exists and is approved.
- **Remove:** not needed to validate changes and not worth carrying.

## Findings to read first

1. **A repository guard does not protect every publish path.** `docker.yml` hard-codes `nousresearch/hermes-agent` (L61). Its main-branch publishing is guarded to `NousResearch/hermes-agent`, but `release-publish` (L510) and `release-publish-manifest` (L607) are not. Only the `container-publish` environment and the absence of Docker Hub credentials stand in the way. `scripts/releases/docker.py:18` hard-codes the same image, and `scripts/releases/r2.py:509` defaults the public URL to `hermes-assets.nousresearch.com`.
2. **Several publishing jobs have no repository guard at all:** `archive-inputs.yml`, `canary-release.yml`, the `deploy-vercel` job of `deploy-site.yml`, the release path of `stable-release.yml`, and the R2 jobs of `termux-verify.yml`. Without credentials most of them should fail, but `canary-release.yml` tags and releases on `origin` using only the built-in token.
3. **No workflow uses `pull_request_target`.** The string appears only in two job conditions that exclude it (`archive-inputs.yml` L26, `termux-verify.yml` L99).
4. **Three workflows use `workflow_run`,** which runs with base-repository privileges after other runs finish. `ci-review-comment.yml` limits itself to same-repo PRs, and it and `stable-release-publication.yml` check out the default branch with `persist-credentials: false`. `install-e2e-red.yml` runs only for scheduled or manual runs and sparse-checks out one tracker script.
5. **Fork pull requests can reach 10 workflows directly:** `ci.yaml`, `docker.yml`, `install-e2e.yml`, `label-rerun.yml`, `nix.yml`, `plugin-catalog-ci.yml`, `pm-bundle.yml`, `sandbox-image.yml`, `termux-verify.yml`, `windows-bundle-sdk.yml`. These use the `pull_request` event, which gives fork code a read-only token and no secrets. `ci.yaml` passes no secrets to the workflows it calls. Job-by-job review belongs to the hardening PR.
6. **Every external `uses:` reference is pinned to a full commit SHA** (122 distinct references per file, 13 distinct third-party actions). This checks `uses:` pinning only, not container images used by `docker run`. Whether each publisher is trusted, and whether each SHA matches its version comment, is still to review.
7. **Larger runners are required by:** `desktop-bundle-smoke.yml`, `desktop-bundled-release.yml`, `docker.yml`, `e2e-desktop-core.yml`, `e2e-desktop-update.yml`, `e2e-desktop.yml`, `install-e2e-windows-run.yml`, `js-tests.yml`, `nix.yml`, `pm-bundle.yml`, `rust-tests.yml`, `sandbox-image.yml`, `tests-os.yml`, `tests.yml`, `windows-bundle-sdk.yml`, `windows-install-update-e2e.yml`, `windows-venv-e2e.yml`. These labels (`*-32-core`, `*-96-core`, `*-32-arm-core`) are organization-specific. If October has no runners with these names, those jobs will wait and never start.
8. **Hard-coded Nous references also exist outside `.github`,** including `scripts/install.sh`, `scripts/install.ps1` and `scripts/termux/stage_apt_repo.py`. Replace decisions need to cover these. This inventory does not list every one.

## Workflows

Triggers: **called** means `workflow_call`, so it runs only when another workflow calls it. **manual** means `workflow_dispatch`. Permissions: **default** means the file sets none and uses the repository default. **Jobs** lists only jobs that set their own. **L** numbers are lines in that workflow at the audited commit. `GITHUB_TOKEN` is omitted from secrets. GitHub-owned `actions/*` are omitted from third-party actions.

| Workflow | Triggers | Permissions | Environments, secrets, vars | External destinations | Third-party actions (all SHA-pinned) | Proposed decision |
|---|---|---|---|---|---|---|
| `archive-inputs.yml` | push (main), called, manual | contents: read | Env: `release-signing`<br>Secrets: `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`<br>Vars: `CLOUDFLARE_R2_BUCKET` | R2 bucket from `vars.CLOUDFLARE_R2_BUCKET` (L34). Job guard is event-only (L26). | none | **replace**: Uploads pinned inputs to R2 on every push to main with no repo guard; needs an October-owned bucket and approval |
| `bootstrap-installer-build.yml` | manual | contents: read, id-token: write | Env: `release-signing`<br>Secrets: `APPLE_API_ISSUER`, `APPLE_API_KEY_ID`, `APPLE_API_KEY_P8`, `CSC_KEY_PASSWORD`, `CSC_LINK`<br>Vars: `AZURE_CLIENT_ID`, `AZURE_SIGN_ACCOUNT`, `AZURE_SIGN_ENDPOINT`, `AZURE_SIGN_PROFILE`, `AZURE_SIGN_PUBLISHER`, `AZURE_TENANT_ID` | Azure and Apple signing identities, from `vars` and `secrets` | `azure/login` | **replace**: Signs installers with inherited identities; needs October-owned Azure and Apple signing |
| `bootstrap-installer.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing; its push goes to a temporary local mirror (L64) |
| `canary-release.yml` | schedule (41 6 * * *), manual | contents: write, actions: write | Env: `release-signing`<br>Secrets: `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`<br>Vars: `CLOUDFLARE_R2_BUCKET` | Git tags and GitHub releases on `origin` through `scripts/release.py --publish` (L47, L74); R2 pruning (L65). No repo guard on either job. | none | **replace**: Daily tag and GitHub release on `origin`; keep disabled until the release flow is approved |
| `case-collision-check.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `ci-review-comment.yml` | workflow_run (CI) | contents: read, actions: read, pull-requests: write | none | PR comments in this repo | none | **keep**: Comments only on same-repo PRs from a trusted checkout (fork PRs excluded by its `if`) |
| `ci.yaml` | manual, called, **pull_request (fork PRs)**, push (main) | contents: read, pull-requests: write | none | none | none | **keep**: Orchestrates the validation workflows; top-level `pull-requests: write` is for later hardening |
| `contributor-check.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `deploy-site.yml` | release (published), push (main), manual | contents: read, actions: read, pages: write, id-token: write | Env: `github-pages`<br>Secrets: `APP_PRIVATE_KEY`, `VERCEL_DEPLOY_HOOK`<br>Vars: `APP_CLIENT_ID` | Vercel deploy hook (L50), whose job has no repo guard. GitHub Pages, guarded to `NousResearch/hermes-agent` (L53). Reads `hermes-agent.nousresearch.com` (L152). | none | **replace**: Publishes to or depends on a Nous-owned destination; needs an October-owned site, hook and domain |
| `desktop-bundle-smoke.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `desktop-bundled-release.yml` | called, manual | contents: write, actions: read, packages: write, id-token: write<br>Jobs: 5 jobs: contents: read; 4 jobs: none; `publish-channel`, `commit-builds-summary`: contents: write, actions: read | Env: `release-signing`<br>Secrets: `APPLE_API_ISSUER`, `APPLE_API_KEY_ID`, `APPLE_API_KEY_P8`, `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`, `CSC_KEY_PASSWORD`, `CSC_LINK`, `MS_STORE_CLIENT_ID`, `MS_STORE_CLIENT_SECRET`, `MS_STORE_SELLER_ID`, `MS_STORE_TENANT_ID`, `TERMUX_APT_GPG_KEY`, `TERMUX_APT_GPG_PASSPHRASE`<br>Vars: `AZURE_CLIENT_ID`, `AZURE_SIGN_ACCOUNT`, `AZURE_SIGN_ENDPOINT`, `AZURE_SIGN_PROFILE`, `AZURE_SIGN_PUBLISHER`, `AZURE_TENANT_ID`, `CLOUDFLARE_R2_BUCKET`, `CLOUDFLARE_R2_PUBLIC_URL`, `MS_STORE_PRODUCT_ID` | R2 (L278), Microsoft Store through `microsoft/microsoft-store-apppublisher`, Termux APT signing keys, Azure and Apple signing | `azure/login`, `microsoft/microsoft-store-apppublisher` | **replace**: Builds, signs and ships desktop releases with inherited identities; needs October-owned signing, store and bucket |
| `docker-lint.yml` | called | contents: read | none | none | `hadolint/hadolint-action`, `ludeeus/action-shellcheck` | **keep**: Validation only; no publishing |
| `docker.yml` | **pull_request (fork PRs)**, push (main), called | contents: read | Env: `container-publish`<br>Secrets: `DOCKERHUB_TOKEN`, `DOCKERHUB_USERNAME` | Docker Hub `nousresearch/hermes-agent` (L61). Main-branch publish is guarded (L326, L425). `release-publish` (L510) and `release-publish-manifest` (L607) are not; only the `container-publish` environment and Docker Hub credentials stand in the way. | `docker/build-push-action`, `docker/login-action`, `docker/setup-buildx-action` | **replace**: Publishes to or depends on a Nous-owned destination; `release-publish` has no repo guard, so it needs an October image name before any release run |
| `docs-site-checks.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `e2e-desktop-core.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `e2e-desktop-update.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `e2e-desktop.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `history-check.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `icons-freshness-check.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `infographic-check.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `install-e2e-macos-run.yml` | called | contents: read | none | Downloads from `hermes-assets.nousresearch.com` by default (L54) | none | **replace**: Test leg downloads Nous's installer by default; repoint to a Loki asset |
| `install-e2e-red.yml` | workflow_run (Install & Update E2E), manual | contents: read<br>Jobs: `track`: contents: read, actions: read, issues: write | none | Issues in this repo | none | **keep**: Tracks scheduled failures as issues in this repo; useful only if `install-e2e.yml` runs on schedule |
| `install-e2e-run.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `install-e2e-windows-run.yml` | called | contents: read | none | Downloads from `hermes-assets.nousresearch.com` by default (L79) | none | **replace**: Test leg downloads Nous's installer by default; repoint to a Loki asset |
| `install-e2e.yml` | manual, called, schedule (20 7,19 * * *), **pull_request (fork PRs)** | contents: read | none | none | none | **keep**: Validation only, once its macOS and Windows legs are repointed |
| `js-autofix.yml` | push (main), manual | contents: read<br>Jobs: `apply-patch`: contents: write, pull-requests: write | Env: `trusted-automation`<br>Secrets: `APP_PRIVATE_KEY`<br>Vars: `APP_CLIENT_ID` | `bot/js-autofix` branch in this repo (L191), auto-merged PR (L214) | none | **remove**: Pushes to a bot branch and auto-merges into main with an App token; not needed to validate changes |
| `js-tests.yml` | called | default | none | none | none | **keep**: Validation only; no publishing |
| `label-rerun.yml` | **pull_request (fork PRs)** (labeled) | actions: write, pull-requests: read | none | Actions reruns in this repo | none | **keep**: Reruns CI in this repo when a label is added; no external destination |
| `lazy-deps-guard.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `lint.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `live-providers.yml` | schedule (17 7 * * *), push (v*), manual | contents: read | Secrets: `LIVE_ANTHROPIC_API_KEY`, `LIVE_GEMINI_API_KEY`, `LIVE_NOUS_API_KEY`, `LIVE_OPENAI_API_KEY`, `LIVE_OPENROUTER_API_KEY`, `LIVE_XAI_API_KEY` | Live model provider APIs. Guarded to `NousResearch/hermes-agent` (L39). | `astral-sh/setup-uv` | **remove**: Spends six live provider keys and is guarded to Nous's repo; add back deliberately if Loki budgets for it |
| `lockfile-diff.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `nix.yml` | **pull_request (fork PRs)**, push (main), called | contents: read | none | none | `cachix/install-nix-action`, `nix-community/cache-nix-action` | **keep**: Validation only; no publishing; builds the local flake (L160) |
| `osv-scanner.yml` | schedule (0 9 * * 1), manual | actions: read, contents: read, security-events: write | none | Code scanning results in this repo | `google/osv-scanner-action/.github/workflows/osv-scanner-reusable.yml` | **keep**: Dependency scanning; reports to this repo's code scanning |
| `plugin-catalog-ci.yml` | **pull_request (fork PRs)** | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `pm-bundle.yml` | manual, called, **pull_request (fork PRs)** | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `profile-artifact-check.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `review-labels.yml` | called | contents: read, pull-requests: read | none | none | none | **keep**: Validation only; no publishing |
| `rust-tests.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `sandbox-image.yml` | **pull_request (fork PRs)**, release (published), manual | contents: read | Env: `container-publish`<br>Secrets: `DOCKERHUB_TOKEN`, `DOCKERHUB_USERNAME` | Docker Hub `nousresearch/hermes-sandbox` (L37). All jobs guarded to `NousResearch/hermes-agent` (L42, L78, L132). | `docker/build-push-action`, `docker/login-action`, `docker/setup-buildx-action` | **replace**: Publishes to or depends on a Nous-owned destination; replace only if Loki ships this image, otherwise remove |
| `skills-index-freshness.yml` | schedule (0 */4 * * *), manual | contents: read, issues: write | Env: `trusted-automation`<br>Secrets: `APP_PRIVATE_KEY`<br>Vars: `APP_CLIENT_ID` | Probes `hermes-agent.nousresearch.com` (L38); issues in this repo (L164). Guarded (L21). | none | **replace**: Probes Nous's docs site and is guarded to Nous's repo; repoint with the site |
| `skills-index.yml` | schedule (0 6,18 * * *), manual, push (main) | contents: read, actions: write | Env: `trusted-automation`<br>Secrets: `APP_PRIVATE_KEY`<br>Vars: `APP_CLIENT_ID` | Dispatches `deploy-site.yml` (L108). Guarded (L21). | none | **replace**: Feeds `deploy-site.yml` and is guarded to Nous's repo; repoint with the site |
| `stable-release-publication.yml` | manual, workflow_run (Stable Release) | contents: write, actions: write | Env: `release-signing`<br>Secrets: `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`, `DOCKERHUB_TOKEN`, `DOCKERHUB_USERNAME`, `MS_STORE_CLIENT_ID`, `MS_STORE_CLIENT_SECRET`, `MS_STORE_TENANT_ID`<br>Vars: `CLOUDFLARE_R2_BUCKET`, `CLOUDFLARE_R2_PUBLIC_URL`, `MS_STORE_PRODUCT_ID` | R2 (L63), Docker Hub login, GitHub releases, Microsoft Store credentials | `docker/login-action`, `docker/setup-buildx-action` | **replace**: Reconciles a release across every inherited destination; needs approved October-owned targets |
| `stable-release.yml` | manual | contents: read<br>Jobs: `admit`: contents: write; `ci`: contents: read, pull-requests: write, actions: read, security-events: write; `termux-checks`, `install-e2e`: contents: read, actions: read; 7 jobs: contents: write, actions: read, packages: write, id-token: write; `complete`: contents: write, actions: read | Env: `release-signing`<br>Secrets: `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`, `DOCKERHUB_TOKEN`, `DOCKERHUB_USERNAME`<br>`secrets: inherit`<br>Vars: `CLOUDFLARE_R2_BUCKET`, `CLOUDFLARE_R2_PUBLIC_URL` | Calls `docker.yml` for publishing (L522) and `desktop-bundled-release.yml` (L538); R2 (L304). `scripts/releases/docker.py:18` hard-codes `nousresearch/hermes-agent`. | `docker/login-action`, `docker/setup-buildx-action` | **replace**: Top-level release driver for every inherited destination; needs approved October-owned targets |
| `supply-chain-audit.yml` | called | pull-requests: write, contents: read | none | PR comments in this repo | none | **keep**: Validation only; no publishing |
| `termux-verify.yml` | push (main), **pull_request (fork PRs)**, manual, called | contents: read, actions: read | Env: `release-signing`<br>Secrets: `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_ACCOUNT_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`<br>Vars: `CLOUDFLARE_R2_BUCKET` | R2 bucket from `vars.CLOUDFLARE_R2_BUCKET` (L126). Job guard is event-only (L99). | none | **replace**: Contract and TUI jobs are validation; the R2 jobs need splitting out or an October-owned bucket |
| `tests-os.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `tests.yml` | called | contents: read | none | none | `astral-sh/setup-uv` | **keep**: Validation only; no publishing |
| `uv-lockfile-check.yml` | called | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `windows-bundle-sdk.yml` | **pull_request (fork PRs)**, manual | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `windows-install-update-e2e.yml` | called, push (wine2e-install/**) | contents: read | none | none | none | **keep**: Validation only; no publishing |
| `windows-venv-e2e.yml` | push (wine2e/**), called | contents: read | none | none | none | **keep**: Validation only; no publishing |

## Local actions

There are no local actions outside `.github/actions`. Reusable workflows are the rows above with a **called** trigger.

| Action | Type | Third-party actions (all SHA-pinned) | Proposed decision |
|---|---|---|---|
| `desktop-build-cache` | composite | none | **keep**: Cache helper; no external destination |
| `detect-changes` | composite | none | **keep**: Path filter for CI; no external destination |
| `e2e-screen-record` | composite | none | **keep**: Test recording helper; no external destination |
| `get-app-token` | composite | none | **keep**: Mints a GitHub App token; used only by workflows marked replace or remove here, so it stays unused until those are approved |
| `nix-setup` | composite | `DeterminateSystems/nix-installer-action`, `cachix/cachix-action` | **remove**: No callers in this repo, and it targets the `hermes-agent` Cachix cache (L16) |
| `plugin-validate` | composite | none | **replace**: Clones Hermes from `NousResearch/hermes-agent` (L27) to validate plugins; repoint to Loki. No callers in this repo |
| `retry` | composite | none | **keep**: Retry helper; no external destination |
| `save-pm-cache` | composite | none | **keep**: Cache helper; no callers found in this repo, so review whether it is still needed |
| `setup-pm` | composite | none | **keep**: Toolchain setup; no destination in this file |
| `setup-pm/prune` | node24 | none | **keep**: Cache cleanup for `setup-pm` |
| `setup-windows-build-deps` | composite | none | **keep**: Build dependency setup; no external destination |

## Keeping the Hermes upstream sync separate

`UPSTREAM.md` defines the sync as a manual process: maintainers fetch a read-only `upstream` remote, review, and merge a `sync/hermes-YYYY-MM-DD` branch through a reviewed pull request. It says not to mirror or merge upstream automatically.

No workflow here fetches from or pushes to that remote, and none automates the sync. The only clone of the Hermes repository in `.github` is in `plugin-validate` (L27). It is read-only and marked replace.

To keep it that way, later PRs should add no scheduled or automatic sync workflow. A sync can also bring back workflows this inventory removes or replaces, so each sync PR should compare `.github` against this document.

## Not covered here

- Hardening `permissions`, reviewing action publishers, or checking scripts for expression injection.
- Listing every required secret with its purpose (criterion 5).
- Choosing October-owned destinations for the replace rows.
- Enabling any workflow.
