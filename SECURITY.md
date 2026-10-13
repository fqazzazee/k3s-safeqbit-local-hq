# Security policy — k3s-safeqbit-local-hq

## Supported versions

Only the current `main` branch is maintained.

## Reporting a vulnerability

Do not open a public issue.

Use [private vulnerability reporting](https://github.com/fqazzazee/k3s-safeqbit-local-hq/security/advisories/new),
or email fadi.qazzazee@gmail.com with the details and, if you have one, a proof of
concept.

Expect an acknowledgment within 72 hours and an assessment within a week. If a fix is
warranted, you will get the timeline and credit in the release notes unless you would
rather not be named.

## Scope

This repository is the GitOps source for a home lab cluster, not software meant for
reuse.

**In scope:** a plaintext credential anywhere in this repository or its history, a
manifest that exposes a service more widely than intended, and anything in the
workflows that could leak or misuse a secret.

**Out of scope:** the presence of `SealedSecret` resources. Their values are encrypted to
the cluster's key and are meant to be public. Also out of scope: vulnerabilities in the
upstream charts and images, which should go to their maintainers.
