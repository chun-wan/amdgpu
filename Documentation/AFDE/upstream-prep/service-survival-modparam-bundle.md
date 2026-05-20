# amdgpu / amdkfd production-validated modparam bundle (upstream-prep)

> Status: **prep doc only** -- no code patches yet.  This document is
> the planning artifact for a series of per-modparam patches that will
> be sent to amd-gfx@lists.freedesktop.org and opened as a corresponding
> PR series against ROCm/amdgpu master.

## Motivation

A production MI300X serving deployment was validated 11+ hours stable
under a customer multi-tenant workload using the modparam combination
listed in section "Modparam bundle" below.  Each modparam in the
bundle has a small, isolated kernel-side code change.  Together they
form the kernel-layer half of a three-layer service-survival stack
(CLR / ROCr / kernel) that lets the userspace runtime opt into a
non-abort recovery path on hardware faults.

This prep doc plans how the bundle will be split for upstream review.

## Submission ground rules (apply to every patch in the series)

Each generated patch under this series MUST:

1. **Subject prefix**: `drm/amdgpu:` for `src-tree/amd/amdgpu/`
   touches, `drm/amdkfd:` for `src-tree/amd/amdkfd/` touches.
2. **Signed-off-by**: required (DCO).  Author identity will be the
   chun-wan identity used in the AFDE internal forks.
3. **One logical change per patch**: one modparam per patch.  20 amdgpu
   + 2 amdkcl = 22 patches in the series (some may merge if their
   code-flows are tied, see "Grouping" below).
4. **No debug-only prints in shipped patches**: `dev_dbg` /
   `pr_debug` allowed only if guarded behind a `module_param` and
   off by default; verbose `printk(KERN_INFO ...)` MUST be removed
   from the patch (kept only in the internal fork for triage).
5. **Maintainer cc**: each patch's send-email cc list will be
   generated via `scripts/get_maintainer.pl <patch>` and pasted in
   the patch's `--cc` line.  Cover letter cc = union of all per-patch
   maintainer lists.
6. **Cover letter (series 0/N)**: describes the bundle as a whole,
   references the userspace-side companion PRs in ROCm/rocm-systems,
   and includes the production deployment recipe.
7. **No internal codenames**: no `V17.x`, `K-N`, `RC4-firewall`, or
   AFDE-internal commit SHAs in patch text.  Reference internal
   sources only as "an internal amdgpu fork" with no branch name.

## Modparam bundle

The 20 amdgpu modparams + 2 amdkcl modparams shipped in the validated
production stack:

```
options amdgpu \
    rdma_dereg_timeout_ms=4000 \
    kfd_wait_max_ms_per_wall=5000 \
    kfd_survival_slow_ioctl_ms=4000 \
    kfd_free_wait_ms=4000 \
    kfd_unpin_drain_ms=3000 \
    kfd_free_on_pinned=1 \
    pin_orphan_timeout_ms=10000 \
    pin_reaper_interval_ms=2000 \
    gtt_lock_timeout_ms=4000 \
    gtt_multi_window=4 \
    rdma_pin_debug=1 \
    sdma_fence_watchdog_ms=30000 \
    ttm_error_fence_timeout_ms=4000 \
    dmabuf_pin_max_mb=0 \
    dmabuf_reject_new_pins=0 \
    bo_sync_wait_max_ms=30000 \
    gtt_cgroup_reserve_mb=0 \
    kfd_pin_queue_svm_pages=0 \
    kfd_pin_queue_svm_max_mb=256 \
    kfd_defer_queue_eviction=1

options amdkcl suballoc_timeout_ms=4000 suballoc_force_reclaim=1
```

The default for each new modparam is the upstream stock behaviour;
the validated production values listed above are recommended only
in conjunction with the userspace runtime envs documented in the
companion PR (rocm-systems / `rocr: ROCR_SIGNAL_WAIT_MAX_MS`).

## Grouping (planned 22 patches -> ~10 logical groups)

Patches that operate on the same struct / file may merge to avoid
churn-only patches:

| Group | Patches | Files |
|---|---|---|
| G1 RDMA-pin | rdma_dereg_timeout_ms, rdma_pin_debug | `amdgpu_drv.c`, `amdgpu_rdma.c` (if present) |
| G2 KFD wait | kfd_wait_max_ms_per_wall, kfd_survival_slow_ioctl_ms | `kfd_events.c`, `kfd_chardev.c` |
| G3 KFD free | kfd_free_wait_ms, kfd_unpin_drain_ms, kfd_free_on_pinned | `amdgpu_drv.c` |
| G4 PIN reaper | pin_orphan_timeout_ms, pin_reaper_interval_ms | `amdgpu_drv.c` |
| G5 GTT contention | gtt_lock_timeout_ms, gtt_multi_window, gtt_cgroup_reserve_mb | `amdgpu_drv.c` |
| G6 SDMA fence | sdma_fence_watchdog_ms | `amdgpu_drv.c` |
| G7 TTM error | ttm_error_fence_timeout_ms | `amdgpu_drv.c` |
| G8 DMA-buf pin | dmabuf_pin_max_mb, dmabuf_reject_new_pins | `amdgpu_drv.c` |
| G9 BO sync wait | bo_sync_wait_max_ms | `amdgpu_sync.c` |
| G10 KFD queue lifecycle | kfd_pin_queue_svm_pages, kfd_pin_queue_svm_max_mb, kfd_defer_queue_eviction | `kfd_queue.c`, `kfd_svm.c` |
| G11 amdkcl suballoc | suballoc_timeout_ms, suballoc_force_reclaim | `amdkcl/` (separate dkms module) |

Each group = 1 PATCH in the mailing-list series.  PR series on
GitHub will mirror the same 11 patches.

## Companion userspace PRs (rocm-systems)

- `users/chun-wan/rocr-bound-interrupt-signal-wait` -- adds
  `ROCR_SIGNAL_WAIT_MAX_MS` to the ROCr runtime (the userspace per-call
  cap that layers above `kfd_wait_max_ms_per_wall` in G2 of this
  bundle).
- `users/chun-wan/rocr-service-survival` -- adds `ROCR_SERVICE_SURVIVAL`
  master gate (non-abort VMFaultHandler / memory error handler).
- `users/chun-wan/rocclr-bound-signal-wait` -- adds `HIP_MAX_SIGNAL_WAIT`
  (CLR-side outer-loop cap).

Reviewers can land the kernel patches in this bundle independently of
the userspace PRs; the kernel modparams default to existing upstream
behaviour, so the bundle is a no-op without the userspace gating envs.

## TODO before sending each patch

- [ ] Generate per-patch diff from internal amdgpu fork using
  `git format-patch -1 --subject-prefix='drm/amdgpu PATCH'`
- [ ] Strip any `pr_info("rc-N firewall: ...")` / `dev_info` debug
  lines that exist in the internal fork commit
- [ ] Add `Signed-off-by:` trailer
- [ ] Run `scripts/checkpatch.pl --strict <patch>` and fix style
- [ ] Run `scripts/get_maintainer.pl <patch>` -> populate `--cc`
- [ ] Sanity build (defconfig + amdgpu) before send
