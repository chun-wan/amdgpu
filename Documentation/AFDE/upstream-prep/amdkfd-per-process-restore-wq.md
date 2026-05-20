drm/amdkfd: per-process kfd_restore_wq to break shared workqueue contention

Today amdkfd uses a single system-wide kfd_restore_work workqueue to run
the per-process queue-restore worker. Under cross-tenant load (vLLM +
torch_service inference fleet on one MI300X), the restore worker for one
slow process can hold the workqueue for >5s, blocking the restore worker
of every other process even though they don't share any GPU memory or
queues with the slow one.

Split this into a per-kfd_process ordered workqueue allocated at process
init. Each process now schedules its own restore work on its own
WQ_UNBOUND queue, so a slow restore on process A no longer stalls the
restore of process B.

The patch is structurally NVIDIA-driver inspired: NVIDIA has per-FD
context, AMD has per-process kfd_process, so per-process WQ is the
correct granularity for us.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/delivery/v17.5-rc7-lock-shard/README.md
