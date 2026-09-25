# Filesystem & Permissions — Cheat Sheet

*Companion to the [Filesystem Hierarchy & Permissions guide](../02-filesystem-hierarchy-permissions.md) and [hands-on lab](../labs/02-filesystem-permissions-lab.md).*

## Permission bits

| Symbol | Bit | Meaning on a **file** | Meaning on a **directory** |
|---|---|---|---|
| `r` | 4 | Read contents | List entries (`ls`) |
| `w` | 2 | Modify contents | Create/delete/rename entries |
| `x` | 1 | Execute as program | **Traverse** into it (`cd`, path resolution) |
| `-` | 0 | No permission | No permission |

`-rw-r--r--` → `u=rw-` `g=r--` `o=r--`. Directory `x` is the one people forget: without it on **every** parent dir, the file is unreachable no matter its own mode.

## Octal quick chart (each digit = one triple)

| Octal | Binary | String | Typical use |
|---|---|---|---|
| 0 | 000 | `---` | No access |
| 1 | 001 | `--x` | Traverse-only (dirs) |
| 2 | 010 | `-w-` | Write-only (rare) |
| 3 | 011 | `-wx` | Write + traverse |
| 4 | 100 | `r--` | Read-only |
| 5 | 101 | `r-x` | Read + execute — **dirs, scripts, binaries** |
| 6 | 110 | `rw-` | Read + write — **regular files** |
| 7 | 111 | `rwx` | Full control |

Common modes: `755` dirs & executables · `644` config/data files · `600` secrets & SSH keys · `700` `~/.ssh` · `777` = never, find another way.

## Special bits (4th octal digit)

| Bit | Octal prefix | Symbol | What it does |
|---|---|---|---|
| SUID | `4` | `s` in owner's `x` (`-rwsr-xr-x`, `4755`) | Runs with the **file owner's** privileges — e.g. `/usr/bin/passwd` runs as root |
| SGID | `2` | `s` in group's `x` (`drwxrwsr-x`, `2775`) | On dirs: new files inherit the **directory's group** — shared project dirs |
| Sticky | `1` | `t` in other's `x` (`drwxrwxrwt`, `1777`) | In a shared dir, only the file's **owner** can delete/rename it — that's `/tmp` |

Uppercase `S`/`T` = special bit set but execute bit missing (usually a misconfiguration).

## Ownership

```bash
chown user:group file        # change owner and group
chown -R deploy:deploy /opt/app
chgrp adm file              # group only
stat -c '%a %A %U:%G %n' file   # compact truth: octal, string, owner:group
```

## umask math

New files start at `666`, new dirs at `777`. umask **subtracts**: `mode = base − umask`.

| umask | New file | New dir | Vibe |
|---|---|---|---|
| `0022` (default) | `644` | `755` | Normal |
| `0002` | `664` | `775` | Group-collaborative |
| `0077` | `600` | `700` | Paranoid |

Files never get `x` from creation — that's deliberate. `umask` in shell profile / systemd `UMask=` / CI env controls what daemons create.

## ACLs (beyond user:group:other)

```bash
getfacl file                # view — look for '+' in ls -l output: -rw-r-----+
setfacl -m u:alice:rw file  # grant a specific user
setfacl -m g:devs:rx dir
setfacl -d -m u:alice:r dir # DEFAULT acl — inherited by new files
setfacl -x u:alice file     # remove one entry
setfacl -b file             # strip all ACLs
```

## File attributes (below the permission model — even root obeys)

```bash
lsattr file
chattr +i file              # immutable — no write/delete/rename, even as root
chattr -i file              # unlock
chattr +a file              # append-only — perfect for logs
```

## `find` one-liners for audits

```bash
find /usr/bin -perm -4000 -ls 2>/dev/null        # SUID binaries (privesc surface)
find / -xdev -type f -perm -0002 2>/dev/null     # world-writable files
find / -xdev \( -nouser -o -nogroup \) 2>/dev/null  # orphaned ownership
find /etc -xdev -type f -cmin -60 2>/dev/null    # permission/content changed in last hour
```

`-perm -4000` = has the bit (match any); `-perm 4000` = exactly that. `-xdev` stays on one filesystem (avoids hanging on network mounts).

## Key directories

| Path | What's there | Why you care |
|---|---|---|
| `/etc` | Config files | `644` configs, `640` secrets; version-control the ones you own |
| `/var/log` | Logs | `640`/`600`; this is where your disk fills up at 3 AM |
| `/var/lib` | App state (DBs, Docker) | Back this up; don't `chmod -R` it blindly |
| `/proc` | **Kernel** process/memory info (virtual) | `cat /proc/cpuinfo`, `/proc/<pid>/fd` — read-only window into the kernel |
| `/sys` | **Kernel** device/driver tunables (virtual) | Write here to tune the running kernel (e.g. `/sys/block/sda/queue/scheduler`) |
| `/tmp` | Scratch, sticky bit (`1777`) | Wiped on reboot (often tmpfs); world-writable by design |
| `/run` | Runtime state: PIDs, sockets (tmpfs) | `/run/app.sock` lives here, not in `/tmp` |
| `/opt` | Third-party software | Your commercial agents and self-installed tools |
| `/srv` | Served data (web roots, FTP) | Alternative to `/var/www` for site data |

## /proc & /sys paths worth knowing

```bash
/proc/cpuinfo /proc/meminfo          # hardware inventory
/proc/loadavg /proc/uptime           # load, uptime — what monitoring scrapes
/proc/<pid>/cmdline /proc/<pid>/fd   # what a process IS and what it HOLDS OPEN
/proc/sys/net/ipv4/ip_forward        # kernel tunables (sysctl interface)
/sys/block/sda/queue/scheduler       # IO scheduler per disk
/sys/class/net/eth0/statistics/      # NIC counters without extra tools
```

## Debugging ladder (memorize this)

1. `ls -l` the file — decode the string cold; spot the `+` (ACL) and `s`/`t` bits
2. `namei -l /full/path` — check `x` on **every** parent directory
3. `getfacl` — hidden ACL entries override what `ls` shows
4. `lsattr` — immutable flag blocks even root
5. `findmnt` — read-only mount? (`ro` in options)
6. `df -i` — "no space" with free GB = inode exhaustion
7. `lsof +L1` / `du` vs `df` mismatch — deleted-but-open files

*Rule of thumb: permissions are checked at every layer of the path, not just on the file. When in doubt, trace the whole chain.*
