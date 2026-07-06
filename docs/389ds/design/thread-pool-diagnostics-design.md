---
title: "Offline Thread Pool Diagnostics"
---

# Offline Thread Pool Diagnostics
----------------

{% include toc.md %}

Overview
--------

When the worker thread pool is saturated, the server cannot answer a cn=monitor search because the search itself needs a worker thread. So the usual monitoring is unavailable at the moment the administrator needs to see what the pool is doing. The access log does not help much either, because a hung operation only gets its RESULT line after it completes.

This feature lets the administrator inspect the thread pool without an LDAP connection. The server continuously publishes the pool state into a memory mapped file, and a new dsctl action reads that file directly from disk:

```
dsctl localhost thread-pool status
```

The command works while the pool is saturated. It also works when the server is hung or was killed, in which case it reports that the data is a stale snapshot. For the normal case where the server is healthy, cn=monitor gets a new threadpoolworker attribute with a summary of the same data.

Design
------

### The status file

At startup, ns-slapd creates slapd-INSTANCE.threadpool in the server run directory (for example /run/dirsrv/slapd-localhost.threadpool). The file starts with a 4KB header followed by one 64 byte slot per worker thread. The layout is defined in ldap/servers/slapd/threadpool_stats.h, and the lib389 reader mirrors the same field list.

The header contains a magic value, the format version, the sizes the reader needs to validate the file, the server pid and start time, a heartbeat timestamp, and the pool gauges: current and maximum busy workers, current and maximum work queue size, operations initiated and completed, and current connections.

Each worker slot contains the worker state (idle, busy, exited), the operation type, the connection id, the operation id, and the start time of the current operation.

The file is only meant to be read on the machine it was written on. All values are fixed width host-endian integers. It contains no DNs, filters, IP addresses or any other request content, only operation metadata.

### Writing the data

The pool gauges already exist as internal counters. Once per second, a callback on the event queue thread copies them into the header and updates the heartbeat timestamp. The event queue thread is not a worker thread, so the heartbeat keeps running when all the workers are busy. When the whole process stops making progress, the heartbeat stops updating, and the reader uses that to warn that the server may be stalled.

The worker slots are updated by the workers themselves, with atomic stores, at the same places in connection.c where the busy worker counters are maintained. The start time field also serves as the "operation in flight" flag, because operation id 0 is a valid value (the first operation on a connection).

The magic value is written last during initialization, so the reader never sees a half initialized file. On clean shutdown the file is removed. After a crash or kill the file stays behind, and the next startup preserves it instead of overwriting it: the leftover is renamed to slapd-INSTANCE.threadpool.YYYYMMDD-HHMMSS (the same naming the log rotation uses) and the five newest archives are kept. Only a leftover from an unclean shutdown is preserved, and there is no configuration option for this. If the file cannot be preserved, it is removed as before.

The run directory is writable by the service user, so the file is created carefully. Any stale file at the path is removed first, the file is opened with O_EXCL and O_NOFOLLOW and mode 0640, and the result is verified to be a regular file owned by the server. If any of this fails (for example SELinux denies removing a symlink someone planted at the path), the feature is disabled with a warning in the errors log and the server starts normally without it. The disk space for the file is reserved up front, so a full filesystem cannot crash the server later when a worker writes into an unbacked page.

### Reading the data

dsctl finds the file through nsslapd-rundir in the instance dse.ldif, maps it read only, and validates the magic, version, header size, slot size and worker count before parsing anything. A file that fails validation is refused. The file mode is 0640, so the command must run as root or as a member of the server's group.

The reader also checks whether the data can be trusted. The pid in the header must belong to a running ns-slapd process, otherwise the data is reported as a stale snapshot from a crashed or killed server. If the heartbeat is older than 30 seconds while the process is alive, dsctl warns that the server may be stalled. And nsslapd-threadnumber from dse.ldif is compared with the worker count in the file, to catch a changed thread number that was not restarted into effect. All of these produce warnings, not errors, and the data is still printed.

When the file does not exist, dsctl explains why: the instance is not running (the file is removed on clean shutdown), or the feature is disabled by nsslapd-thread-pool-stats.

A preserved crash file can be read with the --file option, and the same pid and heartbeat warnings apply to it. When crash archives exist, the normal status output points at the newest one:

```
dsctl localhost thread-pool status --file /run/dirsrv/slapd-localhost.threadpool.20260707-153042
```

### cn=monitor

cn=monitor returns one threadpoolworker value per active worker, read from the same mapped region:

```
threadpoolworker: worker=1 state=busy op=search duration_ns=8824373
threadpoolworker: worker=2 state=busy op=search duration_ns=1721985
threadpoolworker: worker=3 state=idle op= duration_ns=0
threadpoolworker: worker=4 state=busy op=add duration_ns=16733782
```

cn=monitor is readable by anonymous users in default deployments, so these values only contain the state, the operation type and the duration. The connection and operation ids are only available through dsctl.

Usage
-----

```
Instance: standalone1
Path: /run/dirsrv/slapd-standalone1.threadpool
PID: 64442
Uptime: 57.000s
Heartbeat age: 0.737s
Workers: 4/4 busy (max 4)
Queue: 7 current (max 12)
Operations: 5299 initiated, 5288 completed
Current connections: 10

IDX STATE    OP                 CONN        OP-ID     DURATION
    1 BUSY     UNBIND             1157            1       14.6ms
    2 BUSY     SEARCH             1161            0        8.9ms
    3 BUSY     SEARCH                5          388       12.8ms
    4 BUSY     SEARCH                6          387       10.6ms
```

Configuration
-------------

The feature is enabled by default. One new cn=config attribute controls it:

```
nsslapd-thread-pool-stats: on\|off (on by default)
```

A restart is required for a change to take effect, because the value is read once when the file is created at startup.

There is no sizing option. The file size follows nsslapd-threadnumber: a 4KB header plus 64 bytes per worker, which is around 6KB for a 32 thread server.

External Impact
---------------

cn=monitor gets the new threadpoolworker attribute. There is no other impact to other components.

Origin
------

https://github.com/389ds/389-ds-base/issues/7633

Author
------

<spichugi@redhat.com>
