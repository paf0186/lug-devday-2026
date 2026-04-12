# LUG Dev Day 2026 — Workshop Ephemera

Cloud lab + setup scripts for the workshop.  Disposable, public for
attendee convenience.

- [WORKSHOP.md](WORKSHOP.md) — VM inventory, costs, ops cheat sheet
- [docs/ATTENDEE_FLOW.md](docs/ATTENDEE_FLOW.md) — what attendees do on the day
- [bin/create-account](bin/create-account) — self-service per-attendee account
  creation, run by attendees on first login (deployed to `/usr/local/bin/`
  on the host image)

Use `bd ready` to see the workshop-prep punch list.

## Architecture in one paragraph

One GCP `n2-standard-64` (`host1`, 64 vCPU / 256 GiB) with a public IP
and nested virt, running ltvm.  Two static IPs are pre-reserved and
pre-DNSed (`devday1.mulberrytree.us` live, `devday2.mulberrytree.us`
cold spare for disaster recovery).  No bastion, no sibling host —
attendees ssh directly with the shared workshop passphrase, run
`create-account` once to bootstrap their own user, then log out and
ssh back in as themselves.  Optional pubkey paste during account
creation skips password auth thereafter.  Hand-configured once and
optionally snapshot for DR; no scripted bootstrap.

## Attendee quickstart

```sh
# Bootstrap your account (once)
ssh admin@devday1.mulberrytree.us         # passphrase: blew-hurry-throughout-rate
admin@host1$ create-account alice "Alice Example" alice@example.com
admin@host1$ exit

# From now on, log in as yourself
ssh alice@devday1.mulberrytree.us
```

See [docs/ATTENDEE_FLOW.md](docs/ATTENDEE_FLOW.md) for the full attendee
walkthrough including SSH key setup, VSCode Remote-SSH, and gerrit push.
