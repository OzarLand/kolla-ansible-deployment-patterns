# services

How Kolla-Ansible deploys an OpenStack service or an integration (Octavia, Designate, monitoring, logging, external
Ceph and others): what to prepare, what to set, where it fails and which logs to read. One page per service, written
with `templates/deployment-pattern.md`.

- [Octavia with the amphora provider: certificates, the management network and the amphora image](octavia-amphora-certificates-and-management-network.md)
- [Monitoring with Prometheus, Grafana and Alertmanager: what is deployed, and the rules, receivers and dashboards you still have to bring](monitoring-prometheus-grafana-and-alerting.md)
- [Central logging with Fluentd and OpenSearch: where logs go, how to search them by request ID, and how long they are kept](central-logging-fluentd-and-opensearch.md)
- [Designate with the BIND9 back-end: name servers, NS records, Neutron DNS integration, and how to check a record end to end](designate-dns-with-bind9-and-neutron.md)
- [External Ceph for Glance, Cinder, Cinder backup and Nova: pools, users, the files Kolla-Ansible expects, and the libvirt secrets](external-ceph-for-glance-cinder-and-nova.md)
