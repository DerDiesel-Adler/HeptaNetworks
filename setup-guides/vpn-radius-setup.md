*AI-Generated

# VPN & RADIUS Authentication Setup

End-to-end setup of the road-warrior VPN: IKEv2 on FortiGate, EAP-MSCHAPv2
authentication against an external FreeRADIUS server, split tunneling, and
administrative SSH access to the Windows VPN client itself.

**FortiOS:** v7.4.12 build2902 (unlicensed VM — see
[Known limitations](#known-limitations-of-this-fortigate-vm) below)
**FreeRADIUS:** 3.0.27 (Alpine package)
**FortiClient VPN:** 7.4.3 build4726
**VPN client OS:** Windows 10 Tiny (see [`image-integration.md`](image-integration.md))

For the debugging story behind several of the steps below (why local
FortiGate authentication doesn't work here, why IKE version has to be set
explicitly, a stale tunnel object that caused a cryptic validation
failure), see
[`troubleshooting.md`](../troubleshooting/troubleshooting.md). This
document covers the working end state and how to reach it, not every
dead end along the way.

---

## Why an external RADIUS server

FortiGate can authenticate VPN users locally, and normally that's the
simpler option. On this particular VM license, local EAP authentication
is broken at a lower level — the firewall routes EAP requests for local
users to itself over loopback, and that internal request never receives a
response, regardless of configuration. Rather than being a workaround for
this specific license issue, this also happens to be closer to how larger
real-world environments are set up: centralized authentication instead of
duplicated local user databases per device.

---

## RADIUS-HQ (FreeRADIUS)

Baseline networking and SSH setup as described in
[`all-alpine-nodes.md`](all-alpine-nodes.md). Static IP `10.10.30.5`.

### Install

```
apk add freeradius freeradius-eap freeradius-utils
```

`freeradius-utils` alone only provides client-side testing tools
(`radtest`), not the server itself. `freeradius-eap` is a separate
package from the core server — without it, EAP requests are silently
ignored (logged only as `Ignoring "eap"`, easy to miss).

### Add the VPN user

```
vi /etc/raddb/users
```
```
vpnuser Cleartext-Password := "eve"
```

### Register the FortiGate as an allowed client

```
vi /etc/raddb/clients.conf
```
```
client localhost {
    ipaddr = 127.0.0.1
    secret = eve
    require_message_authenticator = no
}

client fortigate-net {
    ipaddr = 10.10.0.0/16
    secret = eve
    require_message_authenticator = no
}
```

### Enable the EAP module

```
ln -s /etc/raddb/mods-available/eap /etc/raddb/mods-enabled/eap
```

The EAP module also initializes TLS submodules on load, even when only
using EAP-MSCHAPv2 (no certificates involved in the actual authentication
flow). Without any certificates present, the module fails to load at all.
Generate the bundled test certificates once:

```
cd /etc/raddb/certs
sh ./bootstrap
```

### Start and verify

```
rc-update add radiusd default
rc-service radiusd start
```

```
echo "127.0.0.1 Radius-HQ" >> /etc/hosts
radtest vpnuser eve localhost 0 eve
```
Expected: `Received Access-Accept`.

---

## FortiGate configuration

### Phase 1 (IKEv2)

```
config vpn ipsec phase1-interface
    edit "RoadWarrior"
        set type dynamic
        set interface "port1"
        set ike-version 2
        set peertype any
        set net-device disable
        set mode-cfg enable
        set proposal des-sha256
        set dpd on-idle
        set dhgrp 14
        set eap enable
        set eap-identity send-request
        set authusrgrp "GRP_SSLVPN_Users"
        set ipv4-start-ip 10.10.200.10
        set ipv4-end-ip 10.10.200.50
        set ipv4-netmask 255.255.255.0
        set dns-mode auto
        set ipv4-split-include "Internal-Net"
        set psksecret Fortigate
        set dpd-retryinterval 60
    next
end
```

Note that `psksecret` is required even with EAP enabled — FortiOS uses it
as a device-level pre-check in addition to EAP's user-level
authentication, not as a replacement for one or the other.

### Split-tunnel address object

```
config firewall address
    edit "Internal-Net"
        set subnet 10.10.0.0 255.255.0.0
    next
end
```

### Phase 2

```
config vpn ipsec phase2-interface
    edit "RoadWarrior"
        set phase1name "RoadWarrior"
        set proposal des-sha256
        set pfs enable
        set dhgrp 14
        set src-subnet 10.10.0.0 255.255.0.0
    next
end
```

The `src-subnet` here has to match the split-tunnel network from Phase 1
— left at the default `0.0.0.0/0.0.0.0`, the split-tunnel setting and the
actual negotiated traffic selectors disagree, and the tunnel silently
stops carrying internal traffic even though the connection itself still
comes up.

### RADIUS server object

```
config user radius
    edit "Radius-HQ"
        set server "10.10.30.5"
        set secret eve
    next
end
```

### User group

```
config user group
    edit "GRP_SSLVPN_Users"
        set member "Radius-HQ"
    next
end
```

### Firewall policies (both directions)

```
config firewall policy
    edit 0
        set name "VPN-to-LAN"
        set srcintf "RoadWarrior"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat disable
    next
    edit 0
        set name "LAN-TO-VPN"
        set srcintf "port2"
        set dstintf "RoadWarrior"
        set srcaddr "all"
        set dstaddr "all"
        set action accept
        set schedule "always"
        set service "ALL"
    next
end
```

Both directions are needed — the first handles VPN clients reaching into
the internal network, the second handles internal hosts (e.g. the jump
host) reaching back out to a connected client.

### Return route on Router-HQ

Router-HQ needs to know how to reach the VPN pool subnet, routed back
through the FortiGate:
```
ip route 10.10.200.0 255.255.255.0 10.10.100.1
```

---

## FortiClient configuration (Windows)

**VPN type:** IPsec VPN — not SSL-VPN, which is a separate FortiGate
feature never used in this setup.

| Field | Value |
|---|---|
| Connection Name | `RoadWarrior-HQ` |
| Remote Gateway | FortiGate's WAN IP |
| Authentication Method | Pre-shared key |
| Pre-shared Key | `Fortigate` |
| Username | `vpnuser` |

**Advanced Settings → VPN Settings:**
- **IKE Version: 2** — FortiClient defaults to Version 1, which will not
  match this Phase 1 config at all (see the IKEv1/IKEv2 mismatch entry in
  `troubleshooting.md`)
- Phase 1 Proposal: Encryption `DES`, Authentication `SHA256`, DH Group `14`
- Phase 2 Proposal: Encryption `DES`, Authentication `SHA256`, DH Group `14`
- Enable Perfect Forward Secrecy (PFS)

On connect, FortiClient prompts separately for username/password —
`vpnuser` / `eve`, matching the RADIUS user created above.

---

## Administrative SSH access to the Windows VPN client

Since the Windows client is a full node in this topology, not just an
external test device, it also gets administrative SSH access through the
jump host — the same access model used for every other node.

### Install and start OpenSSH Server

Via **Settings → Optional Features → Add a feature → OpenSSH Server**
(install the OpenSSH Client feature first if the Server option isn't
listed yet).

```
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

### Deploy the admin key

Windows uses a different, system-wide path for administrator accounts
instead of a per-user `authorized_keys`:

```
C:\ProgramData\ssh\administrators_authorized_keys
```

Create the file (e.g. via Notepad — copy-pasting through a VNC console
can silently corrupt individual characters in a long key string, so
verify with a fingerprint comparison afterward rather than trusting a
visual check), then lock down permissions to only SYSTEM and
Administrators:

```
icacls C:\ProgramData\ssh\administrators_authorized_keys /inheritance:r
icacls C:\ProgramData\ssh\administrators_authorized_keys /grant SYSTEM:F
icacls C:\ProgramData\ssh\administrators_authorized_keys /grant Administrators:F
```

Any broader permissions on this file (e.g. a stray write permission for a
non-admin account) cause the SSH daemon to silently reject the key.

**Verify the deployed key matches what the client actually offers**,
rather than assuming a copy-paste succeeded:
```
ssh-keygen -lf C:\ProgramData\ssh\administrators_authorized_keys
```
Compare against the fingerprint shown by `ssh -v` from the connecting
side (`Offering public key: ... SHA256:...`).

### Test

From the jump host:
```
ssh vpnuser@10.10.200.10
```
(only reachable while the VPN tunnel is actively connected, using the
client's currently assigned pool address)

---

## Verification

With the tunnel connected on the Windows client:

```
ping 8.8.8.8       # direct, not through the tunnel (split-tunnel working)
ping 10.10.99.9    # through the tunnel (internal reachability working)
```

From the jump host or router, reaching back into the tunnel:
```
ping 10.10.200.10
```
Windows blocks inbound ICMP by default — if this fails despite the tunnel
otherwise working, check the Windows Defender Firewall inbound rules for
ICMPv4 before assuming a routing problem.

---

## Backup automation from the Windows VPN client

A road-warrior client isn't reliably connected at a fixed time each day,
so rather than a Cron-style fixed schedule (as used for PC-HQ-MA1 in
[`all-alpine-nodes.md`](all-alpine-nodes.md)), the push here is triggered
by the VPN connection itself.

Windows has no native `rsync` — `scp` (bundled with the OpenSSH client
already installed for administrative access) is used instead for the
actual file transfer.

### Staging-Server-HQ: dedicated push user

```
adduser winpush
mkdir -p /home/winpush/.ssh
echo "<windows-client-public-key>" >> /home/winpush/.ssh/authorized_keys
chown -R winpush:winpush /home/winpush/.ssh
chmod 700 /home/winpush/.ssh
chmod 600 /home/winpush/.ssh/authorized_keys
mkdir -p /home/winpush/incoming
chown winpush:winpush /home/winpush/incoming
```

### Router-HQ: ACL gap

The `Staging-Server-Restrictions` ACL predates the VPN pool subnet and had
no return-traffic rules for it, silently blocking both ping replies and
SSH responses to VPN clients despite the tunnel itself working correctly:

```
ip access-list extended Staging-Server-Restrictions
 permit icmp host 10.10.20.10 10.10.200.0 0.0.0.255 echo-reply
 permit tcp host 10.10.20.10 eq 22 10.10.200.0 0.0.0.255 established
```

### Windows client: SSH key and push script

```
ssh-keygen -t ed25519 -C "vpnuser@Win10-VPN-Client"
```

`C:\Scripts\push-backup.bat`, with a same-day marker file to avoid
repeated pushes while the trigger below fires every few minutes:

```bat
@echo off
set MARKER=C:\Scripts\.last-push
set TODAY=%date%

if exist "%MARKER%" (
    set /p LASTPUSH=<"%MARKER%"
    if "%LASTPUSH%"=="%TODAY%" exit /b
)

scp -i C:\Users\%USERNAME%\.ssh\id_ed25519 -r C:\Users\%USERNAME%\Documents\Work\* winpush@10.10.20.10:/home/winpush/incoming/
echo %TODAY% > "%MARKER%"
```

### Task Scheduler: triggering on VPN connection

The intended approach was an event trigger on VPN connect (Windows
normally logs this under `Microsoft-Windows-RasClient/Operational`). That
event log provider doesn't exist on this Tiny10 image, even after
enabling analytic/debug logs — apparently stripped from the image rather
than just hidden. Worked around instead with a recurring time trigger
combined with a network-availability condition, which only actually runs
the action while the VPN is connected:

- **Trigger:** Daily, repeat every 5 minutes, indefinitely
- **Conditions:** "Start only if the following network connection is
  available" → select the VPN adapter
- **Action:** `C:\Scripts\push-backup.bat`
- **General tab:** "Run whether user is logged on or not"

Three things that weren't obvious while setting this up:

- **The VPN adapter shows up as "Unidentified network", not under a
  recognizable name**, and its "Public"/"Private" classification gives no
  indication either. The reliable way to identify it is by IP range
  (`10.10.200.x`) or by comparing `IPv4Connectivity` in
  `Get-NetConnectionProfile` — the VPN adapter shows `LocalNetwork`
  (no direct internet access through it, consistent with split tunneling),
  while the regular network adapter shows `Internet`.
- **"Run whether user is logged on or not" requires the Windows account
  to have an actual password set** — a blank password causes the task to
  fail immediately with a generic "User account restriction error" that
  doesn't mention passwords at all.

### Verification

```
Get-Content C:\Scripts\.last-push
```
should show today's date after a successful run; on Staging-Server-HQ:
```
ls -la /home/winpush/incoming/
```

---

## Known limitations of this FortiGate VM

- **DES only** for IKE/IPsec encryption — no AES available on this
  unlicensed VM. In a licensed/production deployment, AES would be used
  instead.
- **Maximum 3 firewall policies** in the root VDOM at once — relevant
  when planning additional policies beyond what's listed here.
- **Local EAP authentication does not work** for the reasons described
  above — this is why RADIUS-HQ exists at all in this topology.
