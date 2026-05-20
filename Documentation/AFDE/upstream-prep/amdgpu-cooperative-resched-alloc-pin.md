drm/amdgpu: cooperative cond_resched in BO alloc / pin hot path

amdgpu_amdkfd_gpuvm_alloc_memory_of_gpu() and the surrounding pin path
hold the BO lock and the DRM TTM resource lock for the entirety of a
multi-MB allocation, with no preemption points. Under a 1024MB
peer-direct RDMA pin storm we observed >2s soft-lockup warnings from
the kernel scheduler complaining about the alloc thread.

Insert cond_resched() calls at clear "between sub-units of work" points
in the alloc / pin path (after each 256MB chunk is mapped) so the
scheduler can preempt the holder without dropping the lock. This is
the AMD analogue of NVIDIA's chunked-pinning loop with explicit
yield points.

Behaviour is unchanged on small allocations; the cond_resched() is
a no-op when the scheduler has no work waiting.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/reproducers/multistream_combo/README.md
