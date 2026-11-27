# Processes, systemd, and logs

## Unit state

```bash
systemctl status <unit>
systemctl cat <unit>                    # the effective unit file plus drop-ins
systemctl show <unit> -p <Property>     # one resolved property
systemctl list-units --failed
systemctl list-dependencies <unit>
```

`systemctl cat` is the one to trust — it shows the unit **and every drop-in
override** in application order. Reading `/lib/systemd/system/foo.service`
directly misses overrides in `/etc/systemd/system/foo.service.d/`, which is where
local changes live.

Always edit with `systemctl edit <unit>` (creates a drop-in) rather than editing
the vendor file, which a package upgrade will overwrite.

```bash
systemctl edit <unit>          # drop-in override
systemctl edit --full <unit>   # full copy into /etc
systemctl daemon-reload        # after any manual file change
```

Forgetting `daemon-reload` after hand-editing a unit is the reason "my change did
nothing".

## Why a unit failed

`Active: failed (Result: ...)` — the `Result` names the category:

| Result | Meaning |
|---|---|
| `exit-code` | The process exited nonzero. `status=N` is its exit code. |
| `signal` | Killed by a signal. `status=SIGKILL` often means OOM or a stop timeout. |
| `timeout` | Did not start (or stop) within the timeout. |
| `start-limit-hit` | Restarted too fast, too often. **Not the root cause.** |
| `protocol` | `Type=notify` service never signalled readiness. |
| `core-dump` | Crashed. `coredumpctl list`. |

`start request repeated too quickly` describes the rate limiter, not the fault.
Look further back in the journal for the first failure:

```bash
journalctl -u <unit> --since "1 hour ago" --no-pager | head -50
systemctl reset-failed <unit>       # clear the counter to retry now
```

## Service types

`Type=` determines when systemd considers the unit started, and mismatches cause
"timeout" failures on services that are actually running:

- `simple` — ready as soon as it forks. Default; no readiness signal.
- `exec` — like simple, but waits for a successful `execve`.
- `forking` — the parent exits; systemd tracks the child. Needs `PIDFile=` to be
  reliable. A service that does **not** daemonize declared as `forking` hangs
  until timeout.
- `notify` — the service calls `sd_notify(READY=1)`. If it does not, systemd
  waits and then fails with `protocol` or `timeout`.
- `oneshot` — runs to completion; pair with `RemainAfterExit=yes` for setup units.

## Restart policy

```ini
[Service]
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=300
StartLimitBurst=5
```

`Restart=always` restarts even on clean exit, which hides bugs and makes a
crash-looping service look merely busy. `on-failure` is the better default.

The start limit is evaluated over `StartLimitIntervalSec`. Setting
`StartLimitIntervalSec=0` disables the limiter entirely — occasionally right for
a supervisor-managed daemon, usually a way to hide a crash loop.

## Resource limits and sandboxing

```ini
[Service]
LimitNOFILE=65535
MemoryMax=2G
CPUQuota=200%
PrivateTmp=yes
ProtectSystem=strict
ReadWritePaths=/var/lib/myapp
NoNewPrivileges=yes
```

**Unit limits override shell limits.** `/etc/security/limits.conf` applies to
login sessions via PAM, not to systemd services. If a daemon reports
`Too many open files`, fix `LimitNOFILE=` in the unit.

`ProtectSystem=strict` makes everything read-only except `/dev`, `/proc`, `/sys`
and whatever `ReadWritePaths=` allows. A service that suddenly cannot write after
a hardening change needs the path added, not the protection removed.

```bash
systemd-analyze security <unit>        # scored report of exposure
```

## journald

```bash
journalctl -u <unit> -n 200 --no-pager
journalctl -u <unit> -f
journalctl -p err -b                   # priority err and above, this boot
journalctl --since "2026-08-21 09:00" --until "2026-08-21 10:00"
journalctl -k                          # kernel only
journalctl -b -1                       # previous boot
journalctl --disk-usage
journalctl --vacuum-time=7d
```

Priorities are numeric: `0 emerg` … `3 err`, `4 warning`, `6 info`, `7 debug`.
`-p err` includes everything more severe.

**By default the journal may be volatile.** If `/var/log/journal` does not exist,
logs live in `/run` and are lost on reboot — which is why `journalctl -b -1`
sometimes returns nothing. Create the directory and `systemctl restart
systemd-journald` to make it persistent.

```bash
journalctl -u <unit> -o json-pretty | head -40    # all structured fields
journalctl _PID=1234
journalctl _SYSTEMD_UNIT=foo.service _COMM=bar
```

## Processes

```bash
ps -eo pid,ppid,stat,etime,rss,comm --sort=-rss | head -20
pstree -ps <pid>
cat /proc/<pid>/status                 # state, threads, memory, signals
cat /proc/<pid>/environ | tr '\0' '\n' # actual environment
ls -l /proc/<pid>/cwd /proc/<pid>/exe  # real cwd and binary, even if deleted
strace -p <pid> -f -e trace=network    # what syscalls it is making
```

`STAT` codes worth knowing: `D` uninterruptible sleep (blocked on I/O — cannot be
killed, and drives load average), `Z` zombie (parent has not reaped it), `T`
stopped, `S` interruptible sleep (normal idle).

A pile of zombies is a parent-process bug, not a resource problem; they consume
only a PID slot. Killing zombies does nothing — restart or fix the parent.

`/proc/<pid>/exe` still resolves after a binary is replaced on disk, showing
`(deleted)`. That is how you catch a process running an old version after an
upgrade that never restarted the service.
