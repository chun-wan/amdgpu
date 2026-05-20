drm/amdgpu: kfd_map_memory_timeout_ms modparam for MapMemoryToGPU lock wait

KFD_IOC_MAP_MEMORY_TO_GPU acquires a process-wide lock around the
actual map; under heavy concurrent map traffic from a single kfd_process
the lock acquisition path observes >5s waits.

Introduce kfd_map_memory_timeout_ms (default 0 == unbounded). When set,
the lock acquire returns -ETIME, allowing user-space to back off and
retry instead of parking the caller. Useful in container environments
where the host scheduler can preempt a process for far longer than the
ioctl's effective deadline.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/docs/K3_MAP_VALIDATE_WALL_DESIGN.md
