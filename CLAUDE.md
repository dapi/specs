# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**ВАЖНО: Всегда отвечайте на русском языке.**

## Repository Overview

This is the **Site Reliability Handbook** repository - a comprehensive collection of specifications, patterns, guides, and operational practices for building reliable, scalable applications and services. The repository contains no executable code, only markdown documentation.

## Document Types

The repository uses four document type prefixes:

| Prefix | Type | Description |
|--------|------|-------------|
| **SRS** | Specifications | Technical requirements that software must conform to |
| **SRP** | Patterns | Architectural patterns and design solutions |
| **SRG** | Guides | How-to documentation and implementation guides |
| **SRO** | Operations | Operational practices and processes |

## Document Format

- Each document follows the naming pattern: `{PREFIX}-XXX Title.md`
- Documents support versioning: `{PREFIX}-XXX-Y` where Y is the version number
- Status workflow: DRAFT → PROPOSED → APPROVED → DEPRECATED
- Written in Russian and English

## Multilingual Documentation Rules

### File Naming Conventions
- **English documents**: `{PREFIX}-XXX Title.md` (default)
- **Russian documents**: `{PREFIX}-XXX Title.ru.md` (with `.ru.md` extension)
- **English README**: `README.md`
- **Russian README**: `README.ru.md`

### Link Rules
- **README.md (English)** must link to English documents (`.md` files)
- **README.ru.md (Russian)** must link to Russian documents (`.ru.md` files)
- Each README should include a language switcher link at the top:
  - English README: `:ru: [Русская версия](README.ru.md)`
  - Russian README: `:gb: [English version](README.md)`

### Creating Bilingual Documents
1. Create the English version first: `{PREFIX}-XXX Title.md`
2. Create the Russian version: `{PREFIX}-XXX Title.ru.md`
3. Both files should have the same structure but in respective languages
4. Update both README files with links to the appropriate language version

### Updating Documents
When updating content:
- Update both language versions to maintain consistency
- Ensure links in both README files point to correct language versions

## Key Documents

The repository contains 57 documents:

### SRS - Specifications (20)
Technical requirements: Stateless Services, Environment Variables, Liveness Probes, Circuit Breaker, Graceful Shutdown, Timeouts, Idempotency, Rate Limiting, Authentication, Authorization

### SRP - Patterns (9)
Architectural patterns: Jobs Management, Scaling, Load Shedding, Caching, Load Balancing, Auto-scaling, API Gateway, Service Mesh, Materialized Views

### SRG - Guides (19)
Implementation guides: Metrics, Logging, Error Tracking, Tracing, Alerting, Audit Logging, SLI/SLO/SLA, Database Operations, Feature Flags, Security Monitoring

### SRO - Operations (9)
Operational practices: On-Call, Chaos Engineering, FinOps, DR, Capacity Planning, Platform Engineering, Database HA, Maintenance, Security

## Common Tasks

### Creating a New Document
1. Determine the correct prefix (SRS/SRP/SRG/SRO)
2. Create a new file following the naming pattern: `{PREFIX}-XXX Title.md`
3. Start with status DRAFT
4. Create both English and Russian versions

### Updating Documents
- Create a new version by adding version number: `{PREFIX}-XXX-Y Title.md`
- Maintain backward compatibility notes in the document
- Update both README files with any new links

### Reviewing Documents
- Check for consistency with existing documents
- Ensure proper status labeling (DRAFT/PROPOSED/APPROVED/DEPRECATED)
- Verify links between related documents are correct

## Important Notes

- This is a documentation-only repository - no build or test commands needed
- Documents are available in both Russian and English
- All documents should reference related documents when applicable
