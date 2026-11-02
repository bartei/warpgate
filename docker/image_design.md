# Fork container image — design

## Goal

Publish multi-arch (amd64, arm64) images of this fork to `ghcr.io/bartei/warpgate` from
`main` and from release tags, using the existing `docker/Dockerfile`, so the fork's build
can be deployed. No Helm chart publishing.

## Why not upstream's `docker.yml`

- It hardcodes `IMAGE_NAME: warp-tech/warpgate`, so on the fork it pushes to upstream's
  namespace with the fork's token and fails.
- It carries a `schedule` trigger, which makes GitHub park the whole workflow in the
  `disabled_fork` state; it builds on every branch and every pull request (20+ minutes
  each, two runners); and it signs, attests and SBOMs the image and tests the Helm chart,
  none of which this fork needs now.
- Editing it would conflict with every future upstream merge of that file.

## Decisions

**D1 — A separate workflow, `.github/workflows/image.yml` ("Image").** Upstream's
`docker.yml` gets one job-level guard, `if: github.repository == 'warp-tech/warpgate'`,
so it is a no-op on any fork (its other jobs depend on that one and are skipped) and
needs no manual disabling; the guard is a one-line merge against future upstream changes.
`.gitignore` ignores `.github` but re-includes `.github/workflows`, so the new file is
tracked.

**D2 — Triggers:** `push` to `main`, `push` of `v*.*.*` tags, and `workflow_dispatch` for
a manual build. No schedule (fork-disabled and a daily 2×20 min bill), no pull requests
(the test suite already gates PRs; an image per PR is not needed).

**D3 — Multi-arch exactly as upstream does it:** one build job per native runner
(`ubuntu-24.04`, `ubuntu-24.04-arm`) pushing by digest, then a merge job creating the
manifest list with `docker buildx imagetools`. `provenance: false` and empty `attests`
are kept: the default provenance manifest adds an `unknown/unknown` platform entry
(upstream #1255). The GitHub Actions build cache is scoped per platform. The arm64
runner occasionally dies silently; the merge needs both digests, so a failed build is
re-run rather than published single-arch.

**D4 — Tags:** `main` (branch), `sha-<short>`, `<semver>` and `<major>.<minor>` on
release tags, and `latest` following the default branch. Rejected: upstream's
`branch-main` prefix, which exists only to keep branch images apart from releases;
this workflow builds one branch.

**D5 — The smoke test stays:** `version`, `unattended-setup` into a volume, then
`healthcheck` against a running container, on each architecture before the digests are
merged. It is the only check that the image actually starts.

**D6 — Image name `ghcr.io/${{ github.repository }}`**, which is `bartei/warpgate` here
and would be upstream's name upstream. Permissions: `contents: read`, `packages: write`.
The first push creates the package; its visibility is set in the package settings
afterwards if pulls without a token are wanted.

**D7 — `docker/Dockerfile` is unchanged.** It builds from source inside the image with
the git metadata copied in, and enables `mysql,postgres,rdp-openssl-tls`.

**D8 — Swap on the build runners.** The first `main` run OOM-killed `rustc` on arm64
during the fat-LTO link of the `warpgate` binary (`SIGKILL`, then buildx reporting
"cannot allocate memory"), while amd64 passed and upstream's identical arm64 build
passes on the same runner. The link's peak sits at the edge of the free runner's RAM,
so a second swap file (6 GB at `/swapfile-lto`; the runner image already has an
active 4 GB `/swapfile`, and `fallocate` on that path fails with "Text file busy") is
added before the build on both runners; the build container runs without a memory limit and so uses the host's
swap. Rejected: `lto = "thin"` or the `release-no-lto` profile for arm64 (a different
binary from upstream's and from amd64), fewer `-Zthreads` (the fat-LTO peak is one
LLVM module and does not scale down with threads), and paid larger runners.

## Files

- `.github/workflows/image.yml` (new); this design and `docker/image_todo.md`.

## Deploy / blast radius

- A new workflow on the fork only. Nothing in the product changes. Each `main` push now
  costs two ~20-minute runners plus a short merge job.
- Upstream's Docker workflow is skipped on the fork by its guard; nothing else changes.

## Verification

1. The workflow is valid YAML and `workflow_dispatch` on `main` succeeds.
2. `ghcr.io/bartei/warpgate:main` exists and its manifest lists `linux/amd64` and
   `linux/arm64`.
3. `docker run --rm ghcr.io/bartei/warpgate:main version` prints the fork's version
   locally, and `unattended-setup` + `run` + `healthcheck` pass in the smoke step.
4. The arm64 build job completes the LTO link with the swap step in place (the step
   logs `free -h` so the headroom is visible).
