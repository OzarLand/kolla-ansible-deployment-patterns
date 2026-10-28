# configuration

Inventory layouts, variable organisation, configuration overrides, registries, TLS and secrets handling (never their content). Add a page with [`templates/deployment-pattern.md`](../templates/deployment-pattern.md) (the shared [deployment pattern template](https://github.com/OzarLand/ozarland/blob/main/community/templates/deployment-pattern.md)).

The status of each page is its front matter `status`; see the [repository README](../README.md#-how-much-each-page-was-checked) for what each status means.

| Page | Status |
|---|---|
| [CPU pinning, huge pages and NUMA on Kolla-Ansible computes. Host preparation, nova.conf overrides and the scheduler filter Kolla-Ansible does not enable](cpu-pinning-hugepages-and-numa-on-computes.md) | `draft` |
| [Laying out the configuration. Inventory, globals.yml, per-service overrides and passwords.yml, kept in git without secrets](inventory-globals-overrides-and-secrets-layout.md) | `lab-verified` |
| [A local registry and an offline deployment with Kolla-Ansible. What has to be mirrored, the variables that point at the mirrors, and the Docker setting that does not help](local-registry-and-offline-deployment.md) | `draft` |
| [Rotating the secrets in passwords.yml. Which ones a reconfigure applies, which need manual steps, and which need an outage](rotating-secrets-in-passwords-yml.md) | `draft` |
| [SR-IOV with Kolla-Ansible. VFs on the host, the Neutron SR-IOV agent, the PCI whitelist Kolla-Ansible writes, and the scheduler filter](sriov-neutron-ports-and-nova-pci-alias.md) | `draft` |
| [TLS on the internal and external endpoints. What each switch encrypts, where the certificates go, and what changes on a running platform](tls-on-internal-and-external-endpoints.md) | `draft` |
