---
title: kolla-ansible bootstrap-servers. What it changes on the hosts, and the pitfalls of running it again on a live platform
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, docker]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; the pitfalls matter most on running platforms
  scale: any
  network: any
  storage: any
  os: Ubuntu or Debian, and Red Hat family hosts supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html
  - https://opendev.org/openstack/ansible-collection-kolla/src/branch/stable/2026.1
  - https://docs.openstack.org/kolla-ansible/2026.1/user/security.html
  - https://docs.docker.com/engine/daemon/live-restore/
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# kolla-ansible bootstrap-servers: what it changes on the hosts, and its pitfalls

## Context

`kolla-ansible bootstrap-servers` prepares hosts for Kolla-Ansible: packages, Docker, users, `/etc/hosts`, security
settings. On a new host it is the first command. On a running platform it is run again more often than people expect:
for a new host, after a change of registry or proxy, before an upgrade. Each of those runs can change things on the
hosts that the operator did not mean to change, and several of them restart Docker, and with it every container of
the host.

This page lists what the command changes, from the Kolla-Ansible 2026.1 bootstrap servers guide and the roles that do
the work, in the `openstack.kolla` collection `stable/2026.1` (installed by `kolla-ansible install-deps`): `baremetal`,
`docker`, `etc_hosts`, `kolla_user`, `packages`, `apparmor_libvirt`, `journal`.

## Topology

Runs from the deploy host against every host of the inventory, or the hosts of `--limit`.

## Network design

`/etc/hosts` entries for every host's API address (below). Proxies for Docker with `docker_http_proxy`,
`docker_https_proxy` and `docker_no_proxy`, written as a systemd drop-in for `docker.service`.

## Storage design

Docker's data root can be moved with `docker_runtime_directory` and its storage driver set with
`docker_storage_driver`. Changing either on a host that already runs containers is not covered here.

## Inventory and configuration layout

What a run changes, with the collection's 2026.1 defaults:

| What | Variables (default) | What it does |
|---|---|---|
| Host firewall | `disable_firewall` (**true**) | On every run: `ufw` set to disabled with policy `allow` on Debian/Ubuntu; `firewalld` stopped and disabled on the Red Hat family (`baremetal/tasks/install.yml`). Set it to `false` and `enable_external_api_firewalld: true` to have Kolla-Ansible configure firewalld instead |
| `/etc/hosts` | `customize_etc_hosts` (true) | `localhost` present; one entry per host of `baremetal` for its API address, written on the hosts of the run only; the loopback line (other than `localhost`) that maps the host name removed; cloud-init's `manage_etc_hosts` set to false (`/etc/cloud/cloud.cfg.d/99-kolla.cfg`) |
| User | `create_kolla_user` (false), `create_kolla_user_sudoers` (true), `kolla_user`, `kolla_group` | When enabled: the `kolla` user with the public key of `kolla_ssh_key`, in the `sudo` and `docker` groups (membership of `docker` is root-equivalent), passwordless sudo by default |
| Packages | `ubuntu_pkg_install`, `redhat_pkg_install`, `ubuntu_pkg_removals`, `redhat_pkg_removals` | Installs a few tools (git, ping, tzdata or sudo); **removes** `lxd`, `lxc`, `open-iscsi` on Ubuntu, and `libvirt`, `libvirt-daemon`, `iscsi-initiator-utils` on the Red Hat family (the libvirt ones, and `libvirt-bin` on Ubuntu, when `enable_nova_libvirt_container` is true) |
| Docker packages | `enable_docker_repo`, `docker_apt_package_pin`, `docker_yum_package_pin` (empty) | Docker CE from `download.docker.com`, installed with `state: present`: a rerun upgrades it only if the pin or the repository changes |
| Docker daemon | `docker_registry`, `docker_registry_insecure` (false), `docker_log_max_file` (5), `docker_log_max_size` (`50m`), `docker_custom_config`, `docker_disable_default_iptables_rules` (yes) | `/etc/docker/daemon.json`, by default with Docker's iptables rules, default bridge and IP forwarding turned off; the proxy drop-in `docker.service.d/kolla.conf`; the registry CA under `/etc/docker/certs.d/` |
| systemd | — | `kolla.target` written |
| Ubuntu AppArmor | `apparmor_remove_libvirt_profile` (follows `enable_nova_libvirt_container`) | The `libvirtd` AppArmor profile removed |
| Red Hat family | `change_selinux` (true), `selinux_state` (`permissive`); journal settings | SELinux set to permissive; `net.ipv4.ping_group_range` set; a persistent systemd journal when Fluentd reads it |
| Python | `virtualenv`, `virtualenv_site_packages` (true) | An optional virtual environment on the hosts |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Run it once per new host, limited to that host | Run it on the whole inventory | Nothing changes on the running hosts | `/etc/hosts` entries of the other hosts are not updated (pitfall 3) |
| Batches when it must run on running hosts (`--limit` per batch, or `-e kolla_serial=30%`) | All hosts at once | A Docker restart restarts every container of a host; batches keep quorum | Slower; each batch is a short outage of what runs on it |
| Pin the Docker version (`docker_apt_package_pin`, `docker_yum_package_pin`) | Take what the repository offers | The guide asks to test any operation that updates Docker | One more variable to keep up to date |
| `disable_firewall: false` with firewalld managed by Kolla-Ansible, or by another tool | The default | The default disables the host firewall on every run | Firewall rules to own and maintain |
| Keep `docker_registry_insecure` false | Plain-HTTP registry | The guide says it also disables registry authentication | A TLS registry and its CA to manage |

## The pitfalls on a running platform

**1. Any Docker configuration change restarts Docker, and every container of the host.** The collection's `Restart
docker` handler is a full `systemctl restart` (a reload only if `docker_systemd_reload` is true; default false). It is
triggered by any change to `/etc/docker/daemon.json` (registry, insecure registry, mirrors through
`docker_custom_config`, log options, storage driver), to the proxy drop-in `docker.service.d/kolla.conf`, or to the
registry CA file, as well as by a Docker package upgrade (the guide's *Subsequent bootstrap considerations*). The
guide's ways out:

> ⚠️ **State-changing.** Each run below can restart Docker and every container on the hosts of the batch, whenever
> the Docker package or its configuration changes. Keep a quorum of controllers outside each batch (MariaDB,
> RabbitMQ). Undo: none for a restart that happened.

```bash
# batches with --limit, keeping quorum
kolla-ansible bootstrap-servers -i <inventory> --limit controller-1,compute-01
kolla-ansible bootstrap-servers -i <inventory> --limit controller-2,compute-02
kolla-ansible bootstrap-servers -i <inventory> --limit controller-3,compute-03
# or plays in batches
kolla-ansible bootstrap-servers -i <inventory> -e kolla_serial=30%
```

The third, Docker's `live-restore`, keeps containers running across a daemon restart; the guide says issues have been
reported with it and to use it at your own risk.

**2. The host firewall is turned off by default, on every run.** With `disable_firewall: true` (the default), each run
disables `ufw` with an `allow` policy on Debian/Ubuntu, or stops and disables `firewalld` on the Red Hat family. A
rerun on a live platform drops host firewall rules someone enabled since. Set `disable_firewall: false` before any run
on hosts whose firewall is managed.

**3. `/etc/hosts` is updated only on the hosts of the run.** Entries are written on the hosts in the limit, for every
host of the inventory. A host added with `--limit <new-host>` is unknown to the `/etc/hosts` of the others. For
computes this rarely matters; for controllers the adding and removing hosts guide asks to include every controller,
because RabbitMQ needs every controller in every controller's `/etc/hosts`. The run also removes the loopback line (other than
`localhost`) that maps the host name, and stops cloud-init from managing `/etc/hosts`.

**4. Packages are removed as well as installed.** By default `lxd`, `lxc` and `open-iscsi` on Ubuntu, and with the
libvirt container `libvirt-bin` (Ubuntu) or `libvirt`, `libvirt-daemon`, `iscsi-initiator-utils` (Red Hat family).
Check the lists of your release before a run on hosts that also carry other software.

**5. Security settings change.** The `libvirtd` AppArmor profile is removed on Ubuntu when the libvirt container is
used; SELinux is set to `permissive` on the Red Hat family unless `change_selinux` is false; with
`create_kolla_user`, the `kolla` user gets passwordless sudo and membership of the `docker` group.

**6. The first run is not like the others.** The guide's *Initial bootstrap considerations*: the `kolla` user and
the hosts' virtual environment may not exist yet, so the first run may need another user and the system Python:

> ⚠️ **State-changing.** The first bootstrap of new hosts: everything in the table above happens on them. Run it
> only on hosts with no workload yet. Undo: none simple; reinstall the host.

```bash
kolla-ansible bootstrap-servers -i <inventory> -e ansible_user=<bootstrap-user> -e ansible_python_interpreter=/usr/bin/python3
```

**7. Registry and proxy settings reach Docker only through this command, and changing them restarts Docker.**
`docker_registry_insecure`, mirrors and the proxies are written by `bootstrap-servers`, not by `pull`, `deploy` or
`upgrade` (see
[pull fails with "http: server gave HTTP response to HTTPS client"](../deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md)),
and such a change triggers the Docker restart of pitfall 1: run it in batches, under pitfall 1's ⚠️. A plain-HTTP
registry configured this way also loses registry authentication, as the guide notes.

**8. Facts are gathered for every host.** Like `deploy`, the run gathers facts for every host of the inventory, even
with `--limit`; an unreachable host is reported and makes the run exit non-zero.

## What went wrong during deployment

Seen in this repository: the [upgrade report](../version-notes/upgrade-report-2025.1-to-2026.1-all-in-one-lab.md)
avoided `bootstrap-servers` during an upgrade because it can restart Docker, and applied the registry setting by hand
with a `systemctl reload docker` instead.

## Version notes

Written against Kolla-Ansible 2026.1 and the `openstack.kolla` collection `stable/2026.1`. The 2026.1 roles do not
configure NTP.

## Lessons learned

- Treat `bootstrap-servers` on a running platform as a change with an outage risk, not as a harmless preparation.
- Decide `disable_firewall` before the first run, and keep it decided.
- Keep Docker's version under your control.

## Validity and limits

- Written from the Kolla-Ansible 2026.1 bootstrap servers guide and the `openstack.kolla` collection `stable/2026.1`
  (`roles/baremetal/{defaults/main.yml,tasks/install.yml,tasks/pre-install.yml,tasks/post-install.yml}`,
  `roles/docker/{defaults/main.yml,tasks/config.yml,handlers/main.yml}`, `roles/etc_hosts/tasks/etc-hosts.yml`,
  `roles/kolla_user/`, `roles/packages/defaults/main.yml`, `roles/apparmor_libvirt/defaults/main.yml`). Not run on a
  lab for this page. Status `draft`.
- Podman hosts are not covered (the collection has a `podman` role; the Docker sections do not apply).

## Related

- [Adding a compute node](adding-a-compute-node.md), [replacing a failed controller](replacing-a-failed-controller.md)
- [deploy, reconfigure, upgrade and the others](deploy-reconfigure-upgrade-and-limiting-runs.md)
- Kolla-Ansible 2026.1: [bootstrap servers](https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html),
  [security](https://docs.openstack.org/kolla-ansible/2026.1/user/security.html)
- Docker: [live restore](https://docs.docker.com/engine/daemon/live-restore/)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and the openstack.kolla
collection code. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=operations/bootstrap-servers-what-it-changes-and-its-pitfalls.md&title=Feedback%3A+operations/bootstrap-servers-what-it-changes-and-its-pitfalls.md):
your release, your topology, and what happened.
