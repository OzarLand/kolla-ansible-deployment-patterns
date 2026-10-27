# deployment-errors

Errors met during `deploy`, `reconfigure`, `pull` or `upgrade`, each with its symptom, cause and fix. See the
repository README for what belongs here. Add pages with `templates/deployment-pattern.md`, written as runbooks.

- [deploy fails part-way because Ubuntu upgraded packages during the deploy](unattended-upgrades-during-deploy-breaks-the-mariadb-cluster.md)
- [deploy hangs on MariaDB port liveness: ProxySQL holds port 3306 on a node without HAProxy](mariadb-exits-address-in-use-proxysql-on-a-node-without-haproxy.md)
- [pull fails with "http: server gave HTTP response to HTTPS client"](pull-fails-http-response-to-https-client-insecure-registry.md)
- [deploy times out at "Wait for haproxy to listen on VIP"](deploy-times-out-waiting-for-haproxy-on-the-internal-vip.md)
- [deploy fails with "The following Nova compute services failed to register themselves"](nova-compute-services-failed-to-register.md)
- [prechecks fails with "Ansible version should be between ...": the wrong Ansible or collections on the deploy host](ansible-version-not-supported-or-wrong-ansible-on-the-deploy-host.md)
- [RabbitMQ nodes do not form a cluster during deploy](rabbitmq-nodes-do-not-form-a-cluster.md)
- [deploy fails in the keystone role: Fernet bootstrap, key distribution, or key 0 missing](keystone-fernet-bootstrap-or-key-distribution-fails.md)
