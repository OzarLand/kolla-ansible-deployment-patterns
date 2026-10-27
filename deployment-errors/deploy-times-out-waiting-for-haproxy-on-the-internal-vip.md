---
title: kolla-ansible deploy times out at "Wait for haproxy to listen on VIP". The internal VIP never comes up, or cannot be reached
kind: runbook
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, keepalived, haproxy, docker]
deployment_methods: [kolla-ansible]
environment:
  topology: multinode with enable_haproxy and enable_keepalived (the defaults), 1 or more hosts in the loadbalancer group
  network: any
  storage: any
  os: any host OS supported by the Kolla-Ansible release
risk: state-changing
confirmed_by:
  - { who: "agent:lab-reviewer", type: lab, release: "2026.1", deployment: "kolla-ansible 22.2.0, 3 controllers (loadbalancer group, keepalived unicast, ProxySQL) + 2 computes, nested in an OpenStack cloud", date: 2026-10-27, note: "cause 4 reproduced: VIP removed from the allowed address pairs of the controller ports, reconfigure --tags loadbalancer with a changed keepalived_virtual_router_id timed out on the two hosts not holding the VIP; quick checks, cause 4 resolution (port set) and verification run, reconfigure then passed. The cause 4 undo needed mac-address. Not run: causes 1-3 and their resolutions, the nonlocal_bind and wrong-subnet variants, tcpdump, multicast mode" }
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html
  - https://docs.openstack.org/kolla-ansible/2026.1/user/multi-regions.html
  - https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html
authored_by: agent:ozarland-author
last_verified: 2026-10-27
---

# kolla-ansible deploy times out at "Wait for haproxy to listen on VIP"

## Symptom

`kolla-ansible deploy` (or `reconfigure`, or `upgrade`) stops in the `loadbalancer` role, after the keepalived and
HAProxy containers were (re)started, with:

```text
RUNNING HANDLER [loadbalancer : Wait for haproxy to listen on VIP] *************
ok: [controller-3]
[ERROR]: Task failed: Module failed: Timeout when waiting for 192.0.2.10:61313
Origin: ~/kolla-venv/share/kolla-ansible/ansible/roles/loadbalancer/handlers/main.yml:309:3
...
fatal: [controller-2]: FAILED! => {"changed": false, "elapsed": 301, "msg": "Timeout when waiting for 192.0.2.10:61313"}
fatal: [controller-1]: FAILED! => {"changed": false, "elapsed": 301, "msg": "Timeout when waiting for 192.0.2.10:61313"}
```

Captured on a lab (cause 4, below; Kolla-Ansible 22.2.0). `192.0.2.10` stands for `kolla_internal_vip_address`;
`61313` is `haproxy_monitor_port`. The handler waits for a TCP connection to that address and port, from each host of
the `loadbalancer` group, for 300 seconds. The failure can come on every host or only on some of them: on the lab,
the host holding the VIP (`controller-3`) passed and the two others failed; the run went on with the other plays and
ended with `failed=1` on those two hosts and `exited 2`.

The handler runs only when keepalived is restarted (it is notified by the *Start backup/master keepalived container*
handlers): on the lab, a change to HAProxy's configuration alone restarted HAProxy and ProxySQL, but not keepalived,
and the run did not wait on the VIP.

## Impact

On a first deployment, nothing else is deployed: every service after the load balancer reaches the database and
Keystone through the VIP. On a running platform (`reconfigure` or `upgrade`), the internal VIP may be down or held by
several hosts at once, and every API call and every service-to-service call through the VIP fails or flaps until
it is fixed. Instances keep running and their data plane does not go through the VIP, but their metadata requests
do: the Neutron metadata agents reach `nova-metadata` through the internal endpoint, so cloud-init on instances that
boot or reboot meanwhile fails.

## Quick checks

On each host of the `loadbalancer` group, read-only:

```bash
# 1. Who holds the VIP? Exactly one host should list it.
ip -o addr show | grep -w '<internal-vip>'

# 2. Are both containers running, or restarting?
sudo docker ps -a --format '{{.Names}}\t{{.Status}}' | grep -E '^(keepalived|haproxy)\b'

# 3. What keepalived says: MASTER, BACKUP or FAULT, and why
sudo docker logs --tail 50 keepalived

# 4. Does HAProxy listen on the monitor port?
sudo ss -ltnp | grep ':61313'
```

Check 4 lists the VIP on every host where HAProxy runs, holder or not (HAProxy binds it with `ip_nonlocal_bind`): it
says whether HAProxy listens, not who holds the VIP.

From a host that does not hold the VIP:

```bash
# 5. Can it reach the VIP?
ping -c 3 <internal-vip>
curl -s -o /dev/null -w '%{http_code}\n' http://<internal-vip>:61313/     # any HTTP code means HAProxy answered
```

## Diagnosis

Kolla-Ansible puts keepalived and HAProxy on every host of the `loadbalancer` group. keepalived holds
`kolla_internal_vip_address` (and the external VIP if it differs) on the `api_interface` of one host, elected with
VRRP: every keepalived starts as `BACKUP`, the highest priority one becomes `MASTER` and announces itself with VRRP
advertisements (multicast by default, `keepalived_traffic_mode: multicast`). A track script (`check_alive`, enabled by
default with `keepalived_track_script_enabled`) checks HAProxy through its socket: a host whose HAProxy does not
answer goes to `FAULT` and does not take the VIP. The instances are configured with `nopreempt`: a host that comes
back does not take the VIP back from the current `MASTER`.

So the wait fails for one of three reasons, which the quick checks tell apart:

| What the checks show | Meaning |
|---|---|
| No host holds the VIP; keepalived logs `FAULT` or the check script failing; HAProxy restarting or not listening | HAProxy does not run, so no keepalived takes the VIP (cause 1) |
| More than one host holds the VIP; each keepalived logs that it entered the `MASTER` state | The keepalived instances do not hear each other's VRRP advertisements (cause 2) |
| keepalived logs about advertisements from an unexpected source, or a VRID/password mismatch | Another VRRP cluster uses the same virtual router ID on that network (cause 3) |
| One host holds the VIP, HAProxy listens, but the other hosts cannot reach the VIP | The network does not deliver traffic to the VIP address (cause 4) |

Useful logs: `docker logs keepalived` (the Kolla image runs keepalived with console logging), and `docker logs
haproxy` for HAProxy start errors (its traffic log, through fluentd, is under `/var/log/kolla/haproxy/`).

To see VRRP on the wire, read-only, on a load balancer host:

```bash
sudo tcpdump -ni <api_interface> vrrp
# expected: advertisements from one source only (the MASTER), with the vrid of keepalived_virtual_router_id
```

## Possible causes

### Cause 1: HAProxy is not running on any load balancer host

HAProxy exits at start when its configuration is invalid or a file it needs is missing, most often the TLS
certificate when `kolla_enable_tls_internal` or `kolla_enable_tls_external` is set (the prechecks check for
`kolla_internal_fqdn_cert` and `kolla_external_fqdn_cert`, but only if they were run). With the track script, keepalived
then refuses the VIP everywhere. Tell it apart: `docker ps` shows `haproxy` restarting, and its log names the error.

A variant on the hosts that do not hold the VIP: HAProxy binds the VIP address even where it is not configured, which
needs `net.ipv4.ip_nonlocal_bind=1` (and `net.ipv6.ip_nonlocal_bind=1`). Kolla-Ansible sets these in the
`loadbalancer` role only when `set_sysctl` is true (the default). With `set_sysctl: false` and the sysctls not set by
other means, HAProxy cannot bind and exits on those hosts. Tell it apart: `sysctl net.ipv4.ip_nonlocal_bind` returns
`0`, and the HAProxy log says it cannot bind the VIP address.

### Cause 2: VRRP advertisements do not pass between the load balancer hosts

Every keepalived believes it is alone and takes the VIP: a split brain. Seen when multicast is filtered between the
hosts (some switches, some virtual networks), when a firewall on the hosts drops IP protocol 112 (VRRP), or when
the hosts are instances of another cloud whose ports filter it (security groups, port security). Tell it apart: the VIP
is on several hosts at once, and `tcpdump` on each host sees only its own advertisements.

### Cause 3: another VRRP cluster uses the same virtual router ID on the same network

`keepalived_virtual_router_id` defaults to `51`. Another Kolla-Ansible deployment, another region, or any other
keepalived on the same layer-2 network with ID 51 interferes with the election. The multinode and multi-region guides
ask for a unique value in that case. Tell it apart: `tcpdump` shows advertisements with `vrid 51` from addresses that
are not your load balancer hosts.

### Cause 4: the network does not deliver traffic to the VIP

The VIP is up on one host and HAProxy listens, but packets to the VIP do not arrive. Two common cases:

- the hosts are instances of another OpenStack cloud and port security drops traffic for an address that is not the
  port's own: the VIP must be in the *allowed address pairs* of every load balancer host's port;
- `kolla_internal_vip_address` is not in the subnet of `api_interface` (keepalived adds it to that interface, but
  nothing routes it there). On a first deployment, `kolla-ansible prechecks` catches this: its check *Checking if
  kolla_internal_vip_address is in the same network as api_interface on all nodes* runs on the hosts of the
  `loadbalancer` group, as long as no keepalived container exists on them yet. On a running platform it no longer
  runs.

Tell it apart: from another host, ARP for the VIP gets no answer, or gets one but the TCP connection never completes.
On a lab with the VIP removed from the allowed address pairs, the hosts that did not hold the VIP showed:

```text
2 packets transmitted, 0 received, +2 errors, 100% packet loss, time 1042ms      # ping -c 2 <internal-vip>
000                                                                                # curl … http://<internal-vip>:61313/
192.0.2.10 dev ens3 FAILED                                                         # ip neigh show <internal-vip>
```

while the holder answered `200` and keepalived logged a normal `Entering MASTER STATE` on one host only.

## Resolution

### For cause 1

Read why HAProxy stops, fix the configuration or the missing file, then run the deploy again. For a missing
certificate, generate or copy it to the path the prechecks name (`kolla-ansible certificates` generates test
certificates for a lab only). Run `kolla-ansible prechecks -i <inventory>` before the deploy: the certificate checks are
part of it.

> ⚠️ **State-changing.** Re-running the deploy restarts the load balancer containers again on the hosts whose
> configuration changed. On a running platform the VIP moves during the restart, which interrupts API calls for a few
> seconds. Limit it to the load balancer with `--tags loadbalancer` (see
> [Validity and limits](#validity-and-limits) for what this page did not run).

```bash
kolla-ansible deploy -i <inventory> --tags loadbalancer
```

### For cause 2

Let VRRP through, or stop relying on multicast.

- **Let VRRP through:** allow IP protocol 112 between the load balancer hosts on `api_interface` (host firewall,
  security group of the hosting cloud). On a hosting cloud, both the protocol and multicast must be allowed by its
  network back-end; check with `tcpdump` on each host.
- **Use unicast:** set `keepalived_traffic_mode: "unicast"` in `globals.yml`. Kolla-Ansible then writes a
  `unicast_peer` list with the API addresses of the other load balancer hosts, so no multicast is needed (protocol 112
  still must pass).

> ⚠️ **State-changing.** Rewrites keepalived's configuration and restarts keepalived and HAProxy on the load balancer
> hosts. On a running platform, the VIP moves once during the restart: API calls fail for a few seconds. Undo: the
> previous value and the same command.

```bash
kolla-ansible reconfigure -i <inventory> --tags loadbalancer
```

Then check that only one host holds the VIP.

### For cause 3

Choose an ID from 1 to 255 (the VRRP range, RFC 5798) that no other VRRP cluster uses on that network, and set it in `globals.yml`:

```yaml
keepalived_virtual_router_id: "<unique-id>"
```

> ⚠️ **State-changing.** Same effect as for cause 2: keepalived and HAProxy restart on the load balancer hosts and the
> VIP moves once. Coordinate with whoever runs the other cluster if it is in production too. Undo: the previous
> value and the same command.

```bash
kolla-ansible reconfigure -i <inventory> --tags loadbalancer
```

### For cause 4

- **Hosting cloud:** add the VIP (internal, and external if separate) to the allowed address pairs of every load
  balancer host's port, on the hosting cloud, by whoever administers those ports. On the lab, the VIP answered from
  every host within seconds of the change, without restarting anything; the next `reconfigure` passed the wait:

  > ⚠️ **State-changing.** Changes the hosting cloud's port; it does not touch the Kolla-Ansible deployment. Undo:
  > `openstack port unset --allowed-address ip-address=<internal-vip>,mac-address=<port-mac> <port-id>` (`port set`
  > stores the port's MAC with the address; without it, `unset` answers `Port does not contain allowed-address-pair`).

  ```bash
  openstack port set --allowed-address ip-address=<internal-vip> <port-id>
  ```

- **Wrong subnet:** choose a free address in the `api_interface` subnet for `kolla_internal_vip_address`. Changing the
  VIP of a running platform changes every internal endpoint and is out of scope here; on a first deployment, change it
  in `globals.yml` and deploy again.

## Verification

```bash
# on each load balancer host: exactly one of them lists the VIP
ip -o addr show | grep -w '<internal-vip>'

# from every host of the deployment
curl -s -o /dev/null -w '%{http_code}\n' http://<internal-vip>:61313/

# keepalived: one MASTER, the others BACKUP
sudo docker logs --tail 20 keepalived
```

Then the deploy goes past `Wait for haproxy to listen on VIP` and ends with `failed=0`.

## Validity and limits

- Written from the Kolla-Ansible 2026.1 code (`ansible/roles/loadbalancer/handlers/main.yml`, `tasks/precheck.yml`,
  `templates/keepalived/keepalived.conf.j2`, `defaults/main.yml`) and its HAProxy, multinode and multi-region guides.
  Cause 4 (allowed address pairs missing on a hosting cloud) was reproduced on a lab: Kolla-Ansible 22.2.0, 3
  controllers in the `loadbalancer` group with `keepalived_traffic_mode: "unicast"`, ProxySQL, nested in an OpenStack
  cloud; the failure was triggered by a `reconfigure --tags loadbalancer` that changed `keepalived_virtual_router_id`.
  Causes 1 to 3, the subnet variant of cause 4 and the multicast mode were not reproduced. Status `draft`.
- The causes are those the code and the guides make predictable; no incident was recorded for this page. The exact
  wording of keepalived's log lines depends on its version and is not quoted here on purpose.
- `--tags loadbalancer` follows the play tags of `ansible/site.yml` in 2026.1; check them in your release.
- Not covered: ProxySQL (which has its own `Wait for proxysql to listen on VIP`), IPv6 VIPs, external load balancers
  with `enable_haproxy: "no"`, and changing the VIP of a running platform.

## Related

- The precheck *Checking if kolla_internal_vip_address and kolla_external_vip_address are not pingable from any node*
  fails earlier when the VIP already answers before keepalived runs: another host uses the address.
- [Three controllers and two computes, nested in an OpenStack cloud](../topologies/three-controllers-two-computes-nested-in-an-openstack-cloud.md):
  allowed address pairs, VRRP and a unique `keepalived_virtual_router_id` on a hosting cloud.
- [Single control node versus three controllers](../comparisons/single-control-node-versus-three-controllers.md)
- Kolla-Ansible 2026.1: [HAProxy guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/high-availability/haproxy-guide.html),
  [multinode](https://docs.openstack.org/kolla-ansible/2026.1/user/multinode.html) (`keepalived_virtual_router_id`),
  [multi-regions](https://docs.openstack.org/kolla-ansible/2026.1/user/multi-regions.html)
- keepalived: [configuration synopsis](https://keepalived.readthedocs.io/en/latest/configuration_synopsis.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code.
Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=deployment-errors/deploy-times-out-waiting-for-haproxy-on-the-internal-vip.md&title=Feedback%3A+deployment-errors/deploy-times-out-waiting-for-haproxy-on-the-internal-vip.md):
your release, your topology, and what happened.
