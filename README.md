# restash

A [restic](https://restic.net/) wrapper for backing up selected home directory
paths to online storage. Supports SFTP and Backblaze B2 repositories. Designed
for use with systemd user timers.

## Setup

**1. Install the script**

```sh
cp restash restash-notify ~/.local/bin/
chmod +x ~/.local/bin/restash ~/.local/bin/restash-notify
```

**2. Configure**

```sh
mkdir -p "${XDG_CONFIG_HOME:-$HOME/.config}/restash"
cp restash.conf.example "${XDG_CONFIG_HOME:-$HOME/.config}/restash/restash.conf"
cp excludes.example "${XDG_CONFIG_HOME:-$HOME/.config}/restash/excludes"
```

Edit both files to match your repositories and home directory layout.

**3. Initialize repositories**

Each repository must be initialized before the first backup:

```sh
restash sftp run init
restash b2 run init
```

**4. Install systemd units**

```sh
cp restash-*.service restash-*.timer restash.target \
    "${XDG_CONFIG_HOME:-$HOME/.config}/systemd/user/"
systemctl --user daemon-reload
systemctl --user enable \
    restash-backup@sftp.timer \
    restash-backup@b2.timer \
    restash-check@sftp.timer \
    restash-check@b2.timer \
    restash-maintenance@sftp.timer \
    restash-maintenance@b2.timer
```

Enabling the timers creates the `restash.target.wants/` symlinks so they start
whenever the target is started. How the target itself is managed is up to you:

```sh
# Start/stop manually
systemctl --user start restash.target
systemctl --user stop restash.target

# Or enable it unconditionally on login
systemctl --user enable --now restash.target
```

Or use [nmtrust](https://github.com/pigmonkey/nmtrust) to start and stop the
target automatically on trusted networks.

The timer schedules are defined in the `restash-*.timer` files and can be
adjusted to suit your preferences before copying them into place.

## Usage

```
restash {sftp|b2} {backup|maintain|check|run <restic args>}
```

The `run` subcommand passes arguments directly to restic with the repository
and credentials already configured, making ad-hoc operations straightforward:

```sh
restash b2 run snapshots
restash sftp run init
restash b2 run restore latest --target /tmp/restore
```

The systemd units run backup, maintenance (forget/prune), and rotating
repository checks automatically. Two behaviors worth noting:

- **Transient network failures** exit with code 75 (EX_TEMPFAIL) so systemd
  retries the job rather than triggering an `OnFailure` alert.
- **Lock contention** (exit 11) is handled the same way. If two timers fire
  simultaneously, the one that loses the lock retries after a delay rather than
  failing.
