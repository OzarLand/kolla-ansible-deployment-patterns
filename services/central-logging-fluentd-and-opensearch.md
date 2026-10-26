---
title: Central logging with Fluentd and OpenSearch in Kolla-Ansible. Where logs go, how to search them by request ID, and how long they are kept
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, fluentd, opensearch, haproxy]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for a multinode deployment where "ssh and grep" no longer scales
  scale: any, with OpenSearch on the control group as in the sample inventory
  network: any
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/logging-and-monitoring/central-logging-guide.html
  - https://opensearch.org/docs/latest/im-plugin/index/
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Central logging with Fluentd and OpenSearch in Kolla-Ansible. Where logs go, how to search them by request ID, and how long they are kept

## Context

Every Kolla-Ansible host already runs a `fluentd` container, and every service already writes its logs to files under
`/var/log/kolla/<service>/`. Without central logging, Fluentd does little and the logs stay on each host: finding why
an instance failed means reading `nova-api.log`, `nova-scheduler.log`, `nova-conductor.log` and `nova-compute.log` on
different hosts. `enable_central_logging: "yes"` adds OpenSearch and OpenSearch Dashboards, and Fluentd ships every
log line there, parsed into fields.

This page covers where the logs are in both cases, what central logging deploys, how to search it (above all by
request ID, which ties one API call across services), how retention works, and its security defaults.

## Topology

| Container | Inventory group (sample multinode) | Role |
|---|---|---|
| `fluentd` | every host | Reads `/var/log/kolla/*` (and the systemd journal with central logging), parses, ships |
| `opensearch` | `opensearch` (`control`) | Stores the logs, port 9200 |
| `opensearch_dashboards` | `opensearch-dashboards` (`opensearch`) | Search interface, port 5601 |

With three controllers, OpenSearch forms one cluster (`opensearch_cluster_name: kolla_logging`) across them.

## Network design

Fluentd sends to the OpenSearch hosts' API addresses directly. HAProxy publishes:

| Endpoint | Internal VIP | External VIP | Authentication |
|---|---|---|---|
| OpenSearch API, 9200 | yes | no | **none**: the 2026.1 template sets `plugins.security.disabled: "true"` |
| OpenSearch Dashboards, 5601 | yes | **yes by default** (`enable_opensearch_dashboards_external` follows the dashboards switch) | HAProxy basic auth, user `opensearch`, password `opensearch_dashboards_password` |

Anyone who reaches the internal network can read and delete every log through port 9200. Logs contain user and
project IDs, request parameters and sometimes more: treat the internal network as the boundary, and set
`enable_opensearch_dashboards_external: "no"` if the external VIP faces users.

## Storage design

OpenSearch keeps its data in the `opensearch` Docker volume (`/var/lib/docker/volumes/...`), or in a host directory
set with `opensearch_datadir_volume`. The guide warns: "OpenSearch stores a lot of logs, so if you are running
centralized logging, remember to give `/var/lib/docker` adequate space." On controllers that also hold MariaDB,
RabbitMQ and images, a dedicated disk or directory is safer than the shared Docker root.

The Java heap is 1 GB by default (`opensearch_heap_size: "1g"`), small for a busy platform; raise it with the host's
memory in mind.

### Retention

Kolla-Ansible applies an Index State Management policy to the indices matching `opensearch_log_index_prefix`
(`flog` by default, one index per day): indices are **closed after 30 days** (`opensearch_soft_retention_period_days`)
and **deleted after 60** (`opensearch_hard_retention_period_days`). A closed index uses disk but cannot be searched
until reopened.

Kolla-Ansible "will NOT update an existing retention policy": changing the two variables after the first deploy
does nothing by itself. The guide's options: change the policy in Dashboards or through the API, or delete it and let
Kolla-Ansible apply the new one.

For more than the two ages, the guide lets advanced users override `opensearch_retention_policy` with a whole Index
State Management policy; like the variables, it only takes effect if set before the policy is first created.

## Inventory and configuration layout

```text
/etc/kolla/
  globals.yml                       # enable_central_logging, opensearch_* retention and heap, *_external
  config/
    fluentd/
      input/*.conf                  # extra sources (for example syslog from network equipment)
      filter/*.conf                 # extra filters (tags, field changes)
      format/*.conf                 # extra formatting
      output/*.conf                 # extra destinations (another log system)
```

The four directories are the guide's customisation points.

### Where logs are, with or without central logging

| Where | What |
|---|---|
| `/var/log/kolla/<service>/*.log` on each host | Every service's log file; it is the `kolla_logs` Docker volume, under `/var/lib/docker/volumes/kolla_logs/_data/` on the host |
| `docker logs <container>` (or `podman logs`) | Container output: start-up errors, services that log to stdout (for example Prometheus) |
| `/var/log/kolla/fluentd/fluentd.log` | Fluentd's own log: parse errors, OpenSearch connection errors |
| `journalctl -u docker` (or the Podman units) and the system journal | Host and container engine; shipped to OpenSearch too with central logging (`enable_fluentd_systemd`) |
| OpenSearch, index `flog-<date>` | Everything above, parsed, with central logging |

### The fields to search

OpenStack service logs are parsed by the 2026.1 Fluentd input `00-global.conf.j2`. The fields most useful for a
search:

| Field | Content |
|---|---|
| `Hostname` | The host the line comes from |
| `programname` | The log file's name, for example `nova-compute`, `neutron-server` |
| `log_level` | `INFO`, `WARNING`, `ERROR` ... |
| `request_id`, `global_request_id` | The request ID without `req-`; the global one follows a call across services |
| `python_module` | The Python module that logged |
| `Payload` | The message |

## Searching it

**1. In Dashboards.** Kolla-Ansible creates the `flog-*` index pattern with `@timestamp` as time field, so
**Discover** works at first login. The guide suggests adding the columns `Hostname`, `Payload`, `log_level` and
`programname`, then filtering, for example:

```text
programname:nova-compute AND log_level:ERROR
```

**2. By request ID, the guide's way to follow one failure.** Re-run the failing command with `--debug`, take the
`X-Compute-Request-Id` (or `X-Openstack-Request-Id`) value, and search for it without the leading `req-`. The results
show the request's path from `nova-api` through `nova-scheduler` and `nova-conductor` to `nova-compute`; the `ERROR`
lines carry the cause. For an existing instance, `openstack server event list <server>` gives the request IDs of its
past actions.

```text
request_id:"<request-id-without-req>" OR global_request_id:"<request-id-without-req>"
```

**3. From the command line**, read-only, on a host of the internal network:

```bash
curl -s "http://<internal-vip>:9200/_cat/indices/flog-*?v&s=index"           # daily indices, size, health
curl -s "http://<internal-vip>:9200/flog-*/_search?size=20&sort=@timestamp:desc" \
  -H 'Content-Type: application/json' \
  -d '{"query":{"query_string":{"query":"log_level:ERROR AND programname:nova-compute"}}}' \
  | python3 -m json.tool | grep -E '"(Hostname|Payload)"'
```

**4. Without central logging**, the same search on the hosts:

```bash
sudo grep -h "<request-id-without-req>" /var/log/kolla/nova/*.log      # on each controller and compute concerned
```

## Applying it

> ⚠️ **State-changing.** Creates the OpenSearch and Dashboards containers on the `opensearch` hosts, reconfigures and
> restarts `fluentd` on every host, and adds HAProxy frontends. A new HAProxy frontend restarts the `haproxy`
> containers on every load balancer host (backups first, then the VIP holder): a brief interruption of every API
> behind the VIPs. Plan it in a window. Log files on the hosts are not affected. Undo: see *Undoing it* below;
> setting `enable_central_logging: "no"` is not enough.

```bash
kolla-ansible deploy -i <inventory> --tags common,fluentd,loadbalancer,opensearch
```

The `opensearch` role already configures its own HAProxy frontends under the `opensearch` tag; `loadbalancer` is in
the list so that the HAProxy containers themselves are deployed and checked in the same run, which is harmless if
they already run. In 2026.1 Fluentd has its own role and tag (`fluentd`); the `opensearch` role deploys both OpenSearch and its
Dashboards. Verify (read-only):

```bash
curl -s "http://<internal-vip>:9200/_cluster/health?pretty"     # status green, or yellow with one node
curl -s "http://<internal-vip>:9200/_cat/indices/flog-*?v"      # today's index exists and grows
sudo tail -20 /var/log/kolla/fluentd/fluentd.log                # on any host: no connection errors
```

### Undoing it

`enable_central_logging: "no"` does not remove what was deployed:

- `enable_opensearch` stays true when `enable_osprofiler` is set, or when CloudKitty stores its data in OpenSearch
  (`ansible/group_vars/all/opensearch.yml`): OpenSearch is then still wanted, and only Fluentd's shipping stops.
- The containers and the `opensearch` volume stay on the hosts.
- The HAProxy fragments for OpenSearch and Dashboards stay under `/etc/kolla/haproxy/services.d/` on each load balancer
  host: the `haproxy-config` role writes fragments for enabled services and never deletes one. The Dashboards
  frontend stays on the external VIP, with dead back-ends.

> ⚠️ **State-changing.** Removes the logging containers and their HAProxy frontends. The `reconfigure` restarts the
> `haproxy` containers, with the same brief interruption of every API as above. Precondition: `enable_central_logging`
> is already `"no"` and nothing else needs OpenSearch. Logs stored in OpenSearch stay in the `opensearch` volume until
> you remove it, which deletes them.

```bash
kolla-ansible reconfigure -i <inventory> --tags fluentd      # Fluentd stops shipping to OpenSearch
# on each opensearch host
sudo docker rm -f opensearch opensearch_dashboards           # or: sudo podman rm -f ...
# on each load balancer host: the fragments of OpenSearch and Dashboards
ls /etc/kolla/haproxy/services.d/
sudo rm /etc/kolla/haproxy/services.d/<opensearch-fragment>.cfg
kolla-ansible reconfigure -i <inventory> --tags loadbalancer
```

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| No logs in Dashboards for some hosts | `/var/log/kolla/fluentd/fluentd.log` on those hosts: buffer or connection errors to port 9200 |
| `opensearch` restarts in a loop | `docker logs opensearch` and `/var/log/kolla/opensearch/` on that host (`/var/log/kolla/opensearch-dashboards/` for Dashboards): heap too small, disk watermarks, cluster formation |
| Cluster health `red` or indices read-only | the disk of the OpenSearch hosts: OpenSearch stops writing when its disk watermarks are reached |
| Dashboards asks for a password and refuses it | the `opensearch_dashboards_password` in `passwords.yml`, user `opensearch` |
| Old logs missing | the retention policy: closed after 30 days, deleted after 60 by default |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Kolla-Ansible's OpenSearch | Fluentd outputs to an existing log platform (`config/fluentd/output/`) | Deployed and upgraded with the platform; request-ID searches out of the box | Disk, heap and retention to manage on the controllers |
| OpenSearch on the controllers | Dedicated logging hosts in the `opensearch` group | No extra hosts | Logging load and disk on the hosts that run the control plane |
| Dashboards on the internal VIP only | The 2026.1 default (external too) | Logs are sensitive | Reach Dashboards from the internal network or a VPN |
| Retention set before the first deploy, with the two variables or a whole `opensearch_retention_policy` | Changed later | Kolla-Ansible applies the policy only once | Later changes go through Dashboards or the API |

## What went wrong during deployment

Nothing observed yet: written from the Kolla-Ansible 2026.1 guide and the `fluentd` and `opensearch` roles.

## Version notes

- 2026.1: OpenSearch replaced Elasticsearch and Kibana in Kolla-Ansible several releases ago; the guide says the old
  `elasticsearch_curator_*_retention_period_days` variables are still read if set, and should be migrated.
- 2026.1: `opensearch_log_index_prefix` still reads `kibana_log_prefix` if a deployment kept it.

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- Decide retention, heap and disk before enabling it; retention cannot be changed by Kolla-Ansible afterwards.
- Search by request ID first: it turns a multi-host investigation into one query.
- Port 9200 has no authentication: keep it on a trusted network.

## Validity and limits

Written from Kolla-Ansible 2026.1 (central logging guide, `ansible/group_vars/all/{opensearch,fluentd,common}.yml`,
the `opensearch` and `fluentd` roles and templates, the sample multinode inventory). Not run on a lab: no output on
this page was observed. Not covered: an external OpenSearch or Elasticsearch (`opensearch_address`), OSProfiler and
CloudKitty, which also use OpenSearch, TLS between Fluentd and OpenSearch, and OpenSearch sizing beyond the heap.

## Related

- [Monitoring with Prometheus, Grafana and Alertmanager](monitoring-prometheus-grafana-and-alerting.md): the metrics
  side, and the same HAProxy behaviour on deploy and undo.
- [Laying out the configuration](../configuration/inventory-globals-overrides-and-secrets-layout.md)
- Kolla-Ansible 2026.1: [central logging guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/logging-and-monitoring/central-logging-guide.html)
- OpenSearch: [Index Management plugin](https://opensearch.org/docs/latest/im-plugin/index/)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=services/central-logging-fluentd-and-opensearch.md&title=Feedback%3A+services/central-logging-fluentd-and-opensearch.md):
your release, your topology, and what happened.
