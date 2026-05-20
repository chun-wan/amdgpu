drm/amdkfd: cap WAIT_EVENTS via kfd_wait_max_ms_per_wall modparam

Under cross-GPU IPC events plus heavy SDMA load, the per-event
WAIT_EVENTS ioctl can park in InterruptSignal::WaitRelaxed() forever
when the corresponding SDMA fence never fires (e.g. the GPU is being
reset, or an upstream RDMA pinned buffer counter went underflow and
all subsequent pins are silently rejected). In a multi-tenant serving
fleet this lets one bad request hang every healthy peer thread.

Introduce a wall-clock cap per WAIT_EVENTS ioctl invocation, expressed
in milliseconds via a new module parameter kfd_wait_max_ms_per_wall
(default 0 == unbounded == existing behaviour). When set, the ioctl
returns -ETIME after the deadline, letting user-space (CLR / ROCr)
surface a survivable hipErrorOperationFailed instead of parking
the caller.

The cap is per-poll: each individual poll inside the kfd wait loop
is bounded by min(remaining_caller_timeout, remaining_wall_deadline),
and amdgpu_reset_pending is checked on every poll so a stuck GPU
surfaces -ETIME promptly.

Reported-by: AFDE APAC <afde-apac@amd.com>
Tested-by: chun-wan <chun-wan@users.noreply.github.com>
Signed-off-by: chun-wan <chun-wan@users.noreply.github.com>
Link: https://github.com/AFDEAPAC/alibabaHang/blob/default_backup/reproducers/customer_hang_repro/README.md
