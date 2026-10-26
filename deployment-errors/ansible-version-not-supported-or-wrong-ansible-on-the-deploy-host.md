---
title: kolla-ansible prechecks fails with "Ansible version should be between ...", or plays fail on missing modules. The deploy host runs the wrong Ansible or the wrong collections
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, ansible]
deployment_methods: [kolla-ansible]
environment:
  topology: any; the problem is on the deploy host
  network: any
  storage: any
  os: any deploy host OS; Python virtual environment as in the quickstart
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/user/quickstart.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/virtual-environments.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# kolla-ansible prechecks fails with "Ansible version should be between ...": the wrong Ansible or collections on the deploy host

## Symptom

`kolla-ansible prechecks` stops early with:

```text
TASK [prechecks : Checking Ansible version] ****
fatal: [controller-1]: FAILED! => {"assertion": "...", "changed": false, "evaluated_to": false,
  "msg": "Ansible version should be between 2.19 and 2.20. Current version is 2.16.3 which is not supported."}
```

The task name and the message are those of `ansible/roles/prechecks/tasks/package_checks.yml` in Kolla-Ansible
2026.1 (the surrounding JSON is reconstructed, not copied from a run); the bounds come from `ansible_version_min: '2.19'` and `ansible_version_max: '2.20'` in the prechecks role. The
current version shown here is an example.

Related forms of the same problem, on any command (`deploy`, `reconfigure`, `upgrade`), when prechecks were skipped:

- Ansible stops on a module, filter or plugin it cannot find in a collection (`couldn't resolve module/action`, or a
  filter that "is not available");
- a play fails on syntax or keywords that the installed `ansible-core` does not know, or that a newer one removed.

The exact wording of those Ansible errors depends on the `ansible-core` version and is not quoted here.

## Impact

Nothing has been changed on the hosts: the run stops on the deploy host's own tools. The platform keeps running.
The risk is in the fix: installing a different Kolla-Ansible or Ansible into the wrong environment, and then running
it against production.

## Quick checks

On the deploy host, read-only:

```bash
# 1. Which kolla-ansible and which ansible-playbook will run?
command -v kolla-ansible ansible-playbook ansible-galaxy
ansible-playbook --version | head -1

# 2. What is installed in the virtual environment Kolla-Ansible runs from?
/path/to/venv/bin/pip show kolla-ansible ansible-core | grep -E '^(Name|Version)'

# 3. Which collections Ansible sees, and from where
ansible-galaxy collection list 2>/dev/null | grep -E '^#|community.docker|ansible.utils|community.general'
```

The problem is confirmed when `ansible-playbook` is not the one of the virtual environment, or its version is outside
what the Kolla-Ansible release requires, or the collections listed are not those Kolla-Ansible installed.

## Diagnosis

Three facts from the 2026.1 code explain most cases:

- **Kolla-Ansible runs `ansible-playbook` from the `PATH`.** The CLI builds the command with the bare name
  `ansible-playbook` (`kolla_ansible/ansible.py`). If the virtual environment is not activated and `kolla-ansible` is
  called by its full path, the `ansible-playbook` found first in the `PATH` may be a system package or another
  virtual environment's.
- **Each release pins an `ansible-core` range.** 2026.1 requires `ansible-core>=2.19,!=2.19.0,<2.21`
  (`requirements.txt`), and its prechecks accept 2.19 and 2.20. A `pip install` of Kolla-Ansible installs a matching
  `ansible-core` into its own environment; installing Ansible separately, or upgrading Kolla-Ansible across a series
  without following the operating guide's step for `ansible-core`, breaks that.
- **`kolla-ansible install-deps` installs the Galaxy collections at user level.** It runs `ansible-galaxy collection
  install` with Kolla-Ansible's `requirements.yml` and `requirements-core.yml` and no collections path
  (`kolla_ansible/utils.py`), so they land in Ansible's default collections path, normally under the user's home
  directory, shared by every virtual environment of that user. Two Kolla-Ansible releases on one deploy host, for
  example during an upgrade, share them, and the last `install-deps` wins.

## Possible causes

### Cause 1: the virtual environment is not active

`kolla-ansible` runs, but `ansible-playbook` comes from elsewhere. Tell it apart: `command -v ansible-playbook` is not
in `/path/to/venv/bin/`.

### Cause 2: the wrong ansible-core in the virtual environment

The environment has an `ansible-core` outside the release's range, after a manual install or a partial upgrade. Tell
it apart: `pip show ansible-core` in the environment gives a version outside 2.19–2.20 for 2026.1.

### Cause 3: collections from another release, or none

`install-deps` was not run after installing or upgrading Kolla-Ansible, or another release's `install-deps` ran
later for the same user. Tell it apart: the versions in `ansible-galaxy collection list` do not satisfy the bounds in
`/path/to/venv/share/kolla-ansible/requirements.yml` and `requirements-core.yml`.

## Resolution

### For cause 1

Activate the environment before any `kolla-ansible` command, in every shell, script and scheduled job:

```bash
source /path/to/venv/bin/activate
```

### For cause 2

Install the range the release requires into the environment, as the operating guide does for skip-level upgrades:

> ⚠️ **State-changing.** Changes the Python packages of the deploy host's virtual environment only; nothing on the
> OpenStack hosts. Undo: install the previous version back with the same command. Do it outside a running
> `kolla-ansible` command.

```bash
source /path/to/venv/bin/activate
pip install 'ansible-core>=2.19,!=2.19.0,<2.21'        # the 2026.1 range; read requirements.txt of your release
```

### For cause 3

> ⚠️ **State-changing.** Reinstalls the Galaxy collections for this user, with `--force`; another Kolla-Ansible
> release used by the same user then sees these versions too. Undo: run `install-deps` from the other release's
> environment.

```bash
source /path/to/venv/bin/activate
kolla-ansible install-deps
```

To keep two releases apart on one deploy host, run each from its own system user, or give each environment its own
collections path (Ansible's `ANSIBLE_COLLECTIONS_PATH`, or `collections_path` in an `ansible.cfg`) and install the
collections there with `ansible-galaxy collection install -r <requirements file> -p <path>`. This is Ansible's
mechanism, not a Kolla-Ansible option; test it on a lab first.

## Verification

```bash
command -v ansible-playbook                       # /path/to/venv/bin/ansible-playbook
ansible-playbook --version | head -1              # within the release's range
kolla-ansible prechecks -i <inventory>            # passes "Checking Ansible version"
```

## Validity and limits

- Written from the Kolla-Ansible 2026.1 code (`ansible/roles/prechecks/tasks/package_checks.yml`,
  `ansible/roles/prechecks/vars/main.yml`, `requirements.txt`, `requirements.yml`, `requirements-core.yml`,
  `kolla_ansible/ansible.py`, `kolla_ansible/utils.py`) and the quickstart and operating guides. Not reproduced on a
  lab for this page. Status `draft`.
- Ansible's own error messages for missing collections or unsupported syntax vary by version and are described, not
  quoted.
- Not covered: Kolla-Ansible installed from a distribution package or a container image, and Ansible configuration
  files (`ansible.cfg`) that change the collections path.

## Related

- [deploy, reconfigure, upgrade and the others](../operations/deploy-reconfigure-upgrade-and-limiting-runs.md): keep
  the deploy host's Kolla-Ansible pinned to the platform's release
- [From 2025.1 to 2026.1](../version-notes/2025.1-to-2026.1.md): the upgrade sequence, including Ansible
- Kolla-Ansible 2026.1: [quickstart](https://docs.openstack.org/kolla-ansible/2026.1/user/quickstart.html),
  [virtual environments](https://docs.openstack.org/kolla-ansible/2026.1/user/virtual-environments.html),
  [operating Kolla](https://docs.openstack.org/kolla-ansible/2026.1/user/operating-kolla.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 code and documentation.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/ansible-version-not-supported-or-wrong-ansible-on-the-deploy-host.md&title=Feedback%3A+deployment-errors/ansible-version-not-supported-or-wrong-ansible-on-the-deploy-host.md):
your release, your topology, and what happened.
