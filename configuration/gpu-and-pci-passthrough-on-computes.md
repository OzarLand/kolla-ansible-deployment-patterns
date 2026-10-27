---
title: GPU and PCI passthrough with Kolla-Ansible. device_spec on the computes, the alias on the controllers too, and the scheduler filter
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, nova, placement, libvirt]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for platforms giving whole GPUs or other PCI devices to instances
  scale: any, with the devices on some computes
  network: any
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/pci-passthrough.html
  - https://docs.openstack.org/nova/2026.1/admin/virtual-gpu.html
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/advanced-configuration.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# GPU and PCI passthrough with Kolla-Ansible. device_spec on the computes, the alias on the controllers too, and the scheduler filter

## Context

PCI passthrough gives an instance a whole physical device: a GPU, an accelerator, a NIC used without Neutron. Nova
drives it with three settings (Nova's *Attaching physical PCI devices to guests* guide): `[pci] device_spec` on the
computes says which devices may be passed through, `[pci] alias` names them for flavors, and the
`PciPassthroughFilter` lets the scheduler place instances where a device is free. A flavor then asks for one or more
devices with `pci_passthrough:alias`.

Kolla-Ansible has no variable for any of this, except the SR-IOV whitelist it writes itself (see below): everything is
host preparation and `nova.conf` overrides. Two Kolla-specific points decide whether it works:

1. **The alias must be on the controllers too.** Nova's guide: the alias "must also be configured on controller
   nodes", for `nova-api`. An alias written only in `nova-compute.conf` makes the flavor's request unknown to the API.
2. **`PciPassthroughFilter` is not enabled.** Kolla-Ansible 2026.1 leaves `enabled_filters` at Nova's default, which
   does not contain it.

This page covers whole-device passthrough. Virtual GPUs (one physical GPU shared as several mediated devices) are a
different Nova feature with its own guide and the vendor's host driver; they are only pointed to here.

## Topology

No new role. The devices are on some computes; the scheduler and API on the controllers need the configuration too.

## Network design

Not affected, unless the device is a NIC; for NICs used through Neutron, see the
[SR-IOV page](sriov-neutron-ports-and-nova-pci-alias.md) instead.

## Storage design

Not affected.

## Inventory and configuration layout

### On each compute with devices, before Kolla-Ansible

From Nova's guide: VT-d (or AMD-Vi) enabled in the BIOS, IOMMU enabled on the kernel command line (`intel_iommu=on`
or `amd_iommu=on`), and assignable PCIe devices. For a GPU, the host must also not use the device itself: the host's
graphics driver must not hold it, so that libvirt can hand it to `vfio-pci` (with the default `managed` behaviour) or
find it already bound to `vfio-pci` (with `managed: "no"`, see below). How to keep the host driver off a device is
distribution and vendor specific; check the vendor's documentation.

> ⚠️ **State-changing.** Changes the kernel command line, applied at the next reboot, which stops every instance on the
> host unless they are moved first. Undo: remove the parameter, regenerate the boot configuration, reboot.

```bash
# /etc/default/grub: add intel_iommu=on (or amd_iommu=on) to GRUB_CMDLINE_LINUX, regenerate, reboot
```

Read-only checks after the reboot:

```bash
grep -o -E "(intel|amd)_iommu=on" /proc/cmdline
lspci -nn | grep -i -E "vga|3d|nvidia|amd"       # the device, its [vendor:product] IDs and its address
lspci -k -s <pci-address>                        # "Kernel driver in use": not the host's graphics driver
ls /sys/kernel/iommu_groups/*/devices/ | head     # IOMMU groups exist
```

### nova.conf overrides

The 2026.1 merge order for `nova.conf` in the `nova-cell` role is `config/global.conf`, `config/nova.conf`,
`config/nova/<service>.conf`, `config/nova/<inventory_hostname>/nova.conf`,
`config/nova/<inventory_hostname>/<service>.conf`; the `nova` role (API, scheduler) reads the same files except the
last. So:

```ini
# /etc/kolla/config/nova.conf: the alias, read by nova-api on the controllers and nova-compute
[pci]
alias = { "vendor_id": "<vendor-id>", "product_id": "<product-id>", "device_type": "type-PCI", "name": "gpu-a" }
```

```ini
# /etc/kolla/config/nova/nova-compute.conf: the devices, computes only (or per host under config/nova/<host>/)
[pci]
device_spec = { "vendor_id": "<vendor-id>", "product_id": "<product-id>" }
```

```ini
# /etc/kolla/config/nova/nova-scheduler.conf: one enabled_filters line for every filter you need
[filter_scheduler]
enabled_filters = ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,PciPassthroughFilter
```

From Nova's guide:

- `device_spec` can match by `address` (one device) or by `vendor_id` and `product_id` (every matching device).
- `device_type` in the alias: `type-PCI` for devices without SR-IOV (the default), `type-PF` or `type-VF` otherwise;
  "by default, it is only possible to attach `type-PCI` devices".
- `managed` in `device_spec` (since 2025.1): `"yes"` (default) lets libvirt detach the device from the host and
  re-attach it after; `"no"` means the host has already bound it to `vfio-pci` or a variant driver. The guide warns
  that a wrong value "may result in compute node crashes".

If you already have a scheduler override, for `NUMATopologyFilter`
([CPU pinning](cpu-pinning-hugepages-and-numa-on-computes.md)) or for SR-IOV, add the filter to that line: there is
only one `enabled_filters`.

### When SR-IOV is enabled too

With `enable_neutron_sriov`, Kolla-Ansible writes `[pci] passthrough_whitelist` on the computes from
`neutron_sriov_physnet_mappings`, with `devname` entries (2026.1 `nova-cell` defaults). `passthrough_whitelist` is the
older name of `device_spec`: a `device_spec` override for GPUs and that list are two values for one option, and which
one Nova keeps was not tested here. Put every device in one list instead, by overriding `nova_pci_passthrough_whitelist`
in `globals.yml`.

### PCI tracking in Placement

Nova can track PCI devices in Placement (`[pci] report_in_placement`, off by default; then
`[filter_scheduler] pci_in_placement` for scheduling). Nova's guide warns that once enabled on a compute "it cannot be
disabled there any more", that with it nova-compute "will refuse to start" if `device_spec` uses `devname`, and that
SR-IOV devices for Neutron ports (with `physical_network`) are not tracked this way. Kolla-Ansible's SR-IOV whitelist
uses `devname`, so do not enable `report_in_placement` on computes that carry it without replacing those entries by
PCI addresses. Read the guide's whole section before enabling it.

### Flavors

```bash
openstack flavor set <gpu-flavor> --property "pci_passthrough:alias"="gpu-a:1"   # one device per instance
```

## Applying it

> ⚠️ **State-changing.** Restarts `nova_compute` on the computes in the limit, and `nova_api` and `nova_scheduler` on
> the controllers, where their configuration changed. Running instances keep running; Nova API calls fail or wait
> during the restarts. Undo: remove the overrides and reconfigure again; instances that hold a device keep it until
> they are deleted or moved.

```bash
kolla-ansible genconfig -i <inventory> --tags nova --limit control,<gpu-computes>   # optional: render first
kolla-ansible reconfigure -i <inventory> --tags nova --limit control,<gpu-computes>
```

Verify (read-only, then one instance):

```bash
sudo grep -A3 "^\[pci\]" /etc/kolla/nova-compute/nova.conf      # on a GPU compute: device_spec and alias
sudo grep -A3 "^\[pci\]" /etc/kolla/nova-api/nova.conf          # on a controller: the same alias
```

> ⚠️ **State-changing.** Boots, then deletes, one instance holding a device. Undo: `openstack server delete`.

```bash
openstack server create --flavor <gpu-flavor> --image <image> --network <network> gpu-test
openstack server show gpu-test -c status -c OS-EXT-SRV-ATTR:host
sudo docker exec nova_libvirt virsh dumpxml <domain> | grep -A4 "<hostdev"   # on that compute: the device
```

Inside the instance, `lspci` shows the device; the guest needs the vendor's driver for a GPU.

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| Flavor rejected, or the alias is unknown | `/var/log/kolla/nova/nova-api.log` on the controllers: the alias is missing from the API's `nova.conf` |
| `No valid host was found` | `/var/log/kolla/nova/nova-scheduler.log`: `PciPassthroughFilter` missing, or no compute with a free matching device |
| `nova_compute` does not start after the change | `docker logs nova_compute` (or `podman logs`) and `/var/log/kolla/nova/nova-compute.log`: invalid `device_spec` syntax, PF and VFs both listed, `devname` with Placement tracking |
| Instance fails to start on the compute | `/var/log/kolla/nova/nova-compute.log` and `/var/log/kolla/libvirt/libvirtd.log`: IOMMU off, device held by a host driver, IOMMU group shared with another device |
| Device visible in the guest but unusable | the guest's driver and the vendor's requirements; not an OpenStack problem |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Whole-device passthrough | Virtual GPUs (mediated devices) | No vendor licensing for sharing; full device | One instance per device; no sharing |
| Alias in `config/nova.conf` | Alias in `nova-compute.conf` only | Nova's guide: the API needs it too | The alias is the same everywhere |
| `device_spec` by vendor and product ID | By PCI address | Fewer lines with identical hosts | Every matching device is given away, including one the host may need |
| `managed` left at its default | `managed: "no"` with devices pre-bound to `vfio-pci` | Simplest | libvirt detaches and re-attaches the device at each start and stop |
| Placement tracking off (default) | `report_in_placement` | Cannot be disabled once on; incompatible with Kolla's SR-IOV `devname` entries | Scheduling uses the legacy PCI tracker |

## What went wrong during deployment

Nothing observed yet: written from Nova's PCI passthrough guide and the Kolla-Ansible 2026.1 roles.

## Version notes

- 2026.1: Kolla-Ansible sets `enabled_filters` only with Blazar or the fake driver, so `PciPassthroughFilter` must be
  added; with Blazar, keep `BlazarFilter` in the line.
- Nova 2025.1 added the `managed` and `live_migratable` tags; earlier releases do not accept them. Instances with
  passthrough devices cannot be live-migrated unless the device and its configuration support it (see the guide).

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- Put the alias where both the API and the computes read it.
- Prepare the host (IOMMU, no host driver on the device) before touching Nova.
- Keep one PCI device list when SR-IOV is enabled.

## Validity and limits

Written from Nova 2026.1 (*Attaching physical PCI devices to guests*, *Attaching virtual GPU devices to guests*) and
Kolla-Ansible 2026.1 (advanced configuration guide, the `nova` and `nova-cell` roles). Not run on a lab: no output on
this page was observed, and it needs passthrough-capable hardware. Not covered: virtual GPUs, vendor drivers and
licensing, PCI NUMA affinity policies in detail, live migration of instances with devices, and PCI tracking in
Placement beyond its warnings.

## Related

- [SR-IOV with Kolla-Ansible](sriov-neutron-ports-and-nova-pci-alias.md): the PCI list Kolla-Ansible writes itself.
- [CPU pinning, huge pages and NUMA](cpu-pinning-hugepages-and-numa-on-computes.md): the other scheduler filter and
  the override mechanics.
- Nova 2026.1: [PCI passthrough](https://docs.openstack.org/nova/2026.1/admin/pci-passthrough.html),
  [virtual GPUs](https://docs.openstack.org/nova/2026.1/admin/virtual-gpu.html)

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and the
Kolla-Ansible 2026.1 code. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=configuration/gpu-and-pci-passthrough-on-computes.md&title=Feedback%3A+configuration/gpu-and-pci-passthrough-on-computes.md):
your release, your topology, and what happened.
