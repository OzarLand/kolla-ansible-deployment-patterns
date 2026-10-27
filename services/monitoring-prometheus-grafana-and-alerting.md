---
title: Monitoring with Prometheus, Grafana and Alertmanager in Kolla-Ansible. What is deployed, and the rules, receivers and dashboards you still have to bring
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, prometheus, grafana, haproxy]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for a multinode deployment monitored by its own Prometheus
  scale: any, with a monitoring group (the controllers in the sample inventory)
  network: any
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/logging-and-monitoring/prometheus-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/logging-and-monitoring/grafana-guide.html
  - https://prometheus.io/docs/alerting/latest/configuration/
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Monitoring with Prometheus, Grafana and Alertmanager in Kolla-Ansible. What is deployed, and the rules, receivers and dashboards you still have to bring

## Context

`enable_prometheus: "yes"` gives a Kolla-Ansible deployment a Prometheus server, Alertmanager and a set of exporters,
all scraped automatically; `enable_grafana: "yes"` adds Grafana with Prometheus as a data source. It is a quick way
to get metrics for the whole platform without a separate monitoring project.

What it does not give, in 2026.1, is what turns metrics into monitoring:

- **no alert rules:** Prometheus loads only the `*.rules` files you put in `/etc/kolla/config/prometheus/`, and the
  role copies them only while Alertmanager is enabled (`enable_prometheus_alertmanager`, on by default);
- **no notification:** the default Alertmanager configuration routes every alert to a receiver with nothing in it;
- **no dashboards:** Grafana provisions only the dashboards you put in `/etc/kolla/config/grafana/dashboards/`.

This page lists what is deployed and where, and shows how to add those three parts the way the role expects.

## Topology

Group names are those of the 2026.1 sample multinode inventory, where `monitoring` holds the controllers.

| Container | Inventory group | What it does |
|---|---|---|
| `prometheus_server` | `prometheus` (`monitoring`) | Scrapes every target, evaluates rules, port 9091 |
| `prometheus_alertmanager` | `prometheus-alertmanager` (`monitoring`) | Groups and sends alerts, port 9093; clustered on port 9094 when there are several |
| `prometheus_node_exporter` | `monitoring`, `control`, `compute`, `network`, `storage` | Host metrics, port 9100 |
| `prometheus_cadvisor` | same as node exporter | Container metrics, port 18080 |
| `prometheus_openstack_exporter` | `monitoring` | Metrics from the OpenStack APIs (servers, agents, services) |
| `prometheus_blackbox_exporter` | `monitoring` | Probes the public endpoint of every enabled service, and its internal endpoint too when the internal and external VIPs differ (`kolla_same_external_internal_vip` false) |
| `prometheus_mysqld_exporter`, `prometheus_memcached_exporter`, `prometheus_libvirt_exporter`, ... | where the service runs | One exporter per infrastructure service, each switched by its own `enable_prometheus_*_exporter` variable |
| `grafana` | `grafana` | Dashboards, port 3000 |

RabbitMQ metrics come from RabbitMQ's own Prometheus plugin, not from a separate container: the `rabbitmq` role
enables it when `rabbitmq_enable_prometheus_plugin` is true, which follows `enable_prometheus`, and Prometheus scrapes
it only when `enable_prometheus_rabbitmq_exporter` is true (default: `enable_prometheus` and `enable_rabbitmq`), even
though no exporter container exists for it. Each Prometheus server
scrapes every target on its own; HAProxy sends the internal endpoint to one of them at a time
(`prometheus_active_passive: true` by default).

## Network design

Prometheus scrapes each target on the address of its API interface (`prometheus_target_address` per host overrides
it). Exposure through HAProxy in 2026.1, from the group variables:

| Endpoint | Internal VIP | External VIP |
|---|---|---|
| Prometheus | yes | no (`enable_prometheus_server_external: false`) |
| Alertmanager | yes | **yes by default** (`enable_prometheus_alertmanager_external` follows `enable_prometheus_alertmanager`) |
| Grafana | yes | **yes by default** (`enable_grafana_external` follows `enable_grafana`) |

Prometheus and Alertmanager are protected by basic authentication (`admin` with `prometheus_password` and
`prometheus_alertmanager_password` from `passwords.yml`), Grafana by its own login (`admin` and
`grafana_admin_password`). If the external VIP faces a network you do not trust, set the two `*_external` variables
to `false` and reach the interfaces from the internal network.

## Storage design

Prometheus keeps its time series in a Docker volume on each `prometheus_server` host (`/var/lib/prometheus` inside the
container). Retention is Prometheus's default unless you pass flags, as the guide shows:

```yaml
prometheus_cmdline_extras: "--storage.tsdb.retention.size=30GB"
```

Size the controllers' disk for it: Prometheus's volume competes with MariaDB, RabbitMQ and the images on the same
host when `monitoring` is the controllers.

## Inventory and configuration layout

```text
/etc/kolla/
  globals.yml                         # enable_prometheus, enable_grafana, *_external, prometheus_cmdline_extras
  config/
    prometheus/
      *.rules                         # alert rules, copied to every prometheus_server, listed in rule_files
      prometheus-alertmanager.yml     # replaces the whole Alertmanager configuration
      *.tmpl                          # Alertmanager notification templates, copied as they are
      prometheus.yml.d/*.yml          # merged into prometheus.yml: extra scrape jobs
      <inventory_hostname>/           # per-host prometheus.yml.d/, prometheus-alertmanager.yml, prometheus.yml,
                                      #   web.yml, prometheus-blackbox-exporter.yml (not *.rules or *.tmpl)
      extras/                         # templated files referenced from prometheus.yml (file_sd targets)
    grafana/
      dashboards/*.json               # provisioned dashboards
      grafana_home_dashboard.json     # optional home dashboard
      provisioning.yaml               # optional, replaces the dashboard provider configuration
```

The rules, the Alertmanager file and `prometheus.yml.d` follow the role's `config.yml` in 2026.1; the Grafana paths
follow the Grafana guide. A receiver configuration can contain credentials (an SMTP password, a webhook token): treat
`prometheus-alertmanager.yml` as a secret, like `passwords.yml`.

### Alert rules

Any Prometheus rule file works. A minimal start that uses only metrics Kolla-Ansible's scrape configuration produces:
`up` for every scrape target, and `probe_success` from the blackbox exporter for every service endpoint.

```yaml
# /etc/kolla/config/prometheus/platform.rules
groups:
  - name: platform
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }} target {{ $labels.instance }} is down"
      - alert: EndpointDown
        expr: probe_success == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Endpoint probe failed: {{ $labels.instance }}"
```

The role copies `*.rules` files as they are, without templating, so `{{ $labels.job }}` reaches Prometheus intact.
Many operators start from a published rule collection (node exporter, MySQL, RabbitMQ mixins) rather than from
scratch; check its metric names against the exporters' versions you run.

### Alertmanager receivers

The file replaces the role's template entirely, so it must be a complete Alertmanager configuration
([Alertmanager configuration reference](https://prometheus.io/docs/alerting/latest/configuration/)). An e-mail
example with placeholders; keep the `templates` line of the default if you use the role's Teams template:

```yaml
# /etc/kolla/config/prometheus/prometheus-alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: "<smtp-host>:587"
  smtp_from: "alertmanager@example.org"
  smtp_auth_username: "<smtp-user>"
  smtp_auth_password: "<smtp-password>"
  smtp_require_tls: true
route:
  receiver: ops-mail
  group_by: [alertname, job]
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h
receivers:
  - name: ops-mail
    email_configs:
      - to: "ops@example.org"
templates:
  - '/etc/prometheus/*.tmpl'
```

The role renders this file as a Jinja template (it is picked with `with_first_found` by the `template` module), so a
value can come from an inventory variable or an Ansible Vault-encrypted variable instead of being written in clear.
The other side of it: Alertmanager's own message templates use `{{ }}` too, so put them in a `*.tmpl` file rather
than inline, or wrap them in `{% raw %}`.

### Dashboards

Export dashboards as JSON from a Grafana instance, or take them from a published collection, and place them under
`config/grafana/dashboards/`. The data source Kolla-Ansible provisions points at Prometheus on the internal VIP
(`grafana_prometheus_url`); a dashboard that expects another data source name needs editing.

## Applying it

**1. Enable** in `globals.yml`: `enable_prometheus: "yes"`, `enable_grafana: "yes"`, and the `*_external` choices.

**2. Deploy** the monitoring roles and HAProxy:

> ⚠️ **State-changing.** Creates the Prometheus, exporter and Grafana containers on every host of their groups, and
> adds HAProxy frontends for them. The new frontend configuration **restarts the `haproxy` container on every load
> balancer host**: the `loadbalancer` role's handlers stop and start it on the backup hosts first, then on the host
> holding the VIPs, with keepalived stopped and started around it. This is not a hitless reload: every API behind the
> internal and external VIPs is briefly interrupted, and OpenStack services see failed requests during that window.
> Plan it like any `loadbalancer` change, in a window. OpenStack containers themselves are not restarted. Undo: see
> below; setting the variables back to `"no"` is not enough.

```bash
kolla-ansible deploy -i <inventory> --tags common,loadbalancer,prometheus,grafana
```

**Undoing it.** Setting `enable_prometheus` and `enable_grafana` back to `"no"` removes neither the containers nor
the HAProxy frontends. In 2026.1 the `haproxy-config` role writes `/etc/kolla/haproxy/services.d/<service>.cfg` only
for enabled services and never deletes one, so `prometheus-server.cfg`, `prometheus-alertmanager.cfg` and
`grafana.cfg` stay, and HAProxy keeps their frontends with dead back-ends, on the external VIP too for Grafana and
Alertmanager (and in `external-frontend-map` with `haproxy_single_external_frontend`). Deleting a fragment by hand is
not enough either: a `reconfigure --tags loadbalancer` sees no change, because the restart decision compares only the
files that exist in the configuration directory. HAProxy drops the frontends only when its container restarts, which
copies the directory afresh.

> ⚠️ **State-changing.** Removes the monitoring containers, then restarts the `haproxy` container on every load
> balancer host to drop the frontends: every API behind the VIPs is briefly interrupted on each restart, and the VIP
> may move while the VIP holder restarts, since keepalived tracks the HAProxy process. Restart the backup hosts first,
> the VIP holder last, one at a time, in a window. Precondition: the variables are already `"no"`, so that no later
> run writes the fragments again. The Prometheus volume stays on disk until you remove it. Undo: set the variables to
> `"yes"` again and run the deploy above, which recreates the containers and fragments.

```bash
# on each host running them: the containers of the Topology table that exist there
sudo docker rm -f prometheus_server prometheus_alertmanager grafana     # or: sudo podman rm -f ...
# on each load balancer host
sudo rm -f /etc/kolla/haproxy/services.d/prometheus-server.cfg \
           /etc/kolla/haproxy/services.d/prometheus-alertmanager.cfg \
           /etc/kolla/haproxy/services.d/grafana.cfg       # -f: Alertmanager's exists only where it was enabled
# with haproxy_single_external_frontend only: delete the "# BEGIN grafana" ... "# END grafana" block, and the
# prometheus-alertmanager one, from /etc/kolla/haproxy/external-frontend-map
ip -br addr | grep -w "<internal-vip>"                        # a line only on the VIP holder; one keepalived holds both VIPs
sudo docker restart haproxy                                  # or: sudo podman restart haproxy; backups first, VIP holder last
sudo docker exec haproxy ls /etc/haproxy/services.d/         # or: sudo podman exec ...; the three fragments are gone
```

RabbitMQ metrics need one more step: the `rabbitmq` role writes the plugin into RabbitMQ's configuration, so the
next run of that role changes it.

> ⚠️ **State-changing.** Changes the RabbitMQ configuration and restarts the RabbitMQ containers whose configuration
> changed, which every OpenStack service depends on: plan it in a window, or keep the plugin off with
> `rabbitmq_enable_prometheus_plugin: false`. Undo: set that variable to `false` and run the role again.

```bash
kolla-ansible reconfigure -i <inventory> --tags rabbitmq
```

**3. Add rules, receivers and dashboards**, then apply them:

> ⚠️ **State-changing.** Copies the new files and restarts `prometheus_server`, `prometheus_alertmanager` or `grafana`
> where their configuration changed. Monitoring has a gap while each container restarts. Undo: remove the files and
> reconfigure again.

```bash
kolla-ansible reconfigure -i <inventory> --tags prometheus,grafana
```

**4. Verify** (read-only):

```bash
# from a host on the internal network; the internal VIP and the admin password from passwords.yml
curl -s -u admin:<prometheus_password> http://<internal-vip>:9091/api/v1/targets | python3 -m json.tool | grep -c '"health": "up"'
curl -s -u admin:<prometheus_password> http://<internal-vip>:9091/api/v1/rules | python3 -m json.tool | grep '"name"'
curl -s -u admin:<prometheus_alertmanager_password> http://<internal-vip>:9093/api/v2/status | python3 -m json.tool | head
```

The targets page of the Prometheus interface shows every scrape job and its last error, which is the quickest way
to find an exporter that is down or a firewall that blocks a port. To test the notification path end to end, stop one
node exporter on a lab host for longer than the rule's `for:` delay.

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| Prometheus restarts in a loop after a change | `docker logs prometheus_server` (or `podman logs`): a rule file or a `prometheus.yml.d` fragment that does not parse |
| Alerts fire in Prometheus but nobody is notified | `docker logs prometheus_alertmanager`: SMTP or webhook errors; or the default receiver is still in place |
| A target stays down | the targets page of Prometheus for the error; `docker logs <exporter container>` on the host; the host firewall for the exporter's port |
| Grafana shows no data | `/var/log/kolla/grafana/` on the Grafana hosts; the data source URL and Prometheus credentials |
| Deploy fails in the `prometheus` role | the Ansible output; then `docker logs` of the container named in the failing task |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Kolla-Ansible's Prometheus | An existing monitoring system scraping the exporters | Everything wired at deploy time; targets follow the inventory | Upgraded with OpenStack; its storage lives on the monitoring hosts |
| Rules in `config/prometheus/*.rules` | No rules, dashboards only | Without rules, nobody is told when something breaks | Rules to maintain when exporter versions change metric names |
| Alertmanager and Grafana external access off | The 2026.1 default (on) | Monitoring interfaces stay on the internal network | Reach them through the internal network or a VPN |
| `prometheus_instance_label` set to hostnames | IP addresses (default) | Readable alerts and dashboards | Duplicate series for a while after the change (Prometheus guide) |

## What went wrong during deployment

Nothing observed yet: written from the Kolla-Ansible 2026.1 guides and the `prometheus` and `grafana` roles.

## Version notes

- 2026.1: the guide notes that the instance label may become hostnames by default in a future release; set
  `prometheus_instance_label` explicitly if your rules or dashboards depend on it.
- 2026.1 has a Valkey exporter (`enable_prometheus_valkey_exporter`), following the switch from Redis to Valkey.

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- Enabling Prometheus is not monitoring: plan rules and receivers in the same change, and test one alert end to end.
- Decide the external exposure of Alertmanager and Grafana explicitly; the defaults publish both.
- Give Prometheus a retention limit that fits the controllers' disk.

## Validity and limits

Written from Kolla-Ansible 2026.1 (Prometheus and Grafana guides, `ansible/group_vars/all/{prometheus,grafana}.yml`,
the `prometheus` and `grafana` roles, the sample multinode inventory) and the Alertmanager configuration reference.
Not run on a lab: no output on this page was observed. Not covered: the Ceph manager exporter
(`enable_prometheus_ceph_mgr_exporter`), the OpenStack exporter's metric set and its cost on large clouds, remote
write to a long-term store, Grafana LDAP, and Skyline's use of Prometheus.

## Related

- [Laying out the configuration](../configuration/inventory-globals-overrides-and-secrets-layout.md): where files
  under `/etc/kolla/config/` live, and keeping secrets out of git.
- Kolla-Ansible 2026.1: [Prometheus guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/logging-and-monitoring/prometheus-guide.html),
  [Grafana guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/logging-and-monitoring/grafana-guide.html)
- Prometheus: [Alertmanager configuration](https://prometheus.io/docs/alerting/latest/configuration/)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code, and the
Prometheus documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=services/monitoring-prometheus-grafana-and-alerting.md&title=Feedback%3A+services/monitoring-prometheus-grafana-and-alerting.md):
your release, your topology, and what happened.
