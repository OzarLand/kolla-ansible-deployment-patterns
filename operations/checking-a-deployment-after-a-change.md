---
title: Checking a Kolla-Ansible deployment after a change. kolla-ansible check, cluster state, service lists and one real instance
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, mariadb, rabbitmq, nova, neutron, keystone, docker]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for production after a deploy, reconfigure, upgrade or host change
  scale: any
  network: ml2/ovs or ml2/ovn
  storage: any
  os: any supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/troubleshooting.html
  - https://docs.openstack.org/nova/2026.1/cli/nova-status.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Checking a Kolla-Ansible deployment after a change

## Context

A `kolla-ansible` run that ends with `failed=0` says that every task passed, not that the cloud works: a container
can be up and its service unable to reach RabbitMQ, an agent can be registered and dead, and the API can answer while
instances cannot boot. After every change (a `deploy`, a `reconfigure`, an upgrade, a host added or removed), an
operator runs the same short list of checks, from the cheapest to the most realistic, and records the result.

This page gives that list for Kolla-Ansible 2026.1, in the order to run it. Everything is read-only except the test
instance at the end.

## Topology

Run from the deploy host (Kolla-Ansible and OpenStack client with admin credentials) and, for the cluster checks, on
one controller.

## Network design

Not affected.

## Storage design

Not affected.

## Inventory and configuration layout

Keep the output of each check with the change record (the commit of `/etc/kolla` and the run's log). Comparing with
the same checks taken before the change is what makes a difference visible.

## The checks, in order

**1. Kolla-Ansible's own container check.** `kolla-ansible check` runs each role's `check` tasks: for every enabled
service it lists the containers that should run on each host and fails if one is missing, stopped or unhealthy
(`ansible/roles/service-check`). Some roles add their own probe, for example MariaDB through the VIP.

```bash
kolla-ansible check -i <inventory>
```

On a problem it fails with one of these messages (from the `service-check` role, 2026.1):

```text
The following <service> containers are missing or not running: <containers>
The following <service> containers are unhealthy: <containers>
```

**2. Containers on each host,** for what `check` does not cover (restarts that recovered, recent exits):

```bash
sudo docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -v 'Up .* (healthy)'
# what remains: containers without a health check, restarting, exited or unhealthy; read each one
```

**3. The clustered services,** on one controller:

```bash
# Galera: cluster size, Primary, Synced; -p prompts for database_password from passwords.yml
sudo docker exec -it mariadb mariadb -uroot -p -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment');"

# RabbitMQ: every node running, no partitions, no alarms
sudo docker exec rabbitmq rabbitmqctl cluster_status
```

**4. The OpenStack services and agents,** from the deploy host:

```bash
openstack token issue                        # Keystone answers through the VIP
openstack endpoint list --interface internal # the expected protocol (http or https) and names
openstack compute service list               # every service up; disabled ones are expected and known
openstack network agent list                 # every agent alive (ML2/OVS), or every OVN controller up (ML2/OVN)
openstack volume service list                # if Cinder is deployed
openstack resource provider list             # one per hypervisor (osc-placement plugin)
sudo docker exec nova_api nova-status upgrade check   # on a controller; 0 = all checks passed, 1 = warnings
```

**5. The logs since the change,** on each host, for errors the checks above do not show:

```bash
sudo find /var/log/kolla -name '*.log' -newermt '<time of the change>' -exec grep -l -E ' (ERROR|CRITICAL) ' {} +
```

Read the files it lists; an `ERROR` right after a restart (a reconnection) can be normal, one that repeats is not.

**6. One real instance,** the only test that crosses scheduler, placement, network, storage and metadata:

> ⚠️ **State-changing.** Creates, then deletes, one small instance in a test project, and a floating IP if you test
> external access. Uses quota and a little capacity for a few minutes. Undo: the delete commands below.

```bash
openstack server create --flavor <small-flavor> --image <image> --network <network> --wait check-after-change
openstack server show check-after-change -c status -c addresses -c OS-EXT-SRV-ATTR:host
openstack console log show check-after-change | tail -20     # cloud-init reached the metadata service
openstack server delete --wait check-after-change
```

When the change touched a single host or service, target it: a host with the administrator syntax
`--availability-zone nova:<host>`, a volume back-end with a boot-from-volume instance, the external network with a
floating IP.

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Cheapest checks first | Only a test instance | Finds the broken layer before the slow test | Six steps instead of one |
| `kolla-ansible check` | Only `docker ps` | Knows which containers *should* run on each host | Fails on the first role with a problem; read the whole output |
| A real instance | Trust the service lists | Agents can be listed alive while the data path is broken | A few minutes and a little quota |
| Same checks before and after | After only | A difference is only visible against a baseline | Run them once before the change too |

## What went wrong during deployment

Nothing observed for this page. Typical findings of these checks: a network agent reported down after a network change,
a `nova-compute` service down on a host whose `nova_compute` container restarted but cannot reach RabbitMQ, a Galera
node not `Synced` after a reboot, `401` from Keystone on some calls only (Fernet keys that differ between Keystone
hosts; see the Keystone logs).

## Version notes

`kolla-ansible check` and the `service-check` role are those of Kolla-Ansible 2026.1. `nova-status upgrade check`
return codes are documented by Nova 2026.1 (0 success, 1 warnings, 2 or more failures).

## Lessons learned

- `failed=0` is the start of the checks, not the end.
- Keep a baseline: the same commands' output from a quiet day.
- Delete the test instance even when it failed, and keep its fault message (`openstack server show`) for the record.

## Validity and limits

- Written from the Kolla-Ansible 2026.1 code (`kolla_ansible/cli/commands.py` `check`, `ansible/roles/service-check/`,
  `ansible/roles/mariadb/tasks/check.yml`) and documentation, and the Nova `nova-status` documentation. Not run on a
  lab for this page. Status `draft`.
- Not covered: monitoring-based checks (Prometheus, Grafana), Octavia, Designate and other optional services, and
  Podman (replace `docker`).

## Related

- [deploy, reconfigure, upgrade and the others](deploy-reconfigure-upgrade-and-limiting-runs.md)
- [Adding a compute node](adding-a-compute-node.md), [removing or replacing a compute node](removing-or-replacing-a-compute-node.md)
- [From 2025.1 to 2026.1](../version-notes/2025.1-to-2026.1.md): pre- and post-upgrade checks
- Kolla-Ansible 2026.1: [operating Kolla](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html),
  [troubleshooting](https://docs.openstack.org/kolla-ansible/2026.1/user/troubleshooting.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 code and documentation and the Nova
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=operations/checking-a-deployment-after-a-change.md&title=Feedback%3A+operations/checking-a-deployment-after-a-change.md):
your release, your topology, and what happened.
