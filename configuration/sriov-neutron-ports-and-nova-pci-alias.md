---
title: SR-IOV with Kolla-Ansible. VFs on the host, the Neutron SR-IOV agent, the PCI whitelist Kolla-Ansible writes, and the scheduler filter
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, neutron, nova, libvirt]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for platforms giving instances near line-rate networking through virtual functions
  scale: any, with SR-IOV capable NICs on some or all computes
  network: ml2/ovs or ml2/ovn, SR-IOV ports on flat or VLAN provider networks
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/sriov.html
  - https://docs.openstack.org/neutron/2026.1/admin/config-sriov.html
  - https://docs.openstack.org/nova/2026.1/admin/pci-passthrough.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# SR-IOV with Kolla-Ansible. VFs on the host, the Neutron SR-IOV agent, the PCI whitelist Kolla-Ansible writes, and the scheduler filter

## Context

SR-IOV splits a physical NIC (the PF) into virtual functions (VFs) that can be passed directly to instances: the
instance's traffic bypasses Open vSwitch or OVN on the compute, which gives near line-rate throughput and low latency,
at the cost of features (security groups, easy live migration). Kolla-Ansible supports two ways to use VFs (SR-IOV
guide):

- **Neutron SR-IOV:** `enable_neutron_sriov`, a `neutron_sriov_agent` on the computes, and ports with
  `--vnic-type direct` on a provider network. Neutron manages addresses and VLANs.
- **Nova PCI alias:** a `[pci] alias` and a flavor that requests the device. No Neutron involvement; the guide warns
  that the alias matches vendor and product IDs only, so with several SR-IOV ports of the same model "it is impossible
  to specify a specific NIC port to pull VFs from".

This page is mostly about the first, which is what most operators mean by SR-IOV, and points out where Kolla-Ansible
does part of the work and where it does not: the VFs themselves are created on the host, outside Kolla-Ansible.

## Topology

The `neutron_sriov_agent` container runs on the hosts of the `compute` group when `enable_neutron_sriov` is true
(2026.1 role). Nothing changes on the controllers except Neutron's ML2 configuration and Nova's scheduler filters.

## Network design

SR-IOV ports live on **provider networks**: the physical network name (for example `sriovtenant1`) is mapped to the
PF on each compute, and VLANs on it must be in the ML2 ranges. The PF's switch port must carry those VLANs. See
[provider networks with Kolla-Ansible](../networking/provider-networks-instances-on-flat-or-vlan-networks.md).

From Neutron's SR-IOV guide, *Known limitations*: security groups are not supported on SR-IOV ports, and live
migration of instances with `direct` ports has restrictions (see the guide for the current state).

## Storage design

Not affected.

## Inventory and configuration layout

### On each SR-IOV compute, before Kolla-Ansible

From Neutron's SR-IOV guide: SR-IOV and VT-d enabled in the BIOS, IOMMU enabled on the kernel command line
(`intel_iommu=on` on Intel systems), and VFs created through sysfs.

> ⚠️ **State-changing.** The kernel parameter needs a reboot, which stops the instances on the host unless they are
> moved first. Creating VFs changes the PF's state; on some NICs changing the number again fails with `Device or
> resource busy` until it is set to `0` first (Neutron guide), which removes the existing VFs and cuts any instance
> using them. Undo: remove the kernel parameter and reboot; `echo 0` to `sriov_numvfs`.

```bash
# /etc/default/grub: add intel_iommu=on (and iommu=pt if your platform guide recommends it) to GRUB_CMDLINE_LINUX,
# regenerate the boot configuration, reboot
echo 8 | sudo tee /sys/class/net/<pf-interface>/device/sriov_numvfs
```

The sysfs value does not survive a reboot: make it persistent with the host's network configuration (the Neutron
guide gives `pre-up` and `ifup-local` examples) or a udev rule or systemd unit. Read-only checks:

```bash
cat /proc/cmdline | grep -o "intel_iommu=on"
cat /sys/class/net/<pf-interface>/device/sriov_numvfs
ip -d link show <pf-interface>            # one "vf N" line per VF
lspci | grep -i "virtual function"
```

### In Kolla-Ansible

```yaml
# globals.yml
enable_neutron_sriov: "yes"
neutron_sriov_physnet_mappings:
  sriovtenant1: <pf-interface>            # physical network name : PF interface on the computes
```

```ini
# /etc/kolla/config/neutron/ml2_conf.ini
[ml2_type_vlan]
network_vlan_ranges = sriovtenant1:<first-vlan>:<last-vlan>

[ml2_type_flat]
flat_networks = sriovtenant1
```

What `enable_neutron_sriov` does in 2026.1:

- adds `sriovnicswitch` to the ML2 mechanism drivers (guide);
- deploys `neutron_sriov_agent` on the computes, with `physical_device_mappings` from `neutron_sriov_physnet_mappings`;
- writes **`[pci] passthrough_whitelist`** in the computes' `nova.conf`, built from the same mappings
  (`nova_pci_passthrough_whitelist` in the `nova-cell` role): one entry per PF interface with its physical network.

That last point matters if you also follow the guide's Nova PCI alias section, which writes `[pci] device_spec` in an
override: `passthrough_whitelist` is the older name of `device_spec` in Nova. Two lists under two names for the same
option is a trap; which one Nova keeps was not tested here. If you need extra devices, override
`nova_pci_passthrough_whitelist` in `globals.yml` with the full list instead.

### The scheduler filter

The guide adds `PciPassthroughFilter` to `enabled_filters`. Kolla-Ansible 2026.1 leaves `enabled_filters` at Nova's
default, which does not contain it, so this override is required:

```ini
# /etc/kolla/config/nova/nova-scheduler.conf
[filter_scheduler]
enabled_filters = ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,PciPassthroughFilter
```

The guide writes it in `/etc/kolla/config/nova.conf`; the scheduler-specific file above has the same effect for the
scheduler and wins over `nova.conf` in Kolla-Ansible's merge order. If you already have a scheduler override, for
example for `NUMATopologyFilter` ([CPU pinning, huge pages and NUMA](cpu-pinning-hugepages-and-numa-on-computes.md)),
add the filter to that one line: there is only one `enabled_filters`.

## Applying it

> ⚠️ **State-changing.** Reconfigures Neutron (ML2 drivers; `neutron-server` restarts), deploys the SR-IOV agent on
> every compute, and changes `nova.conf` on the computes and the scheduler, whose containers restart. Running
> instances keep running; API calls to Neutron and Nova may fail during the restarts. Undo: `enable_neutron_sriov:
> "no"`, remove the overrides and reconfigure; the `neutron_sriov_agent` containers stay until removed by hand.

```bash
kolla-ansible reconfigure -i <inventory> --tags neutron,nova
```

Verify, read-only, then with one instance (the guide's sequence):

```bash
openstack network agent list --agent-type nic                 # one "NIC Switch agent" per compute, alive
sudo grep -A2 "^\[pci\]" /etc/kolla/nova-compute/nova.conf    # on a compute: the whitelist with the PF and physnet
```

> ⚠️ **State-changing.** Creates a provider network, a port and an instance in a test project. Provider networks are
> an administrator action.

```bash
openstack network create --provider-network-type vlan --provider-physical-network sriovtenant1 \
  --provider-segment <vlan-id> sriov-test
openstack subnet create --network sriov-test --subnet-range 198.51.100.0/24 sriov-test-subnet
openstack port create --network sriov-test --vnic-type direct sriov-test-port
openstack server create --flavor <flavor> --image <image> --port sriov-test-port sriov-test-vm
```

On the compute where it landed, `ip -d link show <pf-interface>` shows the instance's MAC and VLAN on one VF, and
the kernel log shows the VF being configured (the guide shows `dmesg` output of this kind).

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| `No valid host was found` for an SR-IOV port | `/var/log/kolla/nova/nova-scheduler.log`: `PciPassthroughFilter` missing, or no host with free VFs on that physical network |
| Port binding fails (`binding_failed`) | `/var/log/kolla/neutron/neutron-server.log` and the SR-IOV agent's log under `/var/log/kolla/neutron/` on the compute (its binary is `neutron-sriov-nic-agent`): physical network not mapped, agent down |
| Agent not alive | `docker logs neutron_sriov_agent` (or `podman logs`) on the compute: the PF interface name in `neutron_sriov_physnet_mappings` does not exist there |
| Instance fails to start on the compute | `/var/log/kolla/nova/nova-compute.log` and `/var/log/kolla/libvirt/libvirtd.log`: IOMMU off, no VFs, VF already in use |
| No traffic in the instance | the switch port: VLAN not trunked to the PF; the instance's driver for the VF |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Neutron SR-IOV ports | Nova PCI alias | Neutron manages addresses and VLANs; a chosen physical network per port | Ports must be created with `--vnic-type direct` before boot |
| VFs created by the host's configuration | Kernel module `max_vfs` | The Neutron guide calls `max_vfs` deprecated | One more host setting to keep across reboots and NIC replacements |
| Only some computes with SR-IOV NICs | All computes | Hardware cost | The PF interface name in `neutron_sriov_physnet_mappings` must exist on every host where the agent runs; set it per host in `host_vars` if names differ |
| `nova_pci_passthrough_whitelist` overridden for extra devices | A second list in a `device_spec` override | One list, one option name | Keep the SR-IOV entries in it, or Neutron ports lose their devices |

## What went wrong during deployment

Nothing observed yet: written from the Kolla-Ansible SR-IOV guide, the 2026.1 `neutron` and `nova-cell` roles, and the
Neutron and Nova guides.

## Version notes

- 2026.1: Kolla-Ansible writes the whitelist as `passthrough_whitelist`, the older name of Nova's `[pci] device_spec`.
- OVS hardware offload (`openvswitch_hw_offload: "yes"`) is mentioned by the guide as a separate option; not covered
  here.

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- Prepare the hosts first (BIOS, IOMMU, persistent VFs); Kolla-Ansible only configures OpenStack on top.
- Add `PciPassthroughFilter` explicitly: Kolla-Ansible does not.
- Keep one PCI device list: the one Kolla-Ansible builds, extended if needed.

## Validity and limits

Written from Kolla-Ansible 2026.1 (SR-IOV guide, the `neutron` role defaults, the `nova-cell` role defaults and
`nova.conf.j2`), the Neutron SR-IOV guide and the Nova PCI passthrough guide. Not run on a lab: no output on this page
was observed, and it needs SR-IOV capable hardware. Not covered: OVS hardware offload, `macvtap` and
`virtio-forwarder` ports, SR-IOV with NUMA affinity policies, InfiniBand, and live migration of SR-IOV instances.

## Related

- [CPU pinning, huge pages and NUMA on computes](cpu-pinning-hugepages-and-numa-on-computes.md): the other scheduler
  filter override, to merge with this one.
- [Provider networks with Kolla-Ansible](../networking/provider-networks-instances-on-flat-or-vlan-networks.md)
- Kolla-Ansible 2026.1: [SR-IOV](https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/sriov.html)
- Neutron 2026.1: [SR-IOV](https://docs.openstack.org/neutron/2026.1/admin/config-sriov.html);
  Nova 2026.1: [PCI passthrough](https://docs.openstack.org/nova/2026.1/admin/pci-passthrough.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible, Neutron and Nova 2026.1 documentation and the
Kolla-Ansible 2026.1 code. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=configuration/sriov-neutron-ports-and-nova-pci-alias.md&title=Feedback%3A+configuration/sriov-neutron-ports-and-nova-pci-alias.md):
your release, your topology, and what happened.
