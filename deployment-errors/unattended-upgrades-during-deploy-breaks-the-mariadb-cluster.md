---
title: kolla-ansible deploy fails part-way, a container never starts and MariaDB never forms its cluster, because Ubuntu upgraded packages during the deploy
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [kolla-ansible, mariadb, docker, systemd]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode, 3 controllers, 2 computes
  network: ml2/ovs
  storage: local
  os: ubuntu-24.04 (noble), docker
risk: destructive
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, all-in-one", date: 2026-10-25, note: "an unattended upgrade run during a MariaDB deploy: systemd re-executed and needrestart restarted the Kolla units; prevention, mariadb-recovery and verification run; the Galera symptom needs several controllers and was not reproduced" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/latest/user/quickstart.html
  - https://help.ubuntu.com/community/AutomaticSecurityUpdates
authored_by: agent:ozarland-author
last_verified: 2026-10-25
---

# kolla-ansible deploy fails part-way because Ubuntu upgraded packages during the deploy

## Symptom

A `kolla-ansible deploy` that worked before fails part-way on some hosts, without any configuration change:

- one or more containers are created but never start, or restart in a loop;
- MariaDB never forms its Galera cluster: the deploy waits on the MariaDB tasks and times out;
- on the affected hosts, systemd services were restarted during the deploy (`journalctl` shows `systemd` itself
  re-executed, and units stopping and starting at a time you did not touch them).

In the lab it happened on two hosts out of five, during the daily upgrade run of the morning.

## Impact

The deployment is half-applied on the affected hosts. On a first deploy the platform is not usable; on a
`reconfigure` or `upgrade` of a running platform, services on those hosts may be down and the Galera cluster
degraded.

## Quick checks

On each host that failed:

```bash
# Did apt upgrade packages during the deploy window, and which command did it?
grep -E "^Start-Date|^Commandline|^Upgrade" /var/log/apt/history.log | tail -30
# Which automatic runs are scheduled, and when did they last run?
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer
# Is unattended-upgrades enabled?
cat /etc/apt/apt.conf.d/20auto-upgrades
# Did systemd re-execute (a libc6 or systemd upgrade does that)?
journalctl --since "<deploy start>" | grep -iE "reexecut|Reloading|unattended-upgrade" | head
```

An `Upgrade:` line in the apt history inside the deploy window, listing `libc6`, `systemd` or other base packages,
with `Commandline: /usr/bin/unattended-upgrade`, confirms this runbook. Other lines are expected too:
`bootstrap-servers` runs `apt-get install` itself (on a lab it upgraded `dpkg` and `libbz2-1.0` that way), so read
the `Commandline:` before blaming unattended upgrades.

## Diagnosis

The stock Ubuntu cloud images enable unattended upgrades (`APT::Periodic::Unattended-Upgrade "1"` in
`20auto-upgrades`). `apt-daily-upgrade.timer` fires once a day with a random delay. When it fires during a deploy,
an upgrade of `libc6` and `systemd` makes systemd re-execute and restart several of its services while
Kolla-Ansible is creating and starting containers. In the lab, one container was never started and MariaDB never
formed its cluster. We believe the restarts hit the hosts while Galera was bootstrapping; the exact sequence was not
traced.

On Ubuntu 24.04 the `needrestart` package, installed on the cloud image, restarts the services that use upgraded
libraries at the end of the run, Kolla-Ansible's container units included. Reproduced on an all-in-one lab: during a
`kolla-ansible deploy --tags mariadb`, a manual `unattended-upgrade` re-executed systemd twice, restarted
`containerd`, and stopped and started the `kolla-*-container.service` units (`Stopping
kolla-mariadb-container.service` in `journalctl`). 39 of the 40 containers restarted within three minutes. There, the
deploy had ended 53 s before the restarts and every container came back by itself; on a deploy still running, or on
several controllers bootstrapping Galera, the restarts land in the middle of it.

Kolla-Ansible's `bootstrap-servers` (2026.1, `ansible-collection-kolla`) does not change the automatic upgrade
settings of the hosts, so nothing prevents this.

## Possible causes

| Cause | How to tell |
|---|---|
| Unattended upgrade ran during the deploy | `Upgrade:` entries in `/var/log/apt/history.log` within the deploy window |
| A manual `apt upgrade` by someone else | Same entries, with `Commandline: apt upgrade` and a user |
| Galera failed for another reason (disk, network, clock) | No apt activity in the window; see the MariaDB logs and clock sync |

## Resolution

### Prevent it before deploying

> ⚠️ **State-changing.** Stopping the apt timers disables automatic security updates on the host. Acceptable for a
> short-lived lab; on a long-lived platform, schedule upgrades in maintenance windows instead of removing them.

```bash
# On every host, before any kolla-ansible command
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer
# Wait for a run already in progress to end; never kill a dpkg run half-way
while systemctl is-active --quiet apt-daily.service apt-daily-upgrade.service; do sleep 10; done
systemctl list-jobs | grep -E "apt-daily" || echo "no apt run queued"
```

A run already in progress is never stopped: a half-applied `dpkg` run is worse than waiting.

### Recover the failed deploy

The lab was throw-away, so it was rebuilt from scratch with the prevention above; recovery in place was not tried.
What an operator would do, not observed here:

1. Let any running upgrade finish (above).
2. On a platform that already holds data, recover Galera with Kolla-Ansible's procedure before anything else.

   > 🛑 **Destructive.** `mariadb-recovery` stops MariaDB on every controller, picks the node with the most advanced
   > state, bootstraps a new cluster from it, and makes the other nodes rejoin with a full state transfer that
   > replaces their data. Any transaction that only reached the nodes not chosen is lost. Every OpenStack API is
   > unavailable while it runs. Run it only when the cluster has really lost quorum (no node reports
   > `wsrep_cluster_status` `Primary`, or every `mariadb` container is down), and when no deploy, upgrade or apt run
   > is in progress. Before running it, with the `mariadb` container stopped on each controller, copy its data
   > volume aside (for example `sudo cp -a /var/lib/docker/volumes/mariadb /root/mariadb-before-recovery`) and
   > note the `seqno` in `grastate.dat` on each node. It cannot be undone: going back means restoring those copies.

   ```bash
   kolla-ansible mariadb-recovery -i <inventory>
   ```

3. Re-run the deploy that failed.

   > ⚠️ **State-changing.** Re-running the deploy restarts containers on every host it touches.

   ```bash
   kolla-ansible deploy -i <inventory>    # or the reconfigure or upgrade that failed
   ```

## Verification

```bash
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer    # both inactive, or scheduled in a window you chose
sudo docker ps --filter "status=restarting" --format '{{.Names}}'      # empty, on every host
sudo docker exec -it mariadb mariadb -uroot -p \
  -e "SHOW STATUS LIKE 'wsrep_cluster_size'; SHOW STATUS LIKE 'wsrep_cluster_status';"
# the password is database_password in passwords.yml
# expected: the number of controllers, and Primary
```

## Lab observations

On an all-in-one lab (OpenStack 2026.1, Kolla-Ansible 22.2.0, Ubuntu 24.04 cloud image with 195 pending updates,
`libc6` and `systemd` among them):

| Step | Observed |
|---|---|
| `unattended-upgrade` started by hand, then `kolla-ansible deploy --tags mariadb` with the `mariadb` container removed | Deploy `failed=0` in 133 s. The upgrade ran 23 minutes (73 apt runs) |
| Quick checks | `history.log`: `Upgrade:` lines with `libc6`, under `Commandline: /usr/bin/unattended-upgrade -v`, and earlier ones from `bootstrap-servers`. `journalctl`: `Reexecuting` twice. Timers and `20auto-upgrades` as the page says |
| Effect | `needrestart` restarted `containerd` and the Kolla container units: 39 of 40 containers restarted, all came back |
| Prevention | As written: timers disabled, `no apt run queued` |
| `mariadb-recovery`, with the data volume copied aside first (2.3 GB; `grastate.dat` `seqno: 6073`, `safe_to_bootstrap: 1`) | As written: `failed=0` in 112 s, MariaDB back, the platform's services up |
| Verification | Needed `sudo` for `docker`, and the `mariadb` client: `mysql` is not in the 2026.1 MariaDB 11.4 image. Then `wsrep_cluster_size` 1, `Primary` |

Not reproduced: Galera failing to form across several controllers, which needs a multinode lab.

## Validity and limits

Met once on a real lab deploy (OpenStack 2026.1, Kolla-Ansible 22.2.0, Ubuntu 24.04 cloud image, Docker, 3
controllers and 2 computes): 178 packages upgraded on two hosts during the deploy. The prevention was applied on
every later build of the same lab without recurrence. Not tested: Debian or Rocky hosts (different update
mechanisms), Podman, a `reconfigure` on a production platform. Whether stopping the timers is acceptable outside a
lab is a policy decision for each platform.

## Related

- [Three controllers and two computes, nested inside an existing OpenStack cloud](../topologies/three-controllers-two-computes-nested-in-an-openstack-cloud.md)
- Source: lab builds by @soufian-zaouam (OpenStack 2026.1, Kolla-Ansible 22.2.0).

---

*Provenance: written by agent:ozarland-author, from an incident on lab builds by @soufian-zaouam on 2026.1.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md&title=Feedback%3A+deployment-errors/unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md):
your release, your topology, and what happened.
