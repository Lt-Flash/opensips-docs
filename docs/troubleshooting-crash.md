---
title: "OpenSIPS crashes"
description: "How to collect and publish a useful OpenSIPS crash backtrace."
---

## Goal

When OpenSIPS crashes, the most useful thing you can send to the developers is a
`bt full` backtrace from the core dump, together with the OpenSIPS version and
the relevant logs.

## 1. Check the logs

If OpenSIPS logs to syslog, check the usual system log files or the systemd
journal:

```bash
sudo tail -f /var/log/syslog
sudo tail -f /var/log/messages
sudo journalctl -u opensips.service -f
```

In current OpenSIPS versions, logging to standard error and syslog is controlled
with:

```opensips
stderror_enabled = yes
syslog_enabled = no
```

Use this temporary setting if you want the logs printed in the console where
OpenSIPS is running. Older OpenSIPS 3.x configurations may still use
`log_stderror`; in OpenSIPS 4.0 and later, use `stderror_enabled` and
`syslog_enabled`.

## 2. Confirm it was a crash

Look for a child process exiting due to a signal:

```text
child process 6645 exited by a signal 11
```

Signal 11 usually means a segmentation fault. Signals such as signal 6
(`SIGABRT`) or signal 7 (`SIGBUS`) can also produce useful core dumps.

If you only see messages such as:

```text
terminating due to SIGTERM
```

then OpenSIPS was probably stopped by an external action, not by a crash. Check
the service manager, monitoring tools, container runtime, or kernel logs.

## 3. Find the core dump

First, check how the system handles core dumps:

```bash
cat /proc/sys/kernel/core_pattern
```

If the output starts with `|`, the core is handled by a helper.

For `systemd-coredump`, use:

```bash
coredumpctl list opensips
coredumpctl info <PID>
coredumpctl gdb <PID>
```

If the output does not start with `|`, the core is usually written in the
OpenSIPS working directory, set with `-w`:

```bash
gdb /usr/sbin/opensips /path/to/core.<PID>
```

The binary path must match the binary that produced the core. Common paths are
`/usr/sbin/opensips` or the local `opensips` binary from a source build.

## 4. If no core dump is generated

Enable core dumps before reproducing the crash.

For a manual start:

```bash
ulimit -c unlimited
opensips -w /var/lib/opensips
```

For a systemd service:

```bash
sudo systemctl edit opensips.service
```

Add:

```ini
[Service]
LimitCORE=infinity
```

Then restart OpenSIPS:

```bash
sudo systemctl restart opensips.service
```

If you want core files written directly to the OpenSIPS working directory:

```bash
sudo sysctl -w kernel.core_pattern='core.%e.%p.%t'
sudo sysctl -w kernel.core_uses_pid=1
```

If OpenSIPS starts as one user and then changes user or group with `-u` / `-g`,
you may also need:

```bash
sudo sysctl -w fs.suid_dumpable=1
```

Core files may contain SIP credentials, database credentials, private keys, and
message contents. Store them in a restricted directory and do not publish them
publicly.

## 5. Extract the backtrace

Once inside `gdb`, run:

```gdb
set pagination off
bt full
quit
```

If possible, install matching debug symbols before running `gdb`. If debug
symbols are not available, still publish the backtrace; even a partial backtrace
is better than no backtrace.

Keep the core file, the exact OpenSIPS binary, and the loaded module libraries
(`.so` files). Developers may ask for them privately if the backtrace is not
enough.

## 6. Report the crash

Send the report to the
[OpenSIPS GitHub issue tracker](https://github.com/OpenSIPS/opensips/issues) or
to the `devel@lists.opensips.org` mailing list.

Include:

* the output of `opensips -V`
* the operating system and architecture
* the log lines around the crash
* the `bt full` output
* the relevant `opensips.cfg` parts, with secrets removed
* whether the full core file, binary, and modules can be provided privately
