# ADR-001: Repository classification for Cisco Connection Tracker

## Status
Accepted

## Context
This repository contains the source and packaging assets for **Cisco Connection Tracker**, a Linux auditd-based network connection monitoring system written in Go.

The system captures real-time inbound and outbound network endpoints through audit socket parsing and is intended for enterprise security monitoring and SIEM integration.

The workflow consists of:
1. automated deployment
2. monitoring
3. data collection
4. CSV export
5. reporting and network segmentation analysis

Key repository components include:
- `scripts/connTrac.go` (core service binary)
- `rules/network.rules` (auditd configuration)
- `SPECS/cisco-connection-tracker.spec` (RPM packaging)
- `SOURCES/conntrac.service` (systemd service)

The repository also contains supporting automation and reporting functionality.

A decision is required to determine whether this repository is:
- a multi-product monorepo, or
- a primary product with auxiliary/supporting projects

## Decision
This repository is classified as a:

> **Primary product repository with auxiliary automation/supporting projects**

## Rationale
The repository is centered around a single primary deliverable:

- one core Go service responsible for connection tracking
- one RPM packaging and distribution model
- one systemd-managed runtime
- one audit rule configuration set

All major components contribute to the lifecycle of this single product.

Supporting functionality (CSV export, enrichment, segmentation analysis, automation scripts):
- extends the product workflow
- depends on the primary data output
- is not independently versioned or released

The repository does not exhibit characteristics of a multi-product monorepo such as:
- multiple independently versioned products
- separate release pipelines
- distinct product ownership boundaries
- multiple top-level deliverables of equal standing

## Tradeoffs

### Benefits
- Clear ownership and accountability
- Simplified documentation and onboarding
- Focused CI and release pipelines
- Accurate reflection of current architecture

### Drawbacks
- Supporting tooling may grow and require future separation
- Contributors may assume equal importance across all components unless documented
- Future scaling may require restructuring

## CI Impacts

### Primary product (release-blocking)
- Go build and tests for connection tracker
- audit rule validation
- RPM build verification
- systemd service validation

### Supporting components (non-blocking unless scoped)
- CSV export scripts
- enrichment/analysis tooling
- reporting utilities
- deployment/tenant automation

CI pipelines should clearly distinguish between:
- product-critical checks
- supporting/tooling checks

## Release Ownership

### Primary product
Owned by the Cisco Connection Tracker team:
- versioning and releases
- RPM distribution
- runtime behavior
- audit rule compatibility
- release notes

### Supporting components
- must have defined maintainers
- do not imply independent release lifecycle
- do not automatically gate product releases

## Consequences
- Documentation must clearly distinguish product vs supporting components
- README must reference this ADR
- CI pipelines must reflect ownership boundaries
- Release processes remain centered on the primary product

## Revisit Conditions
Revisit this decision if:
- supporting components gain independent versioning
- separate release pipelines are introduced
- ownership diverges across components
- multiple first-class products emerge in the repository
