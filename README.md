 # 🐧 Ubuntu Linux Operations Project

![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-Scripting-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Administration-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![CompTIA](https://img.shields.io/badge/CompTIA-Security%2B-E31937?style=for-the-badge)
![Cost](https://img.shields.io/badge/Cost-Free-brightgreen?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Complete-blue?style=for-the-badge)

A hands-on Linux operations project built on Ubuntu Server. It recreates the everyday problems a Linux operations team handles: a filesystem filling up, bulk file cleanup, permissions, log analysis, cron automation, runaway processes, a stopped time service, swap, backups, and config edits.

---
## 📺 [Watch Me Build the Project Here](#)

## 📋 Project Overview

| Field | Detail |
|---|---|
| **Certification Alignment** | RHCSA (in progress), CompTIA Linux+ objectives |
| **Operating System** | Ubuntu Server 24.04 LTS |
| **Environment** | VirtualBox, Multipass, or an Azure B-series VM |
| **Time to Complete** | About 2 hours |
| **Estimated Cost** | Free locally (Azure: deallocate the VM when finished) |
| **Career Relevance** | Linux Administrator, Systems Administrator, Cloud Support, NOC / Operations |

---

## 💼 The Business Problem

Production Linux servers fail in predictable ways. A log grows until the disk is full. Thousands of old files pile up. A service stops and timestamps drift. A process pins the CPU. Each of these becomes a ticket, and each one needs a fast, safe, repeatable fix.

This project simulates those tickets on a disposable VM so every fix can be practiced, verified, and documented before it is needed on a real system.

### How This Maps to Real Ticket Types

| Ticket Type | Exercise |
|---|---|
| Disk / filesystem full | Exercise 5 |
| Bulk file cleanup | Exercise 4 |
| Permission denied | Exercise 2 |
| Error investigation in logs | Exercise 3 |
| Scheduled job / monitoring | Exercise 6 |
| High CPU / slowness | Exercise 7 |
| Service down (NTP) | Exercise 8 |
| Memory pressure | Exercise 9 |
| Backup and restore | Exercise 10 |
| Config change | Exercise 11 |

---

## 🎯 Learning Objectives

- Baseline a Linux server: uptime, load, CPU, memory, disk, and network
- Apply user, group, and directory permissions, including SGID
- Search and count log data with `grep`, `wc`, `tail`, `sed`, and `awk`
- Delete large numbers of files safely with `find`
- Diagnose a full filesystem, including deleted files still held open
- Recover a busy or hung mount point
- Automate a health check with Bash and `cron`
- Find and stop runaway processes
- Restart and verify the time synchronization service
- Add swap, compress backups, and edit configs safely

---

## 🏗️ Architecture

### Troubleshooting Workflow

```mermaid
flowchart LR
    A[Alert / Ticket] --> B[Scope<br/>What changed?]
    B --> C[Investigate<br/>df, du, top, lsof, journalctl]
    C --> D[Fix Safely<br/>dry run first]
    D --> E[Verify<br/>re-run checks]
    E --> F[Document<br/>update ticket]
    F --> G[Prevent<br/>cron, logrotate, alerts]
```

### Lab Environment

```text
Ubuntu Server 24.04 VM
├── /srv/opsshare          → shared team folder (SGID 2770)
├── /mnt/labdisk           → 300 MB loop-device filesystem (safe disk-full test)
├── /usr/local/bin/        → healthcheck.sh (runs every 5 minutes via cron)
├── /var/log/              → healthcheck.log
└── ~/linux-project/       → logs, archives, and config copies
```

---

## ✅ Prerequisites

- [ ] An Ubuntu Server 24.04 LTS VM (2 vCPU, 2 GB RAM, 10 GB disk)
- [ ] A user account with `sudo` access
- [ ] SSH or console access to the VM
- [ ] About 2 hours

> ⚠️ **Use a disposable VM.** Several exercises create users, mounts, and swap. Run the [Cleanup](#-cleanup) section when finished.

---

## 🚀 Exercise 1 — Baseline the Server

**Goal:** Know the system before changing it.

```bash
cat /etc/os-release     # OS version
hostnamectl             # hostname and kernel
uptime                  # time up + load average
nproc                   # CPU count
free -h                 # memory and swap
df -h                   # disk space
lsblk                   # disks and partitions
ip -br a                # IP addresses
mkdir -p ~/linux-project && cd ~/linux-project
```

> 💡 **Load average vs. CPU count:** a load average higher than `nproc` means the system is overloaded.

---

## 🔐 Exercise 2 — Users, Groups, and a Shared Folder

**Goal:** Build a team folder only the `opsteam` group can use.

```bash
sudo groupadd opsteam
sudo useradd -m -s /bin/bash -G opsteam alice
sudo useradd -m -s /bin/bash bob

sudo mkdir -p /srv/opsshare
sudo chown root:opsteam /srv/opsshare
sudo chmod 2770 /srv/opsshare       # SGID + owner rwx + group rwx
ls -ld /srv/opsshare                # drwxrws---

sudo -u alice touch /srv/opsshare/alice.txt   # works
sudo -u bob touch /srv/opsshare/bob.txt       # Permission denied
sudo ls -l /srv/opsshare                      # files inherit the opsteam group
```

| Number | Letters | Meaning |
|---|---|---|
| 7 | `rwx` | read + write + execute |
| 6 | `rw-` | read + write |
| 5 | `r-x` | read + execute |
| 4 | `r--` | read only |

---

## 📄 Exercise 3 — Log Analysis

**Goal:** Generate a 500-line log and answer questions with `grep`, `wc`, `tail`, `sed`, and `awk`.

```bash
for i in $(seq 1 500); do
  case $((RANDOM % 4)) in
    0) lvl=ERROR;; 1) lvl=WARN;; *) lvl=INFO;;
  esac
  echo "2026-10-22 10:$(printf %02d $((i % 60))):00 $lvl service=api request_id=$i"
done > app.log

wc -l app.log                      # total lines (500)
grep -c ERROR app.log              # number of errors
grep -v INFO app.log | head        # everything except INFO
tail -n 10 app.log                 # last 10 lines
sed -n '250p' app.log              # line 250 only
grep -n "request_id=42$" app.log   # find a request + its line number
awk '{print $3}' app.log | sort | uniq -c | sort -rn   # count by level
```

> 💡 `tail -n 10` returns the **last** 10 lines. To get a **specific** line, use `sed -n 'Np'` or `awk 'NR==N'`.

---

## 🗑️ Exercise 4 — Bulk File Cleanup

**Goal:** Safely delete 3,000 old files and keep 2,000 new ones.

```bash
mkdir -p ~/linux-project/oldfiles && cd ~/linux-project/oldfiles
touch -d "45 days ago" old_{1..3000}.log
touch new_{1..2000}.log
ls | wc -l                                                # 5000

find . -type f -name "*.log" -mtime +30 | wc -l           # DRY RUN: 3000
find . -type f -name "*.log" -mtime +30 -delete
ls | wc -l                                                # 2000
cd ~/linux-project
```

> ⚠️ **Always dry run first.** `rm *` can fail with `Argument list too long` on very large directories. `find` handles any number of files.

---

## 💾 Exercise 5 — Disk Full, Deleted-but-Open Files, and a Busy Mount

**Goal:** Simulate a full filesystem safely, find the cause, and fix it.

### A. Build a 300 MB Test Disk

A loop device is used so the real disk is never touched.

```bash
fallocate -l 300M ~/labdisk.img
mkfs.ext4 -q -F ~/labdisk.img
sudo mkdir -p /mnt/labdisk
sudo mount -o loop ~/labdisk.img /mnt/labdisk
sudo chown $USER:$USER /mnt/labdisk
df -h /mnt/labdisk
```

### B. Fill It and Investigate

```bash
fallocate -l 120M /mnt/labdisk/app-debug.log
dd if=/dev/zero of=/mnt/labdisk/old-backup.tar bs=1M count=100
df -h /mnt/labdisk                                           # ~80%+ used
du -sh /mnt/labdisk/* | sort -h                              # what is using it
find /mnt/labdisk -type f -size +50M -exec ls -lh {} \;      # large files
```

### C. Fix 1 — Truncate the Active Log

```bash
truncate -s 0 /mnt/labdisk/app-debug.log
df -h /mnt/labdisk                       # space is returned
```

### D. Fix 2 — The Deleted-but-Open File Trap

```bash
sleep infinity < /mnt/labdisk/old-backup.tar &   # a process holds the file open
rm /mnt/labdisk/old-backup.tar
df -h /mnt/labdisk                               # space is NOT freed
lsof +L1 | grep labdisk                          # shows the file as (deleted)
kill %1                                          # stop the process holding it
df -h /mnt/labdisk                               # space is freed now
```

> 💡 **Why truncate instead of `rm`?** If an application still has a log open, deleting it does not free space until the process restarts. Truncating frees it immediately.

### E. Busy / Hung Mount

```bash
(cd /mnt/labdisk && sleep infinity) &
sudo umount /mnt/labdisk            # "target is busy"
sudo fuser -vm /mnt/labdisk         # identify what is using it
kill %1
sudo umount /mnt/labdisk            # succeeds
# If a mount is still stuck: sudo umount -l /mnt/labdisk  (lazy unmount)
```

---

## ⏰ Exercise 6 — Automate a Health Check with Cron

**Goal:** Deploy a Bash script that reports system health and schedule it every 5 minutes.

```bash
sudo tee /usr/local/bin/healthcheck.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
# healthcheck.sh - basic Ubuntu health report
THRESHOLD="${THRESHOLD:-80}"
LOG="${LOG:-/var/log/healthcheck.log}"
{
  echo "===== $(date '+%F %T') | $(hostname) ====="
  echo "Uptime : $(uptime -p)"
  echo "Load   : $(cut -d' ' -f1-3 /proc/loadavg) (CPUs: $(nproc))"
  free -h | awk '/^Mem:/  {print "Memory : used " $3 " of " $2 " (available " $7 ")"}'
  free -h | awk '/^Swap:/ {print "Swap   : used " $3 " of " $2}'
  df -hP -x tmpfs -x devtmpfs -x squashfs -x overlay | awk -v t="$THRESHOLD" \
    'NR>1 {u=$5; sub("%","",u); if (u+0 >= t) print "ALERT  : " $6 " is " u "% full"}'
  echo "Failed services: $(systemctl --failed --no-legend 2>/dev/null | wc -l)"
} >> "$LOG" 2>&1
EOF

sudo chmod 755 /usr/local/bin/healthcheck.sh
sudo env THRESHOLD=0 /usr/local/bin/healthcheck.sh   # test run: alerts on every filesystem
sudo tail -n 12 /var/log/healthcheck.log
```

### Schedule It

```bash
sudo crontab -e
```

Add this line, then save and exit:

```text
*/5 * * * * /usr/local/bin/healthcheck.sh
```

```bash
sudo crontab -l                               # confirm the job
sudo tail -n 8 /var/log/healthcheck.log       # after 5+ minutes
grep CRON /var/log/syslog | tail -n 5         # or: journalctl -u cron
```

| Field | Range | Example (`*/5 * * * *`) |
|---|---|---|
| Minute | 0–59 | `*/5` = every 5 minutes |
| Hour | 0–23 | `*` = every hour |
| Day of month | 1–31 | `*` |
| Month | 1–12 | `*` |
| Day of week | 0–7 (0 and 7 = Sunday) | `*` |

---

## ⚙️ Exercise 7 — Find and Kill a Runaway Process

**Goal:** Create CPU load, find it, and stop it the right way.

```bash
yes > /dev/null &
yes > /dev/null &
uptime                            # load climbs
top -b -n 1 | head -15            # snapshot of top
ps aux --sort=-%cpu | head -5     # biggest CPU users
pgrep -a yes                      # list PIDs
kill <PID>                        # graceful stop (SIGTERM)
pkill yes                         # stop by name
vmstat 1 5                        # confirm the system is back to normal
```

> ⚠️ Use `kill -9` (SIGKILL) only when a process ignores SIGTERM. It gives the process no chance to clean up.

---

## 🕒 Exercise 8 — Stop, Restart, and Verify the Time Service

**Goal:** Recover NTP and confirm the clock is synchronized.

```bash
timedatectl                       # look for "NTP service: active"
systemctl list-units --type=service --all | grep -Ei 'chrony|timesyncd|ntp'
```

**If the server uses `systemd-timesyncd`** (Ubuntu default):

```bash
sudo systemctl stop systemd-timesyncd
timedatectl | grep -i "NTP service"           # inactive
sudo systemctl restart systemd-timesyncd
systemctl status systemd-timesyncd --no-pager
timedatectl timesync-status
```

**If the server uses `chrony`** (common on cloud images, including Azure):

```bash
sudo systemctl restart chrony
systemctl status chrony --no-pager
chronyc tracking
chronyc sources -v
```

> 💡 Accurate time keeps logs, TLS certificates, authentication, and transaction timestamps correct.

---

## 🧠 Exercise 9 — Add and Remove Swap

**Goal:** Add 1 GB of temporary swap and verify it.

```bash
swapon --show ; free -h               # before
sudo fallocate -l 1G /swapfile.lab
sudo chmod 600 /swapfile.lab
sudo mkswap /swapfile.lab
sudo swapon /swapfile.lab
swapon --show ; free -h               # after
cat /proc/sys/vm/swappiness

sudo swapoff /swapfile.lab && sudo rm /swapfile.lab   # remove it
```

> 💡 For permanent swap, add an entry to `/etc/fstab`. It is skipped here so the project leaves no trace.

---

## 📦 Exercise 10 — Back Up and Compress `/etc`

**Goal:** Create, inspect, and restore from a compressed archive.

```bash
cd ~/linux-project
sudo tar -czf etc-backup.tar.gz /etc
ls -lh etc-backup.tar.gz                  # compressed size
sudo du -sh /etc                          # original size
tar -tzf etc-backup.tar.gz | head         # list contents
mkdir -p restore
tar -xzf etc-backup.tar.gz -C restore etc/hostname
cat restore/etc/hostname
```

| Flag | Meaning |
|---|---|
| `c` / `x` / `t` | create / extract / list |
| `z` | gzip compression |
| `f` | archive file name follows |
| `-C` | extract into a directory |

---

## ✏️ Exercise 11 — Edit a Config and Replace a String

**Goal:** Change a value with `sed` and `vim`, safely, on a copy.

```bash
cp /etc/ssh/sshd_config ~/linux-project/sshd_config.lab
cd ~/linux-project
grep -n "Port" sshd_config.lab
sed -i.bak 's/^#Port 22/Port 2222/' sshd_config.lab
diff sshd_config.lab.bak sshd_config.lab

vim sshd_config.lab
# In vim:  :%s/2222/22/gc   then   :wq
```

---

## 🗄️ Exercise 12 — Read the Storage Layout

**Goal:** Identify the partition table type, partitions, LVM, and RAID.

```bash
lsblk -f                          # filesystems + mount points
sudo parted -l                    # msdos (MBR) or gpt
sudo fdisk -l
sudo pvs ; sudo vgs ; sudo lvs    # LVM layers, if used
cat /proc/mdstat                  # software RAID, if any
```

---

## 🧹 Cleanup

```bash
sudo crontab -l | grep -v healthcheck.sh | sudo crontab -
sudo rm -f /usr/local/bin/healthcheck.sh /var/log/healthcheck.log
pkill yes 2>/dev/null ; pkill -f "sleep infinity"
sudo umount /mnt/labdisk 2>/dev/null ; sudo rmdir /mnt/labdisk
rm -f ~/labdisk.img
sudo userdel -r alice ; sudo userdel -r bob ; sudo groupdel opsteam
sudo rm -rf /srv/opsshare
rm -rf ~/linux-project
```

> ⚠️ `pkill -f "sleep infinity"` stops **all** matching processes. Check with `pgrep -af "sleep infinity"` first on any shared system.

---

## 📚 Key Linux Concepts

| Concept | Description |
|---|---|
| **cron** | Time-based job scheduler that runs commands automatically |
| **File permissions** | Read (4), write (2), execute (1) for user, group, and others |
| **SGID on a directory** | New files inherit the directory's group |
| **Logical partition** | MBR workaround for more than 4 partitions, numbered 5+ inside an extended partition |
| **LVM logical volume** | Resizable volume carved from a volume group |
| **RAID** | Combines disks for performance and/or redundancy — not a backup |
| **Swap** | Disk space used as overflow when RAM is full |
| **Deleted-but-open file** | Space stays used until the holding process closes it (`lsof +L1`) |
| **Lazy unmount** | `umount -l` detaches a busy mount once it is no longer in use |

---

## 📸 Portfolio Evidence

> Save screenshots in a `/screenshots` folder in this repository and update the image links below.

### 1. Disk Full — Before and After

![Disk Full Before and After](screenshots/disk-full-before-after.png)

`df -h` showing the test filesystem above 80%, then recovered after truncating the active log.

### 2. Deleted-but-Open File

![Deleted but Open File](screenshots/lsof-deleted-file.png)

`lsof +L1` showing a deleted file still holding space, and `df -h` after stopping the process.

### 3. Cron Health Check Output

![Cron Health Check](screenshots/healthcheck-log.png)

`healthcheck.log` entries written every 5 minutes by cron.

### 4. Permission Denied Test

![Permission Test](screenshots/sgid-permission-test.png)

`alice` creates a file in the shared folder; `bob` is denied. Files inherit the `opsteam` group through SGID.

---

## 🛠️ Tools and Technologies

![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)
![Vim](https://img.shields.io/badge/Vim-019733?style=for-the-badge&logo=vim&logoColor=white)

- **Ubuntu Server 24.04 LTS** — operating system for all exercises
- **Bash** — scripting and automation (`healthcheck.sh`)
- **cron** — scheduled job execution
- **systemd** — service management (`systemctl`, `journalctl`, `timedatectl`)
- **find / grep / sed / awk / wc** — file and text processing
- **lsof / fuser** — open-file and mount troubleshooting
- **tar / gzip** — backup and compression
- **vim** — configuration editing

---

## 🔗 Related Projects

- [ServiceNow ITSM](https://github.com/glenpagesr-dev/Servicenow-ITSM) — incident, change, and service request workflows that track the tickets this project resolves
- [Azure AD / Entra ID Administration](https://github.com/glenpagesr-dev/azure-ad-admin-lab) — identity and access management in Microsoft Entra ID

---

## 👤 Author

**Glen Page** — Linux Administrator | CompTIA Security+

- GitHub: [@glenpagesr-dev](https://github.com/glenpagesr-dev)
- LinkedIn: [Glen Page](https://www.linkedin.com/in/glen-page-862730246)

---

> All exercises were performed on a disposable Ubuntu VM for learning and portfolio purposes. No production systems were used.
