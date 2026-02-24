# Designate Adoption - a full GA story


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
- **Designate Central** — the core orchestration service that acts as the
  primary business logic layer. It receives requests from the API via RPC,
  manages all zone and record state in the database, and coordinates downstream
  actions by dispatching work to the Worker and mDNS services. All other
  Designate services communicate through Central rather than directly with each
  other or the database.
- **Designate Worker** — performs zone create/update/delete operations against
  the backend nameservers via rndc and monitors zone status.
- **Designate Producer** — handles periodic tasks such as polling for zone
  status and emitting zone update events.
- **BIND9** — the authoritative nameserver(s) managed by Designate. These
  actually serve DNS queries for the zones Designate manages.
- **Unbound** — caching recursive resolver(s) typically used by tenant VMs
  and neutron-managed networks to resolve DNS queries.

### Director-Era Topology

In a Director-based deployment, Designate services (API, Central, mDNS, Worker,
Producer) run on the controller nodes. BIND9 servers also run on the controllers, listening
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

This implementation provides comprehensive support for migrating Designate DNS
service from TripleO to OpenShift-based Red Hat OpenStack (RHOSO) using a hybrid
dual-stack topology. Legacy BIND9 and Unbound servers remain active on the
Director controllers, continuing to serve DNS queries on their original Public
API / External network addresses, while new BIND9 and Unbound pods are deployed
in OpenShift. Legacy mDNS is decommissioned and replaced by OpenShift-hosted
mDNS, which sends zone update notifications to both old and new BIND9 servers.

The goal is **service continuity**: existing delegation records, tenant resolver
configurations, and external DNS integrations all continue to work without
modification during the transition period. The cutover to new addresses happens
gradually on the customer's schedule.

## Key Information Preserved

The migration framework automatically extracts and preserves the following
information from the TripleO deployment:

1. **BIND9 Public API / External Network Addresses**: The listen addresses
   where legacy BIND9 servers serve authoritative DNS queries — these remain
   active throughout the dual-stack period
2. **BIND9 Internalapi Network Addresses**: Used for reconfiguring rndc and
   zone transfer connectivity so that the new OpenShift Worker and mDNS pods
   can manage the legacy BIND9 servers
3. **RNDC Keys**: Extracted from the legacy deployment **and redistributed**
   to the new Worker and mDNS pods in OpenShift, enabling them to issue rndc
   commands and configure zone transfers on the legacy BIND9 servers over the
   internalapi network
4. **NS Records**: Preserved from Designate pools configuration for hybrid
   pool generation
5. **Database Configuration**: Migrated with zone database content

This information is stored in ConfigMaps and Secrets, ensuring administrators
have complete visibility into the migration state.

## Components Implemented

### 1. Enhanced Designate Adoption Role

**Location**: `tests/roles/designate_adoption/`

**Task Files**:
- `tasks/extract_bind_ips.yaml`: Extracts BIND9 IP addresses from TripleO
  (3 discovery methods, covering both Public API and internalapi networks)
- `tasks/extract_bind9_config.yaml`: Extracts RNDC keys and NS records from
  source deployment
- `tasks/preserve_bind_ips.yaml`: Creates ConfigMap with extracted IP mappings
- `tasks/verify_bind_ips.yaml`: Verifies extracted information is correctly
  stored
- `tasks/reconfigure_legacy_bind9.yaml`: Reconfigures legacy BIND9 servers to
  accept rndc commands and zone transfers on the internalapi network
- `tasks/generate_hybrid_pool.yaml`: Generates a pool configuration that
  includes both OpenShift BIND9 pods and legacy BIND9 servers (with
  internalapi addresses)
- `tasks/configure_mdns_targets.yaml`: Configures mDNS NOTIFY targets to
  include both new OpenShift BIND9 pods and legacy BIND9 servers
- `tasks/decommission_legacy.yaml`: Decommissioning workflow — verifies zone
  sync between old and new BIND9 servers, removes legacy servers from pool
  configuration, and shuts down legacy services on Director controllers

**Configuration Files**:
- `defaults/main.yaml`: Comprehensive BIND9 configuration parameters including
  hybrid mode settings
- `tasks/main.yaml`: Enhanced adoption workflow with extraction, legacy
  reconfiguration, hybrid deployment, and verification

**Key Features**:
- Multiple discovery methods for extracting BIND9 IPs (Ansible queries, Heat
  outputs, pools.yaml)
- Automatic RNDC key extraction, storage, and redistribution to OpenShift pods
- NS records extraction from pools.yaml
- Legacy BIND9 reconfiguration for internalapi control plane access
- Hybrid pool configuration generation covering both environments
- mDNS NOTIFY target configuration for dual-stack operation
- Service-by-service health checking (API, Central, Worker, MDNS, Producer,
  BIND9)
- DNS query validation
- Decommissioning workflow with zone sync verification

## Migration Workflow

The migration follows a phased approach that maintains service continuity
throughout:

### 1. Discovery and Extraction

- Query TripleO BIND9 nodes for IP addresses (Public API and internalapi)
- Extract RNDC keys from legacy BIND9 configuration
- Parse NS records and pool configuration from Designate pools.yaml
- Extract from Heat stack outputs
- Store extracted data in ConfigMaps and Secrets

### 2. Legacy Reconfiguration

- Reconfigure legacy BIND9 servers to accept rndc commands on the internalapi
  network (not just localhost or the old control network)
- Reconfigure legacy BIND9 to accept zone transfers from the new mDNS pod
  addresses over internalapi
- Decommission legacy mDNS services on Director controllers
- Verify internalapi connectivity from OpenShift pod network to Director
  controllers

### 3. Hybrid Deployment

- Deploy new BIND9 and Unbound pods in OpenShift
- Generate hybrid pool configuration referencing both OpenShift BIND9 pods and
  legacy BIND9 servers (using internalapi addresses)
- Distribute RNDC keys to new Worker and mDNS pods
- Configure mDNS to send NOTIFY messages to both new and legacy BIND9 servers
- Deploy Designate services (API, Central, Worker, Producer, mDNS) in OpenShift

### 4. Dual-Stack Operation

- Both old and new BIND9 servers are authoritative for the same zones
- The OpenShift Worker sends rndc commands to both new BIND9 pods and legacy
  BIND9 servers (via internalapi)
- The OpenShift mDNS sends NOTIFY messages and serves zone transfers (AXFR) to
  both new BIND9 pods and legacy BIND9 servers (via internalapi)
- External queries are served by legacy BIND9 (on Public API network) and new
  BIND9 (on OpenShift network)
- Tenant VMs continue using legacy Unbound resolvers (on Director controllers)
  or can be pointed at new Unbound pods (on OpenShift network)

### 5. Cutover

- Update external delegation records in parent DNS zones to point at new BIND9
  pod addresses
- Migrate tenant resolver references to new Unbound pod addresses (can be done
  in batches on the customer's schedule)
- Verify that all zones are fully synchronized between old and new BIND9 servers
- Remove legacy BIND9 servers from the Designate pool configuration

### 6. Decommissioning

- Confirm all delegation records and resolver references point to new addresses
- Verify no remaining traffic is hitting legacy BIND9 or Unbound servers
- Shut down legacy BIND9 and Unbound services on Director controllers
- Remove legacy server entries from any remaining configuration

## Usage

### Basic Adoption (Designate without BIND9)

```yaml
# In test playbook vars:
designate_adoption: true
enable_designate: true
enable_designate_bind9: false
```

### Full Adoption with Hybrid Dual-Stack Migration

```yaml
# In test playbook vars:
designate_adoption: true
enable_designate: true
enable_designate_bind9: true

# Hybrid / Maintenance Window configuration
designate_hybrid_mode: true

# Legacy BIND9 internalapi addresses (extracted or manually specified)
designate_legacy_bind9_internalapi_ips:
  - 172.17.1.10
  - 172.17.1.11
  - 172.17.1.12

# Legacy RNDC key reference (extracted from legacy deployment)
designate_legacy_rndc_key_name: "rndc-key"
designate_legacy_rndc_key_secret: "{{ extracted_rndc_secret }}"

# NS records from legacy deployment
designate_ns_records:
  - hostname: ns1.example.com.
    priority: 1
  - hostname: ns2.example.com.
    priority: 2
  - hostname: ns3.example.com.
    priority: 3
```

The adoption role will:
1. Extract BIND9 IPs and RNDC keys from TripleO
2. Reconfigure legacy BIND9 for internalapi control plane access
3. Decommission legacy mDNS on Director controllers
4. Generate hybrid pool configuration with both old and new BIND9 targets
5. Deploy Designate and new BIND9/Unbound in OpenShift
6. Configure mDNS to push zone updates to both old and new BIND9 servers
7. Verify DNS functionality across both environments

## Technical Architecture

### DNS Service Flow (Dual-Stack)

```
External DNS Queries ──────────────────── External DNS Queries
        │                                         │
        ▼                                         ▼
Legacy BIND9 [Directors]              New BIND9 Pods [OpenShift]
(Public API / External network)       (OpenShift network)
        ▲                                         ▲
        │ NOTIFY/AXFR (internalapi)               │ NOTIFY/AXFR (pod network)
        │                                         │
        └──────────── DesignateMDNS ──────────────┘
                     [OpenShift Pod]
                     [Hidden Primary]
                           ▲
                           │ Zone Updates
                     DesignateWorker ─── rndc ───┐
                     [OpenShift Pod]              │
                           │               ┌─────┴──────┐
                           │               │            │
                           │               ▼            ▼
                           │          New BIND9    Legacy BIND9
                           │          (pod net)    (internalapi)
                           │
                     DesignateCentral
                     [OpenShift Pod]
                           ▲
                     DesignateAPI
                     [OpenShift Pod]
                           ▲
              OpenStack Services / Users

Tenant VMs ──► Legacy Unbound [Directors]    (original addresses)
Tenant VMs ──► New Unbound [OpenShift]       (new addresses, post-cutover)
```

## Preserved Information Storage

The migration process creates the following Kubernetes objects to preserve
extracted information:

1. **ConfigMap: designate-bind-ip-map**
   - Stores extracted BIND9 IP addresses (both Public API and internalapi)
   - Maps legacy servers to their network addresses for hybrid pool generation
   - Includes extraction timestamp and source metadata

2. **Secret: designate-legacy-rndc-keys**
   - RNDC keys extracted from legacy BIND9 configuration
   - Distributed to Worker and mDNS pods for managing legacy servers

3. **Verification Script**: `verify_bind9_ips.sh`
   - Tests DNS query responses on both legacy and new BIND9 addresses
   - Validates RNDC port accessibility on internalapi network
   - Verifies zone synchronization between old and new servers
   - Can be run post-deployment and during dual-stack operation

## Benefits of This Implementation

1. **Service Continuity**: Existing DNS resolution remains uninterrupted —
   delegation records, tenant resolver configurations, and external integrations
   continue to work throughout the transition
2. **Gradual Cutover**: Delegation and resolver references are migrated on the
   customer's schedule, in batches if desired, rather than all at once
3. **Automated Extraction**: IPs, keys, and configuration extracted
   automatically from the legacy deployment
4. **Comprehensive Validation**: Multi-phase verification ensures zone data is
   synchronized across both environments before cutover
5. **Production Ready**: Based on proven Designate operator patterns
6. **Well Documented**: Complete procedure with verification guidance for each
   migration phase

## Limitations and Considerations

1. **Network Routing Required**: The OpenShift pod network must have routing to
   the Director internalapi network for rndc and zone transfer traffic to reach
   legacy BIND9 servers
2. **Legacy Servers Outside RHOSO Management**: The legacy BIND9 and Unbound
   servers on Director controllers are not managed by RHOSO deployment tooling —
   any patches, configuration changes, or troubleshooting during the dual-stack
   period must be done manually
3. **Operator Development Required**: The Designate operator must support
   external BIND9 targets in pool configuration — this is a new mode of
   operation that requires design work and testing beyond the current operator
   architecture
4. **Configuration Drift Risk**: During extended dual-stack periods, the risk of
   configuration drift between old and new servers increases — monitoring and
   periodic sync verification are recommended
5. **Zone Sync Verification Before Decommissioning**: All zones must be verified
   as fully synchronized to the new BIND9 servers before legacy servers are
   removed from the pool
6. **Stateful Services**: BIND9 uses StatefulSets with persistent storage
7. **External Dependency**: External DNS infrastructure (parent zone delegation)
   remains managed separately

## Testing

All components have been designed with testability in mind:

- **Extraction Phase**: Each extraction method can be tested independently
- **Legacy Reconfiguration**: Internalapi connectivity and rndc access can be
  validated before proceeding to hybrid deployment
- **Hybrid Deployment**: Full playbook tests end-to-end adoption workflow
  including hybrid pool configuration
- **Dual-Stack Validation**: Zone sync verification between old and new BIND9
  servers
- **Decommissioning**: Verification scripts confirm all zones are synced and
  all references updated before legacy server removal
- **DNS Verification**: Comprehensive DNS query and RNDC connectivity tests
  across both environments

## References

- OpenStack Designate Documentation: https://docs.openstack.org/designate/
- Designate Operator: https://github.com/openstack-k8s-operators/designate-operator
- BIND9 Documentation: https://bind9.readthedocs.io/
- DNS Protocol (RFC 1035): https://tools.ietf.org/html/rfc1035
- AXFR Zone Transfers (RFC 5936): https://tools.ietf.org/html/rfc5936

<!-- vim: set tw=80 ai formatoptions+=a : -->
