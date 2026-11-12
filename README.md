# INFO: task hung in xlog_grant_head_check (xfs log space starvation, v7.4-rc2)

Syzkaller-found hang bug on Linux v7.4-rc2, reproduced and verified on the
same kernel build — see "External verification" below for the standalone
runs.

## Summary

Multiple tasks block for more than 140 seconds waiting for XFS log space:

```
INFO: task kworker/0:5:12527 blocked for more than 142 seconds.
      Not tainted 7.4.0-rc2-kernsight-kasan #1
Workqueue: xfs-inodegc/loop6 xfs_inodegc_worker
Call Trace:
 xlog_grant_head_wait+0x144/0xd00 fs/xfs/xfs_log.c:217
 xlog_grant_head_check+0x387/0x610 fs/xfs/xfs_log.c:273
 xfs_log_reserve+0x303/0x8b0 fs/xfs/xfs_log.c:387
 xfs_trans_alloc+0x489/0xea0 fs/xfs/xfs_trans.c:260
 xfs_inactive_ifree+0xe8/0x4a0 fs/xfs/xfs_inode.c:1223
 xfs_inactive+0x74c/0x1120 fs/xfs/xfs_inode.c:1502
 xfs_inodegc_worker+0x2a7/0x800 fs/xfs/xfs_icache.c:1991
INFO: task syz.1.28:12815 blocked for more than 142 seconds.
 xlog_grant_head_check+0x57f/0x610 fs/xfs/xfs_log.c:279
```

## Mechanism

XFS log space reservation (fs/xfs/xfs_log.c): several tasks (an
xfs-inodegc kworker performing inode inactivation and syzkaller worker
tasks) simultaneously wait for log grant space (`xlog_grant_head_wait`,
fs/xfs/xfs_log.c:217). Log space held/unreleased by prior operations keeps
the grant head from moving forward, so the waiters stay in D state for more
than 140 seconds and the hung task detector reports
`INFO: task hung in xlog_grant_head_check`. The title counts this hang as a
use-after-free-class report since the waiters may reference filesystem
objects whose reclamation is blocked.

## How to reproduce

### Prerequisites

1. Boot a Debian trixie x86_64 VM (6 GiB RAM, 4 vCPU, KVM) using the QEMU
   command line in `environment.txt`, with the kernel built from v7.4-rc2
   (see `kernel.config`; CONFIG_KASAN=y, CONFIG_XFS_FS=y).
2. Copy `syz-execprog` and `syz-executor` (built from syzkaller commit
   `853b104bb454141a5b0fc0e4218ec78e9f34ffb1`) into the guest.
3. Copy `reproducer/repro.kernsight.prog` into the guest.

### Run

In the guest, as root:

```sh
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

**`-repeat=0` is critical**: syz-execprog will execute the 67-program set
in an infinite loop. Each pass consumes XFS log transactions; the bug
requires log grant space to be gradually exhausted by repeated passes.
A single pass will not reproduce the hang.

### What you will see

After **~5-10 minutes** the serial console (or `dmesg`) will show:

```
INFO: task kworker/... blocked for more than 143 seconds.
 xlog_grant_head_wait+0x144/0xd00 fs/xfs/xfs_log.c:217
 xlog_grant_head_check+0x387/0x610 fs/xfs/xfs_log.c:273
INFO: task syz.N.NNN:NNNNN blocked for more than 143 seconds.
 xlog_grant_head_check+0x57f/0x610 fs/xfs/xfs_log.c:279
```

The guest does **not** panic or reboot — it stays alive with tasks in D
state. Monitor the serial console (or `dmesg`) for the hung-task reports
rather than waiting for a reboot.

### Why `-repeat=0` (loop form) is required

Each pass through the 67-program set issues a batch of XFS transactions
that consume log grant space. The bug causes some of that space to be
accounted but never fully released. After enough passes the grant head
stops moving forward and waiters accumulate — this is a cumulative
resource-exhaustion hang, not a single-shot race. A single pass through
the program set does not consume enough log space to trigger it.

## External verification

Reproduced on fresh VMs with the plain command above, **2/2 independent
fresh-VM runs**, plus the pipeline's own collateral verification on a
fresh VM (~4 min):

- Run 1: hung-task reports at **315 s after guest boot (~5 min)** —
  console: `logs/external-verification/external-verification-serial.log`.
- Run 2: three tasks blocked at **621.7 s after guest boot (~10 min)**,
  all in `xlog_grant_head_check` — console:
  `logs/external-verification/external-verification-serial-run2.log`.

Both runs show the same double-hang as the pipeline report (the
xfs-inodegc kworker and syz worker tasks simultaneously waiting in
`xlog_grant_head_wait`). Detection record: `logs/external-verification/HIT`;
run book: `logs/external-verification/runner-run.log`. Note: because the
guest does not panic, an automated monitor that probes the guest over ssh
can stall for hours on a hung guest — check the serial console rather
than relying on guest responsiveness.

`reproducer/repro.prog` is the triggering program set from the pipeline.
A C reproducer has not been included yet; it will be added once
syzkaller's C-repro stage passes for this bug.

## Files

- `kernel.config` — kernel configuration used
- `environment.txt` — kernel image info, QEMU command line, reproducer command
- `reproducer/repro.kernsight.prog` — reproducer program set (ordered)
- `reproducer/repro.prog` — triggering program set from the pipeline
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics
- `logs/crash-excerpt.txt` — crash output excerpt
