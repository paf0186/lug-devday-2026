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

Two GCP VMs (`host1`, `host2`) with public IPs and nested virt, both
running ltvm.  Four static IPs are pre-reserved and pre-DNSed
(`host[1-4].mulberrytree.us`); the spare two are cold standby for
failover.  No bastion — attendees ssh directly to a host with the shared
workshop passphrase, run `create-account` once to bootstrap their own
user (on both hosts at once), then log out and ssh back in as
themselves.  Optional pubkey paste during account creation skips
password auth thereafter.  Hosts are deployed from a pre-baked image
(hand-configured once, snapshot, instantiate); no per-host scripted
provisioning.

## Attendee quickstart

```sh
# Bootstrap your account (once)
ssh admin@host1.mulberrytree.us         # passphrase: blew-hurry-throughout-rate
admin@host1$ create-account alice "Alice Example" alice@example.com
admin@host1$ exit

# From now on, log in as yourself
ssh alice@host1.mulberrytree.us
```

See [docs/ATTENDEE_FLOW.md](docs/ATTENDEE_FLOW.md) for the full attendee
walkthrough including SSH key setup, VSCode Remote-SSH, and gerrit push.
