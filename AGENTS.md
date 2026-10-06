# AGENTS.md (fork-local)

This is a fork of an upstream code-interpreter monorepo. Our only goal here is
building and publishing container images that an on-premise orchestrator
pulls and runs. We do not develop, review, or maintain the application code
in this fork.

## Scope of work in this fork

In scope:
- `.github/workflows/images.yml`, `.github/scripts/image-revision.sh`,
  `.github/workflows/open-sync-pr.yml`, `.github/workflows/release.yml` —
  build/publish/sync pipelines.
- `docker/`, `docker-compose*.yml`, and the `Dockerfile*` files themselves,
  strictly for build/publish correctness (targets, platforms, tags, caching).
- Local build validation before triggering GitHub Actions (see below).

Out of scope — never edit, refactor, or "fix" for correctness/style:
- Anything under `api/src`, `service/src`, `packages/`, `launcher/src`,
  `shared/`, or other application source. Upstream owns that code.
- Feature work, bug fixes in application logic, dependency bumps in app code.

## Upstream sync

Snapshots arrive from the internal monorepo on the `sync/main` branch (pushed
via deploy key). `open-sync-pr.yml` opens a PR from `sync/main` into `main` if
one isn't already open. Merging that PR is the release step — `main` accepts
no direct pushes. Do not hand-merge or rebase app code changes outside this
flow; let the sync PR carry them.

## Workflow status

Only `images.yml` ("Images") is active for image builds in
this fork. `ci.yml` and `release.yml` are disabled manually via the GitHub
API (repo Actions settings), not deleted — they stay in the tree so upstream
sync diffs stay clean, but they never run here. `release.yml` also cuts
Helm chart releases and GitHub Releases, which this fork doesn't use since we
only care about the images.

Don't re-enable `ci.yml`/`release.yml` as part of routine work in this fork.

## Image publish pipeline

`images.yml` runs on main pushes that change image inputs, relevant PRs, and
manual dispatch. On main it publishes upstream's image set to
`ghcr.io/<owner>/code-interpreter-<name>`, with immutable `sha-<commit>` tags
and a `main` tag promoted after all images are available. PRs build without
publishing:

- `api` — `service/Dockerfile`, target `api`
- `worker` — `service/Dockerfile`, target `worker`
- `sandbox-runner` — `api/Dockerfile`, target `sandbox-runner-true`
- `sandbox-runner-direct` — `api/Dockerfile`, target `sandbox-runner-false`
- `file-server` — `service/Dockerfile`, target `production`
- `tool-call-server` — `service/Dockerfile.tool-call-server`, target `production`
- `egress-gateway` — `service/Dockerfile.egress-gateway`, target `production`

Target platform is `linux/amd64` only — the on-premise orchestrator runs
amd64 hosts exclusively, no other architecture is needed.

`sandbox-runner` is the heaviest build (nsjail compiled from source, a
Rust/Cargo libkrun launcher, a full baked language-runtime package tree, and
an ext4 rootfs image) — expect it to dominate total pipeline time.

## Test locally before using GitHub runners

GitHub-hosted runners are slow/expensive for these builds and should be used
to confirm a build that has already been validated locally, not to discover
Dockerfile errors. Before pushing a workflow change or re-running `images.yml`:

```
docker compose -f docker-compose.local-dev.yml build sandbox
```

This builds the same effective target as the `sandbox-runner` job
(`api/Dockerfile`, target `sandbox-runner-${KVM_ENABLED:-true}`, which
defaults to `sandbox-runner-true`) using the
local Docker Desktop engine. It builds for the host's native architecture
only (no QEMU), so it validates Dockerfile/build-logic correctness fast, even
though it isn't a byte-for-byte platform match with the amd64 CI build.

For the other matrix images, build the equivalent target/Dockerfile pair
directly with `docker buildx build -f <dockerfile> --target <target> .`
before relying on CI.
