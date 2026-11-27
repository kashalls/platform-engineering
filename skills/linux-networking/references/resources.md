# CPU, memory, disk, and limits

## CPU

```bash
uptime                       # load: 1, 5, 15 min
nproc                        # cores, for context
top -b -n1 | head -20
pidstat 1 5                  # per-process CPU over time
vmstat 1 5                   # r (runnable), b (blocked), wa (io wait)
```

Load average counts running **and uninterruptible-sleep** tasks. High load with
low CPU utilization means processes are blocked on I/O, not compute — look at
`vmstat`'s `b` and `wa` columns rather than adding cores.

Steal time (`st` in `vmstat`/`top`) is CPU the hypervisor gave to someone else.
Persistent steal on a VM is a noisy-neighbour or over-subscription problem you
cannot fix from inside the guest.

## Memory

```bash
free -h
cat /proc/meminfo
ps aux --sort=-%mem | head -10
dmesg -T | grep -i -E 'oom|killed process'
```

Read **`available`**, not `free`. Linux uses spare memory for page cache and
returns it on demand; a machine with almost no `free` memory and plenty of
`available` is working as designed.

`ps` columns mislead in opposite directions:

- **RSS** counts shared pages once per process, so summing RSS across processes
  double-counts shared libraries and overstates usage.
- **VSZ** is address space, including never-touched mappings. It overstates
  wildly and is almost never the number you want.
- **PSS** (in `/proc/<pid>/smaps_rollup`) divides shared pages fairly and is the
  honest per-process figure.

```bash
grep Pss /proc/<pid>/smaps_rollup
```

The OOM killer logs its decision, including the score and what it chose:

```bash
dmesg -T | grep -A5 'Out of memory'
journalctl -k --since "1 hour ago" | grep -i oom
```

In a container, an OOM kill may come from the **cgroup** limit rather than the
host running out — the host can have plenty free while one cgroup is at its cap.

## cgroups (containers)

```bash
# cgroup v2
cat /sys/fs/cgroup/memory.max
cat /sys/fs/cgroup/memory.current
cat /sys/fs/cgroup/memory.events        # look for "oom_kill"
cat /sys/fs/cgroup/cpu.stat             # throttled_usec, nr_throttled
cat /sys/fs/cgroup/cpu.max              # quota period
```

`cpu.stat`'s `throttled_usec` is the number to look at when a container is slow
but not CPU-saturated: CPU limits throttle rather than kill, producing latency
with no error and no restart. Rising `nr_throttled` is proof.

`memory.events` with a nonzero `oom_kill` confirms a cgroup OOM even after the
process is gone.

For cgroup v1 the paths differ (`/sys/fs/cgroup/memory/memory.limit_in_bytes`).
Check which version is mounted:

```bash
stat -fc %T /sys/fs/cgroup      # cgroup2fs = v2, tmpfs = v1
```

## Disk

```bash
df -h                        # space
df -i                        # inodes — a separate exhaustion mode
du -xh --max-depth=1 / | sort -h | tail -20
lsof +L1                     # deleted files still held open
iostat -xz 1 5               # per-device utilization and await
```

Three ways a disk is "full":

1. **No blocks** — ordinary. `du` finds it.
2. **No inodes** — millions of tiny files. `df` looks fine, writes fail with
   `No space left on device`. `df -i` is the only way to see it.
3. **Deleted-but-open files** — a rotated log a process still holds. `du` shows
   nothing; the space returns only when the holder closes it or restarts.
   `lsof +L1` lists them.

`du -x` stays on one filesystem, which stops it wandering into `/proc`, network
mounts, and bind mounts.

In `iostat -xz`, `%util` near 100 means the device is saturated (misleading for
SSDs, which handle parallel requests); `await` rising well above `svctm` means
requests are queueing.

## File descriptors

```bash
ulimit -n                              # current shell's soft limit
cat /proc/<pid>/limits                 # a running process's actual limits
ls /proc/<pid>/fd | wc -l              # how many it currently holds
sysctl fs.file-nr fs.file-max          # system-wide
lsof -p <pid> | wc -l
```

`Too many open files` is per-process by default. A process that leaks sockets
hits its own limit while the system-wide count is fine.

For systemd services, `ulimit` in a shell is irrelevant — the unit's own
`LimitNOFILE=` governs. Setting it in `/etc/security/limits.conf` does nothing
for a systemd-started daemon, which is a very common wasted afternoon.

## Ephemeral ports and TCP tuning

```bash
sysctl net.ipv4.ip_local_port_range
ss -s
sysctl net.ipv4.tcp_tw_reuse
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog
```

A client making many short-lived outbound connections can exhaust the ephemeral
range, producing sporadic `Cannot assign requested address`. Widening the range
helps; connection pooling fixes it.

`somaxconn` too low shows up as dropped connections under burst with
`ss -lnt` reporting a full accept queue (`Recv-Q` at the listener equal to
`Send-Q`).

## Fast whole-host snapshot

```bash
uptime; free -h; df -h; df -i; ss -s
dmesg -T | tail -30
journalctl -p err -b --no-pager | tail -30
systemctl list-units --failed
```

Run this before forming a hypothesis. It catches the majority of "the server is
acting weird" reports in under a minute.
