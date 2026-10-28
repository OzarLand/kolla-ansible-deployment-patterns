# deployment-errors

Errors met during `deploy`, `reconfigure`, `pull` or `upgrade`, each with its symptom, cause and fix. One page per error, written as a runbook (`kind: runbook`) with the shared [runbook template](https://github.com/OzarLand/ozarland/blob/main/community/templates/runbook.md).

The status of each page is its front matter `status`; see the [repository README](../README.md#-how-much-each-page-was-checked) for what each status means.

| Page | Status |
|---|---|
| [kolla-ansible prechecks fails with "Ansible version should be between ...", or plays fail on missing modules. The deploy host runs the wrong Ansible or the wrong collections](ansible-version-not-supported-or-wrong-ansible-on-the-deploy-host.md) | `draft` |
| [kolla-ansible deploy times out at "Wait for haproxy to listen on VIP". The internal VIP never comes up, or cannot be reached](deploy-times-out-waiting-for-haproxy-on-the-internal-vip.md) | `draft` |
| [kolla-ansible deploy fails in the keystone role. Fernet bootstrap refused under --limit, key distribution over SSH failing, or keystone waiting for key 0](keystone-fernet-bootstrap-or-key-distribution-fails.md) | `lab-verified` |
| [kolla-ansible fails with "MariaDB cluster exists but is stopped". Recovering a fully stopped Galera cluster with kolla-ansible mariadb-recovery](mariadb-cluster-exists-but-is-stopped-run-mariadb-recovery.md) | `lab-verified` |
| [kolla-ansible deploy hangs on MariaDB port liveness. MariaDB exits with "Address already in use" because ProxySQL holds port 3306 on a node without HAProxy](mariadb-exits-address-in-use-proxysql-on-a-node-without-haproxy.md) | `lab-verified` |
| [kolla-ansible deploy fails with "The following Nova compute services failed to register themselves](nova-compute-services-failed-to-register.md) | `draft` |
| [kolla-ansible pull fails with "http: server gave HTTP response to HTTPS client". Docker does not know the registry is plain HTTP](pull-fails-http-response-to-https-client-insecure-registry.md) | `lab-verified` |
| [RabbitMQ nodes do not form a cluster during kolla-ansible deploy. Host name resolution, the Erlang cookie, closed ports or old cluster state](rabbitmq-nodes-do-not-form-a-cluster.md) | `draft` |
| [kolla-ansible deploy fails part-way, a container never starts and MariaDB never forms its cluster, because Ubuntu upgraded packages during the deploy](unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md) | `lab-verified` |
