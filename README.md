# integ

A minimal file integrity monitoring (FIM) tool for the command line, written in Bash.

`integ` records a SHA-256 hash for each file you ask it to watch, then tells you later whether that file still hashes to the same value. It is the core idea behind tools like AIDE or Tripwire, stripped down to a single readable script.

---

## Why

Unauthorized modification of configuration files, binaries, and log files is one of the earliest signals of a compromise. A file integrity monitor gives you a cheap tripwire: hash what matters now, re-check it later, investigate anything that moved.

`integ` is deliberately small — one script, no dependencies beyond coreutils — so the whole mechanism fits on one screen.

## How it works

Each monitored file gets one line in a plain-text tracker file:

```
<sha256>  <inode>  <path>
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  8618562  /etc/hosts
```

Files are keyed by **inode**, not by path, so a file that is renamed or moved within the same filesystem stays tracked under its original entry.

```
track   ->  hash the file, append a new line if the inode isn't recorded yet
check   ->  re-hash the file, compare against the stored hash, report match or drift
update  ->  re-hash the file and replace the stored line (use after an intended change)
```

## Requirements

- Bash
- `sha256sum` (GNU coreutils)
- `root` privileges — `integ` refuses to run without them, since the point is to watch files an unprivileged user cannot read or modify

> **Platform note:** the script currently mixes GNU and BSD conventions — it calls `sha256sum` (GNU coreutils) but uses BSD `sed -i ''` for in-place edits. On Linux, `update` will fail until `sed -i ''` is changed to `sed -i`. On macOS, install coreutils (`brew install coreutils`) and make `sha256sum` available on `PATH`. See [Known limitations](#known-limitations).

## Setup

1. Clone the repository and make the script executable:

   ```bash
   git clone https://github.com/igor-woz/integ.git
   cd integ
   chmod +x integ
   ```

2. **Create a tracker file and point the script at it.** Open `integ` and set `TRACKER` at the top of the file — it ships as a placeholder and the script will not work until you change it:

   ```bash
   sudo touch /var/lib/integ/tracker
   sudo chmod 600 /var/lib/integ/tracker
   ```

   ```bash
   TRACKER="/var/lib/integ/tracker"
   ```

   Keep the tracker file readable and writable by root only. If an attacker can edit the tracker, they can hide their own changes.

3. (Optional) Put it on your `PATH`:

   ```bash
   sudo install -m 755 integ /usr/local/bin/integ
   ```

## Usage

```
Usage: integ -m [track | check | update] -f <filename>

Options:
  -h    Show this help message and exit
  -m    Mode in which command will operate
  -f    Specify the input file path
```

### Start monitoring a file

```bash
sudo integ -m track -f /etc/ssh/sshd_config
```

```
[result]: file /etc/ssh/sshd_config added to integrity monitoring
```

Tracking a file that is already monitored is a no-op:

```
[result]: file /etc/ssh/sshd_config is already monitored
```

### Verify a file

```bash
sudo integ -m check -f /etc/ssh/sshd_config
```

```
[result]: integrity confirmed! file /etc/ssh/sshd_config was not altered
```

If the file changed:

```
[result]: hashes for file /etc/ssh/sshd_config differ! investigate the change
```

Checking an untracked file is an error:

```
[fatal] file /etc/passwd is not monitored
```

### Accept a legitimate change

After you deliberately edit a monitored file, re-baseline it so future checks are meaningful:

```bash
sudo integ -m update -f /etc/ssh/sshd_config
```

```
[result]: file /etc/ssh/sshd_config hash updated
```

## Recipes

Baseline a set of sensitive files:

```bash
for f in /etc/passwd /etc/shadow /etc/sudoers /etc/ssh/sshd_config; do
  sudo integ -m track -f "$f"
done
```

Re-check everything already in the tracker:

```bash
sudo awk '{print $3}' /var/lib/integ/tracker | while read -r f; do
  sudo integ -m check -f "$f"
done
```

Run a daily check via cron:

```cron
0 3 * * * /usr/local/bin/integ -m check -f /etc/ssh/sshd_config >> /var/log/integ.log 2>&1
```

## Known limitations

This is a learning project, and the constraints are worth stating plainly:

- **`TRACKER` is hardcoded.** It has to be edited in the script rather than passed as a flag or read from a config file or environment variable.
- **Mixed GNU/BSD assumptions.** `sha256sum` is GNU; `sed -i ''` is BSD. One of the two needs changing depending on your platform.
- **One file per invocation.** There is no recursive directory mode and no bulk re-check.
- **Inode reuse.** If a monitored file is deleted and the inode is recycled by a different file, the old entry silently now refers to the new file.
- **No removal mode.** Entries can only be added or updated; untracking requires editing the tracker by hand.
