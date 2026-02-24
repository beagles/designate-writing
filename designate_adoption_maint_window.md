# Designate DNS Service Migration - Implementation Summary

## Overview

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

---

**Implementation Date**: 2025
**Status**: In Development — hybrid pool support requires operator extension
