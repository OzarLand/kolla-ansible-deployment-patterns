---
title: kolla-ansible pull fails with "http: server gave HTTP response to HTTPS client". Docker does not know the registry is plain HTTP
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, docker]
deployment_methods: [kolla-ansible]
environment:
  topology: any; met on an all-in-one lab during a 2025.1 to 2026.1 upgrade
  network: any
  storage: any
  os: ubuntu-24.04 (noble), docker
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html
  - https://docs.docker.com/reference/cli/dockerd/#insecure-registries
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# kolla-ansible pull fails with "http: server gave HTTP response to HTTPS client". Docker does not know the registry is plain HTTP

## Symptom

`kolla-ansible pull` (or `deploy`, or `upgrade`, which pull too) fails on the first image, on every host, with:

```text
http: server gave HTTP response to HTTPS client
```

The registry named in `docker_registry` serves plain HTTP, and `globals.yml` already says
`docker_registry_insecure: true`.

## Impact

No image can be pulled, so the deployment, reconfiguration or upgrade cannot start. Running containers are not
affected: nothing has been changed yet.

## Quick checks

On a host where the pull failed, read-only:

```bash
# 1. Does the Docker daemon know the registry is insecure?
sudo docker info | grep -A5 'Insecure Registries'

# 2. What the daemon configuration file says
sudo cat /etc/docker/daemon.json

# 3. What Kolla-Ansible is asked to use
grep -E '^(docker_registry|docker_registry_insecure)' /etc/kolla/globals.yml
```

The problem is confirmed when `globals.yml` names the registry with `docker_registry_insecure: true`, but neither
`docker info` nor `daemon.json` lists it under the insecure registries.

## Diagnosis

Docker talks HTTPS to every registry except the ones its daemon lists as insecure (`insecure-registries` in
`daemon.json`). `docker_registry_insecure` in `globals.yml` does not reach the daemon by itself: Kolla-Ansible writes
it into Docker's configuration when it prepares the hosts, with `kolla-ansible bootstrap-servers` (its bootstrap
servers guide lists `docker_registry_insecure` among the Docker settings it applies). `pull`, `deploy` and `upgrade`
do not apply it.

So the error appears whenever the registry, or its insecure setting, changed after the last `bootstrap-servers` run:

- a new local registry set up for an upgrade or for a site without Internet access;
- `docker_registry_insecure` added to `globals.yml` later;
- a host added or reinstalled without `bootstrap-servers`.

## Possible causes

### Cause 1: the registry changed after the hosts were bootstrapped

The case of the lab: the images of the 2026.1 upgrade came from a new plain-HTTP registry, `globals.yml` had
`docker_registry_insecure: true`, and the upgrade steps do not run `bootstrap-servers`.

### Cause 2: the registry serves HTTPS with a certificate Docker does not trust

Then the error differs (a certificate error, not this one). Out of scope here: trust the registry's CA on the hosts,
or see Docker's documentation.

## Resolution

### For cause 1

Two ways to give the daemon the setting. Both apply to every host that pulls images.

**By hand, without restarting containers.** Add the registry to `insecure-registries` in `/etc/docker/daemon.json`,
keeping the other keys of the file, then reload the daemon:

```json
{
  "insecure-registries": ["<registry-host>:<port>"]
}
```

> ⚠️ **State-changing.** Changes Docker's daemon configuration and reloads it. On the lab, `systemctl reload docker`
> applied `insecure-registries` without restarting any container (31 containers kept running). A
> `systemctl restart docker` instead restarts every container on the host. Undo: remove the entry and reload again.

```bash
sudo systemctl reload docker
sudo docker info | grep -A5 'Insecure Registries'      # the registry is now listed
```

**With Kolla-Ansible.** `kolla-ansible bootstrap-servers -i <inventory>` writes the Docker configuration from
`globals.yml`.

> ⚠️ **State-changing.** `bootstrap-servers` can restart Docker, and every container on the host with it. The
> bootstrap servers guide describes how to limit that (`--limit`, `-e kolla_serial=…`, Docker's `live-restore`).
> On a running platform, run it in a maintenance window, or prefer the manual way above.

Then pull again:

```bash
kolla-ansible pull -i <inventory>
```

## Verification

```bash
sudo docker info | grep -A5 'Insecure Registries'     # lists the registry on every host
kolla-ansible pull -i <inventory>                       # ends with failed=0
```

## Lab observations

Met on 2026-10-22 during the [upgrade of an all-in-one lab from 2025.1 to
2026.1](../version-notes/upgrade-report-2025.1-to-2026.1-all-in-one-lab.md) (Kolla-Ansible 22.2.0, Ubuntu 24.04,
Docker), step 6 of the upgrade, `pull`:

- the 2026.1 images came from a local plain-HTTP registry, and `globals.yml` had `docker_registry_insecure: true`;
- `pull` failed with `http: server gave HTTP response to HTTPS client`;
- adding the registry to `insecure-registries` in `/etc/docker/daemon.json` and running `sudo systemctl reload
  docker` fixed it without restarting any of the 31 containers;
- `bootstrap-servers` was not tried during the upgrade, because it can restart Docker.

The [upgrade playbook](../version-notes/2025.1-to-2026.1.md) now applies Docker changes before `pull` (step 5).

## Validity and limits

- Observed once, on one all-in-one lab, during an upgrade. The other triggers (a registry added later, a host added
  without `bootstrap-servers`) follow from the same mechanism but were not run.
- Not covered: Podman (its registry configuration differs), registries with TLS and a private CA, registry
  authentication (the bootstrap servers guide notes that `docker_registry_insecure` disables it), and image mirrors.

## Related

- [Upgrading an all-in-one lab from 2025.1 to 2026.1](../version-notes/upgrade-report-2025.1-to-2026.1-all-in-one-lab.md):
  the run where this was met, and the proxy problem of `bootstrap-servers` in 2025.1.
- [From 2025.1 to 2026.1](../version-notes/2025.1-to-2026.1.md), step 5.
- Kolla-Ansible 2026.1: [bootstrap servers](https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html),
  [deploy a registry](https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html)
- Docker: [insecure registries](https://docs.docker.com/reference/cli/dockerd/#insecure-registries)

---

*Provenance: written by agent:ozarland-author, from a lab upgrade run by agent:lab-reviewer on 2026-10-22 and the
Kolla-Ansible 2026.1 documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md&title=Feedback%3A+deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md):
your release, your topology, and what happened.
