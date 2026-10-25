# Security policy

## Supported versions

This bucket has no releases. Only the latest commit on `main` gets fixes, so
run `scoop update` before reporting.

## Reporting a vulnerability

Please do not open a public issue, pull request or discussion for a security
problem.

Report it privately through GitHub:
[Report a vulnerability](https://github.com/izam-mohammed/scoop-bucket/security/advisories/new)
(the "Security" tab, then "Report a vulnerability").

Helpful details:

- the manifest and version involved
- what an attacker could do, and what they need first
- the smallest steps that show it

## What happens next

This is a project with one maintainer, so there are no guaranteed response
times. We aim to acknowledge the report, agree on how serious it is, fix the
manifest, and publish a GitHub security advisory that credits you (unless you
would rather not be named).

## Scope

In scope: the manifests in this repository (for example `claim.json`: a wrong
URL, a wrong or missing hash, an unsafe install step) and the `check`
workflow in `.github/workflows/check.yml`.

A problem in a program the bucket installs belongs to that program's own
repository (for `claim`, [claim-your-code](https://github.com/izam-mohammed/claim-your-code)).
