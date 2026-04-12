# Workshop Lab — Quick Notes

GCP project: `ltvm-workshop-playground` (us-central1-a)

## VMs

| Name  | Role               | Type           | vCPU | RAM     | Disk   | Public IP             |
|-------|--------------------|----------------|------|---------|--------|-----------------------|
| host1 | live workshop host | n2-standard-64 | 64   | 256 GiB | 512 GB | reserved IP #1 (live) |
| —     | cold spare         | —              | —    | —       | —      | reserved IP #2        |

One mongo host instead of a pair.  The microVM bridge network is
host-local (see the CLAUDE.md note in `lustre-test-vms-v2` about
`fcbr0` / `192.168.100.0/24`), so splitting the workshop across two
hosts would break any exercise that wanted a multi-node Lustre cluster
spanning both machines — not worth the failover story for the room
size we expect.

Nested virt is enabled, and the host lives in `us-central1-a` in the
default VPC subnet.

DNS (manually maintained, both names point at the corresponding reserved IP):
- `devday1.mulberrytree.us` → `34.61.47.200`  (`devday-host1`, live)
- `devday2.mulberrytree.us` → `35.232.23.141` (`devday-host2`, cold spare — no VM attached)

The second IP is pre-DNSed so failover is "spin up a fresh VM, attach
the spare IP" with zero DNS propagation wait.  Recovery story if `host1`
dies mid-workshop:

1. **If you have a recent snapshot** (see `devday-bgd`): create a new
   VM from the snapshot, attach reserved IP #2 — attendees switch from
   `devday1` to `devday2`.  ~5-10 min.
2. **No snapshot**: create a stock VM, run through the hand-configure
   punchlist again.  ~1-2 hours, a mid-workshop disaster.  Take the
   snapshot.

Capacity: ~56 microVMs on one n2-standard-64, roughly the same ceiling
as the old two-host pair (28/host × 2).  Realistic peak: 10 attendees ×
4 VMs = 40.

## Auth

Password-only SSH on the public endpoint, with self-service SSH keys
via `/usr/local/bin/create-account` on first login (sourced from
[bin/create-account](bin/create-account) and baked into the host image).

- **Shared admin** (used once per attendee, to bootstrap their account):
  `ssh admin@devday1.mulberrytree.us` — passphrase `blew-hurry-throughout-rate`
- **Per-attendee** (used for everything else): `ssh <username>@devday1.mulberrytree.us` —
  same passphrase, or key auth if they pasted a pubkey during account
  creation
- **MicroVMs** (from the host): `ssh root@<vm-name>` — managed by `ltvm`

The workshop passphrase (`blew-hurry-throughout-rate`) lives on the host
at `/etc/lab-passphrase` (root, mode 600), baked into the host image.
Rotate by editing the file and re-snapshotting.

`admin` and every attendee account have passwordless sudo (ltvm needs
root for VM lifecycle, bridge config, mounts).

## Cost

Workshop is ephemeral: spin up before, tear down after.  All numbers
approximate — verify on <https://cloud.google.com/products/calculator>.

- **Workshop day** (8h, host running): ~$25 compute (n2-standard-64
  at ~$3.10/hr × 8h) + ~$0.15 disk = **~$25**
- **Pre-workshop idle** (host stopped, disk + 2 reserved IPs attached):
  ~$50/mo disk (512 GB pd-balanced) + ~$7/mo reserved IPs = **~$57/mo**.
  Tear down between workshops to skip this.
- **Reserved IPv4 cost**: ~$0.005/hr per IP ≈ $3.65/mo each, **regardless
  of whether the IP is attached to a running VM**.  GCP changed this in
  Feb 2024 — they used to be free when attached, but now everything
  counts.  2 IPs ≈ **$7.30/mo standing**, or ~$0.75 over a 3-day workshop
  window if you reserve and release around the event.

Roughly cost-neutral with the old 2x n2-standard-32 pair (which was also
~$25 for a workshop day), but simpler to operate and no cross-host
complexity.

## Operations

```bash
# Stop the host when not in use
gcloud compute instances stop host1 --zone=us-central1-a

# Bring it back
gcloud compute instances start host1 --zone=us-central1-a

# Tear down between workshops
gcloud compute instances delete host1 --zone=us-central1-a
# Reserved IPs and DNS stay; next workshop reattaches them.
```

## Provisioning the workshop host

The workflow is **hand-configure once, optionally snapshot for DR**:

1. Spin up `host1` from a stock Rocky 9 GCP image (n2-standard-64),
   attach reserved IP #1
2. Hand-configure it (admin user, `/etc/lab-passphrase`, `create-account`
   in `/usr/local/bin/`, ltvm install, `ltvm fetch rocky9`, agent config
   in `/etc/skel`).  See the bd issues in this repo (`bd ready`) for the
   punch list.
3. **Optional but strongly recommended**: snapshot the disk once the
   host is fully configured.  `gcloud compute images create
   devday-workshop-vN --source-disk=host1`.  This is the disaster
   recovery path — if the host dies mid-workshop, you can spin up a
   replacement from the image in minutes instead of re-doing the
   manual config.

No scripted bootstrap.  The snapshot is for recovery, not for
multi-host deploy (there's only one host).
