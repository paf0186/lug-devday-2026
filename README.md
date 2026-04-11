# LUG Dev Day 2026 — Workshop Ephemera

Cloud lab + setup scripts for the workshop.  Disposable, public for
attendee convenience.

- [WORKSHOP.md](WORKSHOP.md) — VM inventory, costs, ops cheat sheet
- [docs/ATTENDEE_FLOW.md](docs/ATTENDEE_FLOW.md) — what attendees do
- [bin/setup-user](bin/setup-user) — provision a per-attendee account
  on hop + host1 + host2.  Run as `admin` on the hop.

## Quick deploy

```sh
# Pull updates on the hop
cd ~/lug-devday-2026 && git pull

# Provision an attendee
./bin/setup-user alice "Alice Example" alice@example.com
```
