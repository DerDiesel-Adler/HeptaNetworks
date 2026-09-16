*AI-Generated

# Integrating Node Images into EVE-NG

How each image used in this lab was uploaded and made available in EVE-NG,
including the platform-specific issues encountered along the way.

## General upload process

All images are uploaded via WinSCP (or any SCP/SFTP client) to:

```
/opt/unetlab/addons/qemu/<image-folder-name>/
```

After **every** upload of new image files, permissions have to be fixed
manually — newly uploaded files are owned by `root`, which EVE-NG's image
loader cannot work with:

```
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

Skipping this step is the most common reason a freshly uploaded image
doesn't show up (or fails to boot) in the EVE-NG GUI.

## Cisco IOS images

**Layer 3 / Router:** `vios-adventerprisek9-m.vmdk.SPA.156-1.T.bin`
**Layer 2 / Switch & AP:** `viosl2-adventerprisek9-m.ssa.high_iron_20200929.tgz`

Two platform quirks worth noting for this specific L2 image:

- It requires at least **512 MB RAM** — with less, the node continuously
  reboots on its own instead of reporting an out-of-memory error, which
  makes the actual cause easy to miss on first boot.
- The router image (vios, L3) has **no Etherchannel / `channel-group`
  support** — relevant if planning any link aggregation or redundancy
  (see [Future Work](../README.md#things-that-might-be-implemented-in-the-future)
  in the main README).

## FortiGate

**Image:** `FGT_VM64_KVM-v7.4.12.M-build2902-FORTINET.out.kvm`

On first boot, the node produced a cascading series of `fork() failed`
errors instead of starting normally. This turned out to be a resource
problem, not a configuration one — the default node allocation (1 vCPU /
~2 GB RAM) was insufficient for FortiOS to spawn its internal processes.
Resolved by increasing the node to **2 vCPUs and 4096 MB RAM** in the node
properties before the first boot.

## Alpine Linux nodes

**Image:** Alpine Linux 3.21 (`linux-alpine-3.21.3`)

Used for all seven Linux nodes in the topology (Jump Host, Staging Server,
Backup Server, RADIUS server, Syslog server, Admin-HQ, PC-HQ-MA1).

After creating each new node from this image, before first real use:

1. **Wipe the node once** before its first boot, to avoid leftover state
   from the base image being shared unexpectedly across nodes cloned from
   the same source.
2. Set a unique hostname immediately, so nodes don't overwrite each
   other's identity in shared filesystem state:
   ```
   echo "<hostname>" > /etc/hostname
   hostname -F /etc/hostname
   ```
3. If SSH access is needed before a password/key is fully set up, allow
   root login temporarily:
   ```
   passwd
   sed -i 's/#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
   service sshd restart
   ```

Full step-by-step configuration per node (networking, users, installed
services) is covered separately in
[`all-alpine-nodes.md`](all-alpine-nodes.md).

## Windows VPN client

Getting a working Windows image running inside EVE-NG turned out to be one
of the most time-consuming parts of the whole project — two different
Windows versions and several rounds of QEMU configuration changes were
needed before it actually worked.

### First attempt: Windows 11 (Tiny11) — abandoned

A Tiny11 image failed Windows Setup's hardware compatibility check ("This
PC can't run Windows 11"), even after bypassing it via the documented
registry trick (`HKLM\SYSTEM\Setup\LabConfig`, with `BypassTPMCheck`,
`BypassSecureBootCheck`, `BypassRAMCheck`, `BypassStorageCheck`, and
`BypassCPUCheck` all set to `1`). Past that point, the VM blue-screened
with `MULTIPROCESSOR_CONFIGURATION_NOT_SUPPORTED`, and after reducing to a
single vCPU, with `IRQL_NOT_LESS_OR_EQUAL` instead. Rather than keep
chasing compatibility issues with a stripped-down Windows 11 image, the
approach was switched to Windows 10 instead — a version without any
TPM/Secure Boot enforcement at setup time.

### Working setup: Windows 10 (Tiny10)

**Image:** `tiny-10-23-h2`

**Folder naming:** EVE-NG's Windows template only recognizes image folders
matching a `win-<version>-...` naming pattern in its GUI dropdown — a
folder named `win10-tiny` (no hyphen after `win`) was silently not listed,
while `win-10-tiny` was.

**Disk creation:**
```
cd /opt/unetlab/addons/qemu/win-10-tiny/
qemu-img create -f qcow2 hda.qcow2 40G
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

**Disk naming matters more than it looks like it should:** EVE-NG binds
disks to a specific virtual bus based on the filename prefix —
`virtioa.qcow2` forces a VirtIO bus regardless of the `-machine` type
configured, and Windows has no built-in driver for it, resulting in an
unresolvable "no driver found" prompt during setup. Naming the file
`hda.qcow2` instead selects IDE emulation, which Windows recognizes
natively with zero extra driver steps.

**Working QEMU custom options:**
```
-machine type=pc,accel=kvm -cpu qemu64 -vga std -usbdevice tablet -boot order=cd -cdrom /opt/unetlab/addons/qemu/win-10-tiny/win-10-tiny.iso
```

Two settings here were the direct result of trial and error, not the
default template:
- **`-cpu qemu64` instead of `-cpu host`** — the `+fsgsbase` CPU flag that
  comes with `-cpu host` caused the `IRQL_NOT_LESS_OR_EQUAL` blue screen on
  this image; removing it (by switching to the more generic `qemu64` CPU
  model) resolved it.
- **Node CPU count set to 1** — this particular Tiny10 build does not
  support multiple vCPUs; anything higher reproduces the
  `MULTIPROCESSOR_CONFIGURATION_NOT_SUPPORTED` blue screen seen with the
  Windows 11 attempt as well.

**Node settings:** 1 vCPU, 2048–4096 MB RAM, 1 Ethernet interface, console
set to `vnc` (required for a GUI operating system — the default `telnet`
console used for Cisco/Alpine nodes won't work here).
