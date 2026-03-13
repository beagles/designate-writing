# Designate Adoption Tech Preview

## Introduction

This document describes the RHOSP-to-RHOSO adoption scenario that is supported
as tech preview and provides a best-effort outline of the gaps that need further
development. For tech preview, the focus is on a basic, hard cut over that
validates that designate data can be migrated from RHOSP to RHOSO, with the
RHOSO system able to take over API calls and servicing DNS queries using the
newly deployed services in the RHOSO OpenShift environment. There is no
consideration for maintaining continuity with DNS clients using the data managed
by Designate or propogating changes to OpenStack such as Neutron subnets.

## What Is Currently Supported?

* All zone, recordset, etc data that is stored in the database is copied to the
  new system.

* The NS record information for the default pool is extracted from the source
  database.

* Designate can then be started in the RHOSO environment and using the NS record
  information from the source environment.

* Designate initializes it's services according to the new deployment and
  populates the BIND9 instances with the zone data.

At this point, DNS queries can be performed on the new system that return the same
results as the old system.

## Who Might Want This?

Administrators with Designate enabled in the RHOSP deployments that are
considering upgrading to RHOSO. With the adoption basics they can stand up a
RHOSO deployment with their existing data as a test environment.

## What is Not Handled?

* Glue records with DNS servers for subzone delegation.

* Updating registrar information that may reference the Designate BIND9 servers.

* Unmanaged BIND9 deployments (aka Bring-Your-Own-BIND, this was not directly
  supported by RHOSP but it is possible that some customers may have customized
  their deployment to achieve this).

* Maintaining or updating references to the Unbound resolver in Neutron subnets
  and OpenStack workloads.

* If decomissioning legacy BIND9 or Unbound resolver deployments is impractical
  or impossible, it may be possible to maintain a hybrid deployment with the new
  and old BIND9 and Unbound instances co-existing for an extended maintenance
  window until all updates can be propogated. This is also not currently
  handled.

* Reconfiguring networking so OpenStack workloads can access the RHOSO Unbound
resolvers.

* Migrating Unbound customizations such as stub zones, forwarders, etc.


<!-- vim: set tw=80 ai formatoptions+=a : -->

