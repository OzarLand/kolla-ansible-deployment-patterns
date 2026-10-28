---
title: kolla-ansible fails with "MariaDB cluster exists but is stopped". Recovering a fully stopped Galera cluster with kolla-ansible mariadb-recovery
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [kolla-ansible, mariadb, docker]
deployment_methods: [kolla-ansible]
environment:
  topology: two or more hosts in the mariadb group (a Galera cluster); typically after a power loss or all controllers rebooted
  network: any
  storage: any
  os: any host OS supported by the Kolla-Ansible release
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers (Galera behind ProxySQL) + 2 computes, Docker", date: 2026-10-27, note: "cause 1 reproduced: the 3 mariadb containers stopped cleanly together (systemctl), reconfigure --tags mariadb failed with the message; quick checks, step 1 (copy; stop with systemctl), mariadb-recovery (2 min 32 s, highest seqno chosen), step 3 (needs sudo and 'Recovered position:') and verification run. Not run: crash with seqno -1, forced bootstrap with mariadb_recover_inventory_name, cause 2" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/mariadb-backup-and-restore.html
  - https://galeracluster.com/library/documentation/crash-recovery.html
  - https://mariadb.com/kb/en/getting-started-with-mariadb-galera-cluster/
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# kolla-ansible fails with "MariaDB cluster exists but is stopped"

## Symptom

After every controller was down at the same time (power loss, all rebooted, every `mariadb` container stopped),
MariaDB does not come back by itself, the OpenStack APIs return errors, and any `kolla-ansible deploy`,
`reconfigure` or `upgrade` stops in the `mariadb` role with:

```text
TASK [mariadb : Fail on existing but stopped cluster] **************************
[ERROR]: Task failed: Action failed: MariaDB cluster exists but is stopped. Please start it using kolla-ansible mariadb-recovery
Origin: ~/kolla-venv/share/kolla-ansible/ansible/roles/mariadb/tasks/lookup_cluster.yml:38:7
...
fatal: [controller-1]: FAILED! => {"changed": false, "msg": "MariaDB cluster exists but is stopped. Please start it using kolla-ansible mariadb-recovery"}
```

Captured on a lab (`kolla-ansible reconfigure --tags mariadb`, Kolla-Ansible 22.2.0), on every MariaDB host. Earlier
in the same run, one ignored failure per host is expected: `fatal: [controller-1]: FAILED! => {"changed": false,
"elapsed": 10, "msg": "Timeout when waiting for search string MariaDB in 192.0.2.11:3306"}`. Meanwhile the APIs
answered `Bad Gateway (HTTP 502)` (`openstack token issue`, `openstack server list`).
The task name and message are those of `ansible/roles/mariadb/tasks/lookup_cluster.yml` in Kolla-Ansible 2026.1.
The task fails when the inventory has more than one MariaDB host, every host already has a `mariadb` volume, and
none answers on the MariaDB port.

## Impact

Every OpenStack service that uses the database is down or failing: the API calls fail, nothing can be created,
changed or deleted. Running instances keep running. Data is not lost yet; the risk is in recovering the wrong way.

## Quick checks

On each MariaDB host, read-only:

```bash
# 1. Container state: exited, restarting, or running but not serving?
sudo docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -w mariadb

# 2. Galera's saved state on each node: seqno and safe_to_bootstrap
sudo cat /var/lib/docker/volumes/mariadb/_data/grastate.dat

# 3. The end of the MariaDB log
sudo tail -n 50 /var/log/kolla/mariadb/mariadb.log
```

From the deploy host, read-only, for each MariaDB host:

```bash
# 4. Does it answer on the MariaDB port (3306 by default, mariadb_port)?
nc -zv -w 3 <controller-api-address> 3306
```

On a lab where the three containers were stopped within a few seconds (`systemctl stop
kolla-mariadb-container.service`), check 1 showed `Exited (0)` everywhere and check 2 a `seqno` on every node, with `safe_to_bootstrap: 1` only on the
node with the highest `seqno` (the last to stop); check 4 answered `Connection refused`.

## Diagnosis

Galera keeps a node running only while it belongs to a primary component, a majority of the cluster. When every node
stopped, none of them may start alone: each waits for the others, and a plain restart of the containers does not
form a cluster. One node must be started as a new cluster (bootstrapped), and it must be the node with the most
recent data, the highest sequence number (`seqno`). The others then join it and copy what they miss.

`grastate.dat` gives that number when the node stopped cleanly. After a crash it reads `seqno: -1`, and the real
position must be recovered from the data files (`mysqld --wsrep-recover`, which writes `Recovered position:
<uuid>:<seqno>` in the log). `kolla-ansible mariadb-recovery` does exactly this on every node and picks the highest.

What to read in `mariadb.log` (messages of MariaDB and Galera; their exact wording depends on the version):
`WSREP: Recovered position`, errors about `safe_to_bootstrap`, `failed to reach primary view`, or a node waiting for
the others.

## Possible causes

### Cause 1: every node stopped at about the same time

Power loss, a simultaneous reboot of all controllers (for example during patching without a rolling order), or the
MariaDB containers stopped together (`kolla-ansible stop`). The cluster has no primary component left. This is the
case `mariadb-recovery` is made for (the operating guide: "used to recover a completely stopped mariadb cluster").

### Cause 2: some nodes are up but not in a primary component

If one node answers on the port but is not `Synced`, the same roles fail earlier with `MariaDB cluster is not synced.
Please wait for WSREP sync before proceeding.`, not with this message. Look at that node first
(`SHOW STATUS LIKE 'wsrep_%'`) before running a recovery that restarts every node.

## Resolution

### For cause 1

**1. Save the data of every node first.** The recovery bootstraps from one node and the others are then overwritten
by a state transfer from it.

> ⚠️ **State-changing.** Stops a `mariadb` container that is still restarting, then copies its data aside; no data is
> changed. Needs free space equal to the data size on each node. Undo: `sudo systemctl start
> kolla-mariadb-container.service` (the copy can stay).

On Kolla-Ansible 2026.1 the containers are run by systemd units: stop them with `systemctl`, or the unit starts the
container again. On the lab, the copy of a 2.3 GB data directory took about 30 s per node.

```bash
sudo docker ps -a --filter name=mariadb --format '{{.Names}}\t{{.Status}}'   # read-only
sudo systemctl stop kolla-mariadb-container.service    # only if it shows Restarting or Up
sudo cp -a /var/lib/docker/volumes/mariadb/_data /var/lib/mariadb-before-recovery-$(date +%F)
```

**2. Run the recovery.**

> 🛑 **Destructive.** Starts a new Galera cluster from one node's data. Kolla-Ansible stops MariaDB on every node,
> runs `--wsrep-recover` on each to find its position, picks the host with the highest `seqno`, sets
> `safe_to_bootstrap: 1` in its `grastate.dat`, starts it with `--wsrep-new-cluster`, then starts the others, which
> replace their data by a state transfer from it. Any transaction that only a node with a lower `seqno` had is lost.
> Precondition: step 1 done on every node; every MariaDB host reachable by Ansible (a host left out cannot be
> compared). Undo: only by restoring the copies of step 1 (stop all nodes, copy back, recover from the chosen node).

```bash
kolla-ansible mariadb-recovery -i <inventory>
```

On the lab (3 nodes, 2.3 GB of data, clean stop), the run took 2 min 32 s, chose the node with the highest `seqno`
and ended with `failed=0`; the APIs answered again without restarting them.

The role waits for the bootstrap node to be `OPERATIONAL`, for each other node to be `Synced` (the others replace
their data by an incremental or full state transfer), then restarts the bootstrap node normally, so that no node keeps
the bootstrap setting. The role's waits (`recover_cluster.yml`, stable/2026.1): for a joiner's port, 10 retries
6 seconds apart, each attempt waiting up to 60 seconds (`timeout: 60`), so up to about 11 minutes; for `Synced`,
30 retries 10 seconds apart, about 5 minutes. A full transfer of a large database can take longer. If the run times out on a joiner, read
`docker logs mariadb` there for the transfer's progress before running anything again.

**3. Check which host was chosen, and why** (read-only). The role writes the chosen host to
`/tmp/kolla_mariadb_recover_inventory_name_<shard>` on the deploy host (`…_shard_0` on the lab, holding the host name) (not on a forced run with
`mariadb_recover_inventory_name`, which writes none and removes an old one). On each node, the position the
comparison used is the last `Recovered position` line of the MariaDB log. Running the same `grep` on each node
**before** step 2 too (a line from an earlier recovery attempt or a manual `--wsrep-recover`) catches a missing or
stale position before the bootstrap, when the services are not writing yet:

```bash
sudo grep 'Recovered position:' /var/log/kolla/mariadb/mariadb.log | tail -n 1
```

Keep the colon: without it, the pattern also matches `WSREP: Recovered position from storage: <uuid>:<seqno>`, which a
node writes when it joins by state transfer, after the recovery. On the lab, `tail -n 1` without the colon returned
that line on every node, not the line of the recovery (illustrative lines of that form):

```text
2026-10-27 18:18:00 0 [Note] WSREP: Recovered position: 00000000-0000-4000-8000-000000000001:6589
2026-10-27 18:19:51 3 [Note] WSREP: Recovered position from storage: 00000000-0000-4000-8000-000000000001:6682
```

Before step 2, the last line on the lab was from an earlier recovery, about 50 minutes older, as described below.

The comparison has blind spots: a node whose recovery wrote no `Recovered position` line gets an empty `seqno`, is
skipped by every comparison and can end up chosen itself; and the role reads the whole log, so the last line may come
from an earlier run. If the chosen host's line is missing or older than this run, stop and look before the services
start writing again.

**Forcing the bootstrap host.** `-e mariadb_recover_inventory_name=<host>` names the host to bootstrap from instead.

> 🛑 **Destructive.** Skips the `seqno` comparison entirely (`recover_cluster.yml` does not run the wsrep recovery
> block when a host is named): the named host is bootstrapped and the other nodes are started as they are, with their
> own data. Every transaction newer than the named host's position is lost. Nodes whose position is ahead of it
> normally refuse to join (Galera reports that the states diverged, or `Local state seqno ... is greater than group
> seqno`) instead of taking a state transfer, unless their data directory is emptied first. Use it only when you know
> the named host is the right source, for example after restoring a backup on it, and then follow the
> [backup and restore page](../operations/mariadb-backup-and-restore-with-mariabackup.md), which empties the other
> nodes first. Precondition: step 1 done on every node. Undo: only by restoring the copies of step 1.

```bash
kolla-ansible mariadb-recovery -i <inventory> -e mariadb_recover_inventory_name=<mariadb-host>
```

### For cause 2

Wait for the node to finish its state transfer, or find why it cannot join (network between the controllers, disk
space, the log). Do not run `mariadb-recovery` while a node is serving data: it restarts every node.

## Verification

```bash
# on each MariaDB host; -p prompts for database_password from passwords.yml (database_user, root by default);
# -it needs a terminal. Without -it (in a script) there is no prompt: give it as -p<password>, which puts it in the
# process list and shell history
sudo docker exec -it mariadb mariadb -uroot -p -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment');"
# expected: wsrep_cluster_size = number of MariaDB hosts, Primary, Synced

openstack token issue        # Keystone answers again
openstack server list --all-projects --limit 5
```

Then a `kolla-ansible deploy` or `reconfigure` goes past *Fail on existing but stopped cluster*.

## Validity and limits

- Written from Kolla-Ansible 2026.1 code (`ansible/mariadb-recovery.yml`, `ansible/roles/mariadb/tasks/lookup_cluster.yml`
  and `recover_cluster.yml`) and its operating and backup guides, with Galera's crash recovery documentation for the
  background. Cause 1 was reproduced on a lab (Kolla-Ansible 22.2.0, 3 MariaDB hosts with ProxySQL, Docker): the
  three containers stopped cleanly within a few seconds, then `reconfigure --tags mariadb`, steps 1 to 3 and the
  verification. Not run on the lab: a crash (`seqno: -1`), the forced bootstrap with
  `mariadb_recover_inventory_name`, cause 2, a time-out on a joiner.
- Docker paths (`/var/lib/docker/volumes/...`); with Podman, check the volume path.
- Not covered: several shards (the recovery works per shard group), external databases, a cluster split by a network
  partition where some nodes still serve (cause 2), and data corruption, for which the
  [backup and restore page](../operations/mariadb-backup-and-restore-with-mariabackup.md) applies.

## Related

- [Backing up and restoring MariaDB](../operations/mariadb-backup-and-restore-with-mariabackup.md)
- [deploy fails part-way because Ubuntu upgraded packages during the deploy](unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md)
- Kolla-Ansible 2026.1: [operating Kolla](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html)
- Galera: [crash recovery](https://galeracluster.com/library/documentation/crash-recovery.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 code and documentation and the Galera
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/mariadb-cluster-exists-but-is-stopped-run-mariadb-recovery.md&title=Feedback%3A+deployment-errors/mariadb-cluster-exists-but-is-stopped-run-mariadb-recovery.md):
your release, your topology, and what happened.
