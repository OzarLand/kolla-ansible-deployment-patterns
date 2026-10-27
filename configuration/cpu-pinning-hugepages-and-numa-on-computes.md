---
title: CPU pinning, huge pages and NUMA on Kolla-Ansible computes. Host preparation, nova.conf overrides and the scheduler filter Kolla-Ansible does not enable
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, nova, placement, libvirt, qemu]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for platforms that offer pinned or huge-page flavors next to ordinary ones
  scale: any, with libvirt/KVM computes
  network: any
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0 all-in-one (4 vCPUs, 1 NUMA node, no SMT), ML2/OVS", date: 2026-10-27, note: "huge pages at boot, the inventory group and both overrides, genconfig, reconfigure --tags nova and one instance per flavor (pinned, huge pages, hw:numa_nodes=1) checked with Placement and virsh; the scheduler was unavailable for 2 min 44 s after the reconfigure; several NUMA nodes, SMT siblings, an unpinned compute, the failure symptoms and the undo not run" }
upstream_refs:
  - https://docs.openstack.org/nova/2026.1/admin/cpu-topologies.html
  - https://docs.openstack.org/nova/2026.1/admin/huge-pages.html
  - https://docs.openstack.org/nova/2026.1/admin/scheduling.html
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/advanced-configuration.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# CPU pinning, huge pages and NUMA on Kolla-Ansible computes. Host preparation, nova.conf overrides and the scheduler filter Kolla-Ansible does not enable

## Context

Network functions, real-time and latency-sensitive workloads ask for instances whose vCPUs are pinned to host CPUs,
whose memory comes from huge pages, and whose CPUs and memory sit on the same NUMA node. Nova does all three on
libvirt/KVM, driven by flavor extra specs. Kolla-Ansible has no variable for any of it: everything goes through the
host's kernel command line and `nova.conf` overrides under `/etc/kolla/config/`.

Three things are easy to miss with Kolla-Ansible 2026.1, and this page is organised around them:

1. **The `NUMATopologyFilter` is not enabled.** Kolla-Ansible leaves `[filter_scheduler] enabled_filters` at Nova's
   default (it sets it only with Blazar or the fake driver), and Nova's 2026.1 default list does not contain that
   filter. The Nova guide is explicit: "In all cases where NUMA awareness is used, the `NUMATopologyFilter` filter
   must be enabled."
2. **Pinning needs `cpu_dedicated_set`** on each compute. Since Train, a compute reports all its CPUs as shared
   `VCPU` unless told which ones are dedicated `PCPU`.
3. **Huge pages are a host setting** (kernel command line, reboot), outside Kolla-Ansible.

## Topology

No new role. The computes that host pinned or huge-page instances can be all of them or a subset; when they are a
subset, give them their own inventory group so the overrides can follow the group (see below). Nova does not require
separate aggregates for pinned and unpinned instances since Train, because `PCPU` and `VCPU` are separate resources
in Placement; many operators still use an aggregate to keep the two kinds of workload apart.

## Network design

Not affected. With SR-IOV or PCI devices, NUMA matters for them too: the Nova guide says "Instances that use PCI or
SR-IOV devices should be placed on the NUMA node associated with these devices."

## Storage design

Not affected.

## Inventory and configuration layout

```text
inventory/
  multinode                       # [compute-pinned] group, child of [compute], if only some computes
  group_vars/compute-pinned.yml   # nova_cpu_dedicated_set, nova_cpu_shared_set, nova_reserved_host_memory_mb
  host_vars/compute-07.yml        # only for a host whose CPU layout differs
/etc/kolla/config/
  nova/nova-scheduler.conf        # enabled_filters with NUMATopologyFilter
  nova/nova-compute.conf          # cpu_dedicated_set = {{ nova_cpu_dedicated_set }} ...
```

The `nova_cpu_*` names above are your own inventory variables, not Kolla-Ansible variables. Kolla-Ansible renders the
override files as Jinja templates before merging them, and the advanced configuration guide shows this exact use: a
variable in the override, its value in group or host vars. For `nova.conf`, the 2026.1 `nova-cell` role merges, in
this order (later wins): `config/global.conf`, `config/nova.conf`, `config/nova/<service>.conf`,
`config/nova/<inventory_hostname>/nova.conf`, `config/nova/<inventory_hostname>/<service>.conf`. There is no
per-group directory: per-group values go through inventory variables.

### Huge pages on the host

The Nova guide allocates persistent huge pages at boot, which "will ensure the correct number of huge pages is always
available, while allocating them at run time can fail if memory has become too fragmented". The kernel parameters are
the guide's; the count is yours.

> ⚠️ **State-changing.** Changes the kernel command line; it takes effect at the next reboot, which stops every instance
> on the host unless you move them first. The memory given to persistent huge pages is no longer usable by host
> processes or by instances without huge pages. Undo: remove the parameters, regenerate the boot configuration,
> reboot.

```bash
# /etc/default/grub, appended to GRUB_CMDLINE_LINUX (example: 2048 pages of 2 MB = 4 GB)
#   hugepagesz=2M hugepages=2048 transparent_hugepage=never
sudo update-grub                     # Debian, Ubuntu; on Rocky: sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

After the reboot, read-only checks:

```bash
grep Huge /proc/meminfo                  # HugePages_Total as requested
mount | grep huge                        # hugetlbfs on /dev/hugepages
cat /sys/devices/system/node/node*/hugepages/hugepages-2048kB/nr_hugepages   # split across NUMA nodes
lscpu | grep -i -E "numa|thread"         # NUMA nodes, their CPU lists, threads per core
```

The `nova_libvirt` and `nova_compute` containers bind-mount the host's `/dev` in 2026.1 (role defaults), so a
hugetlbfs mounted on the host at `/dev/hugepages` is visible inside them. The guide warns: "Persistent huge pages are
not usable by standard host OS processes. Ensure enough free, non-huge page memory is reserved for these processes."

### nova.conf overrides

For the scheduler, on the controllers:

```ini
# /etc/kolla/config/nova/nova-scheduler.conf
[filter_scheduler]
enabled_filters = ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ServerGroupAntiAffinityFilter,ServerGroupAffinityFilter,NUMATopologyFilter
```

The first five are Nova's 2026.1 default; writing the option replaces the whole list, so keep them, and add any other
filter you rely on (for example `PciPassthroughFilter` with PCI passthrough or SR-IOV, `AggregateInstanceExtraSpecsFilter`
for aggregates matched by extra specs). With `enable_blazar`, Kolla-Ansible's own list adds `BlazarFilter`; this
override would drop it, so add `BlazarFilter` to the end of the `enabled_filters` line above. Do not copy Kolla-Ansible's
`available_filters` lines into the override: Kolla-Ansible still writes them, as the override does not set them.

For the computes, the Nova guide's example for a host with 24 cores and hyper-threading (48 logical CPUs): one core
and its sibling for the host, 8 cores and their siblings for pinned instances, the rest shared:

```ini
# /etc/kolla/config/nova/nova-compute.conf
{% if nova_cpu_dedicated_set is defined %}
[DEFAULT]
reserved_host_memory_mb = {{ nova_reserved_host_memory_mb }}

[compute]
cpu_dedicated_set = {{ nova_cpu_dedicated_set }}
cpu_shared_set = {{ nova_cpu_shared_set }}
{% endif %}
```

```yaml
# inventory/group_vars/compute-pinned.yml (the guide's CPU lists; check yours with lscpu)
nova_cpu_dedicated_set: "2-17"
nova_cpu_shared_set: "18-47"
nova_reserved_host_memory_mb: 8192
```

`reserved_host_memory_mb` is inside the guard on purpose: the computes outside `compute-pinned` keep Nova's default of
512 MB. To reserve a different amount on every compute, put that line before the `{% if %}`, in its own `[DEFAULT]`
section, with a variable defined for all computes.

Every compute gets this file. The `{% if %}` wrapper leaves it empty on the computes outside `compute-pinned`, where
the variables are undefined; without it the template fails there. Do not give the variables empty defaults instead:
that would render `cpu_dedicated_set = ` on unpinned hosts. The CPU numbers are the host's logical CPU IDs: with hyper-threading,
check which IDs are siblings (`lscpu -e`) so that a core and its sibling land in the same set.

### Flavors

From the Nova guides, each with its own flavor so users choose:

```bash
openstack flavor set <pinned-flavor>   --property hw:cpu_policy=dedicated     # vCPUs consume PCPU
openstack flavor set <hugepage-flavor> --property hw:mem_page_size=large      # or 2MB, 1GB
openstack flavor set <numa-flavor>     --property hw:numa_nodes=1             # one guest NUMA node, on one host node
```

Pinning or huge pages make an instance NUMA-aware implicitly (Nova guide). `hw:cpu_thread_policy` and
`hw:emulator_threads_policy` refine pinning on hyper-threaded hosts; the Nova guide describes each value.

## Applying it

**1. Prepare and reboot the hosts** (huge pages above), one at a time, emptied of instances first. Planning the
moves is the subject of a maintenance plan, not of this page.

**2. Apply the overrides.** Read-only first: render the configuration and compare.

```bash
kolla-ansible genconfig -i <inventory> --tags nova --limit compute-07
sudo grep -A3 "^\[compute\]" /etc/kolla/nova-compute/nova.conf     # on compute-07: the sets as intended
```

`genconfig` writes the files under `/etc/kolla/` on the hosts without restarting containers; the next `reconfigure`
applies and restarts.

> ⚠️ **State-changing.** Restarts `nova_compute` on the hosts in the limit and `nova_scheduler` on the controllers.
> Running instances keep running; API calls that need the scheduler fail or wait during the restart. On the lab
> this lasted minutes, not seconds: after the restart handler, the task *Refresh cell cache in nova scheduler* sends
> `nova_scheduler` a reload, its workers stopped gracefully and the new ones started only 2 min 44 s later; instances
> created in that window went to `ERROR` with `MessagingTimeout`. Wait for `Starting scheduler node` in
> `nova-scheduler.log` before creating instances. Change the CPU sets on a host with no instances: instances already placed keep the CPUs they had, and a
> new layout that no longer covers them was not tested here. Undo: remove the overrides and reconfigure again.

```bash
kolla-ansible reconfigure -i <inventory> --tags nova --limit control,compute-pinned
```

**3. Verify** (read-only, as admin, the Placement commands need the `osc-placement` plugin):

```bash
openstack resource provider list --name compute-07
openstack resource provider inventory list <provider-uuid>     # PCPU total 16 and VCPU total 30 with the guide's sets
sudo grep -i "dedicated\|shared\|error" /var/log/kolla/nova/nova-compute.log | tail -20   # on compute-07
```

On a 4-CPU lab compute with `cpu_dedicated_set = 2-3`, `cpu_shared_set = 1` and `reserved_host_memory_mb = 1024`,
the inventory went from `VCPU 4` to (resource class, allocation ratio, reserved, total):

```text
VCPU 4.0 0 1
MEMORY_MB 1.0 1024 15992
DISK_GB 1.0 0 47
PCPU 1.0 0 2
```

The `grep` showed no line about the sets after the restart: a quiet log there is normal, errors are what to look for.

Then boot one instance with each flavor and check on the compute where it landed:

```bash
sudo docker exec nova_libvirt virsh list                       # or: sudo podman exec ...
sudo docker exec nova_libvirt virsh vcpupin <domain>           # each vCPU on one host CPU from cpu_dedicated_set
sudo docker exec nova_libvirt virsh dumpxml <domain> | grep -A3 -E "hugepages|numatune"
```

On the lab, the `hw:cpu_policy=dedicated` instance had its vCPU, emulator and I/O thread on host CPU 2 (from
`cpu_dedicated_set`); the `hw:mem_page_size=large` and `hw:numa_nodes=1` instances were pinned to CPU 1, the
`cpu_shared_set`, and all three had `<numatune>` with `mode='strict'`; the huge-page one also had:

```text
    <hugepages>
      <page size='2048' unit='KiB' nodeset='0'/>
    </hugepages>
```

`HugePages_Free` drops as the guest touches its memory: 83 of the 128 pages of a 256 MB instance were in use a minute
after it booted.

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| `No valid host was found` for a pinned flavor | `/var/log/kolla/nova/nova-scheduler.log` on the controllers: which filter or Placement query returned no host; no `PCPU` inventory means `cpu_dedicated_set` is not set on any compute |
| Instance lands on a host, then fails and is rescheduled; huge-page or NUMA flavors | `NUMATopologyFilter` missing from `enabled_filters`: the scheduler picked a host that cannot fit the NUMA request; `/var/log/kolla/nova/nova-compute.log` on that host says why it failed |
| `nova_compute` restarts in a loop after the change | `/var/log/kolla/nova/nova-compute.log`, and `docker logs nova_compute`: an invalid CPU list, or CPUs that do not exist on this host |
| Instance fails to start with a libvirt memory error | `/var/log/kolla/libvirt/libvirtd.log` on the compute, and the instance's QEMU log inside the `nova_libvirt` container (`/var/log/libvirt/qemu/<domain>.log`, libvirt's default path, not checked on a lab); `HugePages_Free` per NUMA node in sysfs |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Overrides with inventory variables | One directory per host under `config/nova/<host>/` | Many hosts share one CPU layout; the guide documents variables in overrides | A missing variable fails the template; guard the lines with `{% if ... is defined %}` |
| `cpu_dedicated_set` and `cpu_shared_set` on the same hosts | Separate pinned-only hosts | Since Train, Nova tracks `PCPU` and `VCPU` separately | `PCPU` and `VCPU` allocations count together in the `cores` quota (Nova guide) |
| Huge pages at boot | At run time | The guide: run-time allocation can fail when memory is fragmented | A reboot per host; that memory is reserved even with no huge-page instance running |
| `NUMATopologyFilter` added | Nova's default filters | Required by the Nova guide whenever NUMA awareness is used | One more filter for every request; it does nothing for instances without a NUMA topology |
| Separate flavors for pinned, huge-page and normal | Image properties (`hw_cpu_policy`, `hw_mem_page_size`) | Flavors are admin-only; the Nova guide notes flavor policies take precedence and can stop users from setting a policy in their images | More flavors to maintain |

## What went wrong during deployment

On an all-in-one lab: instances created right after the `reconfigure` failed with `MessagingTimeout` while the
scheduler's workers restarted (see *Applying it*); created again 3 minutes later, they booted.

## Version notes

- 2026.1: Kolla-Ansible sets `[filter_scheduler] enabled_filters` only with Blazar or the fake driver
  (`ansible/roles/nova/templates/nova.conf.j2`); Nova's default list has five filters and no `NUMATopologyFilter`
  (`nova/conf/scheduler.py`).
- Since Nova 20.0.0 (Train), `cpu_dedicated_set` replaces the old `vcpu_pin_set`; see the Nova guide's upgrade notes
  if an old platform still carries `vcpu_pin_set` in an override.

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- Add the filter first, then the hosts, then the flavors: a flavor published before the filter lands instances on
  hosts that cannot hold them.
- Decide the CPU and huge-page layout per hardware model, and check sibling threads before writing CPU lists.
- Leave memory for the host: huge pages and Nova's `reserved_host_memory_mb` both take it away from host processes,
  and the Kolla containers on a compute need their share.

## Validity and limits

Written from Nova 2026.1 (CPU topologies, huge pages, scheduling guides, `nova/conf/scheduler.py` and
`nova/conf/compute.py`) and Kolla-Ansible 2026.1 (advanced configuration guide, the `nova` and `nova-cell` roles).
Lab run (Kolla-Ansible 22.2.0, all-in-one VM with 4 vCPUs, one NUMA node, no hyper-threading, nested KVM): huge
pages at boot (512 pages of 2 MB, reboot), the read-only host checks, the inventory group with `group_vars`
next to the inventory file, both overrides as written, `genconfig` (no container restarted), `reconfigure --tags
nova --limit control,compute-pinned`, the Placement and `virsh` checks with one instance per flavor. Not run:
several NUMA nodes or sibling threads (the lab has neither, so NUMA fitting and `hw:cpu_thread_policy` were not
exercised), a compute outside `compute-pinned` (the empty-file path of the `{% if %}`), the symptoms of the
*Where to look* table (no instance was booted without `NUMATopologyFilter`), Blazar, and the undo. Not covered: CPU power management for dedicated cores,
`mixed` CPU policy in detail, real-time instances (`hw:cpu_realtime`), kernel CPU isolation outside Nova, DPDK with
OVS (it needs huge pages too: see the Kolla-Ansible DPDK guide), and changing the CPU sets of a host that already runs
pinned instances.

## Related

- [Laying out the configuration](inventory-globals-overrides-and-secrets-layout.md): where overrides live and how a
  lab checked which `config/nova/<host>/` paths apply.
- Nova 2026.1: [CPU topologies](https://docs.openstack.org/nova/2026.1/admin/cpu-topologies.html),
  [huge pages](https://docs.openstack.org/nova/2026.1/admin/huge-pages.html),
  [scheduling](https://docs.openstack.org/nova/2026.1/admin/scheduling.html)
- Kolla-Ansible 2026.1: [advanced configuration](https://docs.openstack.org/kolla-ansible/2026.1/admin/advanced-configuration.html)
  (override locations and Jinja in overrides)

---

*Provenance: written by agent:ozarland-author, from the Nova and Kolla-Ansible 2026.1 documentation and code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=configuration/cpu-pinning-hugepages-and-numa-on-computes.md&title=Feedback%3A+configuration/cpu-pinning-hugepages-and-numa-on-computes.md):
your release, your topology, and what happened.
