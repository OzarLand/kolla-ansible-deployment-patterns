---
title: Designate with the BIND9 back-end in Kolla-Ansible. Name servers, NS records, Neutron DNS integration, and how to check a record end to end
kind: pattern
status: draft
openstack_releases: ["2026.1"]
components: [kolla-ansible, designate, bind9, neutron, nova]
deployment_methods: [kolla-ansible]
environment:
  purpose: any; written for a multinode deployment that publishes instance names in DNS
  scale: any, with the Designate services on the controllers and mDNS on the network nodes as in the sample inventory
  network: ml2/ovs or ml2/ovn
  storage: any
  os: any host OS supported by Kolla-Ansible 2026.1
risk: state-changing
confirmed_by: []
upstream_refs:
  - https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/designate-guide.html
  - https://docs.openstack.org/designate/2026.1/
  - https://docs.openstack.org/neutron/2026.1/admin/config-dns-int-ext-serv.html
authored_by: agent:ozarland-author
last_verified: 2026-10-26
---

# Designate with the BIND9 back-end in Kolla-Ansible. Name servers, NS records, Neutron DNS integration, and how to check a record end to end

## Context

Designate gives projects DNS zones and records through an API, and with Neutron's DNS integration it creates records
for instances and floating IPs automatically. With `enable_designate: "yes"` and the default `designate_backend:
"bind9"`, Kolla-Ansible deploys the Designate services and a BIND9 server that serves the zones.

Three things decide whether the names are actually resolvable, and they sit partly outside Kolla-Ansible:

1. **the name servers' names** (`designate_ns_record`), which become the NS records of every zone;
2. **where BIND9 listens**, and who can reach it;
3. **the delegation** of your zones from the parent DNS, which Kolla-Ansible does not do.

This page covers those, Neutron's integration, the optional `designate-sink`, and how to check a record from the API
down to a DNS answer.

## Topology

From the 2026.1 sample multinode inventory:

| Container | Group | Role |
|---|---|---|
| `designate_api`, `designate_central`, `designate_producer`, `designate_worker` | `designate` (`control`) | API, zone logic, periodic tasks, pushing zones to the back-end |
| `designate_backend_bind9` | `designate-backend-bind9` (`designate`) | The BIND9 server: authoritative for the zones, port 53 |
| `designate_mdns` | `designate-mdns` (`network`) | Serves zones to BIND9 by zone transfer, port 5354 |
| `designate_sink` | `designate-sink` (`designate`) | Optional: records from Nova and Neutron notifications, off by default |

With more than one `designate_worker`, the guide requires a coordination back-end, and only Valkey is supported:
`designate_coordination_backend` defaults to `valkey` when `enable_valkey` is true, and is empty otherwise. With three
controllers, enable Valkey.

## Network design

In 2026.1, `named.conf.j2` makes BIND9 listen on port 53 on the host's API address, and also on the `dns_interface`
address when that interface differs from the API one. `dns_interface` defaults to `network_interface`. The guide adds
that the mDNS node "requires the `dns_interface` to be reachable from management network".

So decide who must query BIND9:

- **only clients inside the platform:** the API network is enough;
- **resolvers or clients outside:** give the BIND9 hosts a `dns_interface` on a network those resolvers can reach,
  and open port 53 (UDP and TCP) to it.

BIND9 here is **authoritative only**: `designate_recursion` defaults to `"no"`. It does not resolve other names for
instances; their resolver stays the one their subnet's DHCP gives them.

## Storage design

Zones and records live in the Designate database (MariaDB); BIND9 keeps its copies under `/var/lib/named` in its
container's volume. Nothing to size beyond that.

## Inventory and configuration layout

```yaml
# globals.yml
enable_designate: "yes"
designate_backend: "bind9"                  # the default
designate_ns_record:                        # names, not addresses; they become the NS records of every zone
  - "ns1.cloud.example.org"
  - "ns2.cloud.example.org"
dns_interface: "<interface-reachable-by-resolvers>"   # default: network_interface
neutron_dns_domain: "cloud.example.org."    # must end with a dot and differ from "openstacklocal"
enable_valkey: "yes"                        # coordination for several designate_worker
```

`designate_ns_record` defaults to `ns1.example.org`: left as is, every zone announces a name server that is not
yours. The names must resolve, in the parent DNS, to the addresses BIND9 listens on, and the parent zone must
delegate your zones to them; Kolla-Ansible configures neither.

`pools.yaml` is generated from the inventory (one target per `designate-backend-bind9` host, mDNS hosts as masters)
unless you place your own at `/etc/kolla/config/designate/pools.yaml`. The deploy and reconfigure run
`designate-manage pool update` in `designate_worker`, so a change to the NS records reaches existing zones.

### Neutron DNS integration

With Designate enabled, `neutron_dns_integration` defaults to true. Neutron then sets `external_dns_driver = designate`
and `dns_domain = <neutron_dns_domain>`, and loads the `subnet_dns_publish_fixed_ip` extension driver (2026.1 roles).
Records are created in Designate when a port has a `dns_name` and its network has a `dns_domain` that matches an
existing zone; floating IPs get records the same way. See Neutron's *DNS integration with an external service* guide
for the exact rules per network type.

The guide warns: `neutron_dns_domain` "has to be different to `openstacklocal` (its default value) and has to end
with a period".

## Applying it

**1. Create the delegation first** (outside OpenStack): `ns1`/`ns2` names in the parent zone, pointing at the BIND9
hosts' addresses on `dns_interface`, and NS records delegating your zone to them.

**2. Deploy.**

> ⚠️ **State-changing.** Creates the Designate database, service user and endpoints, the Designate and BIND9
> containers, and reconfigures Neutron (`external_dns_driver`, the extension driver), whose containers restart where
> their configuration changed, and Nova: enabling Designate turns on Nova's notifications in `nova.conf` (2026.1
> templates), so the Nova containers restart, on every compute too. Running instances keep running. The new HAProxy frontend for the Designate API restarts the `haproxy` containers on every load balancer
> host, backups first, then the VIP holder: a brief interruption of every API behind the VIPs. Plan it in a window.
> Undo: `enable_designate: "no"` and reconfigure Neutron and Nova; the Designate containers and HAProxy fragment stay
> until removed by hand, as described for the monitoring page.

```bash
kolla-ansible deploy -i <inventory> --tags common,loadbalancer,valkey,designate,neutron,nova
```

**3. Create a zone and use it** (the guide's sequence, with your names):

> ⚠️ **State-changing.** Creates a zone and records; assigning a `dns_domain` to a network makes Neutron create records
> for new ports on it.

```bash
openstack zone create --email dnsadmin@example.org cloud.example.org.
openstack zone show cloud.example.org. -c status -c serial      # ACTIVE once BIND9 has it
openstack network set <network> --dns-domain cloud.example.org.
openstack server create --image <image> --flavor <flavor> --network <network> test-dns
openstack recordset list cloud.example.org.                      # an A record for test-dns
```

**4. Check each hop** (read-only):

```bash
dig +short -p 5354 @<mdns-host-dns-address> test-dns.cloud.example.org. A   # mDNS, what Designate serves
dig +short @<bind9-host-address> test-dns.cloud.example.org. A              # BIND9, the authoritative answer
dig +short test-dns.cloud.example.org. A                                    # through your resolvers: needs the delegation
dig +short NS cloud.example.org.                                            # the names from designate_ns_record
```

When the zone stays `PENDING`, Designate could not push it to BIND9: the worker, mDNS and BIND9 logs below say which
hop failed.

### designate-sink (optional)

The sink creates records from Nova and Neutron notifications, the older alternative to Neutron's integration. It is
off by default (`designate_enable_notifications_sink: true` enables it) and needs the zone ID in
`/etc/kolla/config/designate/designate-sink.conf` under `[handler:nova_fixed]` and `[handler:neutron_floatingip]`,
then `reconfigure --tags designate,neutron,nova` (guide). Use one mechanism, not both, so records are not created
twice: the guide's own example output shows two A records for the same instance.

## Where to look when it fails

| Symptom | First log to read |
|---|---|
| Zone stays `PENDING` or goes to `ERROR` | `/var/log/kolla/designate/designate-worker.log`: rndc to BIND9 on port 953, or the pool targets |
| BIND9 has no copy of the zone | `/var/log/kolla/designate/designate-mdns.log` (zone transfers) and `/var/log/kolla/named/` on the BIND9 host |
| API errors | `/var/log/kolla/designate/designate-api.log`, `designate-central.log` |
| No record for a new instance | `/var/log/kolla/neutron/neutron-server.log`: the external DNS driver's calls to Designate; the network's `dns_domain` and the port's `dns_name` |
| Name resolves on BIND9 but not from outside | the delegation and the `designate_ns_record` names in the parent zone; port 53 reachability on `dns_interface` |
| A container does not start | `docker logs <container>` (or `podman logs`) |

## Key choices and their consequences

| Choice | Alternative(s) | Why this one here | Consequence in operation |
|---|---|---|---|
| BIND9 deployed by Kolla-Ansible | An external BIND9 (`designate_backend_external`) or Infoblox | Nothing else to operate | Your zones are served from the controllers; their reachability on port 53 is yours to provide |
| Neutron DNS integration | `designate-sink` | The default; records follow ports and floating IPs | Records only for networks with a `dns_domain` and ports with a `dns_name` |
| Authoritative only (`designate_recursion: "no"`) | Recursion with forwarders | Avoids an open resolver | Instances keep their own resolvers |
| Valkey enabled | One `designate_worker` | Required by the guide with several workers | One more service to run |

## What went wrong during deployment

Nothing observed yet: written from the Kolla-Ansible 2026.1 Designate guide and the `designate` and `neutron` roles.

## Version notes

- 2026.1: coordination for several workers is Valkey only (Redis was replaced by Valkey in Kolla-Ansible).
- The Infoblox back-end needs the mDNS container to listen on port 53 and therefore to be rebuilt to run as root
  (guide); not covered here.

## Lessons learned

No first-hand lessons yet. What the sources make clear:

- Set `designate_ns_record` and arrange the delegation before the first zone: the NS records of every zone come
  from it.
- Check each hop with `dig`: mDNS on 5354, BIND9 on 53, then your resolvers.
- Choose Neutron's integration or the sink, not both.

## Validity and limits

Written from Kolla-Ansible 2026.1 (Designate guide, `ansible/group_vars/all/designate.yml`, the `designate` role and
its `named.conf.j2` and `pools.yaml.j2` templates, the `neutron` role defaults and templates, the sample multinode
inventory) and the Neutron DNS integration guide. Not run on a lab: no output on this page was observed. Not covered:
Infoblox and external BIND9 back-ends, DNSSEC signing of zones, reverse zones for floating IPs, several pools, and
Designate quotas and tenancy.

## Related

- [Monitoring with Prometheus, Grafana and Alertmanager](monitoring-prometheus-grafana-and-alerting.md): how HAProxy
  frontends are added on deploy and removed by hand on undo, which applies here too.
- [Provider networks with Kolla-Ansible](../networking/provider-networks-instances-on-flat-or-vlan-networks.md)
- Kolla-Ansible 2026.1: [Designate guide](https://docs.openstack.org/kolla-ansible/2026.1/reference/networking/designate-guide.html)
- Neutron 2026.1: [DNS integration with an external service](https://docs.openstack.org/neutron/2026.1/admin/config-dns-int-ext-serv.html)

---

*Provenance: written by agent:ozarland-author, from the Kolla-Ansible 2026.1 documentation and code, and the Neutron
documentation. Reviewed in its pull request (see the page history). Checked by a human operator: not yet.*

**Worked for you, or not?** [Tell us in one line](https://github.com/OzarLand/kolla-ansible-deployment-patterns/issues/new?template=documentation-proposal.yml&page=services/designate-dns-with-bind9-and-neutron.md&title=Feedback%3A+services/designate-dns-with-bind9-and-neutron.md):
your release, your topology, and what happened.
