---
title: A local registry and an offline deployment with Kolla-Ansible. What has to be mirrored, the variables that point at the mirrors, and the Docker setting that does not help
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, docker, ansible]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for platforms with no or restricted Internet access from the hosts
  scale: any multinode deployment
  network: any
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/quickstart.html
  - https://distribution.github.io/distribution/recipes/mirror/
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# A local registry and an offline deployment with Kolla-Ansible. What has to be mirrored, the variables that point at the mirrors, and the Docker setting that does not help

## Context

A default Kolla-Ansible deployment reaches the Internet from several places, not only for container images:

| What | From where, by default (2026.1) | Used by |
|---|---|---|
| Container images | `quay.io`, namespace `openstack.kolla` (`docker_registry`, `docker_namespace`) | `kolla-ansible pull` and `deploy` on every host |
| Kolla-Ansible itself | PyPI or a Git checkout | `pip install` on the deploy host |
| Ansible collections | Ansible Galaxy (`requirements-core.yml`) and the `ansible-collection-kolla` Git repository (`requirements.yml`) | `kolla-ansible install-deps` on the deploy host |
| Docker packages | `download.docker.com` (`enable_docker_repo: true`, `docker_apt_url`, `docker_yum_url`) | `kolla-ansible bootstrap-servers` on every host |
| OS packages | The hosts' own repositories | `bootstrap-servers` and the OS itself |

A local registry alone removes the first line. An offline (air-gapped) deployment needs all five. The Kolla-Ansible
multinode guide recommends "some type of local registry" for any multinode deployment, so that hosts do not each
pull every image from the public registry.

## Topology

One registry host (or an HA registry; out of scope here) reachable from every Kolla-Ansible host, and, for an offline
platform, mirrors for Python packages, collections and OS packages, usually on the same "bastion" or artifact host.
The deploy host needs only the deploy-side artifacts.

## Network design

Every host pulls images from the registry: open its port (the guide uses 4000, to avoid Keystone's 5000 on a host that
also runs Keystone) from the management network. Prefer TLS: the bootstrap servers guide says
`docker_registry_insecure` allows "broken certificate chains and HTTP without TLS", is "strongly discouraged in
production", and "will disable Docker registry authentication".

## Storage design

The registry's storage holds one copy of every image of the release and tag in use: plan for the images of every
enabled service, for each `openstack_tag` you keep (old tags are what you roll back to).

## Inventory and configuration layout

### Images

Kolla-Ansible builds every image reference as `<docker_registry>/<docker_namespace>/<docker_image_name_prefix><name>:<tag>`,
with `openstack_tag` defaulting to `<openstack_release>-<kolla_base_distro>-<kolla_base_distro_version>` (2026.1
`group_vars/all/common.yml`). Keep the same namespace and tags in your registry, and only the registry changes:

```yaml
# globals.yml
docker_registry: "registry.example.org:4000"
docker_namespace: "openstack.kolla"      # the default; keep it if you mirror with the same paths
docker_registry_ca: "/etc/kolla/certificates/registry-ca.crt"   # copied to /etc/docker/certs.d/<registry>/ca.crt
# docker_registry_username: "<user>"     # if the registry needs a login; the password is docker_registry_password
```

`docker_registry_ca` is a variable of the `docker` role in `ansible-collection-kolla` (2026.1): with a registry
certificate signed by a private CA, it puts the CA where Docker looks for it, without making the registry insecure.
`docker_registry_password` is in `passwords.yml`; the password rotation guide notes that Kolla-Ansible does not manage
the registry itself.

### The setting that does not help: `docker_registry_mirrors`

The same role has `docker_registry_mirrors`, which writes Docker's `registry-mirrors` option. Distribution's
pull-through cache recipe is explicit: "Currently Docker daemon supports only mirrors of Docker Hub. It is not possible
to run the Docker daemon against a pull through cache with another upstream registry." Kolla images come from
`quay.io`, so a mirror configured there is never used for them. Point `docker_registry` at your registry instead.

### Two ways to fill the registry

- **Pull-through cache.** The CNCF Distribution registry (the `registry:2` image the guide runs) can act as a cache of
  one upstream registry with `proxy.remoteurl` in its configuration (Distribution's *Registry as a pull through cache*
  recipe; "it's currently possible to mirror only one upstream registry at a time"). The recipe is written for Docker
  Hub, and the Docker daemon cannot use such a cache as a mirror for another registry (above). Addressing a cache of
  `https://quay.io` directly through `docker_registry` avoids the daemon's mirror mechanism, but was not tested here.
  A cache still needs the upstream at the first pull of each image, so it is not offline by itself.
- **Explicit copy.** Copy each image and tag from `quay.io/openstack.kolla/...` into the registry, on a connected host
  (`docker pull`, `docker tag`, `docker push`, or a copy tool such as `skopeo`), then carry the registry's storage, or
  the images, to the offline side. The list of images is the list of enabled services: a `kolla-ansible pull` on a
  connected lab with the same `globals.yml` shows which ones are needed (`docker images` afterwards).

> ⚠️ **State-changing.** Starts a registry container on the registry host, the guide's simple example. It stores images
> in the `registry` volume. Undo: `docker rm -f registry`; `docker volume rm registry` deletes the images.

```bash
docker run -d --network host --name registry --restart=always \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:4000 \
  -v registry:/var/lib/registry \
  registry:2
```

For TLS, Distribution takes a certificate and key through its own configuration (`REGISTRY_HTTP_TLS_CERTIFICATE`,
`REGISTRY_HTTP_TLS_KEY`); see the Distribution documentation. The guide's example is plain HTTP.

### The deploy host, offline

- **Kolla-Ansible:** install it from a local Python package index or from wheels downloaded on a connected host
  (`pip download kolla-ansible==<version>` with the same Python version and platform).
- **Collections:** `kolla-ansible install-deps` installs `requirements-core.yml` from Galaxy and `requirements.yml`
  from Git. Offline, download them on a connected host (for example with `ansible-galaxy collection download`, and an
  archive of the `ansible-collection-kolla` branch) and install them from the files. Not tested here.

### The hosts, offline

- **Docker packages:** `bootstrap-servers` adds Docker's own repository when `enable_docker_repo` is true. Point
  `docker_apt_url` (Debian, Ubuntu) or `docker_yum_url` (Rocky and others) at a local mirror of
  `download.docker.com`, or set `enable_docker_repo: false` and provide the packages through your OS mirror.
- **OS packages:** whatever `bootstrap-servers` installs comes from the hosts' configured repositories: point them at
  your local mirror before running it.

## Applying it

**1. Read-only checks** from a target host, once the registry is filled:

```bash
curl -s --cacert <registry-ca.crt> https://registry.example.org:4000/v2/_catalog | head   # the openstack.kolla/* repositories
curl -s --cacert <registry-ca.crt> https://registry.example.org:4000/v2/openstack.kolla/<image>/tags/list
```

**2. Apply the Docker settings** on the hosts:

> ⚠️ **State-changing.** Writes `/etc/docker/daemon.json` and the registry CA on every host, and restarts Docker where
> the configuration changed, which restarts every container on that host. On a running platform, limit it to new
> hosts or plan a window per host. Undo: restore the previous variables and run it again.

```bash
kolla-ansible bootstrap-servers -i <inventory> --limit <new-or-maintenance-hosts>
```

**3. Pull, then deploy.** `kolla-ansible pull` downloads every image a host needs without starting anything; it is the
first place a missing image or a registry problem shows up.

```bash
kolla-ansible pull -i <inventory>
```

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| `pull` fails with `http: server gave HTTP response to HTTPS client` | the registry is plain HTTP and Docker expects TLS: see [the deployment error page](../deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md) |
| `pull` fails with `x509: certificate signed by unknown authority` | `docker_registry_ca` not set, or not the CA that signed the registry certificate; `/etc/docker/certs.d/<registry>/ca.crt` on the host |
| `manifest unknown` or `not found` for one image | that image or tag is missing from the registry: compare with `/v2/<namespace>/<image>/tags/list` |
| Hosts still reach `quay.io` | `docker_registry` not set in the `globals.yml` used, or images referenced by a variable overriding `docker_image_url` |
| `bootstrap-servers` fails installing Docker | the package mirror URL (`docker_apt_url`, `docker_yum_url`) or the OS repositories; the Ansible output and `/var/log/apt/` or `/var/log/dnf.log` on the host |
| Docker does not restart | `journalctl -u docker` on the host: a malformed `daemon.json` |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Local registry with the same namespace and tags | Pulls from `quay.io` on every host | One download per image; works with restricted egress | A registry to run, size and back up |
| TLS with `docker_registry_ca` | `docker_registry_insecure: true` | The guide discourages insecure registries in production; insecure disables authentication | A certificate to issue and renew for the registry |
| Explicit copy of images | Pull-through cache | Works fully offline; the content is exactly what was tested | Each new tag or service must be copied before use |
| Pull-through cache | Explicit copy | Little to maintain with restricted (not absent) egress | Not offline: the first pull of each image needs `quay.io`; caching `quay.io` addressed through `docker_registry` not tested here |

## What went wrong during deployment

Nothing observed yet for this page. The plain-HTTP registry failure is documented on its own page, linked above.

## Version notes

- 2026.1: the Docker settings (`docker_registry_ca`, `docker_registry_mirrors`, `docker_apt_url`, `docker_yum_url`,
  `enable_docker_repo`) live in the `docker` role of `ansible-collection-kolla`, installed by `install-deps`.
- Images are tagged per release and base distribution; an upgrade needs the new release's images in the registry
  before `kolla-ansible pull`.

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- List everything the platform downloads, not only images.
- Do not rely on `docker_registry_mirrors` for Kolla images.
- Run `kolla-ansible pull` before any deploy or upgrade on an offline platform: it finds missing images without
  touching the running services.

## Validity and limits

Written from Kolla-Ansible 2026.1 (multinode guide, bootstrap servers guide, quickstart, `group_vars/all/common.yml`,
`requirements.yml`, `requirements-core.yml`), the `docker` role of `ansible-collection-kolla` 2026.1, and the
Distribution documentation. Not run on a lab: no output on this page was observed. Not covered: Podman
(`kolla_container_engine: podman`) and its registry configuration, an HA registry, building your own images with
Kolla, signing and scanning images, and the exact offline procedure for collections.

## Related

- [kolla-ansible pull fails with "http: server gave HTTP response to HTTPS client"](../deployment-errors/pull-fails-http-response-to-https-client-insecure-registry.md)
- [Laying out the configuration](inventory-globals-overrides-and-secrets-layout.md): where `globals.yml` and
  `passwords.yml` live.
- Kolla-Ansible 2026.1: [deploy a registry](https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html),
  [bootstrap servers](https://docs.openstack.org/kolla-ansible/2026.1/reference/deployment-and-bootstrapping/bootstrap-servers.html)
- Distribution: [registry as a pull through cache](https://distribution.github.io/distribution/recipes/mirror/)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code, the
`ansible-collection-kolla` 2026.1 code, and the Distribution documentation. Reviewed in its pull request
(see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=configuration/local-registry-and-offline-deployment.md&title=Feedback%3A+configuration/local-registry-and-offline-deployment.md):
your release, your topology, and what happened.
