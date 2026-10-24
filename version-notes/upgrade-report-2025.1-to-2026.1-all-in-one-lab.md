---
title: Upgrading an all-in-one lab from 2025.1 to 2026.1 with Kolla-Ansible, what happened
kind: experience-report
status: draft
openstack_releases: ["2025.1", "2026.1"]
components: [kolla-ansible, ansible, docker, mariadb, rabbitmq, nova, neutron, ovs, keystone, glance, placement, heat, horizon]
deployment_methods: [kolla-ansible]
environment:
  topology: all-in-one, one VM (4 vCPU, 16 GB RAM, 50 GB disk), nested inside an existing OpenStack cloud
  network: ml2/ovs, VXLAN tenant network, external network on a local veth pair
  storage: local disk only; no Cinder
  os: ubuntu-24.04 (noble), docker 29
date_of_events: 2026-10
authored_by: agent:lab-reviewer
last_verified: 2026-10-22
---

# Upgrading an all-in-one lab from 2025.1 to 2026.1 with Kolla-Ansible, what happened

> **Observed on a disposable all-in-one lab, not on production.** One node, no HA, no Cinder, two test instances,
> deleted afterwards. It shows what the commands did and what the instances saw on that lab; production has more
> nodes, more data and more to lose.

## Context

We wanted a first-hand run of the skip-level (SLURP) upgrade from 2025.1 to 2026.1. We followed the
[version notes from 2025.1 to 2026.1](2025.1-to-2026.1.md) as a reader would, and measured what running instances
and floating IPs went through.

**The lab:**
- One VM of an existing OpenStack cloud.
- Ubuntu 24.04 with nested KVM.
- Kolla-Ansible's default set of services: Keystone, Glance, Nova, Neutron ML2/OVS with the L3 agent, Placement, Heat, Horizon, MariaDB, RabbitMQ and Memcached.
- One node, so `enable_haproxy: "no"` and `enable_proxysql: "no"`, with the internal address set to the node's own address.

**Constraints of the hosting cloud:**
- There is no direct Internet access and no external DNS. Everything goes through an HTTPS proxy on a deploy host, `<deploy-host>` (192.0.2.10:3128).
- NTP comes from the same host.
- The 2025.1 images came from quay.io through the proxy.
- The 2026.1 images came from a local registry on the deploy host (192.0.2.10:4000, plain HTTP), under a pinned tag.

**Versions:**

| | Before | After |
|---|---|---|
| OpenStack | 2025.1 | 2026.1 |
| Kolla-Ansible | 20.6.0 | 22.2.0 |
| ansible-core | 2.18.19 | 2.20.10 |
| Images | `2025.1-ubuntu-noble` (quay.io) | `2026.1-ubuntu-noble-<date>` (local registry) |
| MariaDB | 10.11 | 11.4.13 |
| RabbitMQ | 4.0.9 | 4.2.9 |
| Nova API maximum microversion | 2.100 | 2.103 |

## What we planned

1. Deploy 2025.1 on a fresh VM, the way we deploy 2026.1 labs (see
   [all-in-one lab on a single VM](../topologies/all-in-one-lab-on-a-single-vm.md)). Create a router, a tenant
   network, an external network and two CirrOS instances, each with a floating IP. Record their state.
2. Follow the nine steps of the version notes in order:
   1. back up the configuration and the database (Kolla-Ansible's `mariadb-backup`), check both, copy them off the host;
   2. upgrade Ansible and Kolla-Ansible in the same virtual environment;
   3. run `install-deps`;
   4. compare the inventory with the 2026.1 sample;
   5. update `globals.yml` and merge `passwords.yml`;
   6. run `pull` and `prechecks`;
   7. check the RabbitMQ upgrade path;
   8. run `upgrade`;
   9. remove the old images, once the platform is checked.
3. During `upgrade`, measure:
   - a ping every 0.5 s to each floating IP (traffic already flowing);
   - every 5 s, a *new* ping to one instance and an HTTP request to Keystone, Nova and Neutron;
   - the qemu process IDs before and after.
4. Afterwards: check the services, the instances, a new instance, `kolla-ansible check`, and what was left behind.

## What happened

| Step | Duration | Result |
|---|---|---|
| Host preparation, 2025.1 virtual environment, `install-deps` | about 5 min | fine |
| `bootstrap-servers` (2025.1) | 2 min | fine, but see "What broke", item 1 |
| `prechecks` (2025.1) | 25 s | fine |
| `pull` (2025.1) | 11 min | **failed**: no image could be pulled |
| `pull` again, after restarting Docker | 19 min | fine (all images from quay.io through the proxy) |
| `deploy`, `post-deploy` (2025.1) | 20 min | fine: 31 containers, among them `mariadb_clustercheck` |
| Test resources, state recorded | 3 min | two instances `ACTIVE`, reachable by ping and SSH on their floating IPs |
| 1. Configuration backup | 2 s | fine |
| 1. `enable_mariabackup`, `reconfigure -t mariadb`, `mariadb-backup` | 70 s | fine; but our first check of the backup was wrong, see "What broke", item 2 |
| 2. `pip install -U ansible-core==2.20.10 kolla-ansible==22.2.0` | 18 s | fine, `pip check` clean |
| 3. `install-deps` | 55 s | fine |
| 4. Inventory | 1 min | the 2026.1 sample replaced ours (it was the unchanged 2025.1 sample) |
| 5. `globals.yml`, `kolla-genpwd`, `kolla-mergepwd` | 5 s | every existing password kept |
| 6. `pull` (2026.1) | — | **failed**: `http: server gave HTTP response to HTTPS client`, see "What broke", item 3 |
| 6. `pull` again | 15 min | fine |
| 6. `prechecks` | 45 s | fine |
| 7. RabbitMQ path | 30 s | 4.0.9 to 4.2.9 is an allowed path (`rabbitmq_allowed_upgrades`: 4.0 → 4.1 or 4.2) |
| **8. `upgrade`** | **24 min** | **fine on the first run**, `failed=0` |
| `post-deploy`, new instance, `kolla-ansible check` | 1 min | fine; new instance `ACTIVE` in 21 s |
| Removing the old images | 1 min | `prune-images` removed nothing; `docker rmi` of the old tags freed 7 GB |

### What the platform went through during `upgrade`

The playbook recreated the containers service by service:
- base containers, then MariaDB at +3 min and RabbitMQ at +6 min;
- Keystone, Glance, Placement, then Open vSwitch at +10 min;
- Nova from +12 to +18 min;
- Neutron from +20 to +22 min;
- Heat and Horizon last.

| What | Observed |
|---|---|
| Running instances | Never stopped. Same qemu processes before and after (the PIDs did not change), `ACTIVE` and reachable by SSH afterwards |
| Floating IPs, traffic already flowing | **3 requests lost on each (one 2 s gap)**, when the `openvswitch_vswitchd` container was recreated. Nothing else in the 24 minutes touched them |
| Floating IP, new pings every 5 s | One failed sample, at the same moment |
| Keystone API | Unavailable for about 10 s (its container being recreated) |
| Nova API | Unavailable three times, up to about 15 s |
| Neutron API | Unavailable for about 30 s (`neutron_server` recreated, with the new `neutron_rpc_server` and `neutron_periodic_worker`) |
| MariaDB and RabbitMQ restarts | Not seen by the probes. Each probe was an unauthenticated HTTP request every 5 s, so they could not see errors behind the API |

What changed by itself, as the version notes said:
- `mariadb_clustercheck` was removed.
- The Neutron agent wrapper containers appeared, for dnsmasq and for the haproxy of the DHCP and router namespaces.
- `neutron_rpc_server` and `neutron_periodic_worker` were added.

What the notes did not say:
- With `enable_proxysql: "no"` kept, no ProxySQL was deployed, even though 2026.1 defines `enable_proxysql: "{{ enable_mariadb }}"`. MariaDB stayed reached directly.
- MariaDB moved from 10.11 to 11.4.

## What broke

1. **The 2025.1 deployment could not pull any image, because Docker was not using its proxy.**
   - The error: `failed to resolve reference "quay.io/openstack.kolla/cron:2025.1-ubuntu-noble": ... dial tcp: lookup quay.io on <local resolver>:53: ... i/o timeout`. The lab has no external DNS, so every image needs the proxy.
   - `docker info` showed no proxy, and the running `dockerd` had no proxy variable in its environment. Yet the systemd drop-in that `bootstrap-servers` writes (`/etc/systemd/system/docker.service.d/kolla.conf`) had them.
   - In the `bootstrap-servers` run of Kolla-Ansible 20.6.0 (collection `stable/2025.1`), the `Restart docker` handler ran three seconds **before** the drop-in was written. Afterwards only a `daemon-reload` ran, so Docker never restarted with the proxy settings.
   - The same step with Kolla-Ansible 22.2.0 on another lab restarted Docker after writing the drop-in.
   - Fix: `sudo systemctl restart docker` (no container was running yet), then `pull` again. Cost: 11 minutes of failed pulls plus the diagnosis.
2. **Our first check of the database backup checked nothing.**
   - `kolla-ansible mariadb-backup` worked. It writes a **directory** `full-<date>-<id>/` holding `mysqlbackup-<id>.qp.xbc.xbs.gz`, in the `mariadb_backup` Docker volume, which only root can read.
   - Our script expected a single `.gz` file, so it copied only the `last_full_file` marker. It reported no error, because the glob simply matched nothing.
   - We caught it before the upgrade and redid the check:
     1. `sudo` copy of the directory;
     2. `gzip -t`;
     3. `mbstream -x` into a scratch directory inside the running `mariadb` container. Every service database was there, and `xtrabackup_checkpoints` said `backup_type = full-backuped`.
   - Then we copied the backup off the host with its checksum.
3. **The 2026.1 `pull` failed: `http: server gave HTTP response to HTTPS client`.**
   - The upgrade moved the images to a plain-HTTP registry. `docker_registry_insecure: true` in `globals.yml` reaches Docker's `daemon.json` only through `bootstrap-servers`, and the upgrade steps don't run it.
   - Fix: add the registry to `insecure-registries` in `/etc/docker/daemon.json`, then `sudo systemctl reload docker`. That setting is reloaded without restarting the daemon: all 31 containers kept running.
   - Re-running `bootstrap-servers` would probably have done the same, but it can restart Docker and every container with it. We did not try it during the upgrade.
4. **`kolla-ansible prune-images` did not remove the old images.**
   - It ran successfully and removed nothing. It only removes dangling images, and the 30 `2025.1` images were still tagged.
   - `docker rmi` of the `2025.1` tags freed 7 GB of a 48 GB disk; running containers were not affected.

Nothing broke during `kolla-ansible upgrade` itself.

## What we learned

- **On this lab, the skip-level upgrade was uneventful for running instances.** One 2 s gap on the floating IPs, when Open vSwitch was recreated, and short API outages, the longest about 30 s for Neutron. Everything that went wrong happened before the upgrade command: proxy, backup check, registry.
- **Changes to Docker's own configuration are not part of `kolla-ansible upgrade`.** If the registry, the proxy or other daemon settings change with the upgrade, apply them first: by `bootstrap-servers` in a window where restarting Docker is acceptable, or by hand. Then check with `docker info`.
- **Check that the daemon really has its settings,** not only that the files are there: `docker info` (proxy, insecure registries) or `/proc/<pid>/environ`. The proxy issue in item 1 was invisible until a pull needed the Internet.
- **A backup check must fail loudly when it finds nothing.** Know what the backup looks like before writing the check. For Kolla-Ansible's Mariabackup: a directory per full backup, readable by root.
- **The inventory diff against the new sample is worth doing even for an all-in-one.** It showed renamed groups (`kolla-toolbox` and `kolla-logs` → `kolla_toolbox` and `kolla_logs`), new groups for the new Neutron containers, and removed groups for dropped services.
- **Plan disk for two sets of images,** and remove the old ones by tag afterwards.

## What we would do differently

- Set up and check Docker's proxy and registry settings once, with `docker info`, right after `bootstrap-servers`, before the first pull. On both releases.
- Include the registry change in the preparation: write `insecure-registries` (or run `bootstrap-servers`) before step 6. Don't discover it when `pull` fails.
- Write the backup check against a backup taken beforehand on a throw-away deployment, so that its paths and format are known.
- Measure the database and message-queue restarts with authenticated API calls (for example a `server list` every 5 s), not only unauthenticated HTTP requests.

## Open questions

- Is the ordering of the `Restart docker` handler and the proxy drop-in in the 2025.1 collection reproducible on other hosts, or was it timing on this one? We saw it once.
- What does an upgrade with HAProxy in front of MariaDB and `enable_proxysql: "no"` do, now that 2026.1 drops HAProxy support for MariaDB? Not tested here: this lab had no HAProxy.
- How long do the API outages last on a multinode deployment with HAProxy, where each service has several backends? Here each service had one container, so every recreation was an outage.

## Related

- [From 2025.1 to 2026.1: version notes](2025.1-to-2026.1.md), the page this run followed (as of commit `edb2891`, merged in #11).
- [All-in-one lab on a single VM](../topologies/all-in-one-lab-on-a-single-vm.md), how the lab was built.
- [Unattended upgrades during a deploy](../deployment-errors/unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md):
  disabled on this lab before anything else.
- Kolla-Ansible 2026.1, [operating Kolla](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html);
  2025.1, [MariaDB backup and restore](https://docs.openstack.org/kolla-ansible/2025.1/admin/mariadb-backup-and-restore.html).

---

*Provenance: written by agent:lab-reviewer, from a lab upgrade run on 2026-10-22.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
