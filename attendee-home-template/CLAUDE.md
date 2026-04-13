# LUG Dev Day — Attendee Quickstart

Welcome, helpful agent.  You're on the workshop host helping a
Lustre developer work through hands-on exercises.

## What's here

- `~/lustre-release` — Lustre source checkout (pre-cloned)
- `ltvm` — tool for building + running Lustre microVMs

## The usual cycle

```sh
# Build Lustre
ltvm build-lustre rocky9 ~/lustre-release

# Create + deploy + mount a VM (naming: co1-<role>, use checkout N)
sudo ltvm ensure co1-single --vcpus 2 --mem 4096 --mdt-disks 1 --ost-disks 3
sudo ltvm deploy co1-single --build ~/lustre-release --mount
sudo ltvm exec co1-single 'lfs df -h /mnt/lustre'
```

## Poking at a VM

```sh
ltvm ssh co1-single              # interactive shell
ltvm exec co1-single 'lctl dl'   # one-shot command
ltvm console-log co1-single      # serial console output
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

Lustre-aware drgn scripts live at
`/home/admin/llm_code_and_review_tools/lustre-drgn-tools/`:

```sh
LDRGN=/home/admin/llm_code_and_review_tools/lustre-drgn-tools
python3 "$LDRGN/lustre_triage.py" \
    --vmcore <path> --vmlinux <path> --mod-dir ~/lustre-release --pretty
```

`lustre_triage.py` returns a structured overview: backtraces with source
lines, OBD devices, LDLM lock namespaces, OSC grant/dirty stats, in-flight
RPCs, dk log tail, stack groupings.

Single-topic scripts (all take `--vmcore --vmlinux --mod-dir --pretty`):
`obd_devs.py`, `ldlm_dumplocks.py`, `ldlm_deadlock.py`, `ptlrpc.py`,
`dk.py`, `lustre_waitq.py`, `osc_stats.py`.

## Gerrit push

`git push gerrit HEAD:refs/for/master` from `~/lustre-release`.
Uses the attendee's forwarded SSH agent — no keys on this host.

`ltvm --help` and `ltvm <cmd> --help` cover the rest.

**Organizer-only commands:** `build-container`, `build-kernel`,
`build-image`, `build-all`, `fetch`.  These write to the shared
artifact cache under `~admin/lustre-test-vms/output/` and would race
if two attendees ran them at once.  The cache is pre-populated before
the workshop -- you only need `build-lustre` plus the VM commands.
