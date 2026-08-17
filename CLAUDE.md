# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Raw annotated source code from **Lions' Commentary on UNIX 6th Edition** (1976), derived from http://www.tom-yam.or.jp/2238/src/. This is a **read/study project** — it is not buildable with modern compilers because the C dialect is pre-ANSI (circa K&R 1975), using obsolete compound assignment operators (`=+`, `=-`, `=|`, `=&`) that are illegal in C89+.

## Code Navigation

`cscope` index files are pre-built. Use cscope for symbol lookup:

```sh
cscope -d          # browse existing index (no rebuild)
cscope -d -f cscope.out -l   # line-oriented (scriptable)
```

All source lives flat in the root — no subdirectories.

## Language Notes

- **Pre-ANSI C**: `=+` means `+=`, `=|` means `|=`, `=&` means `&=`. These are NOT typos.
- **No types**: `int` is the default return type; functions declared without return types return `int`.
- **Implicit `int *` arithmetic**: pointer arithmetic on raw `int *` moves by word (2 bytes on PDP-11).
- **Assembly** (`m40.s`, `low.s`): PDP-11 assembly. `mfpi`/`mtpi` are non-standard instructions for moving between address spaces.

## Architecture

### Key Data Structures

| File | Structure | Purpose |
|------|-----------|---------|
| `proc.h` | `proc[NPROC]` | Always-resident per-process table (50 slots) |
| `user.h` | `u` (single global) | Per-process data that swaps with the process; lives at kernel virtual `0140000` |
| `inode.h` | `inode[NINODE]` | In-core inode table (100 slots); on-disk layout in `ino.h` |
| `buf.h` | `buf` + `buffers[NBUF]` | Block buffer cache headers + 512-byte data buffers |
| `systm.h` | globals | `coremap`, `swapmap`, scheduler flags (`runin`, `runout`, `runrun`), clock, mount table |
| `file.h` | `file[NFILE]` | Open-file table (100 slots); referenced from `u.u_ofile[]` |
| `text.h` | `text[NTEXT]` | Shared pure-text (code) segment table |

### Execution Flow

**Boot**: `m40.s:start` → sets up kernel segments → calls `_main`

**Kernel init** (`main.c`): zeroes free memory → detects clock → calls `cinit`/`binit`/`iinit` → `iget` root inode → `newproc()` forks process 1 → parent becomes the **swapper** (`sched()`) forever; child exec's `/etc/init` via inline bootstrap code (`icode[]`).

**System calls**: user `sys` trap → `low.s` vectors → `trap()` (`trap.c`) dispatches via `sysent[]` table (`sysent.c`) → `trap1(callp->call)` calls the handler. Arguments are read from user instruction stream with `fuiword()`. Handlers live in `sys1.c`–`sys4.c` and `fio.c`/`pipe.c`/`sig.c`.

**Scheduling** (`slp.c`):
- `sleep(chan, pri)` — suspends current process on channel; `pri < 0` = uninterruptible
- `wakeup(chan)` — makes all processes sleeping on `chan` runnable
- `swtch()` — saves current kernel context (`savu(u.u_rsav)`), switches to proc 0's stack, finds highest-priority runnable in-core process, restores its context (`retu(p->p_addr)` + `sureg()`)
- `sched()` — process 0 loop: swap in ready-to-run processes, swap out sleeping ones to make room

**Process/memory split**: `proc` holds what's needed while the process is swapped out (`p_addr`, `p_size`, `p_stat`, `p_wchan`). `user` (`u`) holds everything else and is physically co-located with the kernel stack for that process.

**Memory management** (`malloc.c`): `coremap` and `swapmap` are resource maps — arrays of (address, size) pairs managed by `malloc(map, size)` / `mfree(map, size, addr)`. Physical memory is addressed in 64-byte clicks.

**File I/O path**: `rdwri.c` → `alloc.c` (block allocation) / `iget.c` (inode ops) → `bio.c` (buffer cache: `bread`/`bwrite`/`brelse`) → `conf.c` device switch (`bdevsw[]`/`cdevsw[]`) → device driver (e.g. `rk.c` for RK disk).

**TTY** (`tty.c`, `tty.h`): canonical line discipline with erase (`#`) and kill (`@`) processing; `clist` character queues.

### Entry Points for Reading

Per README:
- `m40.s` — boot entry
- `sysent.c` — system call table (map call number → handler)
- `conf.c` — block/char device switch tables
- `low.s` — trap and interrupt vectors
- `sched()` in `slp.c` — swapper main loop
