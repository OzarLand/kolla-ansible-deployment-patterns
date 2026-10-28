---
title: RabbitMQ nodes do not form a cluster during kolla-ansible deploy. Host name resolution, the Erlang cookie, closed ports or old cluster state
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, rabbitmq, docker]
deployment_methods: [kolla-ansible]
environment:
  topology: two or more hosts in the rabbitmq group (by default the controllers)
  network: any
  storage: any
  os: any host OS supported by the Kolla-Ansible release
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/message-queues/rabbitmq.html
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/password-rotation.html
  - https://www.rabbitmq.com/docs/clustering
  - https://www.rabbitmq.com/docs/networking
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# RabbitMQ nodes do not form a cluster during kolla-ansible deploy

## Symptom

One of these, on a first deployment, after a controller was added or reinstalled, or after `passwords.yml` changed:

- `kolla-ansible prechecks` fails with `Hostname has to resolve uniquely to the IP address of api_interface` (task
  *Check if each rabbit hostname resolves uniquely to the proper IP address* of the `rabbitmq` role);
- `kolla-ansible deploy` fails or times out at the `rabbitmq` task *Waiting for rabbitmq to start* (`rabbitmqctl wait
  --timeout 60`), or the `rabbitmq` container restarts in a loop;
- the deploy passes, but `rabbitmqctl cluster_status` on each node lists only itself, or not all controllers, and the
  OpenStack services log AMQP errors.

The task names and the message are those of `ansible/roles/rabbitmq/tasks/precheck.yml` and `restart_services.yml` in
Kolla-Ansible 2026.1.

## Impact

Every OpenStack service talks over RabbitMQ. With a split or incomplete cluster, RPC calls time out or go to a node
the other services do not see: instances stuck in `BUILD`, agents reported down, API calls that hang. With
`cluster_partition_handling = pause_minority` (Kolla-Ansible's default), nodes in a minority pause themselves.

## Quick checks

On each RabbitMQ host, read-only:

```bash
# 1. Cluster membership as this node sees it
sudo docker exec rabbitmq rabbitmqctl cluster_status

# 2. The node name RabbitMQ uses, and what the short host name resolves to
hostname -s
getent ahostsv4 "$(hostname -s)"

# 3. The container and its log
sudo docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -w rabbitmq
sudo ls /var/log/kolla/rabbitmq/
sudo tail -n 50 /var/log/kolla/rabbitmq/rabbit@$(hostname -s).log 2>/dev/null || sudo docker logs --tail 50 rabbitmq
```

Between RabbitMQ hosts, read-only:

```bash
# 4. Are the Erlang ports reachable? 4369 (epmd) and 25672 (inter-node), on the api_interface addresses
nc -zv -w 3 <other-controller-api-address> 4369
nc -zv -w 3 <other-controller-api-address> 25672
```

## Diagnosis

How Kolla-Ansible 2026.1 builds the cluster, from the `rabbitmq` role:

- each node is named `rabbit@<short host name>` (`RABBITMQ_NODENAME` in `rabbitmq-env.conf.j2`, from
  `ansible_facts.hostname`);
- the cluster members are listed statically in `rabbitmq.conf` (`cluster_formation.peer_discovery_backend =
  rabbit_peer_discovery_classic_config`, one `rabbit@<host>` per host of the group);
- nodes authenticate each other with the Erlang cookie, `RABBITMQ_CLUSTER_COOKIE` in the container environment, from
  `rabbitmq_cluster_cookie` in `passwords.yml`;
- they talk through epmd (4369) and the inter-node port (`rabbitmq_cluster_port`, 25672), on the `api_interface`
  address (the precheck requires the short host name to resolve to it, and only to it);
- RabbitMQ logs to `/var/log/kolla/rabbitmq/` (`RABBITMQ_LOG_BASE`).

So a node that cannot resolve the others' short names to their API addresses, cannot reach their ports, presents a
different cookie, or remembers an older cluster, does not join. The node's log says which; its exact wording depends
on the RabbitMQ and Erlang versions, so look for messages about name resolution (`nxdomain`), refused or timed-out
connections, a cookie or authentication refusal, or an inconsistent cluster.

## Possible causes

### Cause 1: the short host names do not resolve to the API addresses

The host's own name resolves first to another address (for example a loopback line other than `localhost` that
some distributions add to `/etc/hosts`, or a management address from DNS), or a controller's name does not resolve on
the others. The precheck catches it when it runs. Tell it apart: `getent ahostsv4 <short-name>` (`ahostsv6` when
`api_address_family` is `ipv6`, as the precheck does) on each host does not return exactly that host's
`api_interface` address.

### Cause 2: the Erlang ports are closed between the controllers

A host firewall or a network ACL blocks 4369 or 25672 on the API network. Tell it apart: the `nc` checks fail; the log
shows connection timeouts or refusals to the other nodes.

### Cause 3: the nodes do not share the same Erlang cookie

Different `passwords.yml` used for different runs, or `rabbitmq_cluster_cookie` changed without the procedure of the
password rotation guide (which stops all services and resets RabbitMQ). Tell it apart: `sudo docker exec rabbitmq env
| grep RABBITMQ_CLUSTER_COOKIE | sha256sum` differs between hosts (compare the hashes, never paste the cookie); the
log shows connection attempts refused for authentication.

### Cause 4: a node keeps the state of an old cluster

A controller was reinstalled or renamed, or its `rabbitmq` volume comes from an earlier deployment: its stored
membership does not match the others'. Tell it apart: `cluster_status` on that node lists members that no longer
exist, or the others list it as a stopped member; the log reports an inconsistent cluster.

## Resolution

### For cause 1

Make each short host name resolve to its `api_interface` address on every RabbitMQ host. `kolla-ansible
bootstrap-servers` writes `/etc/hosts` entries for the hosts in its run (`customize_etc_hosts`); otherwise fix DNS or
`/etc/hosts` yourself, and remove lines that map the host's own name to another address.

> ⚠️ **State-changing.** Changes host name resolution on the host; other software on it that relies on the old line
> (rare on a dedicated controller) sees the new address. Undo: restore the previous `/etc/hosts`.

```bash
sudo cp /etc/hosts /etc/hosts.before-rabbitmq-fix
sudoedit /etc/hosts
getent ahostsv4 "$(hostname -s)"      # expected: only the api_interface address
```

A running `rabbitmq` container may keep the resolution it had when it was created (with host networking, Docker
gives the container its own copy of the host's `/etc/hosts`). Check inside it too:

```bash
sudo docker exec rabbitmq getent ahostsv4 <short-name>
```

If it still sees the old entry, the container must be recreated. The deploy does it when its configuration changed;
otherwise remove it and let the deploy create it again, one RabbitMQ host at a time. The `--limit` below is the
narrowest way to recreate one node; the operating guide advises against `--limit` in general (see
[deploy, reconfigure, upgrade and the others](../operations/deploy-reconfigure-upgrade-and-limiting-runs.md) for the
trade-off).

> ⚠️ **State-changing.** Removes the `rabbitmq` container of this host and recreates it: this node leaves the cluster
> until it is back, and the services connected to it reconnect to the other nodes. Its data stays in the `rabbitmq`
> volume. Precondition: the other nodes form a healthy cluster (`cluster_status` on them lists each other as
> running); one host at a time. Undo: the deploy below recreates it from the same volume.

On Kolla-Ansible 2026.1 the container is run by the systemd unit `kolla-rabbitmq-container.service`: stop the unit
first, or it starts the container again while you remove it. The unit stays enabled, and the deploy starts it again
with the new container.

```bash
# on this host
sudo systemctl stop kolla-rabbitmq-container.service
sudo docker rm rabbitmq                                   # or: sudo podman rm rabbitmq
# from the deploy host
kolla-ansible deploy -i <inventory> --tags rabbitmq --limit <host>
```

Then run `kolla-ansible prechecks`, and the full run that failed again, without `--limit`.

### For cause 2

Open TCP 4369 and 25672 (and the AMQP port, 5672 or 5671 with TLS) between the RabbitMQ hosts on the API network,
then run the deploy again.

### For cause 3

Use the one `passwords.yml` of the deployment for every run. If the cookie was changed on purpose, finish the
procedure of the password rotation guide for `rabbitmq_cluster_cookie`; it is an outage (see
[rotating the secrets in passwords.yml](../configuration/rotating-secrets-in-passwords-yml.md)).

### For cause 4

Reset the RabbitMQ state of the node that holds the old membership, so that it joins the current cluster as a new
member.

> 🛑 **Destructive.** Removes this node's RabbitMQ data: its queues and the messages in them, its view of the cluster.
> On a running platform, messages queued only on that node are lost and the services' RPC calls through it fail
> until it rejoins. Precondition: the other nodes form a healthy cluster (`cluster_status` on them lists each other
> as running), and you are on the node with the stale state. Cannot be undone. Copy to take first: the definitions
> (users, vhosts, policies), with `rabbitmqctl export_definitions`; messages in flight cannot be saved.

```bash
# on the node with the stale state only
sudo docker exec rabbitmq rabbitmqctl export_definitions /tmp/definitions-before-reset.json   # read-only for the cluster
sudo docker cp rabbitmq:/tmp/definitions-before-reset.json .                                # keep the copy on the host
sudo docker exec rabbitmq rabbitmqctl stop_app
sudo docker exec rabbitmq bash -c 'rabbitmqctl force_reset || rabbitmqctl reset'
sudo docker exec rabbitmq rabbitmqctl start_app
sudo docker exec rabbitmq rabbitmqctl await_online_nodes <number of rabbitmq hosts>
```

`force_reset` first, then `reset`: this is the order Kolla-Ansible's own `reset-state.yml` uses. A plain `reset` can
refuse on a node whose stored membership names peers it cannot reach; `force_reset` does not exist with Khepri, the
default metadata store from RabbitMQ 4.2 (release note of bug 2150536), hence the fallback.

If a controller was renamed or replaced under a new name, the healthy nodes may still list its old name as a
stopped member. Remove it from one healthy node, while the old name is listed as stopped. (If the replacement host
reuses the old name, do not forget it: reset that node as above instead.)

> 🛑 **Destructive.** Removes the old member from the cluster's membership; any queue whose only replica was on it is
> lost. Precondition: the old node is gone for good (reinstalled or renamed) and is not running. Cannot be undone; the
> node can only join again as a new member.

```bash
sudo docker exec rabbitmq rabbitmqctl forget_cluster_node rabbit@<old-short-name>
```

`kolla-ansible rabbitmq-reset-state` resets **every** RabbitMQ node at once (`reset-state.yml`: `stop_app`,
`force_reset` or `reset`, `start_app`, `await_online_nodes`); it is for the cases the RabbitMQ guide describes, not
for one bad node, and it is a full messaging outage.

## Verification

```bash
# on each RabbitMQ host
sudo docker exec rabbitmq rabbitmqctl cluster_status
# expected: every rabbit@<controller> under running nodes, no partitions, no alarms
```

Then the deploy goes past *Waiting for rabbitmq to start* and ends with `failed=0`, and the OpenStack agents report
`alive` again (`openstack network agent list`, `openstack compute service list`).

## Validity and limits

- Written from Kolla-Ansible 2026.1 code (`ansible/roles/rabbitmq/tasks/precheck.yml`, `restart_services.yml`,
  `reset-state.yml`, `templates/rabbitmq.conf.j2`, `templates/rabbitmq-env.conf.j2`, `defaults/main.yml`,
  `ansible/group_vars/all/rabbitmq.yml`) and RabbitMQ's clustering and networking documentation. Not reproduced on a
  lab for this page. Status `draft`.
- RabbitMQ and Erlang log lines are described, not quoted; their wording changes between versions. The log file name
  under `/var/log/kolla/rabbitmq/` depends on the node name.
- Not covered: RabbitMQ TLS between nodes, the RabbitMQ SLURP and version upgrade steps, quorum queue migration,
  network partitions on a running cluster (`pause_minority` behaviour), and external RabbitMQ.

## Related

- [Rotating the secrets in passwords.yml](../configuration/rotating-secrets-in-passwords-yml.md)
- [Adding a compute node](../operations/adding-a-compute-node.md) and the upstream guide's note that new controllers
  need `/etc/hosts` entries for all controllers
- Kolla-Ansible 2026.1: [RabbitMQ](https://docs.openstack.org/kolla-ansible/2026.1/reference/message-queues/rabbitmq.html),
  [password rotation](https://docs.openstack.org/kolla-ansible/2026.1/admin/password-rotation.html)
- RabbitMQ: [clustering](https://www.rabbitmq.com/docs/clustering), [networking](https://www.rabbitmq.com/docs/networking)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 code and documentation and the RabbitMQ
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/rabbitmq-nodes-do-not-form-a-cluster.md&title=Feedback%3A+deployment-errors/rabbitmq-nodes-do-not-form-a-cluster.md):
your release, your topology, and what happened.
