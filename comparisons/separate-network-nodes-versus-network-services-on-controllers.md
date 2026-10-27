---
title: Separate network nodes versus network services on the controllers. What the network group decides in Kolla-Ansible
kind: comparison
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, neutron, ovs, ovn, haproxy, keepalived]
deployment_methods: [kolla-ansible]
environment:
  purpose: any (lab, pre-production, production)
  scale: three controllers, with or without two or more dedicated network nodes
  network: ml2/ovs or ml2/ovn
  storage: any
  os: any supported by Kolla-Ansible 2026.1
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/production-architecture-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html
  - https://docs.openstack.org/neutron/2026.1/admin/ovn/routing.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Separate network nodes versus network services on the controllers

## What is compared

The `network` group of a Kolla-Ansible inventory decides where north-south traffic leaves the cloud: the Neutron L3,
DHCP and metadata agents with ML2/OVS, the OVN gateway chassis with ML2/OVN, and, in the sample inventories, HAProxy
and keepalived. This page compares two common layouts:

- **Network services on the controllers:** the same hosts are in `control` and `network`. This is what the
  all-in-one and many small multinode deployments do (see
  [three controllers and two computes](../topologies/three-controllers-two-computes-nested-in-an-openstack-cloud.md)).
- **Dedicated network nodes:** two or more hosts only in `network`, the controllers only in `control`.

Neither is right in general. What changes is where routed and floating IP traffic flows, what fails together, what
an upgrade or a reconfiguration restarts, and how many hosts there are to run.

## What the network group carries

From the multinode sample inventory and the roles of Kolla-Ansible 2026.1:

| Service | Group in the multinode sample inventory | ML2/OVS | ML2/OVN |
|---|---|---|---|
| HAProxy and keepalived (the VIPs) | `loadbalancer:children` = `network` | yes | yes |
| Neutron L3, DHCP, metadata agents | `neutron:children` = `network` | yes | not used |
| Open vSwitch | `network`, `compute` | yes | yes |
| OVN controller as gateway chassis | `ovn-controller-network` = `network` (gets `enable-chassis-as-gw`) | not used | yes |
| OVN metadata agent | `compute`, `network` | not used | yes |
| Neutron server, OVN databases and `northd` | `control` | — | — |
| `neutron_external_interface` (`br-ex`) | hosts in `network`; computes too with provider networks, DVR or `neutron_ovn_distributed_fip` | | |

So "dedicated network nodes" in the sample inventory also moves the VIPs off the controllers. That is a choice of
the sample, not a rule: the production architecture guide describes network nodes as hosting the Neutron agents
"along with haproxy / keepalived", but nothing stops an inventory from setting `[loadbalancer:children]` to
`control`.

## What changes in globals.yml and the inventory

```ini
# network services on the controllers
[control]
controller-[1:3]

[network:children]
control

# dedicated network nodes
[control]
controller-[1:3]

[network]
network-[1:2]
```

`globals.yml` does not change between the two, except interfaces: `neutron_external_interface` must exist, with the
same name or per-host values, on every host of `network`. With dedicated network nodes, the controllers no longer
need an interface on the external or provider networks at all (unless computes or controllers still carry provider
networks for another reason), and the network nodes need one on the tunnel network and the API network.

For the agents to fail over, ML2/OVS needs `enable_neutron_agent_ha: true` (L3 HA routers and several DHCP agents
per network) and at least two hosts in `network`. With ML2/OVN, every host of `network` is a gateway chassis and OVN
schedules router gateways across them by priority (and by availability zone when `neutron_ovn_availability_zones`
is set, not covered here).

## What each layout protects against, and what it does not

| Event | Network on the controllers | Dedicated network nodes |
|---|---|---|
| A controller is lost | The routers it hosted fail over (L3 HA or OVN gateway priority) **and** its control services are lost at the same time | Only control services are affected; routing is unaffected |
| A network node is lost | (no such host) | Routers fail over to the other network nodes; the control plane is unaffected |
| `reconfigure` or `upgrade` of the control plane | Restarts API and data plane containers on the same hosts, at the same time | Control and network roles can be restarted at different moments, with `--limit` used with care |
| Heavy north-south or SNAT traffic | Competes with the databases, RabbitMQ and APIs for CPU and NIC bandwidth | Isolated on hosts sized for it |
| Kernel or OVS problem on a gateway host | Can take a controller down with it | Stays on a network node |

Neither layout makes east-west traffic between instances leave the computes: with VXLAN or Geneve it goes compute to
compute. And neither replaces distributed routing: with DVR in `dvr` mode on the computes
(`neutron_compute_dvr_mode: dvr`, ML2/OVS) or `neutron_ovn_distributed_fip` (ML2/OVN), floating IP traffic of
instances leaves from their own compute; only SNAT for instances without a floating IP still goes through the
network group. With `neutron_compute_dvr_mode: dvr_no_external`, floating IP traffic stays centralised on the
network group too.

## What it costs to run

- **Hosts.** Dedicated network nodes are at least two more hosts (one gives no failover), with NICs on the external
  networks.
- **Cabling and VLANs.** External and provider networks reach only the network nodes (and the computes if
  distributed), not the controllers: a smaller exposure of the control plane to outside networks.
- **Upgrades.** More hosts to upgrade, but control plane restarts and data plane restarts can be sequenced
  separately. The operating guide says that using `--limit` "is not recommended", because of known bugs (bug
  2054348), so test such a sequence on a lab.
- **Moving later.** Moving from one layout to the other on a running platform means adding the new hosts to
  `network`, moving routers and DHCP networks to them (ML2/OVS) or letting OVN reschedule gateways, then removing the
  old ones from the group: the procedure the upstream *Adding and removing hosts* guide gives for controllers, applied
  to network agents. With the sample inventory it also moves the VIPs, which is a separate, visible change.

## What an operator gains and loses

| Network on the controllers | Dedicated network nodes |
|---|---|
| Fewer hosts and cables | Routing failures and control plane failures are independent |
| One kind of host to size and patch | Network nodes sized for packet processing, controllers for databases and APIs |
| A controller loss is also a router failover | A controller maintenance does not move routers |
| Simpler for labs, small sites and edge | More hosts, more interfaces, more to monitor |

## Validity and limits

- Written from the Kolla-Ansible 2026.1 multinode sample inventory (`ansible/inventory/multinode`), its production
  architecture and Neutron guides, the `ovn-controller` role (`tasks/setup-ovs.yml`, gateway chassis on
  `ovn-controller-network`) and `ansible/group_vars/all/neutron.yml` (`computes_need_external_bridge`). Not built on
  a lab for this page. Status `draft`.
- No measurement of throughput or failover time is given; those depend on hardware, the number of routers and the
  back-end.
- Not covered: BGP-based designs (`neutron-bgp-dragent`, OVN BGP), hardware offload, and network nodes in separate
  availability zones (`neutron_ovn_availability_zones`).

## Related

- [ML2/OVS versus ML2/OVN with Kolla-Ansible](ml2-ovs-versus-ml2-ovn-with-kolla-ansible.md)
- [Single control node versus three controllers](single-control-node-versus-three-controllers.md)
- [Network interfaces, VLANs and MTU](../networking/interfaces-vlans-and-mtu-ovs-and-ovn.md)
- [Provider networks](../networking/provider-networks-instances-on-flat-or-vlan-networks.md)
- Kolla-Ansible 2026.1: [production architecture guide](https://docs.openstack.org/kolla-ansible/2026.1/admin/production-architecture-guide.html),
  [Neutron](https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html),
  [multinode](https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html)
- Neutron 2026.1: [routing in OVN](https://docs.openstack.org/neutron/2026.1/admin/ovn/routing.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation, sample inventory and
code. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=comparisons/separate-network-nodes-versus-network-services-on-controllers.md&title=Feedback%3A+comparisons/separate-network-nodes-versus-network-services-on-controllers.md):
your release, your topology, and what happened.
