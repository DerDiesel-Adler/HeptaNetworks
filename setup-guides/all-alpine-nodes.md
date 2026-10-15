*AI-Generated

# Alpine Linux Nodes — Setup Guide

All seven Alpine-based nodes in this lab (Admin-HQ, PC-HQ-MA1, Jump-Host-HQ,
Staging-Server-HQ, Backup-Server-HQ, Radius-HQ, Syslog-HQ) share the same
base image and a common baseline setup. This document covers that shared
baseline once, then goes through each node individually — including the
full backup automation chain and the syslog server setup.

**Image:** Alpine Linux 3.21 (see
[`image-integration.md`](image-integration.md) for how it was added to
EVE-NG).

RADIUS-HQ is also an Alpine node, but its FreeRADIUS-specific
configuration is documented together with the rest of the VPN setup in
[`vpn-radius-setup.md`](vpn-radius-setup.md) rather than here, since it
only makes sense in that context. Its network/SSH baseline is still
covered below for consistency.

---

## Baseline setup (every node)

### 1. Check disk persistence

Before making any lasting changes, confirm the node is running from a real
disk, not a RAM-only (diskless) boot:

```
mount | grep " / "
```

`ext4 on /dev/vdaX` means changes persist across reboots — the normal
case for every node in this lab. `tmpfs on /` would mean diskless mode,
requiring `lbu commit -d` after every change to persist it (not needed
for any node here, but worth checking on a freshly cloned node before
assuming).

### 2. Set a unique hostname

```
echo "<hostname>" > /etc/hostname
hostname -F /etc/hostname
```

### 3. Static network configuration

```
vi /etc/network/interfaces
```
```
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
        address <node-ip>
        netmask 255.255.255.0
        gateway <gateway-ip>
```
```
ifup eth0
rc-update add networking default
```

### 4. Time synchronization

```
apk add chrony tzdata
rc-update add chronyd default
rc-service chronyd start
cp /usr/share/zoneinfo/Europe/Berlin /etc/localtime
echo "Europe/Berlin" > /etc/timezone
```

### 5. Admin account (key-based SSH, no direct root login)

```
adduser admin
addgroup admin wheel
```

Set a password immediately when prompted — leaving it empty locks the
account and blocks SSH entirely, including key-based login (see
[`troubleshooting.md`](../troubleshooting/troubleshooting.md) for the
full diagnosis story behind that one).

```
mkdir -p /home/admin/.ssh
echo "<public-key>" >> /home/admin/.ssh/authorized_keys
chown -R admin:admin /home/admin/.ssh
chmod 700 /home/admin/.ssh
chmod 600 /home/admin/.ssh/authorized_keys
```

Confirm `PubkeyAuthentication yes` is active in `/etc/ssh/sshd_config`
(it ships commented out by default on this image — another entry covered
in the troubleshooting doc) and disable direct root login over SSH:

```
sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
service sshd restart
```

---

## Node reference

| Node | IP | Segment | Role |
|---|---|---|---|
| Admin-HQ | 10.10.10.9 | VLAN10 | Admin workstation, whitelisted for jump-host access |
| PC-HQ-MA1 | DHCP (VLAN10 pool) | VLAN10 | Employee client, backup push source |
| Jump-Host-HQ | 10.10.99.9 | VLAN99 | Bastion host |
| Staging-Server-HQ | 10.10.20.10 | VLAN20 | Intermediate storage, backup push target / pull source |
| Backup-Server-HQ | 10.10.0.5 | Server-LAN (no VLAN) | Backup destination, physically isolated |
| Radius-HQ | 10.10.30.5 | VLAN30 | External RADIUS server (see `vpn-radius-setup.md`) |
| Syslog-HQ | 10.10.99.6 | VLAN99 | Central log collector |

---

## Admin-HQ

Represents the admin's workstation. Sits in VLAN10 like a regular
employee device, but has an explicit ACL exception on the router allowing
it (and only it) to reach the jump host — the point being that admin
access is IP-restricted, not just a matter of knowing credentials.

Baseline setup as above, static IP `10.10.10.9`. Its SSH keypair is the
first hop in the admin access chain (Admin-HQ → Jump-Host → target
server):

```
ssh-keygen -t ed25519 -C "admin@Admin-HQ"
```

Public key gets deployed to the jump host's `admin` user as described in
the baseline section.

**Test:**
```
ssh admin@10.10.99.9
```

---

## Jump-Host-HQ

The bastion host — every subsequent administrative hop to another server
originates from here, using its own keypair, separate from the one used
to reach the jump host itself:

```
su - admin
ssh-keygen -t ed25519 -C "admin@Jump-Host-HQ"
```

This key gets deployed to every other server's `admin` user (Staging,
Backup, Radius, Syslog).

**Legacy SSH algorithms for the Cisco devices.** The IOS 15.x images on
Router-HQ, Switch-HQ and AP-HQ only offer `diffie-hellman-group14-sha1`
for key exchange and `ssh-rsa` as host key type, both disabled by default
in current OpenSSH. They are re-enabled on the jump host for the
management segment only, so a plain `ssh admin@10.10.99.12` works without
extra flags:

```
cat >> ~/.ssh/config << 'EOF'
Host 10.10.99.*
    KexAlgorithms +diffie-hellman-group14-sha1
    HostKeyAlgorithms +ssh-rsa
EOF
chmod 600 ~/.ssh/config
```

The `+` appends to the default list instead of replacing it, so the Linux
nodes in the same segment keep negotiating modern algorithms. The Cisco
devices authenticate with local accounts (SSHv2, `login local`), key-based
login applies to the Linux nodes.

---

## PC-HQ-MA1

Represents a regular employee client. Receives its IP via DHCP from the
router's `Employees` pool rather than a static address. Its role in this
lab is exclusively as the source of the backup push — see the
[Backup Chain](#backup-chain) section below.

Employees work as an unprivileged user, not as `root`. The account
`employee` owns the working directory `/home/employee/work/` and the push
key, so no root credentials are involved in the backup push:

```
adduser employee
mkdir -p /home/employee/work
chown -R employee:employee /home/employee
su - employee
ssh-keygen -t ed25519 -C "employee@PC-HQ-MA1"
```

(Set a password at account creation, same reasoning as above.) The public
key goes into `pushuser`'s `authorized_keys` on Staging-Server-HQ.

---

## Staging-Server-HQ

Intermediate storage layer: receives pushed data from employee clients,
and is the source that the backup server later pulls from. Three dedicated
service accounts live here in addition to the baseline `admin` account:

- **`pushuser`** — receives incoming pushes from the Linux employee client
  (PC-HQ-MA1). Never initiates connections itself.
- **`winpush`** — receives incoming pushes from the Windows VPN client
  (see [`vpn-radius-setup.md`](vpn-radius-setup.md)). Never initiates
  connections itself.
- **`backupuser`** — accepts the backup server's pull connections. Also
  never initiates connections itself.

```
adduser pushuser
adduser backupuser
```
(set a password for each immediately, same reasoning as above)

Public keys from the respective remote hosts get added to each user's
`~/.ssh/authorized_keys` following the same pattern as the baseline admin
setup.

```
apk add rsync
```

`backupuser` has to read both incoming directories
(`/home/pushuser/incoming/` and `/home/winpush/incoming/`). The default
home directory permissions (`drwxr-sr-x`) already allow that, which can be
confirmed with `ls -ld /home/pushuser /home/winpush`.

---

## Backup-Server-HQ

Physically isolated — connected directly to the router, not through any
switch or VLAN trunk. Only ever initiates outbound connections; never
accepts inbound connections except administrative SSH from the jump host.

```
apk add rsync openssh-client
ssh-keygen -t ed25519 -C "root@Backup-Server"
```

Public key gets added to `backupuser`'s `authorized_keys` on
Staging-Server-HQ.

### Backup Chain

The full automation, in both directions:

**1. Push: PC-HQ-MA1 → Staging-Server-HQ, daily at 17:30**

On PC-HQ-MA1 (the push runs as `employee`, using that user's key):
```
apk add rsync openssh-client
```

```
cat > /usr/local/bin/push-backup.sh << 'EOF'
#!/bin/sh
rsync -avz -e ssh /home/employee/work/ pushuser@10.10.20.10:/home/pushuser/incoming/ && date > /home/employee/.last-push
EOF
chmod +x /usr/local/bin/push-backup.sh
```

The marker file `.last-push` is only written if `rsync` succeeded (`&&`),
so a failed push is retried by the login fallback below.

```
rc-update add crond default
rc-service crond start
su - employee
crontab -e
```
Add:
```
30 17 * * * /usr/local/bin/push-backup.sh >> /var/log/push-backup.log 2>&1
```

**Login fallback** — catches up a missed push (e.g. laptop was off at
17:30) the next time the user logs in:
```
cat > /etc/profile.d/push-check.sh << 'EOF'
#!/bin/sh
TODAY=$(date +%Y-%m-%d)
LASTPUSH=$(date -r /home/employee/.last-push +%Y-%m-%d 2>/dev/null)

if [ "$TODAY" != "$LASTPUSH" ]; then
    /usr/local/bin/push-backup.sh &
fi
EOF
chmod +x /etc/profile.d/push-check.sh
```

**2. Pull: Backup-Server-HQ → Staging-Server-HQ, daily at 20:00**

On Backup-Server-HQ, hardlink-snapshot approach — unchanged files across
consecutive days cost no additional disk space, since `--link-dest`
hardlinks them instead of copying. Both push targets on the staging
server are pulled into the same dated snapshot: the Linux client's data
directly in the day folder, the Windows VPN client's data in a `vpn-client`
subfolder.

```
cat > /usr/local/bin/backup.sh << 'EOF'
#!/bin/sh
DATE=$(date +%Y-%m-%d)
DEST="/backup/staging/$DATE"
LATEST="/backup/staging/latest"

mkdir -p "$DEST"
rsync -avz --link-dest="$LATEST" -e "ssh -i /root/.ssh/id_ed25519" backupuser@10.10.20.10:/home/pushuser/incoming/ "$DEST"
mkdir -p "$DEST/vpn-client"
rsync -avz --link-dest="$LATEST/vpn-client" -e "ssh -i /root/.ssh/id_ed25519" backupuser@10.10.20.10:/home/winpush/incoming/ "$DEST/vpn-client/"
rm -f "$LATEST"
ln -s "$DEST" "$LATEST"
EOF
chmod +x /usr/local/bin/backup.sh
```

```
rc-update add crond default
rc-service crond start
crontab -e
```
Add:
```
0 20 * * * /usr/local/bin/backup.sh >> /var/log/backup.log 2>&1
```

**Verification:**
```
/usr/local/bin/push-backup.sh              # on PC-HQ-MA1, as employee
ls -la /home/pushuser/incoming/            # on Staging-Server-HQ
/usr/local/bin/backup.sh                   # on Backup-Server-HQ
ls -la /backup/staging/                    # dated folders + a `latest` symlink
ls -la /backup/staging/latest/             # data pushed by PC-HQ-MA1
ls -la /backup/staging/latest/vpn-client/  # data pushed by the VPN client
```
On the very first run, rsync prints `--link-dest arg does not exist` for
the `vpn-client` folder. That is expected, there is no previous snapshot
to hardlink against yet.

To confirm the hardlink behavior is actually working (no duplicate disk
usage for unchanged files across two snapshots):
```
ls -i /backup/staging/<day-1>/<file> /backup/staging/<day-2>/<file>
```
Identical inode numbers confirm it's the same physical data, referenced
twice.

---

## Radius-HQ

Baseline network/SSH setup as above, static IP `10.10.30.5`. Full
FreeRADIUS installation and configuration (packages, EAP module,
certificates, client/user definitions) is documented in
[`vpn-radius-setup.md`](vpn-radius-setup.md), since it only makes sense in
the context of the full VPN authentication flow.

---

## Syslog-HQ

Central log collector. Originally placed on VLAN30 alongside RADIUS-HQ,
later moved to VLAN99 — see the corresponding entry in
[`troubleshooting.md`](../troubleshooting/troubleshooting.md) for why
(switch and AP couldn't route to a separate VLAN, but could reach a
server in their own Layer-2 segment).

Current static IP: `10.10.99.6`.

```
apk add rsyslog
```

Alpine's `rsyslog` package references a `$FileGroup adm` directive by
default, but the `adm` group doesn't exist on a fresh install — check and
create it if needed before starting the service:
```
grep adm /etc/group || addgroup adm
```

Enable UDP/TCP reception on port 514, appended to `/etc/rsyslog.conf`:
```
cat >> /etc/rsyslog.conf << 'EOF'

module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
EOF
```

rsyslog does not automatically pick up the system's local timezone —
without setting `TZ` explicitly, log timestamps are written in UTC
instead of local time:
```
echo 'export TZ="Europe/Berlin"' >> /etc/conf.d/rsyslog
```

```
rc-update add rsyslog default
rc-service rsyslog start
```

**Currently forwarding logs here:** Router-HQ, FortiGate-HQ, Switch-HQ and
AP-HQ (configuration in the files under `configs/`). Switch-HQ and AP-HQ
can only reach the server because it sits in their own Layer-2 segment,
see the single-hop default-gateway limitation documented in
[`troubleshooting.md`](../troubleshooting/troubleshooting.md).

**Verification:**
```
tail -f /var/log/messages
```
watch for entries from `10.10.100.2` (router) or the FortiGate's
management IP after triggering any loggable event on those devices (e.g.
`write memory` on the router).
