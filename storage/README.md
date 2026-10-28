# storage

Storage designs and their trade-offs for Glance, Cinder and Nova: what is shared, what is local, what each choice implies. Add a page with [`templates/deployment-pattern.md`](../templates/deployment-pattern.md) (the shared [deployment pattern template](https://github.com/OzarLand/ozarland/blob/main/community/templates/deployment-pattern.md)).

The status of each page is its front matter `status`; see the [repository README](../README.md#-how-much-each-page-was-checked) for what each status means.

| Page | Status |
|---|---|
| [Cinder and Glance back-ends compared. LVM, Ceph RBD and NFS, and what each choice implies](cinder-and-glance-backends-compared-lvm-ceph-nfs.md) | `draft` |
| [Where instance disks live. Local on each compute, in Ceph RBD, or boot from volume, and what each choice means for migration, evacuation and capacity](nova-instance-disks-local-ceph-rbd-or-boot-from-volume.md) | `draft` |
