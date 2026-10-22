# every manifest is checked by one command, locally and on every PR, with secrets scanned

## The ask

The repository standard asks for lint and format, tests, a coverage floor
of 90 % (target 95 %), and security checks (dependency audit, secret scan,
static analysis where the language has one), all run by one local check
command and by CI on every PR, reusing the shared gate in
`izam-mohammed/.github` rather than a parallel one.

What it means here: this is a Scoop bucket of generated JSON manifests, so
"lint, format and tests" are the manifest checks PR `#2` already runs
(format, Scoop schema, download hashes). The gaps are that those checks
live only inside `.github/workflows/check.yml`, nothing scans for secrets
or lints the workflows, and nothing proves a manifest installs. A 90 %
coverage floor is not applicable: there is no program code to cover. The
check script is instead proved by fixtures that must fail it.

Built so far: `docs/built/BUILT-2026-10-22-1827.md`.

## Alternatives

### Decision 1: where the manifest checks live (two-way door)

Today the three checks are inline `run:` steps in the workflow, so nobody
can run them before pushing.

Searched 2026-10-22: "scoop bucket CI checkver checkurls bucket template
Excavator GitHub Actions"; the ScoopInstaller Excavator README (archived,
"no longer needed"), the Scoop App Manifest Autoupdate wiki, the
ScoopInstaller GithubActions listing.

| Criterion | Weight | Baseline: inline workflow steps | `scripts/check.sh` | Makefile target |
| --- | ---: | ---: | ---: | ---: |
| Performance efficiency | 40 | 4 | 4 | 4 |
| Fit to the ask | 35 | 1 | 5 | 4 |
| Maintainability of the result | 25 | 3 | 4 | 3 |
| **Weighted total** | **100** | **2.70** | **4.35** | **3.75** |

- Efficiency is equal: the same commands run either way; the download
  step dominates (one fetch per `url`).
- Fit: inline steps cannot run locally (1). A POSIX `sh` script runs the
  same on a laptop and in CI (5). A Makefile adds `make`, absent on a
  stock Windows machine where Scoop users are (4).
- Maintainability: one script that CI calls means one place to change; a
  Makefile would still wrap the same shell (3).

**Chosen: `scripts/check.sh`.** One POSIX `sh` script holds the three
checks and CI calls it, so local and CI cannot drift. Revisit if the
bucket grows past a handful of manifests and the download step needs
caching.

### Decision 2: how secrets, workflows and code are scanned (two-way door)

Nothing scans this repository today.

Searched 2026-10-22: "pre-commit.ci vs pre-commit/action GitHub Actions
run hooks on pull request"; the shared gate's README in
`izam-mohammed/.github`, the `repo-template` README.

| Must | Baseline: none | Shared gate (`security-scan.yml`) | gitleaks action alone | pre-commit hooks only |
| --- | --- | --- | --- | --- |
| A committed secret fails the PR | no | yes | yes | no (local only; bot and web edits skip it) |
| **Survives** | no | **yes** | **yes** | no |

| Criterion | Weight | Shared gate | gitleaks action alone |
| --- | ---: | ---: | ---: |
| Performance efficiency | 40 | 3 | 5 |
| Compatibility | 35 | 5 | 2 |
| Security | 25 | 5 | 3 |
| **Weighted total** | **100** | **4.20** | **3.45** |

- Efficiency: the shared gate runs four stages (gitleaks, osv-scanner,
  semgrep, zizmor) where gitleaks alone runs one, so it costs more runner
  time; exact minutes unverified.
- Compatibility: the shared gate is what `izam-mohammed` and
  `izam-mohammed.github.io` already call, with the same sticky PR comment
  (their `.github/workflows/security.yml`).
- Security: zizmor on `check.yml` matters here, since a workflow that
  fetches URLs from manifests is the main attack surface.

**Chosen: the shared gate**, called with `ecosystem: shell` (the check
script is shell), pinned by commit SHA like the other callers. Revisit if
the gate gains a `json` or `data` ecosystem.

### Decision 3: whether a manifest is proved to install (two-way door)

The hash check proves the archive is the one recorded, not that Scoop
can install it and the binary runs.

Searched 2026-10-22: "scoop bucket CI checkver checkurls bucket template
Excavator GitHub Actions"; the Scoop autoupdate wiki (recommends
installing an updated manifest to verify it), the Excavator README.

| Criterion | Weight | Baseline: no install | `scoop install` on a Windows runner | Scoop's `checkver`/`checkurls` scripts |
| --- | ---: | ---: | ---: | ---: |
| Performance efficiency | 40 | 5 | 2 | 3 |
| Fit to the ask | 40 | 2 | 5 | 3 |
| Reliability | 20 | 3 | 4 | 3 |
| **Weighted total** | **100** | **3.40** | **3.60** | **3.00** |

- Efficiency: a Windows runner installing Scoop and one app costs minutes
  per run (2); `checkver` needs PowerShell and Scoop too but downloads
  less (3).
- Fit: only an install and `claim --version` proves the user's path (5).
  `checkver` checks for newer upstream versions, which GoReleaser already
  handles (3).

Sensitivity: shifting 10 points from fit to efficiency flips it to the
baseline (3.70 against 3.30), so this is fragile; it is kept because it
is the only check of what a user does.

**Chosen: `scoop install` on a Windows runner**, in Phase 2, after the
rough cut. Revisit if runner minutes become a constraint.

## The decision

`scripts/check.sh` runs the format, schema and download checks in that
order and stops at the first failure (decision 1);
`.github/workflows/check.yml` calls it instead of holding the logic.
`.github/workflows/security.yml` calls the shared gate (decision 2).
Phase 2 adds an install job on `windows-latest` that adds this bucket from
the checkout and runs each app's `--version` (decision 3).

Dependencies: the shared reusable workflow
`izam-mohammed/.github/.github/workflows/security-scan.yml` (pinned by
SHA), and, in Phase 2, Scoop itself on the Windows runner (installed by
its official script). No secret, no new permission beyond the gate's
`pull-requests: write` for its comment.

Standards: `docs/standards.md` (format, schema and download hashes, today
inline in CI; through `sh scripts/check.sh` once Phase 1 lands).

## Where it lands

| Path | New or edit | Phase | What changes | Firm? |
| --- | --- | --- | --- | --- |
| `scripts/check.sh` | new | 1 | the three manifest checks, POSIX `sh` | expected |
| `tests/fixtures/` | new | 1 | bad manifests the check must refuse | expected |
| `.github/workflows/check.yml` | edit | 1 | steps replaced by `sh scripts/check.sh` | reuse |
| `.github/workflows/security.yml` | new | 1 | calls the shared gate | expected |
| `.github/dependabot.yml` | new | 1 | weekly `github-actions` updates, `ci(deps)` prefix | expected |
| `docs/standards.md` | edit | 1 | the one command, secret scan row | reuse |
| `.github/workflows/check.yml` | edit | 2 | an `install` job on `windows-latest` | expected |

*Reused, not rewritten:*

- `.github/workflows/check.yml`: its three steps move into the script
  unchanged in behaviour, including the vendored schema.
- `izam-mohammed/.github` `security-scan.yml`: the gate itself, called,
  never copied.
- `repo-template` `.github/dependabot.yml` and `.github/workflows/security.yml`:
  copied as the template intends, with `ecosystem: shell`.

## Security

What is protected: the manifests users install from, and the CI that
fetches every URL they name. Who reaches it: pull requests (anyone can
open one), and the release bot pushing to `main`.

| Threat | Here | Handled by | Proved by |
| --- | --- | --- | --- |
| Tampering: a manifest points at a swapped archive | a PR or bot commit changes `url` but not `hash` | the download hash check | a fixture with a wrong hash fails `scripts/check.sh` |
| Information disclosure: a token committed | a hand edit adds a credential | gitleaks in the shared gate, blocking | the gate's stage 1 on the PR |
| Elevation of privilege: a workflow abused through PR input | a manifest URL or field reaching `run:` | the script reads manifests as data through `jq`, never `${{ }}`; `contents: read`; zizmor in the gate | zizmor reports no finding on `check.yml` |

Data: none personal. Secrets: none used. Logged: URLs and hashes only.
Checklist: SLSA build track and the CIS GitHub benchmark for CI settings,
editions to be confirmed when built.

## Declined

- Inline workflow steps (decision 1): cannot run locally.
- A Makefile target (decision 1): adds `make`, which a Windows contributor
  may not have.
- pre-commit hooks only (decision 2): never run for bot or web edits.
- gitleaks action alone (decision 2): a parallel gate beside the shared
  one, and no workflow lint.
- Scoop's `checkver`/`checkurls` (decision 3): version tracking is
  GoReleaser's job here.
- A 90 % coverage floor: no program code to measure.
- A `site/` one-pager: the bucket only delivers; a site belongs to each
  app's own repository.

## What the tests hold

- A manifest with bad JSON, the wrong indent, a schema violation or a
  wrong hash each makes `scripts/check.sh` exit non-zero (fixtures).
- `claim.json` passes unchanged, so GoReleaser's output is accepted.
- On Windows, `scoop install` of each manifest succeeds and the binary
  answers `--version` (Phase 2).

## Follow-ups

- Release-bot commits bypass PRs, so the check reports after publishing.
  Making GoReleaser open a PR instead is a change in each app's
  repository, not here.

## Phases

### Phase 1: the rough cut (needs you: try it)

- [ ] `sh scripts/check.sh` runs the format, schema and download checks on
  every top-level `*.json` and exits non-zero on the first failure
  (expected in `scripts/check.sh`), pinned by fixtures under
  `tests/fixtures/` that each must fail it, run by `sh scripts/check.sh
  --self-test`.
- [ ] `.github/workflows/check.yml` runs `sh scripts/check.sh` and
  `sh scripts/check.sh --self-test` in place of its inline steps.
- [ ] `.github/workflows/security.yml` calls the shared gate pinned by
  commit SHA with `ecosystem: shell`, and `.github/dependabot.yml` updates
  GitHub Actions weekly, both as `repo-template` ships them.
- [ ] `docs/standards.md` names `sh scripts/check.sh` as the one command
  and adds the secret scan row.
- [ ] Check: `sh scripts/check.sh` and `sh scripts/check.sh --self-test`
  pass locally, and the PR shows `check` green and the security gate's
  comment.
- [ ] You try it: run `sh scripts/check.sh` in a clone, then change one
  hash in `claim.json` and run it again, and say whether the failure line
  tells you what is wrong. What you want changed goes into Phase 2.

### Phase 2: prove it installs

- [ ] An `install` job on `windows-latest` adds the bucket from the
  checkout, installs each manifest with `scoop install` and runs its
  binary with `--version` (expected in `.github/workflows/check.yml`).
- [ ] Check: `sh scripts/check.sh` passes, and the `install` job is green
  on the PR with `claim 0.1.0` printed.

### Phase 3: open the PR

- [ ] The branch pushed and its PR opened against the branch it was cut
  from (or, if one is already open for this branch, its body brought up
  to date) with a Conventional Commit title for the whole change, a body
  naming this plan, what changed and why, and the checks run with their
  result. The PR's URL printed as the evidence. Opened, never merged.
