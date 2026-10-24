---
title: Cinder and Glance back-ends compared. LVM, Ceph RBD and NFS, and what each choice implies
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, cinder, glance, nova, ceph, lvm, nfs]
deployment_methods: [kolla-ansible]
environment:
  purpose: any multinode
  scale: any
  network: any
  storage: lvm, ceph-rbd (external), nfs
  os: any supported by Kolla-Ansible 2026.1
risk: read-only
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/storage/cinder-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/storage/external-ceph-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/glance-guide.html
  - https://docs.openstack.org/cinder/2026.1/reference/support-matrix.html
  - https://docs.ceph.com/en/latest/rbd/rbd-openstack/
authored_by: agent:ozarland-author
last_verified: 2026-10-22
---

# Cinder and Glance back-ends compared. LVM, Ceph RBD and NFS, and what each choice implies

## Context

Kolla-Ansible 2026.1 deploys Cinder with the `lvm`, `nfs`, `ceph` and `iscsi` back-ends, and Glance with `file`,
`ceph` and `s3` (Cinder and Glance guides). The choice is usually made once, early, and is expensive to change: it
decides where volume and image data live, what survives the loss of a host, how fast an instance boots from an
image, and what has to be operated besides OpenStack.

This page compares the three most common choices, LVM, Ceph RBD and NFS, as Kolla-Ansible 2026.1 deploys them. It is
written from the 2026.1 documentation, `globals.yml`, the inventory and the group variables, not from one platform.
It describes consequences, not a recommendation: each choice is right somewhere.

## Topology

Where the services run in the 2026.1 multinode inventory (changed in this release, see [Version notes](#version-notes)):

| Group | Default members | Runs |
|---|---|---|
| `cinder` | `control` | `cinder-api`, `cinder-scheduler`, and now `cinder-volume` and `cinder-backup` |
| `cinder-volume-lvm`, `cinder-backup-lvm` | `storage` | The LVM back-end, which needs the volume group on the host |
| `cinder-volume-multiple`, `cinder-backup-multiple` | `cinder` and `storage` | Mixed layouts |
| `glance-api` | `control` | Glance; only one container with a non-shared `file` back-end |

## Network design

Volume data travels between the storage back-end and the computes: iSCSI from the `tgtd` target with LVM, the Ceph
public network with RBD, NFS mounts with NFS. A dedicated storage network is set in the back-end's own
configuration; this page does not cover it.

## Storage design

### Side by side

| | LVM (`enable_cinder_backend_lvm`) | Ceph RBD (external Ceph) | NFS (`enable_cinder_backend_nfs`) |
|---|---|---|---|
| What Kolla-Ansible deploys | `cinder-volume` with the LVM driver; `tgtd` (iSCSI target) on the storage hosts; `iscsid` with `nova-compute` and with `cinder-volume`/`cinder-backup` | Only the clients: Kolla-Ansible "does not provide support for provisioning and configuring a Ceph cluster" (external Ceph guide) | `cinder-volume` with the NFS driver, mounting the shares listed in `/etc/kolla/config/nfs_shares` |
| What you operate besides | A volume group named `cinder-volumes` on each storage host (`pvcreate`, `vgcreate`) | A Ceph cluster, deployed with a dedicated tool (`cephadm`, ceph-ansible), its pools and keyrings | NFS servers and their exports |
| Where a volume lives | On the disks of one storage host | Replicated across the Ceph cluster | On one NFS server's export |
| Loss of the host holding the volume | Its volumes are unavailable until the host returns | Ceph keeps serving from the replicas, within its own redundancy | Its volumes are unavailable until the server returns |
| Several `cinder-volume` hosts | Known issues with LVM in a multi-controller setup (bug 1571211, cited in the Cinder guide) | Active/active with `cinder_cluster_name`, when the driver supports it (Cinder support matrix) | Depends on the driver's active/active support; check the support matrix |
| Glance on the same back-end | No: Glance has no LVM back-end; use `file` or `s3` | Yes, `glance_backend_ceph`; copy-on-write clones for boot-from-image with `show_image_direct_url` | Indirectly: Glance `file` with `glance_file_datadir_volume` on a shared filesystem such as NFS |
| Nova instance disks | Local to each compute | Can also live in Ceph (`nova_backend_ceph`): "instances' ephemeral disks do not need to be copied between hypervisors" on migration (external Ceph guide) | Local to each compute, unless configured separately |
| `cinder-backup` | The default driver is `ceph`: set `cinder_backup_driver` to `nfs` or `s3`, or disable backup | Default `cinder_backup_driver: "ceph"` | `cinder_backup_driver: "nfs"` with `cinder_backup_share` |

### Glance back-ends

From the Glance guide:

- **`file`** (the default when neither Ceph nor S3 is enabled): images are stored in `glance_file_datadir_volume`, a
  Docker volume named `glance` by default, and "only one `glance-api` container can be running". For reliability,
  the guide recommends mounting it on a shared filesystem such as NFS. Glance's rolling upgrade cannot be used with a
  non-shared `file` back-end.
- **`ceph`**: enabled automatically with external Ceph. Copy-on-write cloning needs `show_image_direct_url = True`
  in `/etc/kolla/config/glance.conf`; the guide warns it "can present a security risk if using more than just Ceph as
  Glance backend(s)". Ceph's OpenStack guide recommends raw images for copy-on-write.
- **`s3`**: `glance_backend_s3` with the `glance_backend_s3_*` variables.

### What to set, per choice

LVM, with the volume group created on each storage host beforehand:

```yaml
# globals.yml
enable_cinder_backend_lvm: true
cinder_backup_driver: "nfs"          # or "s3", or enable_cinder_backup: false, when there is no Ceph
```

Ceph RBD, with pools and keyrings created in Ceph, and `ceph.conf` and the keyrings copied under
`/etc/kolla/config/{glance,cinder,nova}/` as the external Ceph guide lists:

```yaml
# globals.yml
cinder_backend_ceph: "yes"
glance_backend_ceph: "yes"
nova_backend_ceph: "yes"             # optional: instance disks in Ceph too
```

NFS, with `/etc/kolla/config/nfs_shares` listing one `<server>:<export>` per line:

```yaml
# globals.yml
enable_cinder_backend_nfs: true
```

A read-only check of what Cinder actually runs:

```bash
openstack volume service list                 # one cinder-volume per back-end host@backend, up
openstack volume backend pool list --long     # capacity per pool
```

## Inventory and configuration layout

```text
inventory/
  hosts                         # [storage] hosts for LVM; [control] runs cinder-volume for Ceph and NFS
globals.yml                     # enable_cinder_backend_*, *_backend_ceph, cinder_backup_driver
config/
  nfs_shares                    # NFS back-end
  glance.conf                   # show_image_direct_url, with Ceph
  glance/ cinder/ nova/         # ceph.conf and keyrings, with Ceph (secrets: see the configuration pattern)
```

Ceph keyrings are secrets; keep them out of git as the
[configuration layout pattern](../configuration/inventory-globals-overrides-and-secrets-layout.md) describes.

## Key choices and their consequences

| Choice | Alternative(s) | Why one would choose it | Consequence in operation |
|---|---|---|---|
| LVM | Ceph, NFS, a vendor array | Nothing else to operate; simple for a lab or a small platform | A volume is as available as its storage host; volumes do not move between hosts by themselves; Glance needs another back-end |
| Ceph RBD for Cinder, Glance and Nova | LVM or NFS for Cinder, `file` for Glance | One replicated store; copy-on-write boots; migrations without copying disks | A Ceph cluster to deploy, size, monitor and upgrade on its own schedule, outside Kolla-Ansible |
| NFS | LVM, Ceph | Reuses an existing NAS; shared storage for Glance `file` too | The NFS server's availability is the volumes' availability; performance depends on the NAS and the network |
| Glance `file` on local disk | Shared `file`, Ceph, S3 | Simplest | One `glance-api` container only; no rolling upgrade of Glance |

## What went wrong during deployment

Nothing recorded yet: this page is written from the upstream documentation and source.

## Version notes

- **2026.1 moved `cinder-volume` and `cinder-backup` to the `cinder` group** (the controllers by default; earlier
  they defaulted to `storage`) and added the `cinder-volume-lvm`, `cinder-volume-multiple`, `cinder-backup-lvm` and
  `cinder-backup-multiple` groups (release note `move-cinder-volume`). An existing inventory needs updating before
  the upgrade. The 2026.1 Cinder guide still tells LVM users to add `storage` as a child of `cinder-volume`; the new
  `cinder-volume-lvm` group in the sample inventory already contains `storage`. Check your inventory against the
  sample.
- 2026.1: `glance-api` now runs under uWSGI and the `glance-tls-proxy` container is removed during the upgrade
  (release note `uwsgi-glance`).

## Lessons learned

No first-hand lessons yet. Contributions with a real platform's numbers (volume counts, failure of a storage host,
a migration from one back-end to another) are welcome.

## Validity and limits

Written from Kolla-Ansible 2026.1 (Cinder guide, external Ceph guide, Glance guide, `etc/kolla/globals.yml`,
`ansible/group_vars/all/{cinder,glance}.yml`, the multinode inventory, release notes) and the Ceph OpenStack guide.
Not applied on a platform.

Not covered: vendor arrays and their drivers, Cinder `iscsi` with external targets, Manila, Swift, Glance S3 in
detail, multi-attach, encryption, and migrating volumes between back-ends.

## Related

- [Laying out the configuration](../configuration/inventory-globals-overrides-and-secrets-layout.md)
- [Volume runbooks](https://github.com/OzarLand/openstack-vm-runbooks/tree/main/volumes)
- Kolla-Ansible 2026.1: [Cinder](https://docs.openstack.org/kolla-ansible/2026.1/reference/storage/cinder-guide.html),
  [external Ceph](https://docs.openstack.org/kolla-ansible/2026.1/reference/storage/external-ceph-guide.html),
  [Glance](https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/glance-guide.html)
- Ceph, [Block devices and OpenStack](https://docs.ceph.com/en/latest/rbd/rbd-openstack/)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and source, and the Ceph documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*
