---
title: deploy, reconfigure, upgrade and the others. Which kolla-ansible command to run after a change, and how to narrow it with --limit and --tags
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, docker]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for running platforms
  scale: any
  network: any
  storage: any
  os: any supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, all-in-one, ML2/OVS, no HAProxy", date: 2026-10-27, note: "safe sequence (prechecks, genconfig, validate-config, diff with -u root, reconfigure --tags) and an untagged reconfigure on Glance, one host; kolla_serial override and --limit fact gathering observed; deploy, deploy-containers, upgrade, stop, destroy, rolling runs, multi-host --limit and the HAProxy restart not run" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/advanced-configuration.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/adding-and-removing-hosts.html
  - https://bugs.launchpad.net/kolla-ansible/+bug/2054348
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# deploy, reconfigure, upgrade and the others

## Context

Kolla-Ansible has one playbook, `site.yml`, and most commands run it with a different `kolla_action`. Knowing what
each action does on a running platform, and what `--limit` and `--tags` really narrow, is what makes the difference
between a change that restarts one service and one that restarts the whole control plane. This page sorts the
commands by what they change, and gives a safe sequence for a configuration change. It follows the Kolla-Ansible
2026.1 operating guide and the CLI and roles of that release.

## Topology

Not specific to a topology. Everything below runs on the deploy host, from the virtual environment that holds the
Kolla-Ansible version of the running platform.

## Network design

Not affected.

## Storage design

Not affected.

## Inventory and configuration layout

The commands read the inventory (`-i`, several allowed), `/etc/kolla/globals.yml`, `/etc/kolla/passwords.yml` and the
overrides under `/etc/kolla/config/` (see [laying out the configuration](../configuration/inventory-globals-overrides-and-secrets-layout.md)).
Keep that directory in version control, and commit before each run, so that "what changed" has an answer.

## What each command changes

From `kolla_ansible/cli/commands.py` and the operating guide, 2026.1:

| Command | `kolla_action` | What it does to running containers | Risk |
|---|---|---|---|
| `prechecks` | `precheck` | Nothing; checks ports, files, variables before a deploy | read-only |
| `genconfig` | `config` | Writes the configuration files under `/etc/kolla/<container-config-dir>/` on the hosts; no container restarts, so not applied yet | state-changing (files) |
| `validate-config` | `config_validate` | Validates the generated configuration files; reports under `/var/log/kolla/config-validate` | read-only |
| `check` | `check` | Checks container status | read-only |
| `pull` | `pull` | Downloads images; no container changes | read-only for services |
| `deploy` | `deploy` | Writes configuration, bootstraps what is new (databases, users, endpoints), creates missing containers, restarts containers whose configuration or definition changed | state-changing |
| `reconfigure` | `reconfigure` | In most roles the same tasks as `deploy` (`reconfigure.yml` imports `deploy.yml`) | state-changing |
| `deploy-containers` | `deploy-containers` | Checks and updates containers without generating configuration | state-changing |
| `upgrade` | `upgrade` | Runs each role's upgrade tasks: new images, database migrations, rolling restarts where the role supports them | state-changing; see the [upgrade playbook](../version-notes/2025.1-to-2026.1.md) |
| `post-deploy` | `post-deploy` | Writes `admin-openrc.sh` on the deploy host | state-changing (local file) |
| `stop` | `stop` | Stops containers; needs `--yes-i-really-really-mean-it` | state-changing, outage |
| `destroy` | `destroy` | Removes containers and volumes; needs `--yes-i-really-really-mean-it`; `--include-images` and `--include-dev` remove more | destructive |

`deploy` and `reconfigure` restart a container only when something changed. Kolla-Ansible compares each running
container with what it should be (image, volumes, environment) and runs `kolla_set_configs --check` inside it against
the newly generated files (`compare_container` in `ansible/module_utils/kolla_docker_worker.py`); only containers that
differ are restarted. So a `reconfigure` after a change in one service's override restarts that service only, even
without `--tags`. What `--tags` saves is time, and exposure to changes you did not mean to apply.

The run's `changed` count does not count restarts. On a 2026.1 all-in-one lab, a full `reconfigure` after a change
in Glance's override ended with `changed=22` (bootstrap containers of Keystone, Glance, Neutron, Heat and Placement,
Keystone user registration, Fernet key distribution …) and restarted `glance_api` only. The restarts show as
`Notify handlers to restart containers for <service>` among the changed tasks; `docker inspect -f
'{{.Name}} {{.State.StartedAt}}'` before and after the run confirms them.

## Narrowing a run

**`--tags <service>`** selects plays and tasks by tag. In `site.yml` each service's play is tagged with its name
(`nova`, `neutron`, `keystone`, `mariadb`, `loadbalancer` …), and the HAProxy configuration of a service is tagged with
the service name too, so `--tags nova` also updates Nova's frontends in HAProxy. Fact gathering is tagged `always` and
always runs.

**`--limit <hosts>`** restricts the hosts. Kolla-Ansible still gathers facts for every host (the `gather-facts` play
delegates it to the limited hosts), so an unreachable host still fails the run. The operating guide warns: "using
the ansible `--limit` option is not recommended", because of known bugs with how `register:` interacts with it (bug
2054348; upgrading parts of Nova is the guide's example). Use it where the upstream documentation does (adding
compute nodes, removing hosts, batching `bootstrap-servers`), and test other uses on a lab.

**`-e kolla_serial=<n or %>`** runs each play in batches of hosts (a rolling run); the bootstrap servers guide uses it
with `bootstrap-servers`. Beware with `reconfigure` and `upgrade`: in the 2026.1 CLI these two commands add
`-e kolla_serial=0` themselves, and they add it after the operator's own `-e` options (`kolla_ansible/ansible.py`
appends the command's variables last; Ansible keeps the last value). A `-e kolla_serial=…` given to `reconfigure` or
`upgrade` is therefore overridden and the run is not rolling. Seen on a lab: `kolla-ansible reconfigure … -e
kolla_serial=1` ran `ansible-playbook` with `-e kolla_serial=1 -e kolla_action=reconfigure -e kolla_serial=0`, in
that order.

**Never** combine `--limit` with clustered services in a way that leaves the cluster without its other members in the
play when membership changes (MariaDB, RabbitMQ): the upstream guide asks to include all controllers (`--limit
control`) when controllers are added or removed.

## A safe sequence for a configuration change

1. Change `globals.yml` or the override, commit it.
2. Read-only: `kolla-ansible prechecks -i <inventory>` (or with `--tags <service>`).
3. Generate and look at the result without applying it:

   > ⚠️ **State-changing.** Rewrites the configuration files under `/etc/kolla/<container-config-dir>/` on the hosts.
   > Running containers keep their current configuration until they restart; a container restarted for another reason
   > (a host reboot, a crash) picks up the new files. Undo: revert the change and run `genconfig` again.

   ```bash
   kolla-ansible genconfig -i <inventory> --tags <service>
   kolla-ansible validate-config -i <inventory> --tags <service>
   # on a host, compare with what the container runs:
   sudo docker exec -u root <container> diff /etc/<service>/<file> /var/lib/kolla/config_files/<file>
   ```

   Inside the container, `/var/lib/kolla/config_files/` is the bind mount of `/etc/kolla/<container-config-dir>/` on
   the host (named after the container, for example `/etc/kolla/glance-api/` or `/etc/kolla/nova-compute/`), where
   `genconfig` wrote the new files; `/etc/<service>/` holds the files the service runs with. That directory is
   `drwxrwx--- root root`, and `docker exec` runs as the service user (`glance` for `glance_api`), hence `-u root`;
   without it `diff` fails with `Permission denied`.

   `validate-config` exits 0 even when it finds errors: read the reports under
   `/var/log/kolla/config-validate/<host>/<service>/<container>/*.err`. On 2026.1 the Glance report holds two `ERROR`
   lines for options Kolla-Ansible itself sets (`os_glance_tasks_store/filesystem_store_datadir is not part of the
   sample config`, same for `os_glance_staging_store`); compare with a report taken before your change.

4. Apply it:

   > ⚠️ **State-changing.** Restarts the containers of that service whose configuration changed, on every host where
   > it runs, all at once (see the `kolla_serial` remark above). API calls to that service fail briefly; for agents of
   > the data plane (Neutron agents, `nova_compute`, `openvswitch`), say what a restart interrupts before running it.
   > If the change touches the service's HAProxy frontends (TLS, ports, external endpoints), `--tags <service>` also
   > restarts the `haproxy` containers, backups first, then the host holding the VIP: every API behind the VIPs is
   > briefly interrupted. Undo: revert the change and run the same command.

   ```bash
   kolla-ansible reconfigure -i <inventory> --tags <service>
   ```

5. Verify: the run ends with `failed=0`, the service's containers are `healthy` (`docker ps`), and a real API call works.

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| `reconfigure --tags <service>` for one service | A full `reconfigure` | Shorter, and does not apply unrelated pending changes | Pending changes in other services stay unapplied until the next full run; know what is pending |
| A full `reconfigure` from time to time | Tags only | Proves the whole configuration still applies | Long; restarts whatever drifted |
| `genconfig` + `validate-config` before applying | Apply directly | See the files before any restart | One more step |
| `kolla_serial` for rolling runs | All hosts at once | Keeps some containers up while others restart | Works with `deploy` and `bootstrap-servers`; with `reconfigure` and `upgrade` the CLI overrides it (above) |
| `--limit` only where upstream documents it | `--limit` for everything | Known bugs (bug 2054348) | Some changes need a full-inventory run |

## What went wrong during deployment

Nothing observed for this page. The common traps it is written against:

- a `deploy` run "to fix one thing" that also applied every pending change in `globals.yml`;
- `--limit` excluding the hosts a task delegates to or registers results on;
- a newer Kolla-Ansible on the deploy host than the platform's release: `deploy` then deploys the new images, which
  is an upgrade without the upgrade tasks.

## Version notes

Written against Kolla-Ansible 2026.1 (cliff-based CLI, `kolla_ansible/cli/commands.py`). Command names and the
`--yes-i-really-really-mean-it` guard are those of that release. `reconfigure.yml` importing `deploy.yml` holds for the
roles read (Glance, MariaDB, Nova cell); check a role before relying on it for another service.

## Lessons learned

- Read the run's `changed` tasks before believing that "nothing restarted".
- Keep the deploy host's Kolla-Ansible pinned to the platform's release.
- Prefer one service at a time with `--tags`, and a full `reconfigure` in a maintenance window.

## Validity and limits

- Written from the Kolla-Ansible 2026.1 operating guide, `kolla_ansible/cli/commands.py`, `ansible/site.yml`,
  `ansible/gather-facts.yml`, some roles' `reconfigure.yml`, and `ansible/module_utils/kolla_docker_worker.py`.
- Lab run, 2026-10-27, Kolla-Ansible 22.2.0 (2026.1) all-in-one, Docker, no HAProxy: the safe sequence ran end to end
  on Glance (`debug = True` in `/etc/kolla/config/glance.conf`): `prechecks --tags glance` 15 s, `genconfig` 10 s
  (no container restarted), `validate-config` 23 s, the in-container `diff` (with `-u root`) showed the change,
  `reconfigure --tags glance` 76 s restarted `glance_api` only (healthy 11 s later). The undo (override removed,
  full `reconfigure` without `--tags`) took 647 s and restarted `glance_api` only. `-e kolla_serial=1` overridden by
  the CLI's `kolla_serial=0` (command line observed). `--limit localhost` with an unreachable host in the inventory
  failed in `Gather facts for all hosts (if using --limit)` (`UNREACHABLE`, rc 4).
- Not run on the lab: `deploy`, `deploy-containers`, `upgrade`, `pull`, `check`, `post-deploy`, `stop`, `destroy`;
  a rolling run (`kolla_serial` with `deploy`); `--limit` across real hosts and bug 2054348; the HAProxy restart by
  `--tags <service>` (HAProxy disabled on the lab); restarts on several hosts at once. Status `draft`.
- `docker exec … diff` assumes the image has `diff`; most Kolla images do, not checked for each.
- Not covered: Podman specifics, `migrate-container-engine`, Bifrost commands, `nova-libvirt-cleanup`,
  `rabbitmq-reset-state`.

## Related

- [Adding a compute node](adding-a-compute-node.md): a documented use of `--limit`
- [From 2025.1 to 2026.1](../version-notes/2025.1-to-2026.1.md): the upgrade sequence
- [Rotating the secrets in passwords.yml](../configuration/rotating-secrets-in-passwords-yml.md)
- Kolla-Ansible 2026.1: [operating Kolla](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html),
  [advanced configuration](https://docs.openstack.org/kolla-ansible/2026.1/admin/advanced-configuration.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=operations/deploy-reconfigure-upgrade-and-limiting-runs.md&title=Feedback%3A+operations/deploy-reconfigure-upgrade-and-limiting-runs.md):
your release, your topology, and what happened.
