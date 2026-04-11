# Attendee Flow

How a workshop attendee gets from "fresh laptop" to "pushing a Lustre
patch from a workshop host."

## Before the workshop (do at home)

You need:

1. **An SSH key on your laptop** (for git, not for logging into the
   workshop).  If `~/.ssh/id_ed25519` already exists, skip this:

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

4. **Pick a Linux username** to use on the workshop boxes.  Lowercase
   letters/digits/`_-`, must start with a letter, max 31 chars.  Tell
   the workshop organizer your username, full name, and email at the
   start of the day so they can create your account.

## Day of: first login

Once your account has been provisioned, log in from your laptop with
agent forwarding (`-A`):

```sh
ssh -A <username>@devday.mulberrytree.us
# password: welcome2026
```

You're now on the **hop** node.  From here you can reach the two
workshop hosts:

```sh
ssh -A host1   # or host2
# password: welcome2026
```

Or skip the manual two-step with ProxyJump:

```sh
ssh -A -J <username>@devday.mulberrytree.us <username>@host1
```

## Verify agent forwarding works

On the workshop host, after `ssh -A`'ing in:

```sh
ssh-add -l
```

You should see your key fingerprint.  If you see *"Could not open a
connection to your authentication agent"*, the `-A` flag was missing
on one of the hops -- log out and try again.

## Cloning Lustre and pushing a patch

The Lustre tree is already cloned at `~/lustre-release` on each host.

```sh
cd ~/lustre-release
git pull              # via https, no auth needed
# ... edit, hack, build ...
git commit -s         # signed-off-by, gerrit requires it
git push gerrit HEAD:refs/for/master
```

Because your SSH agent is forwarded, the `git push` to gerrit uses the
key on **your laptop**, not anything on the workshop host.  No private
key ever lands on the boxes.

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

See the `ltvm` help (`ltvm --help`, `ltvm <subcommand> --help`) and
the [main ltvm repo](https://github.com/lustre-tools/lustre-test-vms)
for details.

## Identity model -- why this works the way it does

You have **three identities** in this setup:

| Identity                 | Where it lives           | Used for                          |
|--------------------------|--------------------------|-----------------------------------|
| Workshop password        | `/etc/shadow` on the boxes| ssh login (laptop -> hop -> host) |
| `~/.gitconfig` name+email| Workshop host home dir   | git commit attribution            |
| SSH private key          | Your laptop only         | git push to gerrit                |

The workshop password is shared and written on a slide.  It only
protects an internal-only network with no real attack surface, and
"type one word" beats "find the right key file" every time.

The SSH key never leaves your laptop.  Agent forwarding lets git on
the host borrow your laptop's key for the duration of an ssh session,
then drops the borrowing the moment you log out.  This means a
compromised workshop host can't impersonate you to gerrit -- worst
case, an attacker could hijack a live session, but nothing persists.

## Troubleshooting

- **`ssh-add -l` says "no identities"** -- run `ssh-add ~/.ssh/id_ed25519`
  on your laptop, then log back in with `-A`.
- **`git push` to gerrit says "Permission denied (publickey)"** -- your
  agent forwarding isn't reaching the host.  Verify `ssh-add -l` on the
  host shows your key.  If it does, check that gerrit knows your public
  key (Settings -> SSH Keys at <https://review.whamcloud.com>).
- **Forgot your workshop password** -- it's `welcome2026`, same on every
  box, same for every attendee.  If `setup-user` was re-run for you,
  the password is reset to this value automatically.
- **Need a fresh start** -- ask the organizer to re-run `setup-user` for
  your username.  It's idempotent and resets your password + git config.
