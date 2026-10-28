# operations

Day-2 procedures on a running Kolla-Ansible deployment: adding and removing hosts, choosing between `deploy`, `reconfigure` and `upgrade`, backups and recoveries, checks after a change. One page per procedure, written as a pattern (`kind: pattern`) with [`templates/deployment-pattern.md`](../templates/deployment-pattern.md) (the shared [deployment pattern template](https://github.com/OzarLand/ozarland/blob/main/community/templates/deployment-pattern.md)), its procedure steps carrying the ⚠️ and 🛑 warnings of the [risk conventions](https://github.com/OzarLand/ozarland/blob/main/docs/conventions/risk-warnings.md).

The status of each page is its front matter `status`; see the [repository README](../README.md#-how-much-each-page-was-checked) for what each status means.

| Page | Status |
|---|---|
| [Adding a compute node to a running Kolla-Ansible deployment](adding-a-compute-node.md) | `draft` |
| [kolla-ansible bootstrap-servers. What it changes on the hosts, and the pitfalls of running it again on a live platform](bootstrap-servers-what-it-changes-and-its-pitfalls.md) | `draft` |
| [Checking a Kolla-Ansible deployment after a change. kolla-ansible check, cluster state, service lists and one real instance](checking-a-deployment-after-a-change.md) | `lab-verified` |
| [deploy, reconfigure, upgrade and the others. Which kolla-ansible command to run after a change, and how to narrow it with --limit and --tags](deploy-reconfigure-upgrade-and-limiting-runs.md) | `draft` |
| [Backing up and restoring the MariaDB cluster with kolla-ansible mariadb-backup](mariadb-backup-and-restore-with-mariabackup.md) | `lab-verified` |
| [Removing or replacing a compute node in a Kolla-Ansible deployment](removing-or-replacing-a-compute-node.md) | `draft` |
| [Replacing a failed controller in a three-controller Kolla-Ansible deployment](replacing-a-failed-controller.md) | `draft` |
