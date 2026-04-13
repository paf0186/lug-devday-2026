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

`ltvm --help` and `ltvm <cmd> --help` cover the rest.

## Gerrit push

`git push gerrit HEAD:refs/for/master` from `~/lustre-release`.
Uses the attendee's forwarded SSH agent — no keys on this host.
