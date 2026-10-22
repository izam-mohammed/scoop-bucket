# the bucket's workflow actions stay current and security reports have a private route

## The ask

The owner asked, on 22/10/2026, to "have dependabot in all of my repos" and
to "have proper security.md file for all of my repos". A licence was asked
only for repositories that are a product; this bucket holds manifests that
point at another repository's releases, so it gets none.

The repository holds one Scoop manifest (`claim.json`), Scoop's schema
copied into `.github/schema/`, and one workflow (`.github/workflows/check.yml`)
that checks every manifest. It has no package manifest of its own, so the
only ecosystem Dependabot can update is `github-actions`.

## Alternatives

### Decision 1: which bot keeps dependencies current? (two-way door)

The workflow pins two actions to full commit SHAs; something must raise a
pull request when either has a new release.

Searched 22/10/2026: "dependabot github-actions sha pinned", "renovate vs
dependabot github actions"; GitHub's Dependabot options reference
(docs.github.com, `cooldown` and `groups`), Renovate's docs on
`github-actions` managers.

| Must | Dependabot | Renovate |
| --- | --- | --- |
| Updates SHA-pinned actions and keeps the version comment | yes | yes |
| Runs with no app install or token to manage | yes | no (needs the Renovate app) |
| **Survives** | **yes** | no |

| Criterion | Weight | Dependabot |
| --- | ---: | ---: |
| Performance efficiency | 40 | 4 |
| Reliability | 30 | 4 |
| Compatibility | 30 | 5 |
| **Weighted total** | **100** | **4.30** |

- Dependabot compatibility 5: the owner's other repositories already run it
  with the same config shape (`decisionsmith/.github/dependabot.yml`).

**Chosen: Dependabot.** It is built into GitHub, needs no app, and matches
the owner's other repositories. Revisit if grouped updates stop being
enough.

## The decision

Add `.github/dependabot.yml` with one `github-actions` entry (weekly,
`cooldown: default-days: 7`, one group, prefix `ci(deps)`), modelled on
`decisionsmith/.github/dependabot.yml` (Decision 1). Add `SECURITY.md`:
the repository is public, so reports go through GitHub private
vulnerability reporting. No release exists, so only the latest commit on
`main` gets fixes. No licence. Dependencies and permissions: none.

Security: no code, input, secret or permission changes; the change adds
one config file and one policy document.

## Where it lands

| Path | New or edit | Phase | What changes | Firm? |
| --- | --- | --- | --- | --- |
| `.github/dependabot.yml` | new | 2 | weekly `github-actions` updates | fixed |
| `SECURITY.md` | new | 3 | the security policy | fixed |

*Reused, not rewritten:*

- `decisionsmith/.github/dependabot.yml` and `decisionsmith/SECURITY.md`:
  the shape both new files follow.

## Declined

- Renovate: needs an app install, and the owner's other repositories run
  Dependabot.
- A licence: the bucket is packaging metadata for another repository, which
  carries its own licence (`"license": "MIT"` in `claim.json`).

## What the tests hold

The workflow `check.yml` still passes on the PR; the YAML parses; GitHub
shows the Dependabot config as valid on the PR.

## Phases

### Phase 1: topics

- [x] The repository topics set to `json-schema`, `manifests`,
  `package-manager`, `scoop`, `scoop-bucket`, `windows` with
  `gh repo edit`; check: `gh repo view --json repositoryTopics` lists them.

### Phase 2: Dependabot

- [x] `.github/dependabot.yml` lands with one `github-actions` entry;
  check: the file parses with `yaml.safe_load`.

### Phase 3: security policy

- [x] `SECURITY.md` lands with supported versions, the private reporting
  link, what happens next and the scope (the manifests and `check.yml`);
  check: every link in it points at this repository.

### Phase 4: open the PR

- [x] The branch pushed and its PR opened against `main` (or, if one is
  already open for this branch, its body brought up to date) with the
  title `chore: dependabot and security policy`, a body naming this plan,
  what changed and why, and the checks run with their result. The PR's
  URL printed as the evidence. Opened, never merged.
