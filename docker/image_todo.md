# Fork container image — todo (see image_design.md for detail)

## Phase 1 — workflow
- [x] image.yml: triggers, build matrix, smoke test, digest upload
- [x] image.yml: merge job creating the multi-arch manifest with the tag set
- [x] YAML validity check

## Phase 2 — land and verify
- [ ] PR to main, merged
- [ ] Upstream Docker workflow disabled on the fork
- [ ] Manual run on main succeeds; manifest lists amd64 and arm64
- [ ] Image runs locally: version, unattended-setup, healthcheck
