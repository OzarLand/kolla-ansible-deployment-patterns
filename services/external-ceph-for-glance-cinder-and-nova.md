---
title: External Ceph for Glance, Cinder, Cinder backup and Nova. Pools, users, the files Kolla-Ansible expects, and the libvirt secrets
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, ceph, glance, cinder, nova, libvirt]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for a multinode deployment using a Ceph cluster deployed by another tool
  scale: any
  network: any, with a storage network that reaches the Ceph monitors and OSDs
  storage: ceph-rbd (external)
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/storage/external-ceph-guide.html
  - https://docs.ceph.com/en/latest/rbd/rbd-openstack/
  - https://docs.ceph.com/en/latest/rados/operations/user-management/
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# External Ceph for Glance, Cinder, Cinder backup and Nova. Pools, users, the files Kolla-Ansible expects, and the libvirt secrets

## Context

Kolla-Ansible "does not provide support for provisioning and configuring a Ceph cluster directly" (external Ceph
guide): the cluster is deployed with `cephadm` or another tool, and Kolla-Ansible only configures the OpenStack
clients. Most integration failures come from the hand-off between the two: a pool or a user missing, a capability too
narrow, a keyring under the wrong name or in the wrong directory, a `ceph.conf` the INI parser cannot read.

Whether to use Ceph, and for which service, is compared elsewhere:
[Cinder and Glance back-ends compared](../storage/cinder-and-glance-backends-compared-lvm-ceph-nfs.md) and
[where instance disks live](../storage/nova-instance-disks-local-ceph-rbd-or-boot-from-volume.md). This page is the
integration itself, for one Ceph cluster: what to create on the Ceph side, which files go where, what Kolla-Ansible
does with them, and how to check each service.

## Topology

No Kolla-Ansible role on the Ceph hosts. The clients run in the containers that use them: `glance_api`,
`cinder_volume`, `cinder_backup`, `nova_compute` and `nova_libvirt`. Cinder needs hosts in the `storage` group even
without local storage: the guide says that with external Ceph "there may be no nodes defined in the storage group",
which "will cause Cinder and related services relying on this group to fail"; put there the hosts where
`cinder-volume` and `cinder-backup` should run, usually the controllers.

## Network design

Every host running one of those containers must reach the Ceph monitors and every OSD, usually on a storage network
or VLAN separate from the API network. Computes carry the instances' disk traffic. The MTU must match end to end. See
[network interfaces, VLANs and MTU](../networking/interfaces-vlans-and-mtu-ovs-and-ovn.md).

## Storage design

### On the Ceph side

From Ceph's *Block devices and OpenStack* guide, run on a Ceph admin host (the pool names are Kolla-Ansible's
defaults; size the placement groups or let the autoscaler do it):

> ⚠️ **State-changing.** Creates pools and client users in the Ceph cluster. Nothing in OpenStack changes yet. Undo:
> `ceph auth del client.<name>`; deleting a pool is destructive and needs the cluster's pool deletion setting, so
> never delete one that has held data without a backup.

```bash
for pool in images volumes backups vms; do
  ceph osd pool create "$pool"
  rbd pool init "$pool"
done

ceph auth get-or-create client.glance mon 'profile rbd' \
  osd 'profile rbd pool=images' mgr 'profile rbd pool=images'
ceph auth get-or-create client.cinder mon 'profile rbd' \
  osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' \
  mgr 'profile rbd pool=volumes, profile rbd pool=vms'
ceph auth get-or-create client.cinder-backup mon 'profile rbd' \
  osd 'profile rbd pool=backups' mgr 'profile rbd pool=backups'
```

`client.cinder` covers `vms` too because Kolla-Ansible's `ceph_nova_user` defaults to `ceph_cinder_user`: Nova uses the
Cinder user unless you create a separate `client.nova` and set `ceph_nova_user: "nova"`. Its read-only access to
`images` is what lets Nova and Cinder clone images copy-on-write.

Export each keyring and a minimal configuration (read-only):

```bash
ceph auth get client.glance -o ceph.client.glance.keyring
ceph auth get client.cinder -o ceph.client.cinder.keyring
ceph auth get client.cinder-backup -o ceph.client.cinder-backup.keyring
ceph config generate-minimal-conf > ceph.conf
```

The guide warns that `generate-minimal-conf` writes **leading tabs**, which "break Kolla Ansible's ini parser":
remove them before copying the file.

## Inventory and configuration layout

The file names are not free: Kolla-Ansible builds them from the cluster name (`ceph_cluster`, `ceph` by default) and
the user, as `<cluster>.client.<user>.keyring`, and fails or silently misses a file that does not match.

```text
/etc/kolla/
  globals.yml
  passwords.yml                                   # rbd_secret_uuid, cinder_rbd_secret_uuid (generated by kolla-genpwd)
  config/
    glance/
      ceph.conf
      ceph.client.glance.keyring
    glance.conf                                   # optional: show_image_direct_url for copy-on-write
    cinder/
      ceph.conf                                   # merged with the per-service files below
      cinder-volume/ceph.client.cinder.keyring
      cinder-backup/ceph.client.cinder.keyring    # backup reads the volumes pool too
      cinder-backup/ceph.client.cinder-backup.keyring
    nova/
      ceph.conf
      ceph.client.cinder.keyring                  # attach Cinder volumes; also the Nova user by default
      # ceph.client.nova.keyring                  # only with a separate ceph_nova_user
```

```yaml
# globals.yml
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"            # instance disks in the vms pool; leave "no" to keep them local
# enable_cinder_backup is on with Cinder; its driver defaults to ceph (cinder_backup_driver)
# defaults, shown for reference:
# ceph_glance_user: "glance"          ceph_glance_pool_name: "images"
# ceph_cinder_user: "cinder"          ceph_cinder_pool_name: "volumes"
# ceph_cinder_backup_user: "cinder-backup"   ceph_cinder_backup_pool_name: "backups"
# ceph_nova_user: "{{ ceph_cinder_user }}"   ceph_nova_pool_name: "vms"
```

The keyrings are secrets: keep them out of git, like `passwords.yml`
([laying out the configuration](../configuration/inventory-globals-overrides-and-secrets-layout.md)). Kolla-Ansible
copies them with Ansible's `template` module, so they are rendered as Jinja: an ordinary keyring is unaffected.

### What Kolla-Ansible does with them (2026.1)

- **Glance:** `ceph.conf` and the keyring go into `glance_api`; with `glance_backend_ceph`, the first entry of
  `glance_ceph_backends` becomes the default store.
- **Cinder:** one back-end section per entry of `cinder_ceph_backends`, named `rbd-1` by default
  (`cinder_backend_ceph_name`), with `rbd_user`, `rbd_pool` and `rbd_secret_uuid = <cinder_rbd_secret_uuid>`. Its
  `ceph.conf` is merged from `config/cinder/<cluster>.conf` and `config/cinder/<service>/<cluster>.conf`.
- **Nova:** the configuration tasks of `nova-cell/tasks/external_ceph.yml` fail when the keyring for the Nova user
  (with `nova_backend_ceph`) or the Cinder user (with `cinder_backend_ceph`) is not found under `config/nova/` or
  `config/nova/<inventory_hostname>/`. The key is extracted from it and written as a **libvirt secret**: UUID
  `rbd_secret_uuid` for Nova's ephemeral disks, `cinder_rbd_secret_uuid` for attached volumes. libvirt uses them to
  open the RBD images; without the right key, attaches and boots fail on the compute even when Cinder is healthy.

## Applying it

**1. Read-only checks** from a host that will be a Ceph client, if it has the Ceph CLI (otherwise from the Ceph admin
host): the user sees its pool.

```bash
rbd --id cinder --keyring ./ceph.client.cinder.keyring -c ./ceph.conf ls volumes   # empty list, no error
kolla-ansible prechecks -i <inventory>
```

**2. Deploy.** On a new deployment, the ordinary `kolla-ansible deploy`. On a running one:

> ⚠️ **State-changing.** Reconfigures and restarts `glance_api`, the Cinder services and `nova_compute`/`nova_libvirt`
> on every compute; writes the libvirt secrets. Instances keep running; API calls to Glance, Cinder and Nova may fail
> while their containers restart. Switching an existing deployment's back-end does not move existing images,
> volumes or instance disks: they stay where they were, and new ones go to Ceph. Undo: set the switches back and
> reconfigure; data already written to Ceph stays there.

```bash
kolla-ansible reconfigure -i <inventory> --tags glance,cinder,nova
```

**3. Verify each service** (read-only, then one test of each):

```bash
openstack volume service list                    # cinder-volume for rbd-1 and cinder-backup up
sudo docker exec nova_libvirt virsh secret-list  # on a compute: the ceph-persistent-cinder secret (and ceph-ephemeral-nova)
```

> ⚠️ **State-changing.** Creates and deletes test resources in a test project.

```bash
openstack image create --disk-format raw --container-format bare --file <image.raw> test-ceph-image
openstack volume create --size 1 --image test-ceph-image test-ceph-vol      # cloned from images into volumes
openstack volume backup create --name test-ceph-backup test-ceph-vol      # written to backups
openstack server create --flavor <flavor> --image test-ceph-image --network <network> test-ceph-vm
openstack server add volume test-ceph-vm test-ceph-vol
# then delete the server, the backup, the volume and the image
```

On the Ceph side, `rbd ls images`, `rbd ls volumes`, `rbd ls backups` and `rbd ls vms` (with `nova_backend_ceph`)
each show the new objects. Raw images let Ceph clone them; with qcow2 images, Glance stores them as they are and
every volume or boot converts them (Ceph's OpenStack guide).

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| Deploy fails on a missing keyring in the Nova role | the file name under `/etc/kolla/config/nova/`: `<cluster>.client.<user>.keyring`, with the Nova user (`ceph_nova_user`) and the Cinder user |
| `cinder-volume` for `rbd-1` down | `/var/log/kolla/cinder/cinder-volume.log`: monitor unreachable, `permission denied` (capabilities), wrong keyring name, or a `ceph.conf` the parser rejected |
| Backups fail | `/var/log/kolla/cinder/cinder-backup.log`: the backup service needs both the Cinder and the backup keyrings |
| Image upload fails | `/var/log/kolla/glance/glance-api.log` |
| Volume attach or Ceph-backed boot fails on a compute | `/var/log/kolla/nova/nova-compute.log` and `/var/log/kolla/libvirt/libvirtd.log`: a missing or wrong libvirt secret, or no route to the OSDs from the compute |
| A container does not start after the change | `docker logs <container>` (or `podman logs`) |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| Nova shares the Cinder Ceph user (default) | A separate `client.nova` and `ceph_nova_user` | One key less to manage | The Cinder key can read and write instance disks too |
| `nova_backend_ceph: "yes"` | Local instance disks | Live migration without copying disks; one store | All instance I/O goes over the storage network to Ceph |
| Raw images in Glance | qcow2 | Copy-on-write clones for volumes and instances | Larger uploads |
| `show_image_direct_url = True` for Glance | Off | Needed for copy-on-write | The guide warns of a security risk when Glance has back-ends other than Ceph |
| `cinder-volume` on the controllers in `storage` | Dedicated hosts | No extra hosts for a client-only service | The `storage` group must not be empty, or Cinder fails to deploy |

## What went wrong during deployment

Nothing observed yet: written from the Kolla-Ansible external Ceph guide, the 2026.1 roles and Ceph's documentation.

## Version notes

- 2026.1: per-host Ceph clusters for computes (`ceph_cluster` per host, files under `config/nova/<host>/`) and several
  Ceph back-ends for Glance and Cinder (`glance_ceph_backends`, `cinder_ceph_backends`) are supported; see the guide.
- RadosGW integration (`enable_ceph_rgw`) is a separate topic, not covered here.

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- Create the pools and users with the capabilities from Ceph's guide, and name the keyrings exactly as Kolla-Ansible
  expects them.
- Strip the tabs from a generated `ceph.conf`.
- Test each path separately (image, volume, backup, attach, boot): each uses a different user, file or secret.

## Validity and limits

Written from Kolla-Ansible 2026.1 (external Ceph guide, `ansible/group_vars/all/{ceph,cinder,glance}.yml`, the
`cinder`, `glance` and `nova-cell` roles) and Ceph's *Block devices and OpenStack* and user management guides. Not run
on a lab: no output on this page was observed. Not covered: several Ceph clusters, Manila with CephFS, Gnocchi,
RadosGW, Ceph without cephx, migrating existing data into Ceph, and Ceph's own sizing and operations.

## Related

- [Cinder and Glance back-ends compared](../storage/cinder-and-glance-backends-compared-lvm-ceph-nfs.md)
- [Where instance disks live](../storage/nova-instance-disks-local-ceph-rbd-or-boot-from-volume.md)
- Kolla-Ansible 2026.1: [external Ceph guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/storage/external-ceph-guide.html)
- Ceph: [Block devices and OpenStack](https://docs.ceph.com/en/latest/rbd/rbd-openstack/),
  [user management](https://docs.ceph.com/en/latest/rados/operations/user-management/)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code, and the Ceph
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=services/external-ceph-for-glance-cinder-and-nova.md&title=Feedback%3A+services/external-ceph-for-glance-cinder-and-nova.md):
your release, your topology, and what happened.
