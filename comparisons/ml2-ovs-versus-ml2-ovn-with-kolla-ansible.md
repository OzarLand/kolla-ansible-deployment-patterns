---
title: ML2/OVS versus ML2/OVN with Kolla-Ansible. What changes in globals.yml, in operations and at upgrade time
kind: comparison
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, neutron, ovs, ovn]
deployment_methods: [kolla-ansible]
environment:
  purpose: any multinode (pre-production, production)
  scale: control, network and compute groups on separate hosts
  network: ml2/ovs or ml2/ovn
  storage: any
  os: any supported by Kolla-Ansible 2026.1
risk: read-only
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html
  - https://docs.openstack.org/neutron/2026.1/ovn/gaps.html
  - https://docs.openstack.org/neutron/2026.1/admin/ovn/features.html
  - https://docs.openstack.org/neutron/2026.1/admin/ovn/routing.html
  - https://docs.openstack.org/neutron/2026.1/admin/config-mtu.html
authored_by: agent:ozarland-author
last_verified: 2026-10-24
---

# ML2/OVS versus ML2/OVN with Kolla-Ansible. What changes in globals.yml, in operations and at upgrade time

## What is compared

Kolla-Ansible 2026.1 deploys Neutron with one of two mechanism drivers, chosen by one variable,
`neutron_plugin_agent`:

- **ML2/OVS** (`"openvswitch"`, the default): Neutron's own agents do the work. The Open vSwitch agent programs
  each host's switch; the L3, DHCP and metadata agents run routers, DHCP servers and the metadata proxy in network
  namespaces on the `network` hosts.
- **ML2/OVN** (`"ovn"`): OVN does the work. Neutron writes the wanted state to OVN's northbound database,
  `ovn-northd` translates it into the southbound database, and `ovn-controller` on each host programs Open vSwitch
  from it. There are no L3 or DHCP agents.

Both are supported and maintained upstream. The Linux Bridge driver, the third option of earlier releases, is
gone from Neutron and from Kolla-Ansible. This page compares the two from the operator's side: configuration,
day-to-day operation, upgrades and migration. It does not say which one is right: that depends on the platform,
the team and the features it needs. The [network interfaces pattern](../networking/interfaces-vlans-and-mtu-ovs-and-ovn.md)
covers the interface, VLAN and MTU layout for both.

## What changes in globals.yml and the inventory

| | ML2/OVS | ML2/OVN |
|---|---|---|
| Selector | `neutron_plugin_agent: "openvswitch"` (default) | `neutron_plugin_agent: "ovn"` |
| Tenant network type | `vxlan` (`neutron_tenant_network_types`) | `geneve` |
| Router high availability | Off by default. `enable_neutron_agent_ha: true` sets `l3_ha = true`, `max_l3_agents_per_router = 3` and `dhcp_agents_per_network = 2` | Built in: a router's gateway port is scheduled on several gateway chassis and fails over with BFD; no flag |
| Distributed floating IPs | DVR, a separate mode (`enable_neutron_dvr`) | `neutron_ovn_distributed_fip: "yes"`; off by default in Kolla-Ansible, so floating IP traffic goes through the gateway chassis |
| DHCP | DHCP agents on the `network` hosts | Answered by `ovn-controller` on each host. `neutron_ovn_dhcp_agent: "yes"` adds DHCP agents, for example for Ironic bare metal ports, which OVN cannot answer |
| Metadata | Metadata agent on the `network` hosts, reached through the router or DHCP namespace | OVN metadata agent on every compute and network host (`neutron-ovn-metadata-agent` group) |
| Extra containers | `neutron_openvswitch_agent`, `neutron_l3_agent`, `neutron_dhcp_agent`, `neutron_metadata_agent` | `ovn_controller`, `ovn_northd`, `ovn_nb_db`, `ovn_sb_db`, `ovn_sb_db_relay`, `neutron_ovn_metadata_agent` |
| Inventory groups that matter | `neutron-l3-agent`, `neutron-dhcp-agent`, `neutron-metadata-agent` (children of `neutron`) | `ovn-controller-compute`, `ovn-controller-network` (the gateway chassis), `ovn-database` (northd, NB, SB and relay; `control` by default) |
| Southbound relays | Not applicable | `enable_ovn_sb_db_relay`, on with OVN. One relay group per 50 `ovn-controller` hosts by default (`ovn_sb_db_relay_compute_per_relay`), or a fixed `ovn_sb_db_relay_count` |
| Optional agents | Metering agent (works only with the L3 agent) | `neutron_enable_ovn_agent: true` deploys the Neutron OVN agent, needed only for QoS on hardware-offloaded ports; `neutron-ovn-vpn-agent` for VPNaaS |

Both drivers keep the Open vSwitch containers (`openvswitch_db`, `openvswitch_vswitchd`) on every host: OVN
programs Open vSwitch, it does not replace it.

## Operating them day to day

| | ML2/OVS | ML2/OVN |
|---|---|---|
| What runs the data plane | Open vSwitch on each host, with flows from the OVS agent; routers and DHCP in namespaces (`qrouter-…`, `qdhcp-…`) on the `network` hosts | Open vSwitch on each host, with flows from `ovn-controller`; one `ovnmeta-…` namespace per network on each host for metadata |
| What can fail, and what it looks like | An agent: `openstack network agent list` shows it down; its routers or DHCP servers stop being updated, or stop if the namespace goes | `ovn-controller` on a host, `ovn-northd`, or the NB/SB database cluster: changes stop reaching the hosts. Open vSwitch keeps the flows already installed |
| Where to look | `openstack network agent list`, `ip netns`, `tcpdump` inside the namespaces, the agents' logs | `docker exec ovn_northd ovn-nbctl show` and `ovn-sbctl show` (the Kolla-Ansible guide suggests running them from the `ovn_northd` container), `ovn-trace` or Neutron's `ml2ovn-trace`, `ovn-controller` logs |
| State to protect | Neutron's database only; the agents rebuild their state from it | Neutron's database and the OVN NB/SB databases (a Raft cluster on the `ovn-database` hosts). `neutron-ovn-db-sync-util` resynchronises the northbound database from Neutron's |
| Restarts | Agent restarts can disturb the data plane; Kolla-Ansible 2026.1 runs the DHCP and L3 agents' subprocesses in wrapper containers by default (`neutron_agents_wrappers`, release note `neutron-agent-wrappers-by-default`), so an agent restart does not take them down | The OVN metadata agent's HAProxy processes run in wrapper containers too (2026.1 prelude) |
| Knowledge the team needs | Linux namespaces, iptables or the OVS firewall, Neutron agents: the model many existing runbooks assume | OVN's logical model (logical switches and routers, chassis, gateway chassis), its databases and tools |
| Tenant MTU on a 1500-byte underlay | 1450 (VXLAN overhead 50 bytes over IPv4) | 1442 (Geneve, with Neutron's default `max_header_size` of 38) |

Features where the two differ, from Neutron's list of [gaps from ML2/OVS](https://docs.openstack.org/neutron/2026.1/ovn/gaps.html):

- **DHCP and custom security groups:** with OVN, DHCP traffic from an instance must be allowed by its security
  groups; ML2/OVS adds that rule itself. The default security group allows it.
- **DNS:** OVN answers DNS queries for tenant names itself, over UDP only and without EDNS options, by spoofing
  replies from the configured resolvers; using the host's resolver needs extra DHCP agents.
- **Not supported by OVN:** IPv6 NDP proxy, fragmentation of east-west traffic through an OVN router, the
  metering agent, QoS packet-rate rules, and floating IP port forwarding on routers with VLAN or flat internal
  networks when distributed floating IPs are on.

In the other direction, OVN brings natively what ML2/OVS needs extra modes or agents for: distributed routing,
router high availability without VRRP namespaces, and DHCP without agents ([OVN features](https://docs.openstack.org/neutron/2026.1/admin/ovn/features.html)).

## Upgrades

- **ML2/OVS:** the upgrade recreates the agent containers. With router HA, `neutron_l3_agent_failover_delay`
  spaces the L3 agent restarts so that not every router fails over at once; the Kolla-Ansible guide suggests
  `40 + 3n` seconds for `n` routers as a starting point, better measured. The Neutron agents' own upgrade
  constraints apply.
- **ML2/OVN:** Kolla-Ansible's `site.yml` applies the `ovn-controller` role before the `ovn-db` role (northd, NB,
  SB). The OVN databases are a second set of stateful services to check before and after the upgrade, next to
  MariaDB and RabbitMQ. In 2026.1, an upgrade adds the new `ovn_sb_db_relay` containers unless
  `enable_ovn_sb_db_relay` is set to `no`; the release note says no data migration or manual step is needed.
- **Both:** Open vSwitch containers are recreated, which can cut traffic briefly. On an all-in-one ML2/OVS lab,
  floating IPs lost about 2 s when `openvswitch_vswitchd` was recreated during a 2025.1 to 2026.1 upgrade
  ([experience report](../version-notes/upgrade-report-2025.1-to-2026.1-all-in-one-lab.md)). No OVN upgrade has
  been recorded here yet.

## Migrating from one to the other

**Kolla-Ansible does not support it.** Its Neutron prechecks stop the deploy if `neutron_plugin_agent` does not
match what is running:

- with `"openvswitch"`, if an `ovn_controller` container or the OVN database volumes exist;
- with `"ovn"`, if a `neutron_openvswitch_agent` container exists;

with the message "Kolla-Ansible does not support this migration operation"
(`ansible/roles/neutron/tasks/neutron_plugin_agent_check.yml`).

Upstream, Neutron removed its ML2/OVS to ML2/OVN migration tool for TripleO when TripleO was retired (release
note `remove-ovn-migration-for-tripleo`). What remains in its 2026.1 tree is a proof-of-concept playbook,
`tools/ovn_migration/migrate-to-ovn.yml`, whose README says it was tested on Ocata, and
`neutron-ovn-db-sync-util`, which now loads migration plugins from other projects. The README also names the MTU
problem: instances on VXLAN networks must first learn a lower MTU through DHCP, because Geneve needs 8 more bytes.

So the choice is, in practice, made once per deployment. Moving an existing cloud means planning it as a project
of its own (a tool outside Kolla-Ansible, a new region or a rebuild, and the MTU change for every instance), not
as a `globals.yml` change.

## What an operator gains and loses

| Choosing | Gains | Loses or takes on |
|---|---|---|
| ML2/OVS | Kolla-Ansible's default; the model most existing runbooks and documentation describe; every feature in Neutron's gaps list (metering, NDP proxy, QoS packet-rate rules); 8 more bytes of tenant MTU | L3, DHCP and metadata agents to keep alive and to scale on the `network` hosts; router HA and distributed routing only through extra modes (`enable_neutron_agent_ha`, DVR); one more failure point per agent |
| ML2/OVN | No L3 or DHCP agents; router HA with BFD built in; distributed floating IPs as an option; DHCP and metadata served on each host; southbound relays for large numbers of hosts | Two more databases to operate, back up and upgrade; a different troubleshooting model to learn; the gaps listed above; no supported way back to ML2/OVS in Kolla-Ansible |

Questions that usually decide it:

1. Does the platform need a feature from the gaps list (metering, NDP proxy, QoS packet-rate rules, Ironic DHCP
   without extra agents)?
2. Who will troubleshoot at night, and which model do they know?
3. How many hosts? OVN's southbound relays exist for scale; ML2/OVS scales by adding `network` hosts and agents.
4. Is this a new deployment? Changing later is a migration project, not a configuration change.

## Validity and limits

Written from Kolla-Ansible 2026.1 (the Neutron guide, `ansible/group_vars/all/neutron.yml` and `ovn.yml`, the
multinode inventory, `neutron.conf.j2` and `ml2_conf.ini.j2`, `site.yml`, the Neutron prechecks, the 2026.1 release
notes) and Neutron 2026.1 (gaps from ML2/OVS, OVN features and routing guides, the MTU guide, `tools/ovn_migration`,
release notes). Neither driver was deployed or compared on a platform for this page; the only measurement quoted
is the ML2/OVS upgrade report linked above.

Not covered: DVR in detail, OVS-DPDK and hardware offload, SR-IOV, BGP, Octavia's OVN provider, scale figures, and
other deployment tools.

## Related

- [Network interfaces, VLANs and MTU, with Open vSwitch and OVN side by side](../networking/interfaces-vlans-and-mtu-ovs-and-ovn.md)
- [From 2025.1 to 2026.1](../version-notes/2025.1-to-2026.1.md)
- Runbooks written on ML2/OVS: [router stuck on a dead L3 agent](https://github.com/OzarLand/openstack-vm-runbooks/blob/main/networking/router-stuck-on-dead-l3-agent.md),
  [restarting the OVS agent](https://github.com/OzarLand/openstack-vm-runbooks/blob/main/networking/restarting-the-ovs-agent-does-not-cut-instances.md),
  [tunnel MTU mismatch](https://github.com/OzarLand/openstack-vm-runbooks/blob/main/networking/tunnel-mtu-mismatch-large-packets-dropped.md)
- Kolla-Ansible 2026.1: [Neutron guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/neutron.html)
- Neutron 2026.1: [gaps from ML2/OVS](https://docs.openstack.org/neutron/2026.1/ovn/gaps.html),
  [OVN features](https://docs.openstack.org/neutron/2026.1/admin/ovn/features.html),
  [OVN routing](https://docs.openstack.org/neutron/2026.1/admin/ovn/routing.html),
  [MTU considerations](https://docs.openstack.org/neutron/2026.1/admin/config-mtu.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible and Neutron 2026.1 documentation, source and
release notes.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=comparisons/ml2-ovs-versus-ml2-ovn-with-kolla-ansible.md&title=Feedback%3A+comparisons/ml2-ovs-versus-ml2-ovn-with-kolla-ansible.md):
your release, your topology, and what happened.
