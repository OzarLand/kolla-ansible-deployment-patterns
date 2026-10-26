---
title: Removing or replacing a compute node in a Kolla-Ansible deployment
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, nova, neutron, placement, docker]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for production, with instances on the host
  scale: any multinode deployment with spare compute capacity
  network: ml2/ovs or ml2/ovn
  storage: any; what can move depends on where instance disks live
  os: any host OS supported by the Kolla-Ansible release
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/adding-and-removing-hosts.html
  - https://docs.openstack.org/nova/2026.1/admin/live-migration-usage.html
  - https://docs.openstack.org/nova/2026.1/admin/evacuate.html
  - https://docs.openstack.org/nova/2026.1/admin/node-down.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Removing or replacing a compute node in a Kolla-Ansible deployment

## Context

A compute node leaves a deployment for good (decommissioning, a smaller cloud), or is replaced by a new machine,
sometimes with the same name. The Kolla-Ansible part is short; what matters is what happens to the instances on the
host and to the records Nova, Placement and Neutron keep about it. A host removed from the inventory but not cleaned
up in the APIs stays listed as `down` forever; a reinstalled host that reuses the name of a record not cleaned up can
fail to start `nova-compute`.

This page follows the *Removing existing compute nodes* section of the Kolla-Ansible 2026.1 guide and adds the
two cases operators meet: the host is still running (planned), or it is dead (unplanned).

## Topology

Nothing changes on the controllers. The host leaves the `compute` group (and any other group it was in).

## Network design

Nothing to change, unless the host carried a role on the network (provider networks with distributed floating IPs,
or an OVN gateway chassis if it was also in `network`): then routers and gateways move with the instances; check
`openstack network agent list --host <host>` before removing it.

## Storage design

Where instance disks live decides how instances leave the host (see
[where instance disks live](../storage/nova-instance-disks-local-ceph-rbd-or-boot-from-volume.md)):

- **shared storage (Ceph RBD, boot from volume):** live migration works while the host runs; evacuation rebuilds an
  instance elsewhere with its disk intact if the host is dead;
- **local disks:** live migration copies the disk (block migration) while the host runs; if the host is dead,
  evacuation rebuilds the instance from its image and **its local disk is lost**.

## Inventory and configuration layout

Remove the host from every group it is in, and its `host_vars` file, after the services are stopped (below). Keep the
inventory change in its own commit.

## Procedure: the host is still running

Run from the deploy host with admin credentials.

**1. Read-only checks.** Is there room elsewhere?

```bash
openstack server list --all-projects --host <host> --long        # what has to move
openstack hypervisor list --long                                  # free capacity on the others
openstack network agent list --host <host>
```

**2. Stop new placements on the host.**

> ⚠️ **State-changing.** The scheduler no longer places instances on this host; running instances are not affected.
> Undo: `openstack compute service set --enable <host> nova-compute`.

```bash
openstack compute service set --disable --disable-reason "decommissioning" <host> nova-compute
```

**3. Move the instances.** The guide's loop, one instance after another, then check every result:

> ⚠️ **State-changing.** Live-migrates each instance: a short pause at the end of each migration; block migration
> of local disks loads both hosts and the network. Check that the instances' flavors allow it (PCI, SR-IOV or GPU
> instances usually cannot be live-migrated; plan a cold migration or an agreed restart with their owners). Undo:
> migrate back.

```bash
openstack server list --all-projects --host <host> -f value -c ID | while read server; do
  openstack server migrate --live-migration --wait "$server"
done
openstack server list --all-projects --host <host>      # expected: empty
```

**4. Stop the services on the host.**

> ⚠️ **State-changing.** Stops every Kolla container on this host only. Precondition: step 3 left no instance on it.
> Undo: `kolla-ansible deploy -i <inventory> --limit <host>`.

```bash
kolla-ansible stop -i <inventory> --yes-i-really-really-mean-it --limit <host>
```

**5. Remove the host from the inventory**, then check that the rest still works (an instance boots, migrations work
between the remaining hosts).

**6. Clean up its records** (the guide's loops).

> 🛑 **Destructive.** Deletes the host's network agents and its compute service. Deleting the compute service also
> deletes its compute node record and, in Placement, its resource provider; Nova refuses it while instances are
> still allocated to that host. Precondition: `openstack server list --all-projects --host <host>` is empty and the
> host's services are stopped (step 4), otherwise they register again. Cannot be undone: the records come back only
> by deploying the host again.

```bash
openstack network agent list --host <host> -f value -c ID | while read id; do
  openstack network agent delete $id
done

openstack compute service list --os-compute-api-version 2.53 --host <host> -f value -c ID | while read id; do
  openstack compute service delete --os-compute-api-version 2.53 $id
done
```

## Procedure: the host is dead

The instances on it are down. Recovering them means evacuating them, which rebuilds each one on another host.

> 🛑 **Destructive.** Evacuation rebuilds the instance on another host. With local disks, the disk is rebuilt from
> the image: its data is lost. Precondition: the dead host is really down and **fenced** (powered off, or cut from the
> storage and the network), otherwise the same instance can run twice and corrupt a shared disk. Nova must see the
> service as `down` and the service must be forced down or disabled. Undo: none for the lost local data.

```bash
openstack compute service set --down <host> nova-compute     # after fencing; tells Nova at once
openstack server evacuate <server-id>                        # one instance at a time; check each
```

Before running it in production, read Nova's *Evacuate instances* and *Recover from a failed compute node* guides.
Then stop what is left of the host's containers if it is reachable (step 4), remove it from the inventory (step 5)
and clean up its records (step 6). If the dead machine is later powered on with its old disks, keep it from starting
`nova-compute` until it has been reinstalled or added again as a new host.

## Replacing a compute node

- **With a new name:** remove the old host (above), then [add the new one](adding-a-compute-node.md).
- **With the same name** (a reinstalled or swapped machine): finish the removal first, including step 6, so that no
  service, compute node or resource provider with that name is left; then add the host as a new one. Reusing a name
  while the old records exist makes Nova find a compute node record that does not match the new machine.

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Disable, then drain, then stop | Stop first | Instances move while the host still serves them | Takes as long as the migrations |
| Live migration one by one | Many in parallel | Easier to watch and stop | Slower; Nova limits parallel outgoing migrations per host anyway (`max_concurrent_live_migrations`) |
| Evacuate only after fencing | Evacuate at once | Two copies of one instance on shared storage corrupt it | Needs out-of-band power control or a storage fence |
| Delete the records | Leave the host `down` in the lists | Clean lists, no stale resource provider, the name can be reused | Not reversible |

## What went wrong during deployment

Nothing observed for this page. Predictable traps: a stopped host whose `nova_compute` is started again by a later
full `deploy` because it is still in the inventory; agents re-registering after deletion because the containers were
not stopped; `compute service delete` refused because instances or migrations still reference the host.

## Version notes

Written against Kolla-Ansible 2026.1 and Nova 2026.1. The `--os-compute-api-version 2.53` in the cleanup commands is
the guide's (service IDs as UUIDs). `openstack compute service set --down` needs microversion 2.11 or later, which
current clients negotiate.

## Lessons learned

- Drain and check before stopping; stopping is the easy part.
- Fence before evacuating, every time.
- Clean up the records in the same change window, or the next person will wonder about a host that is `down`.

## Validity and limits

- Written from the Kolla-Ansible 2026.1 guide *Adding and removing hosts* and Nova's live migration, evacuation and
  failed-node guides. Not run on a lab for this page. Status `draft`.
- That deleting a compute service also removes its resource provider and is refused while instances remain follows
  Nova's documented behaviour since several releases; check it on your release.
- Not covered: Ironic computes, cells (the cleanup is the same through the API), hosts with other roles (controller,
  network, storage), Masakari-driven evacuation.

## Related

- [Adding a compute node](adding-a-compute-node.md)
- [Where instance disks live](../storage/nova-instance-disks-local-ceph-rbd-or-boot-from-volume.md)
- Kolla-Ansible 2026.1: [adding and removing hosts](https://docs.openstack.org/kolla-ansible/2026.1/user/adding-and-removing-hosts.html)
- Nova 2026.1: [live migration](https://docs.openstack.org/nova/2026.1/admin/live-migration-usage.html),
  [evacuate](https://docs.openstack.org/nova/2026.1/admin/evacuate.html),
  [recover from a failed compute node](https://docs.openstack.org/nova/2026.1/admin/node-down.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 and Nova 2026.1 documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=operations/removing-or-replacing-a-compute-node.md&title=Feedback%3A+operations/removing-or-replacing-a-compute-node.md):
your release, your topology, and what happened.
