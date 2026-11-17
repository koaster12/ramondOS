ramondOS README
===============
Version 1.3 - native security OS for reverse engineering, vulnerability
research and threat hunting. Run it in a virtual machine.
(PgUp / PgDn scroll this page. Full documentation: "ramondOS Documentation".)

WHY IT EXISTS
-------------
Analysis tools usually run on general-purpose systems that trust their own
software, whose logs a privileged user can edit, and whose size makes it hard
to say what the system itself is doing. ramondOS is a small OS written for
one job: run hostile code and files on a machine that denies everything by
default and records everything it allows.

DESIGN RULES
------------
 1. Default deny. A program starts with no rights; each is granted.
 2. Small enough to read. One kernel source tree, no hidden services.
 3. Everything leaves a record. The audit log is a SHA-256 hash chain, so
    editing history breaks the chain.
 4. Analysis never trusts the sample. Untrusted code runs in ring 3 cells or
    in an emulator. "NONE OBSERVED" never proves a file is safe.
 5. One narrow network door: a relay on the host (token + allowlist). API
    keys never live inside the OS.

FILESYSTEM
----------
RAM filesystem: 160 nodes, 1 KB blocks, 2048 blocks, 128 KB max per file.
Owners, groups and permission bits; cells run as uid/gid 1000.
  /bin        built-in programs (sh ls cat grep ps windows_exe ...)
  /etc        configuration         /home/user  your working area
  /tmp        scratch               /var/log    logs
  /cases      one folder per investigation
  /evidence   material collected for a case
  /samples    suspect files awaiting analysis (read-only)
  /tools      analysis helpers
Disk: ATA PIO, format RMDFS001, autosave, sector-diff writes.
Creating a new disk format needs:  disk init confirm

CAPABILITIES (rights a cell can be granted)
-------------------------------------------
  FS_READ 1   FS_WRITE 2   NET 4   EXEC 8   CREDS 16   KMEM 32
NET, EXEC, CREDS and KMEM are sensitive: in learn mode they are logged as
attempts and never executed.

WHAT IT CAN DO
--------------
 cells        progs, profiles, run, learn, check, explain, allow, audit
 analysis     info hash hexdump strings entropy pe elf disasm iocs triage
 windows_exe  emulates a 32-bit Windows PE (integer x86 + ~200 fake APIs);
              reports behaviors, IOCs and a score. 32-bit only.
 ghidra       ghidra analyze|status|summary|funcs|decomp|imports|strings
              (Ghidra runs on the host through the relay)
 network      net on|off|ping|test, intel, pull  (relay only)
 processes    preemptive scheduler, fork, execve, waitpid, pipes, signals,
              Ctrl-C / Ctrl-D, ps, mem, kill; watchdog kills runaways
 shell        sh: pipelines, < > >>, &, quotes, cd pwd ps kill help exit
 files        ls cat cp mv rm mkdir touch echo wc head stat chmod id date
 audit        audit verify | head | show N | selftest
 disk         disk, sync   (saved filesystem, autosave)

USE CASES
---------
 - Triage an unknown Windows executable: copy to /samples, run windows_exe.
 - Deep static analysis: ghidra analyze, then funcs, decomp, imports.
 - Watch what a program wants: learn mode shows the capabilities it asks for.
 - Keep a defensible case record in /cases and /evidence, backed by the
   hash-chained audit log (audit verify).
 - Pull threat intel through the relay with no key stored in the OS.
 - Teach how processes, permissions and audit work on a readable system.

HOW IT DIFFERS FROM WINDOWS AND LINUX
-------------------------------------
Windows and Linux run anything the user wants, with security added around
it. ramondOS runs nothing until told to and assumes the code is hostile.
  Starting posture : no rights, each granted   (they: user's rights)
  Size             : one readable source tree  (they: millions of lines)
  Audit trail      : hash-chained, tamper-evident (they: editable logs)
  Network          : one relay door            (they: direct access)
  Secrets          : keys stay on the host     (they: same machine)
  Malware          : caged or emulated by design
It is a specialized instrument used alongside them in a VM, not a
replacement. Its value is being small, strict and inspectable.

LIMITS
------
 - VM only (QEMU, VirtualBox); not hardened for bare metal.
 - windows_exe cannot run 64-bit PE files; the emulator is partial.
 - "NONE OBSERVED" describes one run, not safety.
 - Ghidra runs on the host; Claude is not inside the OS.
 - Small filesystem, one CPU, framebuffer GUI, no USB.
 - No formal verification. The 64-bit port is unfinished.

VERSIONS
--------
 1.3 README on the desktop   1.2 panel layout   1.1 LCARS panel photo
 1.0 multitasking, fork/exec, pipes, signals, shell, POSIX layer
 0.9 hash-chained audit      0.8 disk persistence
 0.4 and earlier: boot, LCARS GUI, ring 3 cells, learn mode

Type  help  for commands,  readme  to show this page again.
