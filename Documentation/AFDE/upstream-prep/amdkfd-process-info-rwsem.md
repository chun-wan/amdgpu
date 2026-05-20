drm/amdkfd: convert amdkfd_process_info.lock from mutex to rw_semaphore

amdkfd_process_info.lock is taken as a writer on every BO alloc / map
/ unmap / restore, and as a reader on every queue create / destroy /
status query / fence check. Today both readers and writers go through
struct mutex, which serializes them all.

Convert to struct rw_semaphore. Writers (alloc/map/unmap/restore)
continue to use down_write(); readers (queue ops, status queries) now
use down_read(), which lets the multi-threaded queue dispatch path
proceed in parallel with each other while still being serialized against
allocator changes.

No behaviour change for single-process workloads; the throughput win
shows up only when N>=4 host threads are concurrently making KFD ioctls
on the same kfd_process.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/delivery/v17.5-rc7-lock-shard/README.md
