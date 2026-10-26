[← OzarLand home](https://github.com/OzarLand/ozarland)

# kolla-ansible-deployment-patterns

**How operators actually deploy OpenStack with Kolla-Ansible, compared side by side.**

Part of [OzarLand](https://github.com/OzarLand/ozarland), a collaborative knowledge-sharing space for OpenStack
operators. Share experiences, document solutions, learn from each other. No imposed tools, no single way of
operating OpenStack.

## Scope

This repository documents the many ways operators deploy OpenStack with [Kolla-Ansible](https://docs.openstack.org/kolla-ansible/latest/),
so that someone planning or changing a deployment can compare approaches and learn from others' experience.

**It does not promote one architecture as correct.** Every pattern states its context, its benefits, its limits
and the lessons its operators learned. Two patterns can contradict each other and both be right in their context.

**In scope:**

- Reference topologies and node roles, from all-in-one labs to multi-controller, multi-network-node production.
- Lab, pre-production and production environments: how they differ and how they are kept comparable.
- Network design: interfaces, management/API/tunnel/external/storage networks, OVN versus OVS, provider networks,
  MTU, bonding.
- Storage choices: Ceph, LVM, NFS and others for Glance, Cinder and Nova; what is shared and what is local.
- Inventories, `globals.yml`, group and host variables, configuration overrides: how they are organised and
  versioned across environments.
- Secrets and configuration files: how `passwords.yml` and certificates are handled (never their content).
- Architectural choices and their operational consequences.
- Errors met during deployment and reconfiguration, with their causes and fixes.
- Differences between Kolla-Ansible and OpenStack releases that affect a pattern, and the precautions they require.

**Out of scope:** running instances (see `openstack-vm-runbooks`), upgrading (see `openstack-upgrade-playbooks`),
other deployment tools (welcome as separate pattern repositories when their operators show up), and anything
that reproduces the upstream documentation instead of adding operational experience to it.

## How content is organised

```text
topologies/        one pattern per topology, from all-in-one to large multinode
networking/        network designs and their trade-offs
storage/           storage designs and their trade-offs
configuration/     inventory layouts, variable organisation, environment separation, secrets handling
deployment-errors/ errors met during deploy/reconfigure/upgrade-prepare, cause and fix
operations/        day-2 procedures on a running deployment: adding and removing hosts, backups, recoveries
version-notes/     per Kolla-Ansible / OpenStack release: what changed that affects deployments
comparisons/       side-by-side comparisons of patterns on one dimension (e.g. OVN vs OVS, Ceph vs LVM)
templates/         deployment-pattern template
```

Each pattern follows the shared [deployment pattern template](https://github.com/OzarLand/ozarland/blob/main/community/templates/deployment-pattern.md):
context, topology, network, storage, inventory and configuration layout, key choices and consequences, what went
wrong, version notes, lessons learned, validity and limits.

## Index

| Pattern | Purpose | Scale | Network | Storage | Releases |
|---|---|---|---|---|---|
| [Three controllers and two computes, nested inside an existing OpenStack cloud](topologies/three-controllers-two-computes-nested-in-an-openstack-cloud.md) | lab | 3 controllers, 2 computes | OVS, VXLAN | local instance store | 2026.1 |
| [All-in-one lab on a single VM, nested inside an existing OpenStack cloud](topologies/all-in-one-lab-on-a-single-vm.md) | lab | 1 node (all-in-one) | OVS | local root disk | 2026.1 |

| Networking pattern | Releases | Status |
|---|---|---|
| [Network interfaces, VLANs and MTU for a multinode deployment, with Open vSwitch and OVN side by side](networking/interfaces-vlans-and-mtu-ovs-and-ovn.md) | 2026.1 | draft |
| [Provider networks with Kolla-Ansible. Instances directly on a flat or VLAN network, without tunnels or routers](networking/provider-networks-instances-on-flat-or-vlan-networks.md) | 2026.1 | draft |

| Deployment error | Releases | Status |
|---|---|---|
| [kolla-ansible deploy fails part-way because Ubuntu upgraded packages during the deploy](deployment-errors/unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md) | 2026.1 | lab-verified |
| [kolla-ansible deploy hangs on MariaDB port liveness: ProxySQL holds port 3306 on a node without HAProxy](deployment-errors/mariadb-exits-address-in-use-proxysql-on-a-node-without-haproxy.md) | 2026.1 | draft |
| [kolla-ansible pull fails with "http: server gave HTTP response to HTTPS client": Docker does not know the registry is plain HTTP](deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md) | 2026.1 | draft |

| Operations | Releases | Status |
|---|---|---|
| [Adding a compute node to a running Kolla-Ansible deployment](operations/adding-a-compute-node.md) | 2026.1 | draft |

| Storage pattern | Releases | Status |
|---|---|---|
| [Cinder and Glance back-ends compared. LVM, Ceph RBD and NFS, and what each choice implies](storage/cinder-and-glance-backends-compared-lvm-ceph-nfs.md) | 2026.1 | draft |
| [Where instance disks live. Local on each compute, in Ceph RBD, or boot from volume](storage/nova-instance-disks-local-ceph-rbd-or-boot-from-volume.md) | 2026.1 | draft |

| Configuration pattern | Releases | Status |
|---|---|---|
| [Laying out the configuration. Inventory, globals.yml, per-service overrides and passwords.yml, kept in git without secrets](configuration/inventory-globals-overrides-and-secrets-layout.md) | 2026.1 | lab-verified |
| [Rotating the secrets in passwords.yml. Which ones a reconfigure applies, which need manual steps, and which need an outage](configuration/rotating-secrets-in-passwords-yml.md) | 2026.1 | draft |

| Comparison | Releases | Status |
|---|---|---|
| [ML2/OVS versus ML2/OVN with Kolla-Ansible. What changes in globals.yml, in operations and at upgrade time](comparisons/ml2-ovs-versus-ml2-ovn-with-kolla-ansible.md) | 2026.1 | draft |
| [Single control node versus three controllers. What high availability really costs in Kolla-Ansible](comparisons/single-control-node-versus-three-controllers.md) | 2026.1 | draft |

| Version notes | Releases | Status |
|---|---|---|
| [From 2025.1 to 2026.1. What changes for a Kolla-Ansible deployment, from the release notes](version-notes/2025.1-to-2026.1.md) | 2025.1, 2026.1 | draft |

| Experience report | Releases | Status |
|---|---|---|
| [Upgrading an all-in-one lab from 2025.1 to 2026.1 with Kolla-Ansible, what happened](version-notes/upgrade-report-2025.1-to-2026.1-all-in-one-lab.md) | 2025.1, 2026.1 | draft |

## First pages to write

1. Three controllers, OVN, Ceph RBD: a common production baseline, with its consequences.
2. Separated storage and storage-replication networks: why, and what it costs.
3. Keeping lab, pre-production and production inventories aligned: one layout that worked.
4. Deployment errors: the five most common `kolla-ansible deploy` failures and what they meant.

## Contributing

- **Ask a question or compare approaches:** [open a Q&A discussion](https://github.com/OzarLand/ozarland/discussions/new?category=q-a)
  in `ozarland`, where all OzarLand discussions live.
- **Share your pattern:** copy `templates/deployment-pattern.md`, open a pull request. Describe your context first.
- **Confirm a page, add a variant or a consequence to it:** open a pull request or a
  [*Documentation proposal* issue](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml).
- **Report a deployment error you solved:** open an issue with the
  [*Problem solved* form](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=problem-solved.yml).

Rules that are not optional: no secrets (never `passwords.yml` content), no real hostnames, addresses or
unredacted logs; say where your pattern runs and for how long. Full guide: [CONTRIBUTING.md](CONTRIBUTING.md).

## License

All content, documentation and code alike, is licensed under the Apache License 2.0 ([LICENSE](LICENSE)). Reuse it
freely with attribution. See [licensing](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/licensing.md).

OpenStack, Kolla and Kolla-Ansible are projects of the OpenInfra Foundation community. OzarLand is an independent
community initiative and is not affiliated with or endorsed by the OpenInfra Foundation.
