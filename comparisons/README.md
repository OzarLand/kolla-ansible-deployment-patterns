# comparisons

Side-by-side comparisons of two approaches on one dimension (container engine, ML2 driver, controller count, where the network services run). There is no comparison template yet: copy the structure of an existing page (`kind: comparison`, a *Validity and limits* section, the shared front matter of the [content metadata conventions](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/content-metadata.md)).

The status of each page is its front matter `status`; see the [repository README](../README.md#-how-much-each-page-was-checked) for what each status means.

| Page | Status |
|---|---|
| [Docker versus Podman as the container engine of a Kolla-Ansible deployment. What changes for operators, and what migrating costs](docker-versus-podman-with-kolla-ansible.md) | `draft` |
| [ML2/OVS versus ML2/OVN with Kolla-Ansible. What changes in globals.yml, in operations and at upgrade time](ml2-ovs-versus-ml2-ovn-with-kolla-ansible.md) | `draft` |
| [Separate network nodes versus network services on the controllers. What the network group decides in Kolla-Ansible](separate-network-nodes-versus-network-services-on-controllers.md) | `draft` |
| [Single control node versus three controllers. What high availability really costs in Kolla-Ansible](single-control-node-versus-three-controllers.md) | `draft` |
