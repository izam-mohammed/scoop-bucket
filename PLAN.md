# scoop-bucket: what is left to build, in order

The bucket already has its banner (PR `#1`) and manifest checks (PR `#2`).
A `site/` one-pager is not applicable: the bucket only delivers apps, and
each app's site belongs in its own repository. What is left is one local
check command, the shared security gate and an install proof.

Each reference is one plan in `plans/`, worked phase by phase and ticked
here when every box in it is ticked.

### Phase 1: the checks, one command, on every PR

- [ ] `plans/01-quality-gates.md` - the manifest checks as one script,
  the shared security gate, and an install proof on Windows.

### Phase 2: repository hygiene

- [x] `plans/02-repo-hygiene.md` - Dependabot and a security policy.
