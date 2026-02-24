# Designate Adoption


## Introduction

This document describes the scenarios for adopting the Designate DNS-as-a-Service
from a Director-based TripleO deployment to a RHOSO (Red Hat OpenStack Services
on OpenShift) deployment. It is intended for the engineering team implementing
the adoption tooling and covers the trade-offs, requirements, and breakage risks
of each approach. The objective is not to decide on a particular scenario but to
examine adoption cases our customers are likely to encounter based on standard
use cases designate was designed for.

### Components Involved

- **Designate API** — the REST API service for managing DNS zones and records.
- **Designate mDNS** — the mini-DNS service that sends NOTIFY messages and
  serves zone transfers (AXFR/IXFR) to the authoritative nameservers.
- **Designate Worker** — performs zone create/update/delete operations against
  the backend nameservers via rndc and monitors zone status.
- **Designate Producer** — handles periodic tasks such as polling for zone
  status and emitting zone update events.
- **BIND9** — the authoritative nameserver(s) managed by Designate. These
  actually serve DNS queries for the zones Designate manages.
- **Unbound** — caching recursive resolver(s) typically used by tenant VMs
  and neutron-managed networks to resolve DNS queries.

### Director-Era Topology

In a Director-based deployment, Designate services (API, mDNS, Worker, Producer)
run on the controller nodes. BIND9 servers also run on the controllers, listening
on the Public API / External network to serve authoritative DNS queries. Unbound
resolvers run on the controllers as well, providing recursive DNS resolution to
tenant VMs and internal services. Control plane communication (rndc commands,
zone transfers from mDNS to BIND9) happens over the controller's internal
networks. The Designate database runs on the controllers as part of the shared
MariaDB/Galera cluster.

### RHOSO Topology

In RHOSO, Designate services run as pods within OpenShift. BIND9 and Unbound are
deployed as OpenShift-managed pods with networking provided by OpenShift's SDN.
The Designate database is migrated into the OpenShift-hosted MariaDB instance.
Service endpoints get new IP addresses allocated from the OpenShift network
ranges rather than the Director-era Public API network. Control plane traffic
(rndc, zone transfers) flows over the OpenShift pod network rather than the
controller's internal networks. This fundamental shift in network topology is the
primary source of complexity in each adoption scenario.


## Common Prerequisites

Before executing any of the scenarios below, several common steps apply.

### Database Migration

The Designate database must be migrated from the Director-era MariaDB/Galera
cluster to the RHOSO MariaDB instance. This includes all zone data, recordset
data, pool configurations, and tenant/project associations. The database schema
is expected to be compatible; the migration is a data transfer rather than a
schema transformation.

### Pool and NS Record Extraction

All scenarios require extracting the pool configuration and NS records from the
original Designate database. This information is used to generate the new pool
configuration in the RHOSO deployment. The NS records define which nameservers
are authoritative for each zone, and these must be updated to reflect the new
BIND9 server addresses in RHOSO (or preserved if legacy servers are being kept).

### Network Considerations

The common networking challenge across all scenarios is the shift from
controller-hosted services with static IPs on well-known Director networks to
OpenShift-managed pods with addresses allocated from the OpenShift network. Key
considerations include:

- BIND9 authoritative servers will have new IP addresses, affecting delegation
  records in parent zones and any external systems that reference them directly.
- Unbound resolvers will have new IP addresses, affecting any VMs or Neutron
  network configurations that reference them as nameservers.
- Control plane traffic (rndc, zone transfers, mDNS NOTIFY) must be routable
  between the new Designate services and whichever BIND9 servers they manage.


## The CI Friendly Scenario

Everything is transferred to the OpenShift cluster and only the newly deployed
services will be kept up to date. This is a clean cutover: the Director-era
BIND9 and Unbound services are decommissioned and all DNS functionality moves
to the new OpenShift-managed pods.


### Network Topology Changes

This scenario represents a complete network cutover. The old BIND9 servers on
the Director controllers stop serving DNS, and their Public API network addresses
become unreachable once the controllers are decommissioned. The new BIND9 pods in
OpenShift get new IP addresses from the OpenShift network ranges.

The implications are:

- **External delegation records** in parent DNS zones that point at the old
  BIND9 server IPs must be updated to point at the new BIND9 pod addresses.
  Until this update propagates, delegated subzones will be unresolvable from
  outside the cloud.
- **Unbound resolver references** configured in Neutron subnets, network DHCP
  options, or directly in tenant VMs will point at the old controller IPs. These
  become stale immediately. Tenants will lose DNS resolution until they are
  reconfigured to use the new Unbound pod addresses.
- **The old Public API network** used by the Director BIND9/Unbound services
  is no longer relevant. If the new OpenShift pods need to be reachable on
  specific external networks, additional OpenShift network configuration
  (MetalLB, NodePort, or similar) may be necessary.

This is the simplest network topology — there is no hybrid period and no need to
maintain routing between old and new environments for DNS control plane traffic.


### Operator Behavior

The Designate operator behaves in its standard mode. It manages the full
lifecycle of the new BIND9 pods deployed within OpenShift: creating them,
configuring rndc keys, setting up zone transfers from mDNS, and monitoring
their health. It has no awareness of or interaction with the legacy Director
BIND9 servers. No special operator modes or additional development is required
for this scenario.


### What this provides

While it is possible that this will be suitable for some customers, it allows us
to test that we can transfer data that is configured in the Director based
Designate to a RHOSO Designate deployment. This is the baseline adoption
scenario and validates that the core database migration, pool reconfiguration,
and zone creation pipeline works end to end. It is the foundation on which the
other scenarios build.


### What does this require

This is a standard service adoption scenario. The Designate-specific operations
are:

- Extraction of the NS records from the original Designate database so they can
  be used in the generation of the new pool information in the RHOSO Designate
  deployment.
- Updating external delegation records in parent DNS zones to point at the new
  BIND9 pod addresses.
- Updating Neutron subnet DNS nameserver references and any other configurations
  that point VMs at the old Unbound resolver addresses.
- Coordination with the customer to schedule the cutover, since DNS resolution
  will be disrupted for tenants during the transition window.


### What does this break


#### Integration with External DNS Systems

Records in parent zones for delegating to the Director BIND9 servers for
subzones will need to be updated. The Public API network used for the Director
based BIND9 servers will also no longer be available so additional network
configuration may be necessary.


#### Nameservers in VMs May Need Updating

References to the Director Unbound Resolver which may have been configured
through Neutron and/or OpenStack networks will no longer be valid and will need
to be migrated. This includes DHCP-assigned nameservers in existing leases,
statically configured resolvers in tenant VMs, and any hardcoded references in
tenant application configurations.


## The Half Measure

Everything is transferred to the OpenShift cluster, only the newly deployed
services will be kept up to date but the Director Unbound servers will remain
running. This is a database migration that will create the zones in the new
BIND9 servers. Keeping the Unbound resolvers in place avoids forcing immediate
changes to tenant VM DNS configurations.


### Network Topology Changes

In this scenario, the new BIND9 servers run in OpenShift pods with new IP
addresses, while the old Unbound resolvers continue running on the Director
controllers with their original addresses.

The key insight is that Unbound, as a recursive resolver, generally handles DNS
queries by performing recursive resolution itself or forwarding to configured
upstream DNS servers. It does **not** typically reference the Designate-managed
BIND9 servers directly. This means that in the common case, the old Unbound
resolvers will continue functioning normally after the BIND9 cutover — they
resolve queries through the standard DNS hierarchy and are not dependent on the
local BIND9 instances.

The exception is **stub zones**. If the Unbound configuration includes stub zone
entries that point at the old BIND9 server addresses (e.g., for resolving
Designate-managed zones without going through the public DNS hierarchy), these
references will break when the old BIND9 servers are decommissioned. The stub
zone queries will fail because the target IPs are no longer serving DNS.

The network paths break down as:

- **Common case (recursive resolution / forwarding):** Unbound continues to
  work. It resolves queries through the DNS hierarchy or its configured
  forwarders. No disruption to tenants using the old Unbound addresses.
- **Stub zone case:** Any stub zone configurations pointing at old BIND9 server
  IPs will fail. These must be either reconfigured to point at the new BIND9 pod
  addresses (which requires network connectivity from the controllers to the
  OpenShift pod network) or removed entirely (relying on public delegation
  records to route queries to the new BIND9 servers through the normal DNS
  hierarchy).

The external delegation record situation is the same as the CI Friendly scenario:
parent zone records pointing at old BIND9 IPs must be updated.


### Operator Behavior

The Designate operator manages only the new BIND9 pods deployed within
OpenShift. It has no involvement with the legacy Unbound resolvers running on the
Director controllers.

This means that the legacy Unbound instances are outside of any automated
management. If Unbound configuration changes are needed (such as updating stub
zone targets, changing forwarder addresses, or applying security patches), these
must be performed manually on the Director controller nodes. There is no
mechanism in the RHOSO deployment tooling to push configuration updates to legacy
Director-era services. The customer or operations team must accept responsibility
for maintaining these Unbound instances for as long as they remain in service.


### What this provides

This scenario provides a practical middle ground: the authoritative DNS
infrastructure is fully migrated to RHOSO while tenant-facing DNS resolution
remains uninterrupted (in the common case). It avoids the need to immediately
update every tenant VM's DNS configuration, which can be a significant
operational burden in large deployments. The BIND9 migration and operator
behavior are identical to the CI Friendly scenario and will be well-tested in CI.
The exception is the legacy Unbound resolvers, which depending on how they were
used, may continue to function normally.


### What does this require

This is a standard service adoption scenario. In addition to extracting the NS
records from the original Designate database for pool generation in the RHOSO
deployment:

- The Unbound resolver configuration must be audited for stub zone entries or
  any other references to the legacy BIND9 servers. Any such references must be
  reconfigured to point at the new BIND9 pod addresses or removed.
- The Director controller nodes must remain running (at least the Unbound
  service) for as long as tenants depend on the old resolver addresses.
- A plan for eventually migrating tenants off the old Unbound addresses must be
  established, since the legacy resolvers represent an ongoing maintenance
  burden with no automated management.


### What does this break


#### Integration with External DNS Systems

Records in parent zones for delegating to the Director BIND9 servers for
subzones will need to be updated. The Public API network used for the Director
based BIND9 servers will also no longer be available so additional network
configuration may be necessary.


#### Unbound Configuration

Reconfiguration of the legacy Unbound instances will not be handled by the
deployment tools. Stub zone configurations or any other references to the legacy
BIND9 servers will no longer work and must be updated manually. The legacy
Unbound instances become an unmanaged dependency — they will continue to serve
DNS, but no tooling exists to update their configuration, apply patches, or
monitor their health as part of the RHOSO deployment.


## The Maintenance Window Approach

A hybrid approach where new BIND9 servers and Unbound resolvers are deployed in
the OpenShift cluster but the old instances are left in place. The legacy servers
are reconfigured to use the internalapi network for control plane operations
(rndc, zone transfers) while continuing to serve DNS queries on the old Public
API / External network. The legacy mDNS servers are decommissioned and the new
mDNS servers in OpenShift take over zone update notifications to both the new
and legacy BIND9 servers.


### Network Topology Changes

This scenario creates a hybrid topology that spans both the Director controllers
and the OpenShift cluster simultaneously.

The legacy BIND9 and Unbound servers remain running on the Director controllers.
They continue to serve DNS queries on their original Public API / External
network addresses, so existing delegation records and tenant resolver
configurations remain valid. However, their control plane connectivity is
reconfigured: rndc and zone transfer traffic moves to the internalapi network,
which must be routable from the OpenShift pod network where the new Designate
mDNS and Worker services run.

The new BIND9 and Unbound pods in OpenShift serve on new addresses allocated from
the OpenShift network ranges. These new servers also receive zone data from the
new mDNS service.

During the dual-stack period:

- **Both old and new BIND9 servers** are authoritative for the same zones. The
  Designate Worker and mDNS services must push zone updates to both sets of
  servers to keep them in sync.
- **Zone transfers** flow from the OpenShift mDNS to both the new BIND9 pods
  (over the OpenShift pod network) and the legacy BIND9 servers (over the
  internalapi network). This requires the internalapi network to be reachable
  from the OpenShift environment.
- **rndc commands** from the Designate Worker must reach both the new BIND9
  pods and the legacy BIND9 servers. The legacy servers must be reconfigured to
  accept rndc on the internalapi network rather than localhost or the old control
  network.
- **Unbound resolvers** exist in both environments. The old resolvers continue
  serving tenants on the original addresses. The new resolvers are available on
  new addresses. No stub zone reconfiguration is necessary on the old Unbound
  instances since the old BIND9 servers are still running at their original
  addresses.

The eventual cutover involves:

1. Updating external delegation records to point at the new BIND9 pod addresses.
2. Migrating tenant resolver references to the new Unbound pod addresses.
3. Removing the legacy BIND9 servers from the Designate pool configuration.
4. Decommissioning the legacy BIND9 and Unbound services on the Director
   controllers.


### Operator Behavior

**This scenario requires new development in the Designate operator.** The
operator does not currently support managing BIND9 servers that run outside of
OpenShift. The following capabilities would need to be implemented:

- **Pool configuration with external BIND9 targets.** The operator must be able
  to define pools that include both OpenShift-managed BIND9 pods and external
  legacy BIND9 servers. The pool configuration must include the legacy servers'
  internalapi network addresses, rndc keys, and zone transfer endpoints.

- **rndc access to legacy servers over internalapi.** The Worker service running
  in OpenShift must be able to issue rndc commands to the legacy BIND9 servers
  over the internalapi network. This requires network routing between OpenShift
  pods and the internalapi network, as well as rndc key distribution to the
  Worker pods.

- **Zone transfer setup spanning both environments.** The mDNS service must send
  NOTIFY messages to both the new BIND9 pods and the legacy BIND9 servers. The
  legacy BIND9 servers must be configured to accept zone transfers from the new
  mDNS pod addresses on the internalapi network.

- **Decommissioning workflow.** The operator needs a mechanism to remove legacy
  BIND9 servers from the pool once migration is complete. This should include
  verifying that all zones are fully synchronized to the new BIND9 pods before
  the legacy servers are removed from the pool, and possibly a drain period where
  the legacy servers are still receiving zone updates but are no longer listed
  in NS records.

**This is a significant development gap.** The current operator assumes it has
full lifecycle control over all BIND9 instances it manages (create, configure,
monitor, destroy). Supporting externally managed BIND9 servers that the operator
can configure but not provision or destroy is a new mode of operation that would
require design work and testing beyond the current operator architecture.


### What this provides

This approach provides full service continuity. Existing delegation records,
tenant resolver configurations, and external DNS integrations all continue to
work without modification during the transition period. The cutover to new
addresses can happen gradually and on the customer's schedule, allowing them to
update references in batches rather than all at once. This is the lowest-risk
scenario from a tenant-facing perspective.


### What does this require

This requires:

- Reconfiguration of the legacy BIND9 servers to accept rndc commands and zone
  transfers on the internalapi network, with appropriate firewall rules and rndc
  key configuration.

- Network routing between the OpenShift pod network and the internalapi network
  on the Director controllers, to enable the Designate Worker and mDNS pods to
  communicate with the legacy BIND9 servers.

- **New development in the Designate operator** to support managing external
  BIND9 targets in pool configurations, as described in the Operator Behavior
  section above.

- The legacy controller servers must continue running for the duration of the
  dual-stack period, which may be weeks or months depending on the customer's
  migration timeline.

- A clear decommissioning plan and tooling to remove the legacy servers from the
  pool once all references have been migrated to the new addresses.


### What does this break

While no Designate functionality is broken during the dual-stack period, this
scenario introduces operational complexity that affects supportability:

- The legacy BIND9 and Unbound servers are not managed by the RHOSO deployment
  tooling. Any patches, configuration changes, or troubleshooting on these
  servers must be done manually.
- The hybrid pool configuration (OpenShift pods plus external servers) is not a
  tested or supported operator mode today. Until the operator is extended, this
  scenario cannot be implemented.
- The longer the dual-stack period persists, the greater the risk of
  configuration drift between the old and new servers.
- It is not clear that standard support can be extended to the legacy server
  components during the transition period, since they fall outside the RHOSO
  management boundary.


## Scenario Comparison

| Consideration              | CI Friendly         | Half Measure          | Maintenance Window    |
|----------------------------|---------------------|-----------------------|-----------------------|
| **Service continuity**     | Disruption during   | Partial — resolvers   | Full — no disruption  |
|                            | cutover             | stay, BIND9 cuts over | during transition     |
| **Operator complexity**    | Standard — no new   | Standard — no new     | Significant — new     |
|                            | development needed  | development needed    | development required  |
| **Network changes**        | Full cutover — all  | BIND9 addresses       | Minimal — old IPs     |
|                            | IPs change at once  | change, Unbound stays | stay active           |
| **Maintenance burden**     | None — legacy       | Moderate — legacy     | High — legacy servers |
|                            | servers removed     | Unbound must be       | must be maintained    |
|                            |                     | maintained manually   | throughout transition |
| **Risk level**             | Moderate — tenant   | Low-Moderate —        | Low (tenant-facing)   |
|                            | disruption during   | BIND9 cutover risk    | but High (engineering |
|                            | transition          | but resolvers stable  | and support risk)     |
| **Development readiness**  | Ready — standard    | Ready — standard      | Not ready — operator  |
|                            | adoption path       | adoption path         | gaps must be filled   |


# Adding the Maintenance Window Approach to the data-plane-adoption repository

The objective is to add documentation and an ansible roles support for the
migration window approach.



<!-- vim: set tw=80 ai formatoptions+=a : -->
