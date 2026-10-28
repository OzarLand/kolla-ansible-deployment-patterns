---
title: Single control node versus three controllers. What high availability really costs in Kolla-Ansible
kind: comparison
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, haproxy, keepalived, mariadb, proxysql, rabbitmq, memcached, glance]
deployment_methods: [kolla-ansible]
environment:
  purpose: any (lab, pre-production, production)
  scale: one or three hosts in the control group
  network: any
  storage: any
  os: any supported by Kolla-Ansible 2026.1
risk: read-only
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/production-architecture-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/adding-and-removing-hosts.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/message-queues/rabbitmq.html
authored_by: agent:ozarland-author
last_verified: 2026-10-24
---

# Single control node versus three controllers. What high availability really costs in Kolla-Ansible

## What is compared

The `control` group of a Kolla-Ansible inventory runs the API services, MariaDB and RabbitMQ; in the multinode
sample inventory it also feeds the `mariadb`, `rabbitmq` and OVN database groups. This page compares one host in
that group with three:

- **One control node:** every control service runs once. Losing the host stops the control plane.
- **Three controllers:** the API services run three times behind HAProxy and a keepalived virtual IP, and MariaDB
  (Galera) and RabbitMQ form clusters that keep working when one member is lost.

It does not say which one is right. A lab, a small edge site and a production region weigh the costs below
differently. What it shows is what each choice costs in hosts, configuration, operations and upgrades, and what
three controllers do and do not protect against.

## Why three, not two

The Kolla-Ansible production architecture guide is explicit: MariaDB and RabbitMQ need a quorum, a majority vote,
so "deploying 2 instances of these do not provide (by default) any HA as a failure of one causes a failure of the
other one". The recommended number is 3, which tolerates one failure; 5 tolerates two, and more usually brings no
benefit. The `control` group "should have odd number of nodes for quorum". Two controllers cost twice the hardware
of one without the availability of three.

## What changes in globals.yml and the inventory

| | One control node | Three controllers |
|---|---|---|
| Inventory | One host under `[control]` (and, usually, `[network]`, which feeds `loadbalancer`) | Three hosts under `[control]`; `[loadbalancer]` (children of `network` in the sample) holds HAProxy and keepalived |
| HAProxy and keepalived | On by default (`enable_haproxy: true`, `enable_keepalived` follows it); can be turned off with `enable_haproxy: false` and `enable_keepalived: false`; the all-in-one lab pattern also turns ProxySQL off explicitly (`enable_proxysql: "no"`) | Required for HA: keepalived moves `kolla_internal_vip_address` between the `loadbalancer` hosts; HAProxy spreads requests over the three API backends |
| Virtual IP | With HAProxy on, a VIP on the one host; with HAProxy off, `kolla_internal_vip_address` set to the host's own address (all-in-one lab pattern) | `kolla_internal_vip_address` (and `kolla_external_vip_address` if separate), unused addresses on the API network. `keepalived_virtual_router_id` (default `51`) must be unique on the layer 2 segment; VRRP runs in multicast by default (`keepalived_traffic_mode`) |
| MariaDB | One server, no cluster; ProxySQL on by default unless turned off | A three-node Galera cluster; ProxySQL (`enable_proxysql`, on with MariaDB) in front of it |
| RabbitMQ | One node | A three-node cluster with `cluster_partition_handling = pause_minority`; with RabbitMQ 4, queues are quorum queues, replicated on the cluster, and fanouts use streams by default (`om_enable_rabbitmq_stream_fanout`) |
| Glance `file` back-end | Works as is | Only one `glance_api` container is started, unless the image directory is shared storage (the multinode guide's note): Glance stays a single point of failure until its back-end is shared (Ceph, or `file` on a shared filesystem) |
| Failover tuning | Not applicable | `haproxy_host_ipv4_tcp_retries2` shortens the time clients keep dead connections to the VIP after a failover, "especially helpful for connections to MariaDB" (HAProxy guide) |

## What three controllers protect against, and what they do not

| Event | One control node | Three controllers |
|---|---|---|
| One controller host lost (power, kernel, disk) | Control plane down: no API, no new instance, no change. Running instances keep running, with the network they had | The VIP moves, the two remaining members keep quorum, the APIs answer through HAProxy |
| Rolling restart of MariaDB or RabbitMQ (reconfigure, upgrade) | The database or the message bus is down during the restart | Kolla-Ansible restarts MariaDB and RabbitMQ in batches of a third of the hosts (`serial: "33%"` in `mariadb.yml` and `rabbitmq.yml`), so one member at a time |
| Other API containers recreated (reconfigure, upgrade) | The API is unavailable while its container restarts | Kolla-Ansible restarts them on all hosts of the play at once by default (`kolla_serial` unset), so a short gap can still be seen; `-e kolla_serial=…` is documented for `bootstrap-servers` |
| Two controllers lost at once | Not applicable | Quorum lost: MariaDB and RabbitMQ stop serving on the remaining node. Recovery is manual: `kolla-ansible mariadb-recovery` "is used to recover a completely stopped mariadb cluster" (operating guide) |
| A whole site lost (power, network) | Down | Down, if the three controllers are in the same room or rack |
| A bad change (wrong config, bad upgrade) | Down | Down on all three: the same change is applied to every controller |
| Data corruption, an accidental deletion | Restore from backup | Replicated to all members within seconds: restore from backup too |

The control plane is not the data plane: on both, running instances keep running when the control plane stops, as
long as their compute and network nodes are up. What stops is everything that needs an API: creating, deleting,
migrating, attaching, and the services that call the APIs themselves.

## What it costs to run

| | One control node | Three controllers |
|---|---|---|
| Hosts | One | Three, with the same CPU, memory and fast disk for MariaDB and RabbitMQ (the production guide recommends fast storage for `/var/lib/docker`) |
| Network | The host's addresses | Two more addresses (internal and, if used, external VIP), VRRP allowed on the segment, the virtual router ID coordinated with any other keepalived cluster |
| Monitoring | Is the host up, are its containers up | Also: Galera cluster size and state, RabbitMQ cluster and partitions, which host holds the VIP, HAProxy backend states |
| Failure modes to know | One: the host is down | Also: a Galera cluster that lost quorum or stopped completely, a RabbitMQ partition, a VIP held by two hosts (split brain if VRRP traffic is blocked), a backend silently out of HAProxy |
| Removing or replacing a controller | Rebuild and restore | One at a time, keeping quorum, after moving the DHCP agents' networks and the L3 agents' routers it hosts (adding and removing hosts guide) |
| Upgrades | Every service down while it is recreated | Clustered services keep serving during their batched restart; the upgrade takes longer; the database schema migration still runs once, for the whole cluster |
| Backups | Needed | Still needed: replication is not a backup (`kolla-ansible mariadb-backup`) |

## What an operator gains and loses

| Choosing | Gains | Loses or takes on |
|---|---|---|
| One control node | Fewer hosts, simpler network (no VIP needed), fewer failure modes; quick to rebuild a lab | The control plane stops with the host, during every MariaDB or RabbitMQ restart, and during upgrades; maintenance of the host means an API outage |
| Three controllers | The control plane survives the loss or the maintenance of one controller; MariaDB and RabbitMQ restart one member at a time | Three times the control hardware; VIP, VRRP and cluster monitoring; cluster recovery procedures to know and test; Glance (`file`) and other single-instance services still to make redundant |

Questions that usually decide it:

1. What does an hour without API cost here: can users wait for a host to be repaired or rebuilt?
2. Does the team know how to recover a stopped Galera cluster or a partitioned RabbitMQ, and has it practised?
3. Are the three controllers really independent (power, switches, racks), or do they share the failure they are
   meant to survive?
4. Is the rest of the platform redundant too (network nodes, Glance's back-end, storage)? Three controllers in front
   of a single point of failure protect only part of the path.

## Validity and limits

Written from Kolla-Ansible 2026.1: the production architecture guide, the HAProxy guide, the multinode guide, the
adding and removing hosts guide, the operating guide, the RabbitMQ guide, `ansible/group_vars/all/` (`haproxy.yml`,
`keepalived.yml`, `proxysql.yml`), the `loadbalancer`, `mariadb` and `rabbitmq` roles, `mariadb.yml`,
`rabbitmq.yml` and `site.yml`, and the multinode and all-in-one inventories. Neither layout was failed over or
upgraded on a platform for this page.

Not covered: five or more controllers, MariaDB sharding, separate database or message-bus hosts, stretched clusters
across sites, Masakari and instance HA, OVN database clustering in detail, and HAProxy without keepalived (an
external load balancer).

## Related

- [All-in-one lab on a single VM](../topologies/all-in-one-lab-on-a-single-vm.md): one node, HAProxy and ProxySQL
  turned off.
- [Three controllers and two computes, nested inside an existing OpenStack cloud](../topologies/three-controllers-two-computes-nested-in-an-openstack-cloud.md)
- [Cinder and Glance back-ends compared](../storage/cinder-and-glance-backends-compared-lvm-ceph-nfs.md): Glance
  `file` and shared storage.
- [Unattended upgrades during a deploy break the MariaDB cluster](../deployment-errors/unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md)
- [ML2/OVS versus ML2/OVN](ml2-ovs-versus-ml2-ovn-with-kolla-ansible.md)
- Kolla-Ansible 2026.1: [production architecture guide](https://docs.openstack.org/kolla-ansible/2026.1/admin/production-architecture-guide.html),
  [HAProxy guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html),
  [multinode](https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html),
  [adding and removing hosts](https://docs.openstack.org/kolla-ansible/2026.1/user/adding-and-removing-hosts.html),
  [RabbitMQ](https://docs.openstack.org/kolla-ansible/2026.1/reference/message-queues/rabbitmq.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=comparisons/single-control-node-versus-three-controllers.md&title=Feedback%3A+comparisons/single-control-node-versus-three-controllers.md):
your release, your topology, and what happened.
