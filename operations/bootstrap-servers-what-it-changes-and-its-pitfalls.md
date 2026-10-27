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
  - https://docs.openstack.org/kolla-ansible/2026.1/user/security.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/virtual-environments.html
  - https://docs.docker.com/engine/daemon/live-restore/
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# kolla-ansible bootstrap-servers: what it changes on the hosts, and its pitfalls

## Context

`kolla-ansible bootstrap-servers` prepares hosts for Kolla-Ansible: packages, Docker, users, `/etc/hosts`, security
settings. On a new host it is the first command. On a running platform it is run again more often than people expect:
for a new host, after a change of registry or proxy, before an upgrade. Each of those runs can change things on the
hosts that the operator did not mean to change, and one of them, a Docker upgrade, restarts every container of a host.

This page lists what the command changes, from the Kolla-Ansible 2026.1 bootstrap servers guide, and the pitfalls of
each change on a live platform. The implementation is the `baremetal` role of the `openstack.kolla` collection
(installed by `kolla-ansible install-deps`); its defaults were not read for this page.

## Topology

Runs from the deploy host against every host of the inventory, or the hosts of `--limit`.

## Network design

`/etc/hosts` entries for every host's API address (below). Proxies for Docker with `docker_http_proxy`,
`docker_https_proxy` and `docker_no_proxy`.

## Storage design

Docker's data root can be moved with `docker_runtime_directory` and its storage driver set with
`docker_storage_driver`. Changing either on a host that already runs containers is not covered here.

## Inventory and configuration layout

Everything is driven by variables in `globals.yml` or the inventory. The ones the guide names:

| What | Variables (guide's defaults) | What it does |
|---|---|---|
| `/etc/hosts` | `customize_etc_hosts` (true) | `localhost` present, and one entry per host for its API interface address |
| User and group | `create_kolla_user` (true), `kolla_user`, `kolla_group` (`kolla`), `create_kolla_user_sudoers` | The `kolla` user, in the `sudo` group, with the public key of `kolla_ssh_key` from `passwords.yml`; passwordless sudo if the sudoers variable is set |
| Config directory | `node_config_directory` (`/etc/kolla/`) | Created, owned by the `kolla` user if it is created |
| Packages | `debian_pkg_install`, `ubuntu_pkg_removals`, `redhat_pkg_install`, `redhat_pkg_removals` | Packages installed **and removed** |
| Docker packages | `enable_docker_repo`, `docker_apt_*`, `docker_yum_*` | Docker CE repository from `download.docker.com` |
| Docker daemon | `docker_registry`, `docker_registry_insecure` (false), `docker_log_max_file` (5), `docker_log_max_size` (`50m`), `docker_custom_config` | Written to `daemon.json` |
| Firewall | `disable_firewall`, `enable_external_api_firewalld` | firewalld only; the guide says to set `disable_firewall: false` and `enable_external_api_firewalld: true` to have it configured |
| Python | `virtualenv`, `virtualenv_site_packages` (true) | An optional virtual environment on the hosts |
| Ubuntu | — | The `libvirtd` AppArmor profile is removed |
| Red Hat family | `change_selinux` (true), `selinux_state` (`permissive`) | SELinux set to permissive |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Run it once per new host, limited to that host | Run it on the whole inventory | Nothing changes on the running hosts | `/etc/hosts` entries of the other hosts are not updated (see the pitfalls) |
| Batches when it must run on running hosts (`--limit` per batch, or `-e kolla_serial=30%`) | All hosts at once | A Docker upgrade restarts every container of a host; batches keep quorum | Slower; each batch is a short outage of what runs on it |
| Pin the Docker version, or test the package update first | Take whatever the repository offers | The guide asks to test any operation that updates Docker | One more variable to keep up to date |
| Keep `docker_registry_insecure` false | Plain-HTTP registry | The guide says it also disables registry authentication | A TLS registry and its CA to manage |
| Firewall managed by Kolla-Ansible (`enable_external_api_firewalld`) | Firewall managed elsewhere | One owner for host rules | Only firewalld is supported |

## The pitfalls on a running platform

**1. A Docker upgrade restarts every container of the host.** The guide's *Subsequent bootstrap considerations*:
if the Docker package is updated, the engine restarts with all its containers. Its three ways out:

> ⚠️ **State-changing.** Each run below can restart Docker and every container on the hosts of the batch. Keep a quorum
> of controllers outside each batch (MariaDB, RabbitMQ). Undo: none for a restart that happened.

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

**2. `/etc/hosts` is updated only on the hosts of the run.** Entries are written on the hosts in the limit, for every
host of the inventory. A host added with `--limit <new-host>` is unknown to the `/etc/hosts` of the others. For
computes this rarely matters; for controllers the adding and removing hosts guide asks to include every controller,
because RabbitMQ needs every controller in every controller's `/etc/hosts`.

**3. Packages are removed as well as installed.** `ubuntu_pkg_removals` and `redhat_pkg_removals` list packages the
role removes. Read them in your release's collection before a run on hosts that also carry other software.

**4. Security settings change.** On Ubuntu the `libvirtd` AppArmor profile is removed; on Red Hat family hosts SELinux
is set to `permissive` unless `change_selinux` is false. Firewall configuration depends on `disable_firewall` and
`enable_external_api_firewalld`; check their values in your configuration and the collection's defaults before a
first run on hosts whose firewall is managed by something else.

**5. The first run is not like the others.** The guide's *Initial bootstrap considerations*: the `kolla` user and
the hosts' virtual environment may not exist yet, so the first run may need another user and the system Python:

> ⚠️ **State-changing.** The first bootstrap of new hosts: everything in the table above happens on them. Run it
> only on hosts with no workload yet. Undo: none simple; reinstall the host.

```bash
kolla-ansible bootstrap-servers -i <inventory> -e ansible_user=<bootstrap-user> -e ansible_python_interpreter=/usr/bin/python3
```

**6. Registry settings reach Docker only through this command.** `docker_registry_insecure` and the proxies are
written to Docker's configuration by `bootstrap-servers`, not by `pull`, `deploy` or `upgrade` (see
[pull fails with "http: server gave HTTP response to HTTPS client"](../deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md)).
A plain-HTTP registry configured this way also loses registry authentication, as the guide notes.

**7. Facts are gathered for every host.** Like `deploy`, the run gathers facts for every host of the inventory, even
with `--limit`; an unreachable host is reported and makes the run exit non-zero.

## What went wrong during deployment

Seen in this repository: the [upgrade report](../version-notes/upgrade-report-2025.1-to-2026.1-all-in-one-lab.md)
avoided `bootstrap-servers` during an upgrade because it can restart Docker, and applied the registry setting by hand
instead.

## Version notes

Written against the Kolla-Ansible 2026.1 bootstrap servers guide. The variable names and defaults are the guide's;
the role lives in the `openstack.kolla` collection of the same series, whose defaults may differ in detail.

## Lessons learned

- Treat `bootstrap-servers` on a running platform as a change with an outage risk, not as a harmless preparation.
- Read the package removal lists and the security defaults before the first run on hosts you share with other tools.
- Keep Docker's version under your control.

## Validity and limits

- Written from the Kolla-Ansible 2026.1 bootstrap servers guide, the security and virtual environment guides, and
  `ansible/kolla-host.yml`. The `baremetal` role itself (in `openstack.kolla`) was not read. Not run on a lab for this
  page. Status `draft`.
- Podman hosts are not covered (the guide's Docker sections do not apply).

## Related

- [Adding a compute node](adding-a-compute-node.md)
- [deploy, reconfigure, upgrade and the others](deploy-reconfigure-upgrade-and-limiting-runs.md)
- Kolla-Ansible 2026.1: [bootstrap servers](https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html),
  [security](https://docs.openstack.org/kolla-ansible/2026.1/user/security.html)
- Docker: [live restore](https://docs.docker.com/engine/daemon/live-restore/)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=operations/bootstrap-servers-what-it-changes-and-its-pitfalls.md&title=Feedback%3A+operations/bootstrap-servers-what-it-changes-and-its-pitfalls.md):
your release, your topology, and what happened.
