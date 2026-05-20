drm/amdgpu: bound amdgpu_sync_wait worst-case to surface -ETIME on stuck SDMA/KFD fences

amdgpu_sync_wait() walks the per-BO fence list with dma_fence_wait()
on each fence. If any one fence belongs to a wedged SDMA queue, the
wait hangs forever, holding the BO mmap_sem and blocking every
subsequent ioctl on this kfd_process.

Convert the per-fence dma_fence_wait() into
dma_fence_wait_timeout(remaining) where remaining is the wall-clock
budget passed by the caller. If any fence times out, return -ETIME
so the caller can decide how to recover (typically: mark the BO dead
and surface the error to user-space).

This preserves existing semantics on healthy hardware; only stuck
fences see the new behaviour.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/reproducers/customer_hang_repro/README.md
