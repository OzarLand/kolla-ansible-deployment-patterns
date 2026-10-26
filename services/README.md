# services

How Kolla-Ansible deploys an OpenStack service or an integration (Octavia, Designate, monitoring, logging, external
Ceph and others): what to prepare, what to set, where it fails and which logs to read. One page per service, written
with `templates/deployment-pattern.md`.

- [Octavia with the amphora provider: certificates, the management network and the amphora image](octavia-amphora-certificates-and-management-network.md)
- [Central logging with Fluentd and OpenSearch: where logs go, how to search them by request ID, and how long they are kept](central-logging-fluentd-and-opensearch.md)
