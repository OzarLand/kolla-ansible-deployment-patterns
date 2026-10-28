# services

How Kolla-Ansible deploys an OpenStack service or an integration (Octavia, Designate, monitoring, logging, external Ceph and others): what to prepare, what to set, where it fails and which logs to read. One page per service, written with [`templates/deployment-pattern.md`](../templates/deployment-pattern.md) (the shared [deployment pattern template](https://github.com/OzarLand/ozarland/blob/main/community/templates/deployment-pattern.md)).

The status of each page is its front matter `status`; see the [repository README](../README.md#-how-much-each-page-was-checked) for what each status means.

| Page | Status |
|---|---|
| [Central logging with Fluentd and OpenSearch in Kolla-Ansible. Where logs go, how to search them by request ID, and how long they are kept](central-logging-fluentd-and-opensearch.md) | `draft` |
| [Designate with the BIND9 back-end in Kolla-Ansible. Name servers, NS records, Neutron DNS integration, and how to check a record end to end](designate-dns-with-bind9-and-neutron.md) | `draft` |
| [External Ceph for Glance, Cinder, Cinder backup and Nova. Pools, users, the files Kolla-Ansible expects, and the libvirt secrets](external-ceph-for-glance-cinder-and-nova.md) | `draft` |
| [Monitoring with Prometheus, Grafana and Alertmanager in Kolla-Ansible. What is deployed, and the rules, receivers and dashboards you still have to bring](monitoring-prometheus-grafana-and-alerting.md) | `draft` |
| [Octavia with the amphora provider. Certificates, the management network and the amphora image](octavia-amphora-certificates-and-management-network.md) | `draft` |
