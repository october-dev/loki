# Upstream maintenance

Loki is a standalone derivative of
[Nous Research's Hermes Agent](https://github.com/NousResearch/hermes-agent).
It retains the upstream Git history so maintainers can review and merge later
changes without pretending the code appeared from nowhere.

## Repository roles

- `origin`: `https://github.com/october-dev/loki.git` — Loki's source of truth
- `upstream`: `https://github.com/NousResearch/hermes-agent.git` — read-only
- imported base: `18a76be124d7c16ed98b629a358b23fef76a7f46`

For a new clone, configure the upstream remote once:

```bash
git remote add upstream https://github.com/NousResearch/hermes-agent.git
git remote set-url --push upstream no_push
git remote -v
```

The disabled push URL is a local safety rail. It does not prevent creating an
intentional patch or pull request for Hermes through a separate contributor
fork.

## Review upstream changes

```bash
git fetch upstream --tags
git log --oneline main..upstream/main
git diff --stat main...upstream/main
```

Do not automatically mirror or merge upstream. First review security fixes,
dependency changes, migrations, installers, release automation, and changes at
Loki's compatibility boundary.

## Merge an upstream batch

```bash
git switch main
git pull --ff-only origin main
git switch -c sync/hermes-YYYY-MM-DD
git merge --no-ff upstream/main
```

Resolve conflicts with these invariants:

1. Loki's public name, `loki` command, repository URLs, README, identity, and
   security contact remain Loki-owned.
2. `hermes_cli`, `HERMES_*`, `~/.hermes`, and legacy commands remain available
   until a documented migration replaces them.
3. Release and publishing workflows never target Nous Research accounts or
   package names from the Loki repository.
4. Original authorship, commit history, and MIT notices remain intact.
5. Upstream behavior changes receive the same tests and review as native Loki
   changes.

Run the validation described in [CONTRIBUTING.md](CONTRIBUTING.md), push the
sync branch to `origin`, and merge it through a reviewed pull request. Prefer a
small cherry-pick for an isolated urgent fix; use a merge commit for a coherent
upstream batch so ancestry remains clear.

## Contributing back

When a Loki fix applies cleanly to Hermes Agent, prepare it without Loki-only
branding or assumptions and submit it through the upstream project's process.
Keep the upstream commit or pull-request link in the corresponding Loki change.
