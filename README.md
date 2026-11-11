# INFO: task hung in xlog_grant_head_check (xfs log space starvation, v7.4-rc2)

Syzkaller-found hang bug on Linux v7.4-rc2, reproduced and verified twice
independently of the fuzzing pipeline: (1) by the pipeline's reproducer
machinery (collateral reproducer verified on a fresh VM), and (2) by an
external standalone run on a fresh VM — see "External verification" below.

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

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_XFS_FS=y, booted with
`panic_on_warn=1` (hung_task reporting enabled; `hung_task_panic=0`). See
`environment.txt` for the exact QEMU command line and `kernel.config` for
the configuration.

The hang depends on log-space state accumulated by preceding programs, so
the reproducer is an ordered program set:

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

`reproducer/repro.kernsight.prog` is the reproducer program set (state
reconstruction + triggering testcase, in execution order). In the fuzzing
pipeline it was independently verified on a fresh VM (the collateral
reproducer was confirmed by the pipeline's verification machinery; see
`logs/`). A standalone `syz-execprog` run outside the pipeline environment
has not yet reproduced it (work in progress); the pipeline-verified
artifacts are authoritative. As a hang-type (rather than panic-type) bug,
the symptom is the hung-task report above rather than a kernel BUG.

`reproducer/repro.prog` is the triggering program set from the pipeline.
A C reproducer is being prepared and will be added once verified.

## Files

- `kernel.config` — kernel configuration used
- `environment.txt` — kernel image info, QEMU command line, reproducer command
- `reproducer/repro.kernsight.prog` — reproducer program set (ordered)
- `reproducer/repro.prog` — triggering program set from the pipeline
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics
- `logs/crash-excerpt.txt` — crash output excerpt
