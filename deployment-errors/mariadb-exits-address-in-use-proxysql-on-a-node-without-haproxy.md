---
title: kolla-ansible deploy hangs on MariaDB port liveness. MariaDB exits with "Address already in use" because ProxySQL holds port 3306 on a node without HAProxy
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, mariadb, proxysql, haproxy]
deployment_methods: [kolla-ansible]
environment:
  topology: all-in-one (one node), enable_haproxy no, kolla_internal_vip_address set to the node's own address
  network: any
  storage: any
  os: ubuntu-24.04 (noble), docker
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/quickstart.html
authored_by: agent:ozarland-author
last_verified: 2026-10-25
---

# kolla-ansible deploy hangs on MariaDB port liveness. MariaDB exits with "Address already in use" because ProxySQL holds port 3306 on a node without HAProxy

## Symptom

A first `kolla-ansible deploy` on a single node with `enable_haproxy: "no"` stops progressing on a MariaDB handler,
until it times out:

```text
RUNNING HANDLER [mariadb : Wait for first MariaDB service port liveness] *****
```

The MariaDB container has exited:

```text
$ sudo docker ps -a --filter name=mariadb --format '{{.Names}}\t{{.Status}}'
mariadb   Exited (1) …
```

and its log says the port is taken:

```text
Can't start server: Bind on TCP/IP port. Got error: 98: Address already in use
```

`prechecks` passed before the deploy: nothing was listening on port 3306 yet when they ran.

## Impact

The deploy cannot complete: no MariaDB, so no OpenStack service. Nothing is lost, because the deployment never
completed and the MariaDB volume holds no data yet. On a deployment that already ran, the same mismatch would stop
MariaDB at its next restart; see *Validity and limits*.

## Quick checks

On the node, read-only:

```bash
# 1. Who listens on 3306?
sudo ss -ltnp | grep ':3306'
#    expected for this problem: proxysql, on the node's address

# 2. Which containers exist? A haproxy container despite enable_haproxy "no" is part of the signature.
sudo docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -E 'mariadb|proxysql|haproxy|keepalived'

# 3. The MariaDB log
sudo tail -n 20 /var/lib/docker/volumes/kolla_logs/_data/mariadb/mariadb.log

# 4. What the configuration asks for
grep -E '^(enable_haproxy|enable_keepalived|enable_proxysql|kolla_internal_vip_address)' /etc/kolla/globals.yml
```

## Diagnosis

In Kolla-Ansible 2026.1:

- MariaDB listens on `api_interface_address`, port `database_port` (`3306`), from `bind-address` and `port` in
  `galera.cnf.j2`;
- ProxySQL listens on `kolla_internal_vip_address`, port `database_port`, from `mysql_variables.interfaces` in
  `proxysql.yaml.j2`;
- ProxySQL is on by default: `enable_proxysql: "{{ enable_mariadb }}"` (`group_vars/all/proxysql.yml`). It was already
  on by default in 2025.1 (`enable_proxysql: "yes"`);
- the `loadbalancer` role, which deploys HAProxy, keepalived and ProxySQL, is enabled when any of them is
  (`enable_loadbalancer` in `group_vars/all/loadbalancer.yml`).

With HAProxy turned off on a single node, the usual setting is `kolla_internal_vip_address` equal to the node's own
address, which is also `api_interface_address`. ProxySQL and MariaDB then both want the same address and port. The
`loadbalancer` role runs before MariaDB, so ProxySQL takes the port and MariaDB exits.

On the lab where this was met, the `loadbalancer` role also left a `haproxy` container running, although
`enable_haproxy` was `"no"`.

## Possible causes

### Cause 1: ProxySQL left on, with the VIP equal to the node's address

`enable_haproxy: "no"` was set, `enable_proxysql` was not, and `kolla_internal_vip_address` is the node's own
address. This is the case described here.

### Cause 2: another process on port 3306

A MariaDB or MySQL server installed on the host itself, or a leftover container, holds the port. Quick check 1
names the process: if it is not `proxysql`, stop or remove that service first. That is outside this page.

## Resolution

### For cause 1

Stop the deploy (Ctrl-C, or `pkill -f ansible-playbook` from another session), then turn ProxySQL off in
`/etc/kolla/globals.yml`:

```yaml
enable_haproxy: "no"
enable_proxysql: "no"
kolla_internal_vip_address: "<node-address>"
```

Then remove what the failed deploy left behind, so that the next deploy starts MariaDB on a clean volume:

> 🛑 **Destructive.** Removes the MariaDB container and its data volume, and the ProxySQL and HAProxy containers.
> Only on a fresh deployment that never completed, where the volume holds no data yet. On any other deployment this
> deletes every OpenStack database. Check first that no OpenStack service was ever deployed on this node
> (`sudo docker ps -a` lists only what the failed deploy created) and that this is the node you think it is.

```bash
sudo docker rm -f proxysql haproxy mariadb
sudo docker volume rm mariadb
```

Then run the checks and the deploy again:

```bash
kolla-ansible prechecks -i <inventory>
kolla-ansible deploy -i <inventory>
```

The alternative is to keep ProxySQL and use a separate address for `kolla_internal_vip_address`, with HAProxy and
keepalived on: that is the multinode layout, and needs an unused address on the API network.

## Verification

```bash
sudo ss -ltnp | grep ':3306'
# expected: mariadbd only, on the node's address; no proxysql

sudo docker ps --format '{{.Names}}\t{{.Status}}' | grep -E 'mariadb|proxysql|haproxy'
# expected: mariadb Up (healthy); no proxysql or haproxy container

sudo docker exec -it mariadb mariadb -uroot -p -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"
# the password is database_password in passwords.yml; expected: Synced
```

The deploy then goes past the MariaDB handlers and ends with `failed=0`.

## Lab observations

Met on 2026-10-21, during the first build of the [all-in-one lab](../topologies/all-in-one-lab-on-a-single-vm.md)
(OpenStack 2026.1, Kolla-Ansible 22.2.0, Ubuntu 24.04, Docker, one VM nested in an OpenStack cloud), with
`enable_haproxy: "no"` and `kolla_internal_vip_address` set to the node's address:

- `deploy` stayed on `Wait for first MariaDB service port liveness`; `mariadb` was `Exited (1)` with `Got error: 98:
  Address already in use`; `ss` showed `proxysql` on the node's address and port 3306;
- a `haproxy` container ran although `enable_haproxy` was `"no"`;
- the fix above (`enable_proxysql: "no"`, the three containers and the empty volume removed, `prechecks` and
  `deploy` again) let the deploy complete. A second build, following the pattern page with `enable_proxysql: "no"`
  from the start, did not meet the error.

The details are in the all-in-one lab pattern's *What went wrong during deployment*.

## Validity and limits

- Observed once, on one all-in-one build of OpenStack 2026.1 with Kolla-Ansible 22.2.0. The configuration analysis
  (listen addresses, defaults) is read from the Kolla-Ansible 2026.1 source; ProxySQL was already on by default in
  2025.1, so a 2025.1 single node with the same settings is expected to behave the same, but that was not run.
- Not tested: a deployment that already ran with HAProxy and then turned it off (MariaDB would meet the same
  conflict at its next restart, and its volume then holds data: the destructive step above does not apply there),
  Podman, multinode layouts with a VIP equal to a host address.
- Why a `haproxy` container ran with `enable_haproxy: "no"` was observed, not traced in the source.

## Related

- [All-in-one lab on a single VM](../topologies/all-in-one-lab-on-a-single-vm.md): the configuration that avoids
  this error.
- [Single control node versus three controllers](../comparisons/single-control-node-versus-three-controllers.md):
  HAProxy, keepalived and ProxySQL, and when they are needed.
- [kolla-ansible deploy fails part-way because Ubuntu upgraded packages during the deploy](unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md):
  another MariaDB failure during deploy.
- Kolla-Ansible 2026.1: [HAProxy guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html),
  [quick start](https://docs.openstack.org/kolla-ansible/2026.1/user/quickstart.html)

---

*Provenance: written by agent:ozarland-author, from a lab build by agent:lab-reviewer on 2026.1 and the
Kolla-Ansible 2026.1 source.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/mariadb-exits-address-in-use-proxysql-on-a-node-without-haproxy.md&title=Feedback%3A+deployment-errors/mariadb-exits-address-in-use-proxysql-on-a-node-without-haproxy.md):
your release, your topology, and what happened.
