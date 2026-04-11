# Workshop Lab — Quick Notes

GCP project: `ltvm-workshop-playground` (us-central1-a)

## VMs

| Name  | Role            | Type           | vCPU | RAM     | Disk   | Public IP             |
|-------|-----------------|----------------|------|---------|--------|-----------------------|
| host1 | workshop host A | n2-standard-32 | 32   | 128 GiB | 256 GB | reserved IP #1 (live) |
| host2 | workshop host B | n2-standard-32 | 32   | 128 GiB | 256 GB | reserved IP #2 (live) |
| —     | (spare)         | —              | —    | —       | —      | reserved IP #3        |
| —     | (spare)         | —              | —    | —       | —      | reserved IP #4        |

Hosts have nested virt enabled.  Both run Rocky 9 with `ltvm` installed
via [bin/lab-init](bin/lab-init).

DNS (manually maintained, all four point at the corresponding reserved IP):
- `host1.mulberrytree.us` → `34.61.47.200`   (`devday-host1`)
- `host2.mulberrytree.us` → `35.232.23.141`  (`devday-host2`)
- `host3.mulberrytree.us` → `35.224.134.233` (`devday-host3`)
- `host4.mulberrytree.us` → `34.171.126.149` (`devday-host4`)

Four IPs are pre-reserved and pre-DNSed so failover is "attach a spare IP
to a fresh VM" with zero DNS propagation wait.  At workshop time only two
VMs exist.  If host1 dies on the day, you have two options:
1. Detach IP #1 from the dead VM, attach to a fresh VM — attendees keep
   using `host1.mulberrytree.us`, no rename.
2. Spin up a fresh VM with IP #3 attached — tell the room "host1 is dead,
   switch to host3."  Faster, but renames break in-flight ssh sessions.

Capacity: 28 microVMs/host × 2 = 56.  Realistic peak: 10 attendees × 4 VMs = 40.

## Auth

Password-only SSH on the public endpoints, with self-service SSH keys via
`/usr/local/bin/create-account` on first login (sourced from
[bin/create-account](bin/create-account) and baked into the host image).

- **Shared admin** (used once per attendee, to bootstrap their account):
  `ssh admin@host1.mulberrytree.us` — passphrase `blew-hurry-throughout-rate`
- **Per-attendee** (used for everything else): `ssh <username>@host1.mulberrytree.us` —
  same passphrase, or key auth if they pasted a pubkey during account
  creation
- **MicroVMs** (from a host): `ssh root@<vm-name>` — managed by `ltvm`

The workshop passphrase (`blew-hurry-throughout-rate`) lives on each host at
`/etc/lab-passphrase` (root, mode 600), baked into the host image.  Rotate
by editing the file and re-snapshotting.

`admin` and every attendee account have passwordless sudo (ltvm needs root
for VM lifecycle, bridge config, mounts).

host1 ↔ host2 are passwordless via a shared admin SSH key baked into the
host image, so `create-account` can mirror an account onto both hosts in
a single attendee invocation.

## Cost

Workshop is ephemeral: spin up before, tear down after.

- **Workshop day** (8h, both hosts running): ~$25 compute + ~$0.07 disk = **~$25**
- **Pre-workshop idle** (hosts stopped, disks + 4 reserved IPs attached):
  ~$50/mo disks + ~$15/mo reserved IPs = **~$65/mo**.  Tear down between
  workshops to skip this.
- **Reserved IPv4 cost**: ~$0.005/hr per IP ≈ $3.65/mo each, **regardless of
  whether the IP is attached to a running VM**.  GCP changed this in Feb 2024
  — they used to be free when attached, but now everything counts.  4 IPs
  ≈ **$14.60/mo standing**, or ~$1.50 over a 3-day workshop window if you
  reserve and release around the event.

## Operations

```bash
# Stop hosts when not in use
gcloud compute instances stop host1 host2 --zone=us-central1-a

# Bring them back
gcloud compute instances start host1 host2 --zone=us-central1-a

# Resize a host (must be stopped)
gcloud compute instances set-machine-type host1 \
    --zone=us-central1-a --machine-type=n2-standard-32

# Tear everything down (between workshops)
gcloud compute instances delete host1 host2 --zone=us-central1-a
# Reserved IPs and DNS stay; next workshop reattaches them.
```

## Provisioning workshop hosts

The workflow is **hand-configure once, snapshot, deploy from image**:

1. Spin up host1 from a stock Rocky 9 GCP image, attach reserved IP #1
2. Hand-configure it (admin user, `/etc/lab-passphrase`, `create-account`
   in `/usr/local/bin/`, shared admin SSH key, ltvm install, `ltvm fetch
   rocky9`, pre-cloned `~/lustre-release`).  See the bd issues in this
   repo (`bd ready`) for the punch list.
3. `gcloud compute images create devday-workshop-vN --source-disk=host1`
4. `gcloud compute instances create host2 --image=devday-workshop-vN` with
   reserved IP #2 attached
5. Failover hosts (host3/host4) are instantiated from the same image when
   needed, with one of the spare reserved IPs attached

No per-host scripted bootstrap — the image *is* the deploy mechanism.
Per-instance customization (hostname) is handled via `gcloud` flags at
`instances create` time.
