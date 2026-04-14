## Component Inventory & Ownership

The following table defines responsibility, runtime entrypoints, and ownership for all major components in this repository.

| Component | Path | Runtime Entrypoint | Owner | Backup Reviewer | CI Ownership |
|----------|------|-------------------|-------|-----------------|--------------|
| Connection Tracker Service | `scripts/connTrac.go` | Go binary (`main.go`) via auditd socket `/var/run/audispd_events` | NCT Core Team | Platform Team | Product CI (build, test, release) |
| Audit Rules | `rules/network.rules` | Loaded via auditd (`auditctl`) | Security Engineering | NCT Core Team | Product CI (rules validation) |
| Systemd Service | `SOURCES/conntrac.service` | `systemd` unit (`conntrac.service`) | Platform Team | SRE Team | Product CI (service validation) |
| RPM Packaging | `SPECS/cisco-connection-tracker.spec` | `rpmbuild -bb` | Release Engineering | Platform Team | Product CI (RPM build pipeline) |
| Source Assets | `SOURCES/` | Used during RPM build | Release Engineering | NCT Core Team | Product CI (packaging integrity) |
| Tests | `tests/` | `go test ./...` | NCT Core Team | QA Team | Product CI (test pipeline) |
| CI Pipeline | `Jenkinsfile` | Jenkins pipeline execution | DevOps Team | Platform Team | DevOps CI ownership |
| Docker Test Build | `Dockerfile.test` | Docker build/test runtime | DevOps Team | QA Team | DevOps CI ownership |
| Network Segmentation Analysis | (post-processing scripts / external) | CSV + JSON processing pipeline | Security Analytics | Data Engineering | Supporting CI (non-blocking) |
| CSV Export | (Splunk / pipeline output) | Export from Splunk queries | Security Analytics | NCT Core Team | Supporting CI (non-blocking) |
| Tenant Automation Script | `apm-tenant-creation-script/` | Script execution | Platform Team | DevOps Team | Supporting CI (non-blocking) |

> See [ADR-001: Repository classification](docs/adr/ADR-001-repository-classification.md) for how components are categorized.