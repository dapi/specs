# Migration Guide: SRS → Site Reliability Handbook

This document maps old SRS document IDs to the new categorized structure.

## New Document Categories

| Prefix | Type | Description |
|--------|------|-------------|
| **SRS** | Specifications | Technical requirements that software must conform to |
| **SRP** | Patterns | Architectural patterns and design solutions |
| **SRG** | Guides | How-to documentation and implementation guides |
| **SRO** | Operations | Operational practices and processes |

---

## ID Mapping Table

### SRS - Specifications (20 documents)

| Old ID | New ID | Title |
|--------|--------|-------|
| SRS-002 | SRS-001 | Stateless Services |
| SRS-004 | SRS-002 | Environment Variables |
| SRS-005 | SRS-003 | Application Versioning |
| SRS-007 | SRS-004 | Expose Application Version |
| SRS-010 | SRS-005 | Liveness Probes |
| SRS-021 | SRS-006 | Liveness Probes Command |
| SRS-012 | SRS-007 | Circuit Breaker |
| SRS-014 | SRS-008 | Graceful Shutdown |
| SRS-015 | SRS-009 | Blocking Timeouts |
| SRS-016 | SRS-010 | Request Idempotency |
| SRS-017 | SRS-011 | Deadline Propagation |
| SRS-019 | SRS-012 | Stand-Independent Images |
| SRS-020 | SRS-013 | Retry Pattern |
| SRS-022 | SRS-014 | Fallback |
| SRS-025 | SRS-015 | Bulkhead Pattern |
| SRS-027 | SRS-016 | Rate Limiting |
| SRS-028 | SRS-017 | Database Connection Pooling |
| SRS-029 | SRS-018 | Secrets Management |
| SRS-040 | SRS-019 | Service Authentication |
| SRS-041 | SRS-020 | Authorization Pattern |

### SRP - Patterns (9 documents)

| Old ID | New ID | Title |
|--------|--------|-------|
| SRS-001 | SRP-001 | Jobs Management |
| SRS-003 | SRP-002 | Scaling and State |
| SRS-013 | SRP-003 | Load Shedding |
| SRS-018 | SRP-004 | Distributed Caching |
| SRS-023 | SRP-005 | Load Balancing |
| SRS-024 | SRP-006 | Auto-scaling |
| SRS-038 | SRP-007 | API Gateway |
| SRS-044 | SRP-008 | Service Mesh |
| SRS-062 | SRP-009 | Materialized Views |

### SRG - Guides (19 documents)

| Old ID | New ID | Title |
|--------|--------|-------|
| SRS-006 | SRG-001 | Metrics Collection |
| SRS-008 | SRG-002 | Logging |
| SRS-009 | SRG-003 | Error Tracking |
| SRS-011 | SRG-004 | Distributed Tracing |
| SRS-026 | SRG-005 | Alerting Rules |
| SRS-031 | SRG-006 | Audit Logging |
| SRS-032 | SRG-007 | SLI SLO SLA |
| SRS-033 | SRG-008 | Synthetic Monitoring |
| SRS-035 | SRG-009 | Database Migrations |
| SRS-036 | SRG-010 | Backup Recovery |
| SRS-042 | SRG-011 | Feature Flags |
| SRS-048 | SRG-012 | Security Monitoring |
| SRS-052 | SRG-013 | Database Replication |
| SRS-053 | SRG-014 | Database Monitoring |
| SRS-054 | SRG-015 | Query Optimization |
| SRS-056 | SRG-016 | Database Sharding |
| SRS-057 | SRG-017 | Table Partitioning |
| SRS-058 | SRG-018 | Read Replicas |
| SRS-061 | SRG-019 | Database Performance Tuning |

### SRO - Operations (9 documents)

| Old ID | New ID | Title |
|--------|--------|-------|
| SRS-034 | SRO-001 | On-Call Incident Response |
| SRS-043 | SRO-002 | Chaos Engineering |
| SRS-045 | SRO-003 | Cost Optimization FinOps |
| SRS-046 | SRO-004 | Multi-Region DR |
| SRS-047 | SRO-005 | Capacity Planning |
| SRS-049 | SRO-006 | Platform Engineering |
| SRS-055 | SRO-007 | Database High Availability |
| SRS-059 | SRO-008 | Database Maintenance |
| SRS-060 | SRO-009 | Database Security |

---

## Quick Reference

If you're looking for a document by its old ID:

```
SRS-001 → SRP-001 (Jobs Management - now a Pattern)
SRS-002 → SRS-001 (Stateless Services)
SRS-003 → SRP-002 (Scaling and State - now a Pattern)
SRS-004 → SRS-002 (Environment Variables)
SRS-005 → SRS-003 (Application Versioning)
SRS-006 → SRG-001 (Metrics Collection - now a Guide)
SRS-007 → SRS-004 (Expose Application Version)
SRS-008 → SRG-002 (Logging - now a Guide)
SRS-009 → SRG-003 (Error Tracking - now a Guide)
SRS-010 → SRS-005 (Liveness Probes)
SRS-011 → SRG-004 (Distributed Tracing - now a Guide)
SRS-012 → SRS-007 (Circuit Breaker)
SRS-013 → SRP-003 (Load Shedding - now a Pattern)
SRS-014 → SRS-008 (Graceful Shutdown)
SRS-015 → SRS-009 (Blocking Timeouts)
SRS-016 → SRS-010 (Request Idempotency)
SRS-017 → SRS-011 (Deadline Propagation)
SRS-018 → SRP-004 (Distributed Caching - now a Pattern)
SRS-019 → SRS-012 (Stand-Independent Images)
SRS-020 → SRS-013 (Retry Pattern)
SRS-021 → SRS-006 (Liveness Probes Command)
SRS-022 → SRS-014 (Fallback)
SRS-023 → SRP-005 (Load Balancing - now a Pattern)
SRS-024 → SRP-006 (Auto-scaling - now a Pattern)
SRS-025 → SRS-015 (Bulkhead Pattern)
SRS-026 → SRG-005 (Alerting Rules - now a Guide)
SRS-027 → SRS-016 (Rate Limiting)
SRS-028 → SRS-017 (Database Connection Pooling)
SRS-029 → SRS-018 (Secrets Management)
SRS-031 → SRG-006 (Audit Logging - now a Guide)
SRS-032 → SRG-007 (SLI SLO SLA - now a Guide)
SRS-033 → SRG-008 (Synthetic Monitoring - now a Guide)
SRS-034 → SRO-001 (On-Call Incident Response - now Operations)
SRS-035 → SRG-009 (Database Migrations - now a Guide)
SRS-036 → SRG-010 (Backup Recovery - now a Guide)
SRS-038 → SRP-007 (API Gateway - now a Pattern)
SRS-040 → SRS-019 (Service Authentication)
SRS-041 → SRS-020 (Authorization Pattern)
SRS-042 → SRG-011 (Feature Flags - now a Guide)
SRS-043 → SRO-002 (Chaos Engineering - now Operations)
SRS-044 → SRP-008 (Service Mesh - now a Pattern)
SRS-045 → SRO-003 (Cost Optimization FinOps - now Operations)
SRS-046 → SRO-004 (Multi-Region DR - now Operations)
SRS-047 → SRO-005 (Capacity Planning - now Operations)
SRS-048 → SRG-012 (Security Monitoring - now a Guide)
SRS-049 → SRO-006 (Platform Engineering - now Operations)
SRS-052 → SRG-013 (Database Replication - now a Guide)
SRS-053 → SRG-014 (Database Monitoring - now a Guide)
SRS-054 → SRG-015 (Query Optimization - now a Guide)
SRS-055 → SRO-007 (Database High Availability - now Operations)
SRS-056 → SRG-016 (Database Sharding - now a Guide)
SRS-057 → SRG-017 (Table Partitioning - now a Guide)
SRS-058 → SRG-018 (Read Replicas - now a Guide)
SRS-059 → SRO-008 (Database Maintenance - now Operations)
SRS-060 → SRO-009 (Database Security - now Operations)
SRS-061 → SRG-019 (Database Performance Tuning - now a Guide)
SRS-062 → SRP-009 (Materialized Views - now a Pattern)
```
