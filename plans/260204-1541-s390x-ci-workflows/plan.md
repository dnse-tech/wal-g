---
title: "Add s390x CI/CD Workflows for WAL-G"
description: "Disable upstream workflows, add s390x build support for PostgreSQL, Redis, etcd with upstream sync"
status: approved
priority: P1
effort: 2-3 hours
branch: master
tags: [ci, s390x, github-actions]
created: 2026-02-04
decisions:
  approach: "Binary Release Only (Approach 1)"
  sync_frequency: "Daily auto-sync at 17:00 UTC"
  libsodium: "Include encryption support (USE_LIBSODIUM=1)"
  etcd_flag: "ETCD_UNSUPPORTED_ARCH=s390x required"
---

# Add s390x CI/CD Workflows for WAL-G

## Context

WAL-G is a multi-database backup tool. This plan adds s390x architecture support for:
- **PostgreSQL** (fully supported)
- **Redis** (fully supported)
- **etcd** (supported with `ETCD_UNSUPPORTED_ARCH=s390x`)

**Not building**: MySQL (no XtraBackup for s390x), MongoDB, SQL Server, FDB, Greenplum.

## Current State

**Existing workflows to disable:**
| Workflow | Purpose | Action |
|----------|---------|--------|
| `dockertests-par.yml` | Parallel integration tests | Disable |
| `dockertests-seq.yml` | Sequential integration tests | Disable |
| `dockertests.yml` | Legacy docker tests | Disable |
| `golangci-lint.yml` | Code linting | Disable |
| `golvulncheck.yml` | Vulnerability scan | Disable |
| `release.yml` | Release builds (amd64/arm64) | Disable |
| `unittests.yml` | Unit tests | Disable |
| `windows-native.yml` | Windows builds | Disable |

**New workflows to create:**
| Workflow | Purpose |
|----------|---------|
| `sync-upstream.yml` | Daily sync from wal-g/wal-g releases |
| `release-s390x.yml` | Build s390x binaries on release |
| `build-s390x.yml` | CI validation on push/PR |

---

## Approach 1: Binary Release Only (Recommended)

Build native s390x binaries using QEMU emulation, upload to GitHub releases.

### Pros
- Simple, proven pattern from existing release.yml
- No Docker image management needed
- Users download binaries directly
- Consistent with upstream release format

### Cons
- QEMU emulation slower than native builds
- No container image distribution

### Workflow Design

```
wal-g/wal-g (upstream)
      │ releases v3.0.0
      ▼
┌─────────────────────────────────────┐
│ sync-upstream.yml (daily 17:00 UTC) │
│  Check upstream → Create local tag  │
└─────────────────────────────────────┘
      │ triggers
      ▼
┌─────────────────────────────────────┐
│      release-s390x.yml              │
│  Matrix: pg, redis, etcd            │
│  OS: ubuntu 20.04, 22.04, 24.04     │
│  Build via QEMU → Upload binaries   │
└─────────────────────────────────────┘
```

### Files to Create

1. `.github/workflows/sync-upstream.yml`
2. `.github/workflows/release-s390x.yml`
3. `.github/workflows/build-s390x.yml`

### Files to Disable (rename to .disabled)

All 8 existing workflows.

---

## Approach 2: Docker Image + Binary Release

Build both binaries AND Docker images, publish to GHCR.

### Pros
- Container-ready deployment
- Multi-arch manifest support
- Better for Kubernetes users

### Cons
- More complex workflow
- Need to maintain Dockerfile for s390x
- GHCR storage costs
- Must handle upstream Docker images (amd64/arm64)

### Additional Requirements

- Create s390x-compatible Dockerfile
- Handle base image architecture selection
- Multi-arch manifest creation

---

## Recommendation

**Approach 1 (Binary Release Only)** - Simpler, matches upstream pattern, sufficient for s390x users who typically build from source or use binaries.

---

## Implementation Steps

### Phase 1: Disable Upstream Workflows

```bash
cd .github/workflows
for f in dockertests-par.yml dockertests-seq.yml dockertests.yml \
         golangci-lint.yml golvulncheck.yml release.yml \
         unittests.yml windows-native.yml; do
  mv "$f" "${f}.disabled"
done
```

### Phase 2: Create Sync Workflow

**`.github/workflows/sync-upstream.yml`**
- Schedule: daily at 17:00 UTC
- Check latest release from `wal-g/wal-g`
- Create matching local release if not exists

### Phase 3: Create Release Workflow

**`.github/workflows/release-s390x.yml`**
- Trigger: release published OR manual dispatch
- Matrix: `db: [pg, redis, etcd]` × `os: [20.04, 22.04, 24.04]`
- Build via QEMU linux/s390x
- Upload: binary + tarball + sha256 checksums

### Phase 4: Create CI Workflow

**`.github/workflows/build-s390x.yml`**
- Trigger: push/PR to master
- Validate s390x builds without upload
- Fast feedback on compatibility

---

## Build Matrix

| Database | s390x Support | Notes |
|----------|---------------|-------|
| pg | ✅ | Full support |
| redis | ✅ | Builds from source |
| etcd | ✅ | Needs `ETCD_UNSUPPORTED_ARCH=s390x` |

**OS Matrix:** Ubuntu 20.04, 22.04, 24.04

**Total combinations:** 3 db × 3 os = 9 builds per release

---

## Risk Assessment

| Risk | Mitigation |
|------|------------|
| QEMU build slow | Parallel matrix, build only on release |
| Libsodium s390x issues | Test in CI, fallback to no-libsodium build |
| Upstream API changes | Sync workflow uses stable gh CLI |
| Build failures | CI workflow catches issues before release |

---

## Success Criteria

- [x] All upstream workflows disabled
- [x] Sync workflow creates matching releases
- [x] Release workflow builds all 9 combinations
- [x] CI workflow validates s390x on push/PR
- [ ] Binaries work on actual s390x hardware (requires testing)

---

## Decisions Made

| Question | Decision |
|----------|----------|
| Sync frequency | ✅ Daily auto-sync at 17:00 UTC |
| Libsodium | ✅ Include encryption support (`USE_LIBSODIUM=1`) |
| etcd build | ✅ Use `ETCD_UNSUPPORTED_ARCH=s390x` flag |
| Approach | ✅ Approach 1: Binary Release Only |

## Build Environment Variables

```bash
USE_BROTLI=1
USE_LIBSODIUM=1
USE_LZO=1
ETCD_UNSUPPORTED_ARCH=s390x  # For etcd builds only
```

---

## Validation Summary

**Validated:** 2026-02-04
**Questions asked:** 6

### Confirmed Decisions

| Decision | User Choice |
|----------|-------------|
| Sync behavior | Latest release only |
| Libsodium failure | Fail the build (strict) |
| CI scope | All 9 combinations on every push/PR |
| Artifact naming | `wal-g-{db}-ubuntu{os}-s390x` (e.g., `wal-g-pg-ubuntu24.04-s390x`) |
| Release notes | Copy full release notes from upstream |
| Manual trigger | `workflow_dispatch` with tag input |

### Implementation Notes

1. **CI builds all 9 combos** - slower but comprehensive; expect ~30-45 min CI time
2. **Strict libsodium** - no fallback; ensure s390x libsodium works before release
3. **Artifact naming** - matches upstream pattern: `wal-g-{db}-ubuntu{os}-s390x`
4. **Sync workflow** - include `inputs.tag` for manual override

### Action Items

- [x] Implement all 9 combinations in `build-s390x.yml` matrix
- [x] Add `gh release view -R wal-g/wal-g --json body` to copy release notes
- [x] Add `workflow_dispatch.inputs.tag` to `sync-upstream.yml`
