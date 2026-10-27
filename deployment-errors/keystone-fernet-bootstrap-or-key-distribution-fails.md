---
title: kolla-ansible deploy fails in the keystone role. Fernet bootstrap refused under --limit, key distribution over SSH failing, or keystone waiting for key 0
kind: runbook
status: lab-verified
openstack_releases: ["2026.1"]
components: [kolla-ansible, keystone, docker]
deployment_methods: [kolla-ansible]
environment:
  topology: any; distribution only matters with two or more hosts in the keystone group
  network: any
  storage: any
  os: any host OS supported by the Kolla-Ansible release
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, multinode (3 controllers, 2 computes)", date: 2026-10-27, note: "cause 1 (every keystone_fernet stopped, deploy with --limit) and its start-then-deploy resolution; cause 2 simulated with a DROP of port 8023 on one controller, resolved by removing it during the wait; cause 3 (keys removed on one host) with steps 1-4; quick checks and Verification run; start/stop with systemctl, not docker. A first deployment with --limit, kolla_serial, a bootstrap on an empty cluster, a push from an empty host, a different keystone_ssh_key and keystone_ssh stopped not run" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/keystone-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/admin/password-rotation.html
  - https://docs.openstack.org/keystone/2026.1/admin/fernet-token-faq.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# kolla-ansible deploy fails in the keystone role: Fernet bootstrap, key distribution, or key 0 missing

## Symptom

One of these, during `kolla-ansible deploy`, `reconfigure` or `upgrade`, in the `keystone` role:

```text
TASK [keystone : Fail if any hosts need bootstrapping and not all hosts targeted] ****
[ERROR]: Task failed: Action failed: Some hosts (controller-1) need Fernet key bootstrapping, but not all Keystone hosts are in the target list. ...
fatal: [controller-1]: FAILED! => {"changed": false, "msg": "Some hosts (controller-1) need Fernet key bootstrapping, but not all Keystone hosts are in the target list. Stopping as it may be unsafe to proceed. Please run without --limit or --serial to bootstrap these hosts.\n"}
```

```text
TASK [keystone : Waiting for Keystone SSH port to be UP] ****
ok: [controller-2]
ok: [controller-1]
FAILED - RETRYING: [controller-3]: Waiting for Keystone SSH port to be UP (10 retries left).
TASK [keystone : Run fernet key distribution] ****
fatal: [controller-1]: FAILED! => ... "cmd": "docker exec -t keystone_fernet /usr/bin/fernet-push.sh" ...
```

or the `keystone` container restarts, and its log shows:

```text
ERROR: Fernet keys have not been populated, rechecking in 5 seconds
...
CRITICAL: Waited for 3 minutes - failing
```

The task names and messages are those of `ansible/roles/keystone/tasks/bootstrap_service.yml`,
`distribute_fernet.yml` and `templates/keystone-startup.sh.j2` in Kolla-Ansible 2026.1. The surrounding Ansible
output is reconstructed, not copied from a run, except the bootstrap message, the `RETRYING` line and the
`keystone` log lines, which were captured on a lab (see *Lab observations*). The bootstrap message names only the
hosts in the run.

## Impact

Keystone issues and validates every token. Without Fernet keys on a Keystone host, that host's API fails; with keys
that differ between hosts, a token issued by one host is rejected by another, which shows as random `401` errors
behind the load balancer. Running instances are not affected.

## Quick checks

On each Keystone host, read-only:

```bash
# 1. Containers: keystone, keystone_fernet, keystone_ssh
sudo docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -E '^keystone'

# 2. The keys: key 0 must exist, owned by keystone:keystone; compare file names and hashes between hosts
sudo docker exec keystone_fernet ls -l /etc/keystone/fernet-keys/
sudo docker exec keystone_fernet sh -c 'sha256sum /etc/keystone/fernet-keys/*'

# 3. The container's own check: key 0 present and a dry-run rsync to the other hosts
sudo docker exec keystone_fernet /usr/bin/fernet-healthcheck.sh; echo "exit=$?"
```

Between Keystone hosts, read-only:

```bash
# 4. The keystone_ssh port (keystone_ssh_port, 8023 by default) on the api_interface addresses
nc -zv -w 3 <other-controller-api-address> 8023
```

Logs: `docker logs keystone`, `docker logs keystone_fernet`, `docker logs keystone_ssh`, and
`/var/log/kolla/keystone/`.

## Diagnosis

How Kolla-Ansible 2026.1 handles Fernet keys, from the `keystone` role:

- **Bootstrap** runs once, on the first Keystone host, only if no host runs a `keystone_fernet` container: it starts a
  one-off container that runs `keystone-manage fernet_setup`. If some hosts need bootstrapping and the run does not
  include every Keystone host (`--limit`, or a batch with `kolla_serial`), the role stops on purpose, because, in its
  own words, bootstrapping keys on an existing cluster "would overwrite existing keys, and invalidate tokens created
  from them".
- **Distribution:** the role waits for the `keystone_ssh` port on each host (10 tries, 5 seconds apart; each host
  checks its own `api_interface_address`, and each try can last up to `wait_for`'s default timeout of 300 s), then runs
  `fernet-push.sh` once, in `keystone_fernet`, on **the first Keystone host in inventory order whose
  `keystone_fernet` container was running** when the role checked (`keystone_fernet_running_True`), or the first
  Keystone host if none was. The script does an `rsync --delete` of `/etc/keystone/fernet-keys/` over SSH to every
  other Keystone host, as user `keystone`, on `keystone_ssh_port`, with the key pair `keystone_ssh_key` from
  `passwords.yml`. The source is chosen by its running container, not by whether it holds keys: **if that host's key
  directory is empty, the push empties every other host's**. The keys live in the `keystone_fernet_tokens` volume,
  shared by the `keystone` and `keystone_fernet` containers.
- **Start-up:** the `keystone` container waits up to 3 minutes for key `0` before starting its API, then fails.
- **Health:** the Docker health check of every `keystone_fernet` container requires key `0` owned by
  `keystone:keystone` and a successful dry-run `rsync` to every other host, so an `unhealthy` `keystone_fernet` in
  `docker ps` is the quickest pointer to cause 2 or 3.
- **Rotation** later runs from cron inside `keystone_fernet`, every `fernet_key_rotation_interval` seconds (by
  default `fernet_token_expiry` plus the allowed expired window); `fernet-rotate.sh` ends with the same
  `fernet-push.sh`, so a host with an empty key directory and a running `keystone_fernet` can also empty the others
  at its next rotation.

## Possible causes

### Cause 1: a run with --limit or kolla_serial when no keystone_fernet runs anywhere

First deployment with a limit, or every `keystone_fernet` container stopped (after a `kolla-ansible stop`, or a
power loss) followed by a limited run. The role's code notes that it does not detect a stopped container or a volume
without a container as "bootstrapped". Tell it apart: the bootstrap message above.

### Cause 2: the keystone_ssh path between controllers is broken

The `keystone_ssh` container is not running, port 8023 is blocked between controllers, or the SSH key pair differs
between hosts (different `passwords.yml` used for different runs). Tell it apart: the SSH port wait fails, or
`fernet-push.sh` fails with an SSH or `rsync` error; `fernet-healthcheck.sh` exits non-zero.

### Cause 3: key 0 is missing or has the wrong owner on a host

The `keystone_fernet_tokens` volume of a host was removed or emptied (a reinstalled controller), and distribution
has not reached it yet. Tell it apart: `ls -l /etc/keystone/fernet-keys/` on that host is empty or not owned by
`keystone:keystone`; the `keystone` container logs the waiting messages above.

## Resolution

### For cause 1

Run the command on every Keystone host, without `--limit` and without `kolla_serial`, so that the role can decide
safely. If every `keystone_fernet` was only stopped and the keys still exist in their volumes, start the containers
first, so that the role sees them running and does not bootstrap. Check each host's keys first (read-only; the
container is stopped, so look at the volume):

```bash
sudo ls -l /var/lib/docker/volumes/keystone_fernet_tokens/_data/    # expected: files 0, 1, ... owned by the keystone user (42425 on the host)
# with Podman: /var/lib/containers/storage/volumes/keystone_fernet_tokens/_data/
```

> ⚠️ **State-changing.** Starts the stopped `keystone_fernet` container of a host; nothing else. Start it **only on
> hosts whose key directory is not empty**: the first running one becomes the source of the next push, which deletes
> on the other hosts whatever it does not have. Undo: `sudo systemctl stop kolla-keystone_fernet-container.service`.

```bash
# on each Keystone host whose key directory holds keys, if the container exists but is stopped
sudo systemctl start kolla-keystone_fernet-container.service    # Kolla-Ansible 2026.1: systemd units, not docker start
```

> ⚠️ **State-changing.** A full run of the keystone role on every host. If no `keystone_fernet` runs anywhere, it
> runs `fernet_setup` on the first Keystone host: on an empty key repository this creates new keys, every existing
> token becomes invalid and users must authenticate again (whether `fernet_setup` replaces keys already present was
> not checked for this page). If the push source has an empty key directory, the keys of every other host are
> deleted and every Keystone API fails. Undo: none for the invalidated tokens or the deleted keys.

```bash
kolla-ansible deploy -i <inventory> --tags keystone
```

### For cause 2

Start `keystone_ssh` where it is down, open `keystone_ssh_port` between the controllers on the API network, and use
the one `passwords.yml` of the deployment. Then run the keystone role again (above, the same ⚠️ applies), which
redistributes the keys.

### For cause 3

Make sure the push will come from a host that has the keys, then let the role redistribute them.

1. On the host with the missing keys, read-only: `sudo docker exec keystone_fernet ls -l /etc/keystone/fernet-keys/`
   (or the volume path above if the container is stopped). If the directory is empty and that host's
   `keystone_fernet` is running, stop it, so that it cannot be the push source, neither in the deploy nor at its next
   cron rotation:

   > ⚠️ **State-changing.** Stops `keystone_fernet` on the host with no keys: its rotation and pushes stop; its
   > `keystone` API was already failing without keys. The deploy below starts it again. Undo: `sudo systemctl start
   > kolla-keystone_fernet-container.service`.

   ```bash
   sudo systemctl stop kolla-keystone_fernet-container.service
   ```

2. Check that at least one other Keystone host has a running `keystone_fernet` and a full key directory (the
   `sha256sum` command of the quick checks).
3. Run the keystone role on every host: `kolla-ansible deploy -i <inventory> --tags keystone` (the same ⚠️ as
   above). The push now comes from the first host, in inventory order, whose `keystone_fernet` was running (the host
   stopped in step 1 is not among them, even though the deploy starts it again: the role chooses the source before it
   restarts containers).
4. Compare the key hashes on every host (*Verification*).

Do not copy keys by hand between hosts, and never run `fernet_setup` yourself on one host of a running cluster: that
host's keys would no longer match the others'.

## Verification

```bash
# on each Keystone host: the same key files, with the same hashes
sudo docker exec keystone_fernet sh -c 'sha256sum /etc/keystone/fernet-keys/*'
sudo docker exec keystone_fernet /usr/bin/fernet-healthcheck.sh; echo "exit=$?"     # exit=0

# from a client, several times (the load balancer spreads the calls over the hosts)
openstack token issue
openstack project list
```

## Lab observations

OpenStack 2026.1, Kolla-Ansible 22.2.0 (ansible-core output format), 3 controllers with Keystone, 2026-10-27.

- Cause 1: `keystone_fernet` stopped with `systemctl` on the three controllers, then `kolla-ansible deploy --tags
  keystone --limit controller-1`: failed in 49 s at *Fail if any hosts need bootstrapping and not all hosts
  targeted*, message as shown in *Symptom* (only the targeted host named). No key was touched. The volume paths
  showed `0` and `1` owned by `42425`; `systemctl start` on the three hosts made them `healthy`.
- Cause 2, simulated with an `iptables` DROP of port 8023 on controller-3: `fernet-healthcheck.sh` on controller-1
  exited 1 and `nc` timed out. The full `deploy --tags keystone` then sat in *Waiting for Keystone SSH port to be UP*:
  three `FAILED - RETRYING` lines for controller-3 in about 18 minutes, consistent with 300 s per try (10 retries
  would take about 55 minutes before the failure). After the rule was removed, the next try passed, the
  distribution ran, and the deploy ended with no failure.
- Cause 3: the key directory of controller-3 emptied, `keystone` restarted there: `ERROR: Fernet keys have not been
  populated, rechecking in 5 seconds`, then `CRITICAL: Waited for 3 minutes - failing` and the container exited;
  tokens kept working through the other two hosts. Steps 1 to 4 ran as written (with `systemctl stop`): the deploy
  took 1.5 minutes, the push ran from controller-1, and the three hosts ended with the same two key hashes,
  `fernet-healthcheck.sh` exit 0, and `token issue` / `project list` working six times out of six.
- Not run: a first deployment with `--limit`, a run with `kolla_serial`, a bootstrap on an empty cluster, a push
  from an empty host, a different `keystone_ssh_key`, and `keystone_ssh` stopped.

## Validity and limits

- Written from Kolla-Ansible 2026.1 code (`ansible/roles/keystone/tasks/bootstrap_service.yml`,
  `distribute_fernet.yml`, `config.yml`, `templates/fernet-*.sh.j2`, `templates/keystone-startup.sh.j2`,
  `defaults/main.yml`, `ansible/group_vars/all/keystone.yml`) and the Keystone Fernet FAQ. Reproduced on a lab for the
  three causes, as described in *Lab observations*.
- `sha256sum` of the key files is used to compare hosts without printing the keys; never paste the keys anywhere.
- Not covered: rotating `keystone_ssh_key` (see the password rotation guide), federation, Keystone behind an external
  load balancer, and Podman.

## Related

- [Rotating the secrets in passwords.yml](../configuration/rotating-secrets-in-passwords-yml.md)
- [deploy, reconfigure, upgrade and the others](../operations/deploy-reconfigure-upgrade-and-limiting-runs.md): what
  `--limit` and `kolla_serial` do
- Kolla-Ansible 2026.1: [Keystone guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/shared-services/keystone-guide.html)
- Keystone 2026.1: [Fernet token FAQ](https://docs.openstack.org/keystone/2026.1/admin/fernet-token-faq.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 code and documentation and the Keystone
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/keystone-fernet-bootstrap-or-key-distribution-fails.md&title=Feedback%3A+deployment-errors/keystone-fernet-bootstrap-or-key-distribution-fails.md):
your release, your topology, and what happened.
