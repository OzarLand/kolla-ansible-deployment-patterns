---
title: kolla-ansible deploy fails with "The following Nova compute services failed to register themselves"
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, nova, libvirt, rabbitmq, placement, docker]
deployment_methods: [kolla-ansible]
environment:
  topology: any with libvirt/KVM computes; first deployment or a compute added later
  network: any
  storage: any
  os: any host OS supported by the Kolla-Ansible release
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/adding-and-removing-hosts.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/compute/nova-guide.html
  - https://docs.openstack.org/nova/2026.1/admin/cells.html
  - https://docs.openstack.org/nova/2026.1/admin/troubleshooting/troubleshooting.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# kolla-ansible deploy fails with "The following Nova compute services failed to register themselves"

## Symptom

`kolla-ansible deploy` (often with `--limit` on a new compute) retries a task in the `nova-cell` role for about 200
seconds, then fails on one or more compute hosts:

```text
TASK [nova-cell : Waiting for nova-compute services to register themselves] ****
FAILED - RETRYING: [compute-03 -> controller-1]: Waiting for nova-compute services to register themselves (20 retries left).
...
TASK [nova-cell : Fail if nova-compute service failed to register] ****
fatal: [compute-03]: FAILED! => {"changed": false, "msg": "The following Nova compute services failed to register themselves: compute-03"}
```

The task names and the message are those of `ansible/roles/nova-cell/tasks/wait_discover_computes.yml` in
Kolla-Ansible 2026.1; the retry lines follow Ansible's usual format and are reconstructed, not copied from a run. The
name after the colon is the host's node name (`ansible_facts.nodename`), which may be a full name.

## Impact

The host is not added to Nova: no instance can be scheduled on it, and the host discovery step
(`nova-manage cell_v2 discover_hosts`) has not run for it. The other hosts of the run are not failed unless
`nova_compute_registration_fatal` is true. Running instances elsewhere are not affected.

## Quick checks

From the deploy host or a controller, with admin credentials:

```bash
# 1. Which compute services exist, under which host names?
openstack compute service list --service nova-compute

# 2. What Kolla-Ansible expected: the node name of the failed host
ansible -i <inventory> compute-03 -m setup -a 'filter=ansible_nodename'
```

On the failed compute host:

```bash
# 3. Are the containers running?
sudo docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -E 'nova_compute|nova_libvirt|nova_ssh'

# 4. What nova-compute says
sudo tail -n 100 /var/log/kolla/nova/nova-compute.log
sudo docker logs --tail 50 nova_compute
```

## Diagnosis

Kolla-Ansible starts `nova_compute`, then asks the Nova API, from a cell conductor host, for the list of
`nova-compute` services, 20 times 10 seconds apart, until it contains the node name of every compute in the run. A
service appears in that list once `nova-compute` has started, connected to libvirt and to RabbitMQ, and written its
service record through the conductor. Then Kolla-Ansible runs `nova-manage cell_v2 discover_hosts --by-service` to
map the host to its cell.

So the failure means one of: `nova-compute` never got that far, or it registered under another name. The log tells
which:

| In `nova-compute.log` | Points to |
|---|---|
| `Connection to libvirt failed` | libvirt (cause 1) |
| `AMQP server on <address>:5672 is unreachable` | RabbitMQ (cause 2) |
| The service starts normally, and `openstack compute service list` shows the host under another name | a host name mismatch (cause 3) |
| Errors about the placement API or Keystone (connection refused, `401`, certificate errors) | the API endpoints from the compute (cause 4) |

`nova-compute` logs to `/var/log/kolla/nova/nova-compute.log` (Kolla's `log_dir`); `docker logs nova_compute` shows
the container's start-up. libvirt logs to `/var/log/kolla/libvirt/` and `docker logs nova_libvirt`.

## Possible causes

### Cause 1: nova-compute cannot connect to libvirt

`nova_libvirt` is not running or restarts, often because hardware virtualisation is missing (no `/dev/kvm`, nested
virtualisation off on a lab VM) or because a libvirt on the host itself holds the socket. Tell it apart: `docker ps`
shows `nova_libvirt` restarting, or the log line above.

### Cause 2: nova-compute cannot reach RabbitMQ

The compute cannot open the RabbitMQ port of the controllers: a firewall, a wrong `network_interface` on this host,
an MTU problem on the path. Tell it apart: the AMQP line repeats with `Trying again in N seconds`.

### Cause 3: the service registered under another name

Nova registers a libvirt compute under its `[DEFAULT] host` option, which Kolla-Ansible leaves at Nova's default (the
host name of the machine), while Kolla-Ansible waits for `ansible_facts.nodename`. A `host =` set in a
`/etc/kolla/config/nova.conf` or `nova/nova-compute.conf` override, or a host renamed after its facts were gathered
(fact cache), makes the two differ. Tell it apart: `openstack compute service list` shows a service for this host
under another name, `up`.

### Cause 4: nova-compute cannot reach the API endpoints it needs

It needs Keystone and Placement through the internal endpoint. A wrong `kolla_internal_fqdn` resolution on the
compute, or a CA it does not trust with internal TLS, stops it. Tell it apart: placement or Keystone errors in the
log.

## Resolution

Fix the cause on the host, then run the same deploy again: it is idempotent, waits again, and runs the discovery.

### For cause 1

Make hardware virtualisation available (`ls -l /dev/kvm` on the host), or, on a lab without it, set
`nova_compute_virt_type: "qemu"` in `globals.yml` (much slower instances; labs only). Stop and disable any
libvirt installed on the host itself.

> ⚠️ **State-changing.** Stopping a host libvirt stops any virtual machine it runs; on a new compute there should be
> none. Undo: `systemctl enable --now libvirtd`.

```bash
sudo systemctl disable --now libvirtd
```

### For cause 2

Open the path from the compute to the RabbitMQ port of every controller, on the `api_interface` network, and check
the MTU end to end (see [network interfaces, VLANs and MTU](../networking/interfaces-vlans-and-mtu-ovs-and-ovn.md)).

### For cause 3

Remove the `host =` override, or keep it consistently on purpose and know that Kolla-Ansible's wait will not see it.
Clear a stale fact cache if the host was renamed. If a wrong service record was created, delete it once the right
one exists.

> ⚠️ **State-changing.** Deletes a compute service record and, with it, its compute node record in Nova. Only for a
> record with no instances: check with `openstack server list --all-projects --host <wrong-name>` first. Cannot be
> undone; the service registers again if the `nova-compute` that created it is still running under that name.

```bash
openstack compute service delete <service-id>
```

### For cause 4

Fix the name resolution of `kolla_internal_fqdn` on the compute, or the CA trust (`kolla_copy_ca_into_containers`,
`openstack_cacert`), then deploy again.

### Then, for every cause

> ⚠️ **State-changing.** Deploys again to the failed host, (re)starts its compute containers and registers it.

```bash
kolla-ansible deploy -i <inventory> --limit compute-03
```

## Verification

```bash
openstack compute service list --service nova-compute --host <node-name>   # up, enabled
openstack hypervisor list                                                   # the host is listed
```

The deploy goes past *Waiting for nova-compute services to register themselves* and ends with `failed=0`.

## Validity and limits

- Written from the Kolla-Ansible 2026.1 code (`ansible/roles/nova-cell/tasks/wait_discover_computes.yml`,
  `discover_computes.yml`, `templates/nova.conf.j2`, `defaults/main.yml`) and the Nova documentation. Not reproduced
  on a lab for this page. Status `draft`.
- The log lines in the diagnosis table are the messages of Nova's libvirt driver and of oslo.messaging's RabbitMQ
  driver; their exact wording can differ between releases.
- Not covered: Ironic compute services (they register under `<host>-ironic` or a custom name), the fake driver, and
  cells deployments with several conductors groups.

## Related

- *Adding a compute node to a running deployment* (`operations/adding-a-compute-node.md`, proposed in pull request 30)
- Kolla-Ansible 2026.1: [adding and removing hosts](https://docs.openstack.org/kolla-ansible/2026.1/user/adding-and-removing-hosts.html),
  [Nova guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/compute/nova-guide.html)
- Nova 2026.1: [cells v2](https://docs.openstack.org/nova/2026.1/admin/cells.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 code and documentation and the Nova
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/nova-compute-services-failed-to-register.md&title=Feedback%3A+deployment-errors/nova-compute-services-failed-to-register.md):
your release, your topology, and what happened.
