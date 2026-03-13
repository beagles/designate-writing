# Designate Adoption Tech Preview

## Introduction

This document describes the RHOSP-to-RHOSO adoption scenario supported as tech
preview and sets expectations for what it does and does not handle. The tech
preview covers what the GA adoption document calls the "CI Friendly Scenario": a
clean, hard cutover of all Designate data and services from the Director-based
deployment to RHOSO, with no attempt to maintain continuity for existing DNS
clients or external DNS integrations during the transition.

This is the baseline adoption scenario. It validates that zone data, recordsets,
and pool configuration can be migrated from the Director-era MariaDB/Galera
cluster to the RHOSO MariaDB instance, and that a freshly deployed RHOSO
Designate can take over API calls and serve DNS queries using that data. It is
the foundation on which more complete adoption approaches will build.

The scenario is called "CI Friendly" because it is well-suited to automated test
environments where a clean environment can be set up and torn down, and where
disruption to DNS resolution during the cutover is acceptable. It is not designed
for production cutovers where continuity of DNS service must be maintained.


## What Is Provided

### Database Migration

All zone data stored in the Designate database is copied to the RHOSO deployment.
This includes:

- Zones and their associated metadata (TTL, status, tenant/project associations)
- Recordsets and individual records within each zone
- Pool configurations, including the NS record information for each pool

The migration is a data transfer rather than a schema transformation — the
Designate database schema is expected to be compatible between the RHOSP and
RHOSO versions.

### NS Record Extraction and Pool Initialization

The NS records for the default pool are extracted from the source database.
These are used to generate the pool configuration for the RHOSO Designate
deployment. This step is what allows the new RHOSO Designate to understand which
nameservers are authoritative for the zones it manages.

### Zone Population in New BIND9 Instances

Once Designate starts in the RHOSO environment it initializes its services
against the newly deployed BIND9 pods within OpenShift. Designate mDNS sends
NOTIFY messages to the new BIND9 pods and serves zone transfers (AXFR) so that
the zone data from the migrated database is loaded into the authoritative
nameservers.

The Designate operator manages the full lifecycle of these new BIND9 pods:
creating them, configuring rndc keys, setting up zone transfers from mDNS, and
monitoring their health. This is standard operator behavior — no new development
or special modes are needed for this scenario.

### Functional DNS Service on the New System

After migration, the new RHOSO Designate deployment is able to:

- Accept DNS management API calls (creating, updating, and deleting zones and
  records) through the RHOSO Designate API.
- Serve authoritative DNS queries for all migrated zones through the new BIND9
  pods deployed in OpenShift, returning the same results as the old system.

This makes the tech preview suitable for validating migration feasibility,
testing zone data integrity after migration, and using the new deployment as a
development or staging environment.


## Who Might Want This

Operators with Designate enabled in their RHOSP deployments who are evaluating
an upgrade path to RHOSO. The tech preview lets them:

- **Validate migration feasibility** by confirming that their zone data migrates
  correctly and that the new RHOSO Designate can serve it.
- **Run adoption CI pipelines** against a real dataset, which is precisely the
  scenario this path was designed for.
- **Stand up a test or staging environment** in RHOSO that mirrors their
  production DNS data, without needing to immediately solve the harder problems
  of updating DNS client references or external delegation records.

This is not yet suitable as a complete production adoption path. The gaps
described below must be addressed before a production cutover can proceed without
DNS service disruption.


## What Is Not Handled

The following items are explicitly out of scope for the tech preview. They
represent the remaining work required to move from a validated data migration to
a complete, production-grade adoption.

### External DNS Delegation Records

The new BIND9 pods in RHOSO receive IP addresses allocated from the OpenShift
network ranges — these are different from the Public API / External network
addresses used by the Director-era BIND9 servers. Any records in parent DNS
zones that delegate subzones to the old BIND9 server addresses (NS records,
glue records) must be updated to point at the new addresses. Until these are
updated, delegated subzones will be unresolvable from outside the cloud.

Updating registrar information that references the old Designate BIND9 servers
also falls into this category and is not handled.

### Unbound Resolver References in Neutron and Tenant Workloads

The Director-era Unbound resolvers run on the controller nodes with well-known
IP addresses. These addresses are typically propagated to tenant VMs via Neutron
DHCP options or are statically configured in tenant applications. The new RHOSO
Unbound pods have different addresses allocated from the OpenShift network.

The tech preview does not handle:

- Updating Neutron subnet DNS nameserver references to point at the new RHOSO
  Unbound pod addresses.
- Reconfiguring networking so tenant VMs and OpenStack workloads can reach the
  new Unbound resolvers.
- Migrating any Unbound customizations from the legacy deployment — stub zones,
  forwarders, or other configuration that was applied to the Director Unbound
  instances — to the new pods.

Until these are addressed, existing tenant VMs will continue pointing at the
old Unbound resolver addresses. If the Director controllers are decommissioned
without updating these references, DNS resolution will fail for those workloads.

### Bring-Your-Own-BIND Deployments

Designate in RHOSP officially managed BIND9 servers running on the controller
nodes. Some operators may have customized their deployment to integrate external
or self-managed BIND9 servers. These configurations are not accounted for in the
adoption tooling.

### Service Continuity During Cutover

The tech preview is a hard cutover. There is no mechanism for maintaining a
hybrid period where both the old Director-era and new RHOSO Designate
deployments serve DNS simultaneously. If the Director BIND9 servers are
decommissioned before external delegation records and tenant resolver references
are updated, DNS resolution will be disrupted.

The two more complete adoption approaches — the "Half Measure" (which keeps the
legacy Unbound resolvers in place) and the "Maintenance Window Approach" (which
keeps both old BIND9 and Unbound running while the new RHOSO deployment is
brought up alongside them) — are not part of the tech preview. These require
additional operator development and are documented separately in the GA adoption
planning materials.


<!-- vim: set tw=80 ai formatoptions+=a : -->
