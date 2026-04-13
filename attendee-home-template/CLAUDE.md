# LUG Dev Day — Attendee Quickstart

Welcome, helpful agent.  You're on the workshop host helping a
Lustre developer work through hands-on exercises.

## What's here

- `~/lustre-release` — Lustre source checkout (pre-cloned)
- `ltvm` — tool for building + running Lustre microVMs
- `/shared/<username>/` — per-user drop zone; anyone on the host
  can read and write here.  Use it to pass vmcores, patches, logs,
  etc. between attendees without faffing with scp.

## The usual cycle

```sh
# Build Lustre
ltvm build-lustre rocky9 ~/lustre-release

# Create + deploy + mount a VM (naming: co1-<role>, use checkout N)
sudo ltvm ensure co1-single --vcpus 2 --mem 4096 --mdt-disks 1 --ost-disks 3
sudo ltvm deploy co1-single --build ~/lustre-release --mount
ssh root@co1-single 'lfs df -h /mnt/lustre'
```

## Poking at a VM

```sh
ssh root@co1-single              # interactive shell (VM name resolves via dnsmasq)
ssh root@co1-single 'lctl dl'    # one-shot command
ltvm console-log co1-single      # serial console output (through QEMU, not ssh)
```

VMs share a host-local bridge (`192.168.100.0/24`) and can `ssh`
each other by name — handy for multi-node cluster exercises.

## Panic + crashdump

```sh
sudo ltvm nmi co1-single                            # inject NMI → panic + kdump
sudo ltvm crash-collect co1-single --mod-dir ~/lustre-release
```

`crash-collect` waits for kdump to finish, pulls the vmcore back to
the host, and runs the Lustre crash recipe.  `--trigger` combines the
two steps.

## drgn crash analysis

[drgn](https://drgn.readthedocs.io/) is a programmable debugger for
Linux kernels and vmcores -- Python-scriptable access to kernel
structs without the quirks of the traditional `crash` REPL.  Installed
globally on this host (`python3 -c "import drgn"` works).

Lustre-aware scripts live at
`/home/admin/llm_code_and_review_tools/lustre-drgn-tools/` and run as
`python3 <script>.py --vmcore <path> --vmlinux <path> --mod-dir ~/lustre-release --pretty`.
All emit JSON by default; add `--text` for a human-readable dump.

Start with `lustre_triage.py`: one-shot run of everything below, a
single JSON report covering panic backtrace, OBD devices, LDLM locks,
in-flight RPCs, OSC state, D-state tasks, and the dk log tail -- what
you want on first contact with an unfamiliar crash.

### End-to-end: crash → triage

```sh
# 1. Build + deploy Lustre to a running VM
ltvm build-lustre rocky9 ~/lustre-release
sudo ltvm deploy co1-single --build ~/lustre-release --mount

# 2. Panic the VM (or reproduce the bug you're chasing)
sudo ltvm nmi co1-single

# 3. Wait for kdump + reboot, pull vmcore, run triage.
#    --mod-dir points at your Lustre tree so triage can resolve
#    Lustre symbols (mdc_*, osc_*, ldlm_*); without it you only
#    get kernel symbols.  ltvm auto-resolves vmlinux from the
#    target's build output.
sudo ltvm crash-collect co1-single --mod-dir ~/lustre-release
# → /tmp/crashes/crash-co1-single-<ts>/vmcore  +  triage JSON on stdout
```

### Running individual drgn scripts

After `crash-collect`, you have a vmcore on the host.  Point any
single-topic script at it, along with the matching `vmlinux` from
ltvm's build output and your Lustre tree for the module symbols:

```sh
VMCORE=/tmp/crashes/crash-co1-single-<ts>/vmcore
VMLINUX=/home/admin/lustre-test-vms/output/rocky9/kernels/*/vmlinux
LDRGN=/home/admin/llm_code_and_review_tools/lustre-drgn-tools

python3 $LDRGN/ldlm_dumplocks.py \
    --vmcore "$VMCORE" --vmlinux $VMLINUX \
    --mod-dir ~/lustre-release --pretty
```

Single-topic scripts, use when triage points you at a specific
subsystem:

| Script                  | Answers                                                |
|-------------------------|--------------------------------------------------------|
| `obd_devs.py`           | What OBD devices (MDC/OSC/MDT/OST/LOV/LMV) exist and what state they're in. |
| `ldlm_dumplocks.py`     | Every lock in every LDLM namespace: holder, mode, resource, flags.          |
| `ldlm_deadlock.py`      | Cross-namespace cycle detection -- "who's waiting on whom."                 |
| `ptlrpc.py`             | In-flight RPCs per import/service: xid, opcode, phase, age.                 |
| `osc_stats.py`          | Per-OST grant, dirty pages, lost grant -- grant accounting bugs.            |
| `lustre_waitq.py`       | Uninterruptible tasks blocked in Lustre, grouped by wait point.             |
| `dk.py`                 | The Lustre debug (`lctl dk`) ring buffer from the crashed kernel.           |
| `dump_lustre_hashes.py` | `cfs_hash` + `rhashtable` occupancy across every Lustre subsystem.          |

## Gerrit push

`git push gerrit HEAD:refs/for/master` from `~/lustre-release`.
Uses the attendee's forwarded SSH agent — no keys on this host.

## Lustre commit style

Before pushing, sanity-check the patch:

```sh
git diff HEAD | ./contrib/scripts/checkpatch.pl
```

Commit message format:

```
LU-nnnnn component: short description under 64 columns

Body text wrapped to 60 columns.  Explain *why*, not *what* --
the diff already tells you what changed.

Test-Parameters: testlist=sanity
Signed-off-by: Your Name <you@example.com>
```

Rules:
- Subject line: `LU-nnnnn <component>:` prefix, ≤64 columns.  Use
  `LU-0000` if no ticket applies yet.
- ASCII only in the message -- no em dashes or curly quotes (use
  `--`).  Gerrit's tooling chokes on smart punctuation.
- On `git commit --amend`: preserve the existing `Change-Id:` trailer.
  Never invent one, and never add one manually to a new commit --
  the `commit-msg` hook generates it.
- Trailer block (`Signed-off-by`, `Test-Parameters`, `Change-Id`, ...)
  must be contiguous -- no blank lines between trailers.
- `Test-Parameters:` is only for test-only patches; don't add it if
  you touched kernel or userspace code.  Don't invent test names.

`ltvm --help` and `ltvm <cmd> --help` cover the rest.

**Organizer-only commands:** `build-container`, `build-kernel`,
`build-image`, `build-all`, `fetch`.  These write to the shared
artifact cache under `~admin/lustre-test-vms/output/` and would race
if two attendees ran them at once.  The cache is pre-populated before
the workshop -- you only need `build-lustre` plus the VM commands.
