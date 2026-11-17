# ramondOS Documentation

## Why ramondOS was built and how the build went

ramondOS is a from-scratch x86 operating system built so a reverse engineer or threat hunter can run hostile code and hostile files on a machine that denies everything by default and records everything it allows.

The starting problem: analysis tools run on general-purpose systems that trust their own software. Malware samples, vulnerable binaries and unknown executables get opened on an OS whose defaults favor convenience, whose audit trail can be edited by the same user doing the analysis, and whose size makes it hard to say what the system itself is doing. A small OS written for this one job can be read end to end, and every permission in it can be explained.

The build went in stages, each one tested before the next began:

1. **Early versions (through v0.4):** a bootable kernel (GRUB multiboot), a 1280x720 framebuffer GUI in an LCARS style, ring 3 user-mode "cells", and learn mode.
2. **Networking and analysis (v0.5 to v0.7):** a relay-only network path, threat intel pulls, a built-in x86 disassembler, a Windows PE emulator called windows_exe, and Ghidra headless analysis driven through the relay.
3. **Persistence (v0.8):** an ATA disk driver and a saved filesystem, so cases and evidence survive a reboot.
4. **Tamper-evident audit (v0.9):** a SHA-256 hash chain over every security-relevant event.
5. **POSIX layer and multitasking (v1.0):** a Unix-style filesystem, an ELF loader, a small C library, then a preemptive scheduler with fork, exec, pipes, signals and a shell.
6. **Polish (v1.1 to v1.3):** the LCARS side panel image, layout fixes, an in-OS README and a blinking CELLS ARMED indicator.
7. **64-bit port (in progress):** a 64-bit kernel that keeps today's 32-bit programs running unchanged.

Testing used two emulators, QEMU and VirtualBox. v1.0 passed 19 of 19 fork tests and 106 of 106 POSIX tests, plus fault, runaway-process and fork-bomb checks.

## Design principles

Five rules shaped every decision, and a feature that broke one was changed or left out.

- **Default deny.** A program starts with no rights. Each right is a capability bit that must be granted: file read, file write, network, exec, credentials, kernel memory.
- **Small enough to read.** One kernel translation unit, no inherited drivers, no hidden services. What the OS does is what the source says.
- **Everything leaves a record.** Allowed and denied actions go into an audit log where each entry is chained to the previous one by SHA-256, so editing history breaks the chain.
- **Analysis never trusts the sample.** Untrusted code runs in ring 3 cells or inside an emulator. Emulated behavior is reported, never assumed safe.
- **The network is a single narrow door.** The kernel talks only to a relay on the host. The relay holds API keys, checks a token and applies an allowlist, so no key ever lives inside the OS.

A sixth, practical rule: keep Unix conventions where they cost nothing. Programs use familiar calls (open, read, fork, pipe), so ordinary C tools compile for it.

## Architecture overview

ramondOS boots through GRUB into a kernel that runs the GUI, the scheduler and all drivers in ring 0, while every program runs in ring 3 and enters the kernel only by system call. (A diagram is on the project website.)

Programs in ring 3 can only call the kernel, the kernel enforces capabilities on every call, and the only way out of the machine is the relay on the host.

## The filesystem

The filesystem lives in RAM as a Unix-style tree of 160 nodes and is saved to disk on request or by autosave. Files are stored in 1 KB blocks from a pool of 2,048 blocks, and one file can hold at most 128 KB.

It follows Unix rules: every node has an owner, a group and permission bits, and programs running in cells run as user and group 1000. The layout mixes standard Unix directories with four directories made for investigations.

| Directory | Purpose |
| --- | --- |
| /bin | Built-in programs: sh, ls, cat, grep, ps, forktest, windows_exe and more |
| /etc | System configuration |
| /home/user | Working area for the analyst |
| /tmp | Scratch space |
| /cases | One folder per investigation: notes, reports, findings |
| /evidence | Material collected for a case |
| /samples | Suspect files awaiting analysis |
| /tools | Analysis helpers and utilities |

**Persistence.** The disk is an ATA drive read and written in PIO mode, formatted with a signature called RMDFS001. Writing a fresh format requires the explicit command `disk init confirm`, so a stray command cannot wipe a disk. After that, the system saves automatically and writes only the sectors that changed.

**Process view of files.** Each process has its own table of 16 file descriptors pointing into a shared table of 32 open files. Pipes (8, each 4 KB) and the console use the same descriptors, which is what lets shell pipelines work.

## Capabilities: everything the OS can do

ramondOS covers six areas: isolation, analysis, networking, files, processes and audit.

| Area | What it does |
| --- | --- |
| Cells | Runs a program in ring 3 with only the capabilities granted to it; anything else is denied and logged |
| Learn mode | Shows what a program tries to do and asks for approval; sensitive capabilities (network, exec, credentials, kernel memory) are recorded but never actually executed |
| windows_exe | Emulates a 32-bit Windows PE file with an x86 interpreter and about 200 fake Windows API handlers, then reports behavior categories, indicators of compromise and a heuristic score; limited to 30 million instructions and 15 seconds of CPU |
| Disassembler | Built-in x86 disassembly for inspecting code |
| Ghidra via relay | `ghidra analyze`, `status`, `summary`, `funcs`, `decomp`, `imports` and `strings` run Ghidra 12.1.4 headless on the host and return results to the OS |
| Threat intel | `intel` and `pull` fetch indicator data through the relay |
| Networking | `net on`, `off`, `ping`, `test`; the only destination is the relay |
| Multitasking | Preemptive round-robin scheduler, up to 16 tasks, 5-tick slices at 100 Hz |
| Process control | fork, execve, waitpid, pipes, signals, process groups, Ctrl-C and Ctrl-D; kernel commands `ps`, `mem`, `kill` |
| Shell | /bin/sh with pipelines, redirection (`<`, `>`, `>>`), background jobs (`&`), quoting and builtins (cd, pwd, ps, kill, help, exit) |
| Programs | A freestanding C library lets ELF programs run from /bin: cat, echo, ls, cp, mv, rm, mkdir, grep, wc, head, touch, stat, chmod, id, date, uname and others |
| Audit | SHA-256 hash-chained log of 256 entries with `audit verify`, `head`, `show N` and `selftest` |
| Persistence | Saved filesystem with autosave on a data disk |
| GUI | LCARS-style 1280x720 interface with a built-in README (`readme`) |

A watchdog protects the whole system: every task has a CPU tick budget (200 ticks by default, 1,500 for windows_exe, none for the shell), and a task that exceeds it is killed.

## Security model

Every program is untrusted until granted rights, and the kernel checks each request against six capability bits.

| Capability | Value | Allows |
| --- | --- | --- |
| FS_READ | 1 | Reading files |
| FS_WRITE | 2 | Writing files |
| NET | 4 | Network access through the relay |
| EXEC | 8 | Starting other programs (fork, execve) |
| CREDS | 16 | Credential access |
| KMEM | 32 | Kernel memory access |

NET, EXEC, CREDS and KMEM are the sensitive set. In learn mode they are logged as attempts and never executed.

How the rules apply in practice:

- **Grants are narrow.** Only the shell and a few test programs in /bin receive EXEC. A program placed anywhere else gets none, so it cannot spawn children.
- **Resources are bounded.** A fork bomb or a spinning loop hits the task limit or the CPU watchdog, is killed, and leaks no memory.
- **The audit log resists tampering.** Each entry's SHA-256 hash covers the one before it. `audit verify` recomputes the chain, and a changed or deleted entry makes it fail.
- **Secrets stay outside.** API keys live on the host relay, which listens only on 127.0.0.1, requires a token and enforces an allowlist.

One caution applies to the emulator: "NONE OBSERVED" in a windows_exe report means nothing suspicious was seen in that run. It does not prove the file is safe. Run ramondOS inside a virtual machine, as designed.

## Use cases

ramondOS fits work where the analyst needs to know exactly what a piece of code did and to prove the record was not altered.

1. **Triage an unknown Windows executable.** Copy the file to /samples and run it with windows_exe. The report lists behavior categories, indicators of compromise and a heuristic score, with nothing touching a real Windows system.
2. **Deep static analysis with Ghidra.** From inside the OS, `ghidra analyze` sends the sample through the relay to Ghidra on the host, then `funcs`, `decomp`, `imports` and `strings` bring the results back. This was run on a real binary, whoami.exe.
3. **Watch what a program wants.** Run an unfamiliar program in a cell in learn mode and read which capabilities it asks for. A tool that requests network and credentials when it should only read a file is a finding.
4. **Keep a defensible case record.** Notes and findings go in /cases, material in /evidence, and the hash-chained audit log shows what was done and when. `audit verify` supports the integrity of the record.
5. **Pull threat intelligence safely.** `intel` and `pull` fetch indicator data through the relay, so no API key is ever stored in the analysis environment.
6. **Teach and test.** The small readable source and the Unix-style layer make it a clear platform for teaching how processes, permissions and audit work, and for testing detection ideas.

The larger goal is a sandbox where reverse-engineering tools and an AI assistant work together: ramondOS contains and records the analysis, and the assistant directs it through the relay. That orchestration is a design so far, not a finished feature.

## How ramondOS differs from Windows and Linux, and why it matters

Windows and Linux are built to run anything the user wants, and security is added around that. ramondOS is built the other way: it runs nothing until told to, and it assumes the thing being run is hostile.

| Question | Windows and Linux | ramondOS |
| --- | --- | --- |
| Starting posture | Programs run with the user's rights; restrictions are opt-in | Programs get no rights; each is granted explicitly |
| Size and readability | Millions of lines; no one person can read it all | One kernel source tree a single person can read |
| Audit trail | Logs exist but a privileged user can edit or delete them | Hash-chained log; edits break the chain |
| Network | Direct access to any host | One door: a relay with token and allowlist |
| Secrets | API keys sit on the same machine as the work | Keys stay on the host relay |
| Running malware | Real system at risk; needs a separate sandbox product | Emulated or caged by design |
| Hardware and software support | Huge | Deliberately tiny |

Why this matters:

- **Fewer places to hide.** Malware often relies on the complexity of a general OS: services, drivers, hooks. A small system has few of those to abuse, and each is visible.
- **The tool cannot be quietly subverted.** When the analysis environment itself keeps tamper-evident records, a result can be defended, which counts in forensics.
- **Containment is the default, not an add-on.** Sandboxes layered on a general OS are only as strong as that OS. Here the isolation is the OS.
- **It is explainable.** Every denied action has a named capability behind it, so the analyst can say why something was allowed or blocked.

It does not replace Windows or Linux. It is a specialized instrument, used alongside them in a virtual machine, and its value comes from being small, strict and inspectable.

## Limits, stated plainly

ramondOS is a working research system, not a finished product, and these limits are real.

- **Run it in a VM.** It is tested on QEMU and VirtualBox only. It has not been hardened for bare metal.
- **32-bit samples only.** windows_exe cannot emulate 64-bit PE files; a 64-bit binary is refused.
- **The emulator is partial.** It interprets integer x86 code and fakes about 200 Windows APIs. Real malware can use features it does not model.
- **Absence of evidence is not safety.** "NONE OBSERVED" describes one run only.
- **Ghidra runs on the host, not in a cell.** The OS sends work through the relay and receives results; no AI assistant runs inside ramondOS.
- **Small filesystem.** 160 nodes, 2,048 KB of block space and a 128 KB file limit.
- **Limited hardware.** One CPU, ATA PIO disk access, a framebuffer GUI, and no USB or modern network drivers.
- **No formal verification.** The kernel is tested, not proven correct.
- **64-bit is unfinished.** The 64-bit kernel is a work in progress. A 64-bit userland, C library and ELF64 loader are not started.

## Version history and how to build and run

The current release is v1.3 (32-bit); a 64-bit v2.0 is in progress.

| Version | What it added |
| --- | --- |
| v1.3 | In-OS README (button and `readme` command), blinking CELLS ARMED indicator |
| v1.2 | Panel layout evened out |
| v1.1 | LCARS side panel image |
| v1.0 | Preemptive multitasking, fork/exec, pipes, signals, shell, POSIX-style filesystem and ELF programs |
| v0.9 | SHA-256 hash-chained audit log |
| v0.8 | Disk persistence with autosave |
| v0.4 and earlier | Boot, LCARS GUI, ring 3 cells, learn mode |

**Build.** The source is a single kernel translation unit (kernel.c) with headers for fonts, networking, the disassembler, the reverse-engineering tools, the POSIX layer, the scheduler and persistence. The `build.sh` script compiles the kernel, builds the user programs (sh, forktest, ps, sleep, spin, grep, cat, echo, wc, head, ls and others, plus windows_exe), and produces a bootable ISO.

**Run.** Boot the ISO in QEMU or VirtualBox. For saved data, attach a second disk and run `disk init confirm` once. To use networking and Ghidra, start the relay on the host (bound to 127.0.0.1, with its token and allowlist), then use `net on`.

**Test.** Inside the OS, run `forktest` and `posixtest`. A QEMU driver script scripts keyboard input and takes screenshots for regression checks.
