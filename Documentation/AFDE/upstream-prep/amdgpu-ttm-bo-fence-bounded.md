drm/amdgpu: bound TTM error path + BO fence OOM fallback waits

Two callers of dma_fence_wait_any_timeout(MAX_SCHEDULE_TIMEOUT) inside
the TTM error-recovery path and the BO-fence OOM fallback path can park
on a wedged fence forever, blocking the eviction worker thread.

Bound both calls with a new modparam ttm_error_fence_timeout_ms
(default 0 == unbounded for back-compat). When set, the wait returns
-ETIME and the caller falls through to the existing error-cleanup
path, which logs + marks the BO error rather than holding the eviction
worker.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/docs/TIMEOUT_DEATH_MAP.md
