---
title: Docker versus Podman as the container engine of a Kolla-Ansible deployment. What changes for operators, and what migrating costs
kind: comparison
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, docker, podman]
deployment_methods: [kolla-ansible]
environment:
  purpose: any (lab, pre-production, production)
  scale: any
  network: any
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/advanced-configuration.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/troubleshooting.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html
  - https://docs.podman.io/en/stable/Commands.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# Docker versus Podman as the container engine of a Kolla-Ansible deployment

## What is compared

Kolla-Ansible 2026.1 supports two container engines, chosen with one variable in `globals.yml`:

```yaml
kolla_container_engine: "docker"   # the default; or "podman"
```

The images, the roles and the commands of `kolla-ansible` are the same. What changes is on the hosts: which daemon
or tool runs the containers, where their volumes live, which commands an operator types, and what a restart of the
engine does to the platform. This page compares the two from an operator's side, and says what the supported
migration between them costs. It does not recommend one.

## What is the same

- **Images and configuration.** The same Kolla images, the same `/etc/kolla/<container>/` configuration files, the same
  `kolla_logs` volume behind `/var/log/kolla` on the host.
- **The `kolla-ansible` commands.** `deploy`, `reconfigure`, `upgrade`, `check`, `stop` work with either engine.
- **Restarts managed by systemd, with both engines.** In 2026.1, Kolla-Ansible writes one systemd unit per container,
  `kolla-<container>-container.service` under `/etc/systemd/system/`, and lets systemd restart the containers
  (`ansible/module_utils/kolla_systemd_worker.py`). With Docker the units depend on `docker.service`; with Podman on
  `network-online.target`.

## What is different

| Aspect | Docker | Podman |
|---|---|---|
| Engine process | A daemon, `dockerd`, that all containers depend on | No central daemon; each container is started by Podman under its systemd unit |
| Restarting the engine | `systemctl restart docker` stops and starts every container of the host, unless Docker's `live-restore` is used (the bootstrap servers guide warns about it) | No daemon to restart; upgrading the Podman package does not by itself restart running containers |
| Volume path on the host | `/var/lib/docker/volumes/<volume>/_data` (`docker_volumes_path`) | `/var/lib/containers/storage/volumes/<volume>/_data` (`podman_volumes_path`) |
| Commands in runbooks | `docker ps`, `docker logs`, `docker exec` | `podman ps`, `podman logs`, `podman exec` (the troubleshooting guide says to use `podman` instead of `docker` where applicable) |
| Registry configuration | Docker's daemon configuration (`daemon.json`: `insecure-registries`, mirrors), written by `bootstrap-servers` | Podman's own registries configuration; how `bootstrap-servers` handles it lives in the `openstack.kolla` collection and was not checked here |
| Libvirt guests at host shutdown | `tools/kolla-libvirt-guests-docker.service` | `tools/kolla-libvirt-guests-podman.service` |
| Prechecks | *Checking Docker version* against `docker_version_min`, and the Docker SDK for Python | Those two checks do not apply |

Most pages of this repository, and most of the upstream examples, use Docker commands and paths. With Podman, an
operator translates them: same subcommands, the other volume path.

## Migrating from one to the other

The advanced configuration guide describes a supported migration, in both directions:

1. Set `kolla_container_engine` to the new engine in `globals.yml`; change nothing else.
2. Stop every virtual machine of the cloud: the guide says rolling migration is not supported and the migration can
   fail otherwise.
3. Run `kolla-ansible migrate-container-engine`. The playbook prepares the hosts (`container-engine-migration` role),
   redeploys everything with the new engine (`site.yml` with `kolla_action: deploy`), then runs `post-deploy`.

The guide warns that the container volumes are migrated to the new engine, and that the old engine's packages are
removed from the system **with all their resources and data**. So: a full outage of every instance, and a backup of
the databases and of `/etc/kolla` first (see
[backing up and restoring MariaDB](../operations/mariadb-backup-and-restore-with-mariabackup.md)). This page gives no
commands for it on purpose: follow the guide of your release, on a lab first.

## What an operator gains and loses

| Docker | Podman |
|---|---|
| The default: most examples, runbooks and upstream CI coverage use it | Fewer examples; every Docker command in a runbook needs translating |
| One daemon to watch; one restart that can restart every container | No daemon whose restart takes every container down; more units to look at |
| `daemon.json` handled by `bootstrap-servers` | Registry settings in Podman's own configuration |
| Moving away later is a full outage (migration) | Same in the other direction |

## Validity and limits

- Written from Kolla-Ansible 2026.1: `doc/source/admin/advanced-configuration.rst` (*Migrate container engine*),
  `doc/source/user/troubleshooting.rst`, `ansible/group_vars/all/common.yml` (`kolla_container_engine`, volume
  paths), `ansible/module_utils/kolla_systemd_worker.py`, `ansible/migrate-container-engine.yml`,
  `ansible/roles/prechecks/tasks/{service_checks,package_checks}.yml`, and the libvirt guests unit files under
  `tools/`. Not run on a lab for this page. Status `draft`.
- What a Podman package upgrade does to running containers depends on the distribution's packaging; it was not
  tested here.
- The host preparation (`bootstrap-servers`, the `baremetal` role) is in the `openstack.kolla` collection, installed
  by `install-deps`; it was not read for this page.
- Not covered: rootless Podman (Kolla-Ansible runs the containers as root), Podman's own auto-update, and Podman's
  registries configuration.

## Related

- [pull fails with "http: server gave HTTP response to HTTPS client"](../deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md): the Docker side of registry configuration
- [deploy, reconfigure, upgrade and the others](../operations/deploy-reconfigure-upgrade-and-limiting-runs.md)
- Kolla-Ansible 2026.1: [advanced configuration](https://docs.openstack.org/kolla-ansible/2026.1/admin/advanced-configuration.html),
  [troubleshooting](https://docs.openstack.org/kolla-ansible/2026.1/user/troubleshooting.html)
- Podman: [commands](https://docs.podman.io/en/stable/Commands.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=comparisons/docker-versus-podman-with-kolla-ansible.md&title=Feedback%3A+comparisons/docker-versus-podman-with-kolla-ansible.md):
your release, your topology, and what happened.
