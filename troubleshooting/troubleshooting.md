# Troubleshooting

A detailed look at problems encountered while building this network that aren't already covered in the README highlights

## Contents

- [Layer-2 switches only route one hop via their default gateway](#layer-2-switches-only-route-one-hop-via-their-default-gateway)
- [A management SVI dropped to down after every reload, despite a correct config](#a-management-svi-dropped-to-down-after-every-reload-despite-a-correct-config)
- [An empty password at account creation locks the account entirely](#an-empty-password-at-account-creation-locks-the-account-entirely)
- [FreeRADIUS silently ignored EAP entirely](#freeradius-silently-ignored-eap-entirely)
- [A misleading error hid a session-context problem, twice over](#a-misleading-error-hid-a-session-context-problem-twice-over)
- [FortiClient negotiates IKEv1 by default, not IKEv2](#forticlient-negotiates-ikev1-by-default-not-ikev2)
- [Legacy Cisco IOS images rejected SSH from a modern OpenSSH client](#legacy-cisco-ios-images-rejected-ssh-from-a-modern-openssh-client)

## Network & ACL Logic

### Layer-2 switches only route one hop via their default gateway

The switch and access point repeatedly failed to reach services placed one VLAN away — an NTP source, a syslog server, even the VPN client pool — while the router reached the exact same targets without issue.

**`ip default-gateway`** on a pure L2 switch is a single-hop pointer, not real routing. Confirmed by comparison testing: a ping to the switch's own gateway worked, one hop further didn't. Rather than fight the limitation, affected services (like the syslog server) were moved into the same Layer-2 segment as the switch and AP instead — sidestepping the need for routing entirely.

### A management SVI dropped to down after every reload, despite a correct config

After each switch restart, the management interface was administratively enabled but simply not reachable — with nothing in the saved configuration indicating a problem.

**`show ip interface brief`** showed the interface down, even though no **`shutdown`** line existed anywhere in the config. Adding an explicit **`shutdown`** followed by **`no shutdown`** directly in the startup config didn't help either — IOS evaluates only the final configured state when booting, not the sequence of commands that produced it. The only reliable fix was a manual bounce at runtime, with a real time gap between **`shutdown`** and **`no shutdown`**, repeated after every boot.


## Linux & Alpine Specifics

### An empty password at account creation locks the account entirely

SSH logins failed repeatedly with a correctly deployed public key, on more than one server, with more than one user, ruling out a one-off mistake.

Starting **`sshd`** in foreground debug mode showed the actual reason explicitly: the account was locked. Alpine's **`adduser`** marks an account as locked internally when the password prompt is left empty, rather than treating it as passwordless — and a locked account rejects every authentication method, key included. Fix going forward: always set a password at creation time, even when only key-based login is planned.

### FreeRADIUS silently ignored EAP entirely

A properly configured EAP setup rejected every authentication attempt, with only a barely noticeable log line indicating the EAP module was being ignored.

The **`freeradius-utils`** package only provides client-side testing tools; the actual EAP module ships in a separate **`freeradius-eap`** package that hadn't been installed. After adding it, a second issue surfaced: the module also requires working TLS submodules to load at all, even for EAP-MSCHAPv2 with no certificates involved, resolved using the bundled bootstrap script to generate test certificates.


### A misleading error hid a session-context problem, twice over

**`ssh-copy-id`** kept failing with confusing "file not found" errors that had nothing to do with the key itself. The root cause: commands were run as **`root`** in one terminal and as the intended admin user in another, so SSH was generating and looking for keys in two different home directories and the problem reappeared even after switching users, for the reason above (**`su`** vs. **`su -`**). Resolved by checking **`$HOME`** directly in each session instead of assuming it matched whichever user was currently active.

## VPN & Authentication

### FortiClient negotiates IKEv1 by default, not IKEv2

A tunnel configured on the firewall side for IKEv2 failed with a proposal mismatch, even though every visible parameter matched.

Firewall-side debug showed incoming IKEv1 Aggressive Mode packets, not IKEv2 at all. The IKE version turned out to be a separate, easy-to-miss field in FortiClient's advanced connection settings, defaulting to version 1. Setting it explicitly to 2 resolved the mismatch.

### Legacy Cisco IOS images rejected SSH from a modern OpenSSH client

SSH from the Alpine jump host to Router-HQ, Switch-HQ and AP-HQ failed
with a "no matching key exchange method found" error, even though SSHv2
was enabled on all three devices.

The IOS 15.x images only offer `diffie-hellman-group14-sha1` for key
exchange and `ssh-rsa` as host key algorithm, both of which current
OpenSSH (Alpine 3.21) disables by default. Re-enabling them explicitly
on the client side, for the Cisco hosts only, resolved it:

    ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
        -oHostKeyAlgorithms=+ssh-rsa admin@10.10.99.12

In production, the proper fix would be a newer IOS release with modern
algorithms rather than weakening the client.