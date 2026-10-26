---
title: Rotating the secrets in passwords.yml. Which ones a reconfigure applies, which need manual steps, and which need an outage
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, mariadb, rabbitmq, memcached, nova, keystone, horizon, heat]
deployment_methods: [kolla-ansible]
environment:
  purpose: any (lab, pre-production, production)
  scale: any
  network: any
  storage: any
  os: any supported by Kolla-Ansible 2026.1
risk: destructive
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/password-rotation.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Rotating the secrets in passwords.yml. Which ones a reconfigure applies, which need manual steps, and which need an outage

## Context

`passwords.yml` holds every internal secret of a Kolla-Ansible deployment: service passwords, database and message
queue credentials, keys. Rotating them is needed after a leak, when someone with access leaves, or by policy. Kolla-
Ansible's password rotation guide (2026.1) splits them into three groups, and the group decides the cost:

1. secrets that a `kolla-ansible reconfigure` applies;
2. secrets that need a manual step first, then `reconfigure`;
3. RabbitMQ's secrets, which need every service stopped and RabbitMQ rebuilt.

This page lays out those groups and the order to follow, from the guide. It does not replace it: the guide has the
exact commands for each manual step.

## Topology

Not affected: the rotation changes credentials, not roles.

## Network design

Not affected.

## Storage design

Not affected, except that the RabbitMQ step destroys the `rabbitmq` Docker volume on each RabbitMQ node.

## Inventory and configuration layout

The rotation edits `passwords.yml` and then runs Kolla-Ansible. If `passwords.yml` is kept encrypted with Ansible
Vault, the [configuration layout pattern](inventory-globals-overrides-and-secrets-layout.md) describes how to run
`kolla-genpwd` on a plain working copy and encrypt the result again: `kolla-genpwd` reads and writes plain YAML.

### Regenerating secrets

The guide's way: back up the file, empty the values to rotate, let `kolla-genpwd` fill them again.

```bash
cp /etc/kolla/passwords.yml ./passwords.yml.bak       # keep it until the rotation is checked
# in passwords.yml, change  foo: "old"  to  foo:   for each secret to rotate
```

> ⚠️ **State-changing.** `kolla-genpwd` rewrites `/etc/kolla/passwords.yml` in place, filling every empty value.
> Nothing reaches the platform until a `reconfigure` or `deploy`. Undo: copy the backup back.

```bash
kolla-genpwd -p /etc/kolla/passwords.yml
```

The guide's warning: "Always back up your `passwords.yml` file before making any changes. Otherwise, it is easy to
make unrecoverable mistakes." Keep the backup out of any repository, like the file itself.

## Key choices and their consequences

| Group | Secrets (from the guide) | How it is applied | Cost |
|---|---|---|---|
| Applied by `reconfigure` | `*_keystone_password`, `*_database_password` except `nova_database_password`, `*_ssh_key` except `kolla_ssh_key`, `keystone_admin_password`, `designate_rndc_key`, `keepalived_password`, `libvirt_sasl_password`, `metadata_secret`, `opensearch_dashboards_password`, `osprofiler_secret`, `prometheus_alertmanager_password`, `qdrouterd_password`, `valkey_master_password` | `kolla-ansible reconfigure` | The containers whose configuration changes restart |
| Manual step, then `reconfigure` | `horizon_secret_key`, `grafana_admin_password`, `database_password`, `nova_database_password`, `heat_domain_admin_password` | The guide's step for each, in this order, then `reconfigure` | From the Nova database password on, the guide warns that the API may be disrupted |
| Outage | `rabbitmq_cluster_cookie`, `rabbitmq_password` | Stop every service, destroy RabbitMQ's containers and volumes, `deploy`, flush Memcached | All APIs down until the deploy completes |
| Not handled by Kolla-Ansible | `kolla_ssh_key` (no mechanism, per the guide); `docker_registry_password` (Kolla-Ansible does not manage registries) | Outside Kolla-Ansible | Depends on the tool used |

### The manual steps, in the guide's order

- **`horizon_secret_key`:** Horizon accepts two keys at once, the new one first, written as
  `horizon_secret_key: newsecret' 'oldsecret`. Remove the old one and reconfigure again once old links have expired,
  about one to two weeks according to the guide.
- **`grafana_admin_password`:** reset with `grafana-cli admin reset-admin-password` inside a Grafana container.
- **`database_password`:** the MariaDB `root` password, changed inside the `mariadb` container with
  `SET PASSWORD FOR 'root'@'%' = …`, for every `root` entry that still has the old password.
- **`nova_database_password`:** the guide's warning starts here: "From this point onward, API service may be
  disrupted." The `cell0` database connection stored by Nova holds the password, so it is updated with
  `nova-manage cell_v2 update_cell` from the `nova_conductor` container.
- **`heat_domain_admin_password`:** a Keystone user password, changed with `openstack user set` in the
  `heat_user_domain` domain.

Then one `kolla-ansible reconfigure` applies these and every secret of the first group.

### RabbitMQ: plan an outage

The guide has no seamless process for `rabbitmq_cluster_cookie` and `rabbitmq_password`: reconfiguring every service
for new RabbitMQ credentials takes time, so it recommends stopping everything and rebuilding RabbitMQ.

> 🛑 **Destructive.** Stops every OpenStack service of the deployment, then deletes the RabbitMQ containers and their
> volumes on every RabbitMQ node. Every API is down until `deploy` completes; messages in flight are lost. Running
> instances keep running, but nothing can be created, changed or deleted meanwhile. Precondition: a maintenance
> window, the new values already in `passwords.yml`, and its backup kept. `deploy`, not `reconfigure`, restarts the
> services, because the RabbitMQ containers no longer exist.

```bash
kolla-ansible stop -i <inventory> --yes-i-really-really-mean-it
# on each RabbitMQ node
sudo docker stop rabbitmq
sudo docker rm rabbitmq
sudo docker volume rm rabbitmq
# then, from the deployment host
kolla-ansible deploy -i <inventory>
```

The upstream guide writes `kolla-ansible stop -i inventory` without `--yes-i-really-really-mean-it`. In 2026.1 that
flag is required (`kolla_ansible/cli/commands.py`), and the command exits with an argument error without it.

After the deploy, the guide flushes Memcached on every Memcached container: data stored under the old Memcached
password can no longer be read. From a host that reaches the Memcached network (a Memcached host, if unsure), with a
telnet client installed:

```bash
# on every host running Memcached: the address after -l and the port after -p
sudo grep command /etc/kolla/memcached/config.json
```

> ⚠️ **State-changing.** `flush_all` empties the cache, including cached tokens: services fetch them again, and the
> first requests are slower. Nothing persistent is lost.

```bash
telnet <address> <port>
flush_all
quit
```

## What went wrong during deployment

Nothing recorded: no rotation has been run on the OzarLand labs.

## Version notes

- 2026.1: the groups and steps above are those of Kolla-Ansible's password rotation guide on `stable/2026.1`. The
  guide says it "does not cover every possible `passwords.yml` variable, only the most common ones".

## Lessons learned

No first-hand lessons yet. What the guide stresses:

- Back up `passwords.yml` before touching it.
- Rotate the first group freely; schedule the manual group in a window, in the guide's order, expecting API
  disruption from Nova's database password on; treat RabbitMQ as a planned outage.
- Rotating `horizon_secret_key` is a two-step change: add the new key now, remove the old one later.

## Validity and limits

Written from Kolla-Ansible 2026.1 (password rotation guide, operating guide). Not run on a lab: the group lists, the
order and the commands are the guide's. The guide covers the most common secrets only, and assumes Docker; it says
the same commands work with Podman by replacing `docker` with `podman`.

Not covered: Fernet and credential keys (rotated by Keystone's own mechanism, not by `passwords.yml`), TLS
certificates, Ceph keyrings, HashiCorp Vault as the password store, and secrets of services not listed in the guide.

## Related

- [Laying out the configuration](inventory-globals-overrides-and-secrets-layout.md): where `passwords.yml` lives,
  and how to run `kolla-genpwd` and `kolla-mergepwd` with Ansible Vault.
- [Every API call fails at once: expired certificates or inconsistent Fernet keys](https://github.com/OzarLand/openstack-vm-runbooks/blob/main/diagnostics/expired-certificates-and-fernet-keys-break-every-api-call.md)
- Kolla-Ansible 2026.1: [password rotation](https://docs.openstack.org/kolla-ansible/2026.1/admin/password-rotation.html),
  [operating Kolla](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=configuration/rotating-secrets-in-passwords-yml.md&title=Feedback%3A+configuration/rotating-secrets-in-passwords-yml.md):
your release, your topology, and what happened.
