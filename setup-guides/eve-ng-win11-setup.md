*AI-Generated

# EVE-NG on Windows 11 — Host Setup & Nested Virtualization

**Host OS:** Windows 11 25H2 26.200 9445
**Hypervisor:** VMware Workstation Pro 26.0.0.25388281
**EVE-NG:** 6.2.0-4

## The problem

EVE-NG relies on nested virtualization — running QEMU with KVM hardware
acceleration *inside* a VM that is itself running on the Windows host. This
requires VMware Workstation to pass CPU virtualization extensions (VT-x)
through to the guest, on a system where Windows 11 itself wants to use those
same extensions for its own security features (Hyper-V, Virtualization-Based
Security, Windows Hello).

Without nested virtualization working correctly, EVE-NG can still boot
nodes, but without `/dev/kvm` acceleration — meaning heavier images
(FortiGate, larger Cisco images) become extremely slow or effectively
unusable.

## What actually happened

Getting this to work reliably took most of a day, including two situations
where Windows itself became unbootable (Windows Hello sign-in broken) and
had to be recovered. Several combinations of enabling/disabling Windows
features (Hyper-V, Virtual Machine Platform, Windows Sandbox) were tried in
between — the exact sequence that didn't work isn't fully reconstructable
afterward, since troubleshooting under time pressure wasn't documented step
by step at the time.

The setting that ultimately resolved it, run from an elevated Command
Prompt:

```
bcdedit /set hypervisorlaunchtype off
```

This disables the Windows-native hypervisor at boot, freeing up VT-x for
VMware Workstation to pass through to the EVE-NG VM. After a full reboot,
nested virtualization inside VMware Workstation could be enabled for the
EVE-NG VM (VM Settings → Processors → **Virtualize Intel VT-x/EPT or
AMD-V/RVI**), and `/dev/kvm` became available inside EVE-NG.

## Trade-off, and why this wasn't investigated further

Turning off the Windows hypervisor at the bootloader level is a blunt
instrument — it also disables Windows security features that build on
virtualization (Credential Guard, some Windows Hello protections). For a
dedicated lab machine used primarily for this project, that trade-off was
acceptable. It would **not** be an appropriate fix on a daily-use or
work-managed device.

Since the current setup works reliably, the underlying cause (which
specific Windows feature was actually in conflict) was not narrowed down
further — this is a working fix, not a fully understood one, and is
documented as such rather than presenting a cleaner story than what
actually happened.


