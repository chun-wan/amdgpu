drm/amdkfd: convert kfd_process.event_mutex to rw_semaphore with shared wait path

kfd_process.event_mutex is held for the full duration of
kfd_wait_on_events(), which can be 100ms-multi-second for the wait
itself. Today struct mutex serializes EVERY kfd_wait_on_events()
caller of the same process, even when they're waiting on completely
different events. With N concurrent worker threads (vLLM + small HIP
services on the same kfd_process), only one thread can be inside
WAIT_EVENTS at a time, and the others queue behind it.

Convert event_mutex to struct rw_semaphore. Event create / destroy /
notify (writers) continue to take down_write(); kfd_wait_on_events()'s
init phase + finalize phase use down_read() by default, falling back
to down_write() only when bit 0 of kfd_lock_shard_mask modparam is
clear (preserving the old serialized behaviour as an emergency rollback).

This lets multiple WAIT_EVENTS callers of the same process truly run
in parallel, with mutual ordering relative to event-creation writers
preserved via the rwsem read-write contract.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/delivery/v17.5-rc7-lock-shard/README.md
