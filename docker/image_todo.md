# Fork container image — todo (see image_design.md for detail)

## Phase 1 — workflow
- [x] image.yml: triggers, build matrix, smoke test, digest upload
- [x] image.yml: merge job creating the multi-arch manifest with the tag set
- [x] YAML validity check

## Phase 2 — land and verify
- [x] PR to main, merged
- [x] Upstream Docker workflow guarded to run only in warp-tech/warpgate
- [x] arm64 build: swap step for the LTO link, verified by a dispatch run
- [ ] Same swap step in build.yml; arm64 Build job green on the PR
- [ ] Manual run on main succeeds; manifest lists amd64 and arm64
- [ ] Image runs locally: version, unattended-setup, healthcheck
