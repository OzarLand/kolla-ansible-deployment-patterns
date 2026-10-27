# operations

Day-2 procedures on a running Kolla-Ansible deployment: adding and removing hosts, choosing between `deploy`,
`reconfigure` and `upgrade`, backups and recoveries, certificate renewal, checks after a change. One page per
procedure, written with `templates/deployment-pattern.md`.

- [Adding a compute node to a running deployment](adding-a-compute-node.md)
- [deploy, reconfigure, upgrade and the others: which command to run, and --limit and --tags](deploy-reconfigure-upgrade-and-limiting-runs.md)
- [Removing or replacing a compute node](removing-or-replacing-a-compute-node.md)
- [Backing up and restoring the MariaDB cluster with kolla-ansible mariadb-backup](mariadb-backup-and-restore-with-mariabackup.md)
- [Checking a deployment after a change](checking-a-deployment-after-a-change.md)
- [Replacing a failed controller](replacing-a-failed-controller.md)
