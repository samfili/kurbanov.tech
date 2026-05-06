---
title: "DRBD resync took down Jenkins disk I/O for 8 hours"
date: 2026-05-06
description: "w_await on the primary node hit 103ms during a 1.1 TB Jenkins partition resync. Switching from a fixed resync-rate to the adaptive controller brought the peak down to 5.7ms without stopping Jenkins."
tags: ["drbd", "jenkins", "storage", "disaster-recovery"]
---

During a full resync of the Jenkins DRBD partition (1.1 TB), `w_await` on the primary node shot up to 103ms. Builds started degrading, agents began timing out. The cause: a fixed `resync-rate` that ignored application I/O. The fix took 15 minutes.

## Context

Jenkins runs in active-standby DR: the primary node holds the DRBD partition in `Primary/UpToDate` state, secondary in `Secondary/UpToDate`. Both nodes run on OpenStack Ubuntu 22.04, with the `jenkins` partition occupying 1.1 TB of a 1.3 TB block device.

After planned maintenance, the secondary node fell out of sync. DRBD started a full resync.

## Problem

Right after resync started, `iostat` on the primary showed normal working load:

```text
Device  r/s   w/s    wkB/s   w_await  %util
vdb     945  2270   12969     5.71    34.4
```

Ten minutes later, under active Jenkins load:

```text
Device  r/s   w/s    wkB/s   w_await  %util
vdb     980    54    5613   103.00    76.0
```

At 103ms, `w_await` means the disk queue is backing up. Builds started hanging, some agents went into a JNLP reconnect loop.

The resync config at the time of the incident:

```bash
drbdsetup disk-options jenkins --c-plan-ahead=0 --resync-rate=512000k
```

`c-plan-ahead=0` disables the adaptive controller. DRBD was reading ~945 r/s from disk to send to the secondary — without throttling back even when Jenkins was writing heavily.

## Solution

Switched to adaptive resync on the fly — no Jenkins downtime, no replication interruption:

```bash
drbdsetup disk-options jenkins \
  --c-plan-ahead=20 \
  --c-fill-target=50k \
  --c-min-rate=50000k \
  --c-max-rate=500000k
```

What each parameter does:

`c-plan-ahead=20` — the controller's planning horizon in units of 0.1s, i.e. 2 seconds. The docs recommend ≥ 5x RTT; with 0.23ms ping between nodes, 20 is overkill, but harmless.

`c-fill-target=50k` — how much data to keep in-flight, in sectors (1 sector = 512 bytes, 50k ≈ 25 MB). The controller adjusts read speed to keep the queue from growing.

`c-min-rate=50000k` — the resync floor. Even when Jenkins is writing hard, the controller won't drop below ~50 MB/s.

`c-max-rate=500000k` — the ceiling. When the disk is idle, resync can use up to ~500 MB/s.

## Result

`iostat` two minutes after changing the parameters:

```text
Device  r/s   w/s    wkB/s   w_await  %util
vdb     870  2269   12965     5.71    34.4   # Jenkins writing actively
vdb       0     0       0     0.00     0.0   # Jenkins idle
vdb       1     1       4     0.00     0.0   # Jenkins idle
```

`w_await` peaked at 5.7ms, down from 103ms. Builds stopped hanging.

Resync speed dropped to ~36 MB/s — DRBD steps aside for Jenkins and takes what's left. Estimated time to full sync: 8.5 hours. Jenkins ran without interruption throughout.

## Takeaways

A fixed `resync-rate` with `c-plan-ahead=0` on a production node is a misconfiguration. The adaptive controller is on by default in DRBD 9 (`c-plan-ahead=20`), but an explicit `--c-plan-ahead=0` in `drbdsetup disk-options` turns it off.

If you're tuning parameters via `drbdsetup`, check that `c-plan-ahead` isn't zero. Parameters apply without restarting the resource or breaking replication.

Next time: start with adaptive, then decide if a fixed rate is actually needed.

---

- [DRBD 9 man page: disk-options](https://manpages.debian.org/unstable/drbd-utils/drbd.conf-9.0.5.en.html)
- [LINBIT: Tuning the DRBD Resync Controller](https://kb.linbit.com/drbd/tuning-the-drbd-resync-controller/)
