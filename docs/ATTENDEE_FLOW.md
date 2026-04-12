# Attendee Flow

How a workshop attendee gets from "fresh laptop" to "pushing a Lustre
patch from a workshop host."

## Before the workshop (do at home)

You need:

1. **An SSH key on your laptop** (for git, and optionally for logging
   into the workshop hosts).  If `~/.ssh/id_ed25519` already exists,
   skip this:

   ```sh
   ssh-keygen -t ed25519 -C "you@example.com"
   ```

2. **The key loaded into your SSH agent**.  Verify with `ssh-add -l`.
   If empty, run:

   ```sh
   ssh-add ~/.ssh/id_ed25519
   ```

   On macOS, persist it across reboots:

   ```sh
   ssh-add --apple-use-keychain ~/.ssh/id_ed25519
   ```

3. **A Whamcloud Gerrit account** at <https://review.whamcloud.com>.
   Sign in with GitHub or Google, then upload your **public** key under
   *Settings → SSH Keys*.  Paste the contents of `~/.ssh/id_ed25519.pub`
   (the `.pub` file, never the private one).

4. **Pick a Linux username** to use on the workshop boxes — lowercase
   letters/digits/`_-`, must start with a letter, max 31 chars.  You
   create the account yourself on the day, no organizer involvement.

5. **(Optional but recommended) VSCode + the Remote-SSH extension** if
   you want to edit Lustre source on the workshop host from your laptop.

## Day of: bootstrap your account

The workshop has two hosts, `host1` and `host2`, both directly reachable
from the public internet.  You'll create your account on both at once.

Step 1 — ssh in as the shared admin user with the workshop passphrase
(`blew-hurry-throughout-rate`, also on the slide):

```sh
ssh admin@devday1.mulberrytree.us
# password: blew-hurry-throughout-rate
```

Step 2 — run `create-account` with your chosen username, full name, and
email.  It will prompt you to paste your SSH public key (or skip and use
password auth):

```sh
admin@host1$ create-account alice "Alice Example" alice@example.com

Paste your SSH public key — the contents of ~/.ssh/id_ed25519.pub
from your laptop (the file ending in '.pub', not the private key).
Press Enter on a blank line to skip and use password auth instead.
key> ssh-ed25519 AAAAC3Nz...K3l alice@laptop

==> Provisioning alice on host1... done
==> Provisioning alice on host2... done

Account ready: alice
...
```

The script creates your account on **both** hosts (so failover is just
"switch the hostname"), drops your pubkey into `~/.ssh/authorized_keys`,
sets up your gitconfig, and gives you passwordless sudo.

Step 3 — log out and ssh back in as yourself:

```sh
admin@host1$ exit
$ ssh alice@devday1.mulberrytree.us
[alice@host1 ~]$
```

If you pasted a pubkey, this login uses key auth (no password).  If you
skipped, you'll be prompted for the same workshop passphrase.

### Where the SSH key lives

To be very clear: your **private** key never leaves your laptop.  When
`create-account` asks for your "public key," it wants the contents of
the file ending in `.pub` — typically `~/.ssh/id_ed25519.pub`.  If you
accidentally try to paste your private key, the script will refuse and
tell you to paste the `.pub` file instead.

### Forgot the passphrase, or want to rotate your SSH key

Re-run `create-account` with `--force-password`:

```sh
ssh admin@devday1.mulberrytree.us
admin@host1$ create-account --force-password alice "Alice Example" alice@example.com
```

This resets the password back to the workshop passphrase, replaces your
authorized_keys with whatever pubkey you paste, and updates your
gitconfig.  It's the recovery path for any "I locked myself out"
situation.

## VSCode Remote-SSH

Once your account exists, in VSCode:

1. *View → Command Palette → Remote-SSH: Connect to Host...*
2. Type `alice@devday1.mulberrytree.us` (substitute your username)
3. Pick the platform (Linux), authenticate (key or passphrase)
4. *File → Open Folder* — `/home/alice` is your home

No `~/.ssh/config` editing required.  If you want a friendlier alias for
later sessions, you *can* add one to `~/.ssh/config` on your laptop, but
it's not necessary.

## Cloning Lustre and pushing a patch

The Lustre tree is already cloned at `~/lustre-release` on each host.

```sh
cd ~/lustre-release
git pull              # via https, no auth needed
# ... edit, hack, build ...
git commit -s         # signed-off-by, gerrit requires it
git push gerrit HEAD:refs/for/master
```

Because your SSH agent is forwarded (`ssh -A` if you used the CLI;
VSCode does this automatically), the `git push` to gerrit uses the key
on **your laptop**, not anything on the workshop host.  No private key
ever lands on the boxes.

If you logged in *without* `-A`, agent forwarding isn't on and `git
push gerrit` will fail with "Permission denied (publickey)."  Log out,
re-login with `ssh -A alice@devday1.mulberrytree.us`, and verify with
`ssh-add -l`.

## Doing actual workshop work

The workshop hosts run microVMs via `ltvm` for Lustre testing.  Build
your modified Lustre against the test kernel, then deploy to a fresh
microVM:

```sh
sudo ltvm build-lustre rocky9 ~/lustre-release
sudo ltvm create co1-test --os rocky9 --mdt-disks 1 --ost-disks 2
sudo ltvm deploy co1-test --build ~/lustre-release --mount
sudo ltvm exec co1-test 'lctl dl'
```

`ltvm list` shows everyone's VMs on the host along with who created
each one, so it's easy to spot whose VM is hung when the room is
debugging.  See `ltvm --help` and `ltvm <subcommand> --help` for
details, and the [main ltvm
repo](https://github.com/lustre-tools/lustre-test-vms) for the full
story.

## Identity model — why this works the way it does

You have **three identities** in this setup:

| Identity                  | Where it lives             | Used for                          |
|---------------------------|----------------------------|-----------------------------------|
| Workshop passphrase       | `/etc/shadow` on the hosts | ssh login (laptop → host)         |
| `~/.gitconfig` name+email | Workshop host home dir     | git commit attribution to gerrit  |
| SSH private key           | Your laptop only           | git push to gerrit                |

The workshop passphrase is shared and printed on a slide.  It only
protects a short-lived test lab on a small known network — "type one
phrase" beats "find the right key file" every time.  If you paste your
pubkey during account creation, you skip even that for normal logins.

The SSH key never leaves your laptop.  Agent forwarding lets git on the
host borrow your laptop's key for the duration of an ssh session, then
drops the borrowing the moment you log out.  A compromised workshop
host can't impersonate you to gerrit — worst case, an attacker could
hijack a live session, but nothing persists.

## Troubleshooting

- **`ssh-add -l` says "no identities"** — run `ssh-add ~/.ssh/id_ed25519`
  on your laptop, then log back in with `-A`.
- **`git push` to gerrit says "Permission denied (publickey)"** — your
  agent forwarding isn't reaching the host.  Verify `ssh-add -l` on the
  host shows your key.  If it does, check that gerrit knows your public
  key (Settings → SSH Keys at <https://review.whamcloud.com>).
- **Forgot the workshop passphrase** — it's on the workshop slide.  If
  you locked yourself out of your *user account* (somehow rotated the
  password locally), have an organizer run `create-account
  --force-password` for you.
- **`create-account` says "<username> already exists"** — either pick a
  different username, or re-run with `--force-password` if it's actually
  your account and you want to reset it.
- **Pasted the wrong key into `create-account`** — re-run with
  `--force-password`; the new pubkey replaces the old one.
- **devday1 is unreachable / unresponsive** — try `devday2.mulberrytree.us`.
  Your account exists on both hosts, so failover is just changing the
  hostname.  If both are down, ask an organizer.
