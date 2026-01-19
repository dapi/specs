# SRO-006 Platform Engineering

| Field | Value |
|-------|-------|
| **Status** | APPROVED |
| **Version** | 1.0 |
| **Priority** | P2 |
| **Complexity** | High |
| **Target Audience** | Platform Engineers, DevOps, SRE, Architects |

## 1. Introduction

### 1.1 Purpose

This specification defines standards for building Internal Developer Platforms (IDP) that enable self-service infrastructure, improve developer experience, and accelerate software delivery. Platform Engineering treats the platform as a product, providing golden paths for common workflows while maintaining flexibility for advanced use cases.

### 1.2 Scope

This specification covers:
- Internal Developer Platform architecture
- Developer Portals (Backstage, Port, Cortex)
- Service Catalog and software templates
- Golden Paths and paved roads
- Self-service infrastructure provisioning
- Developer Experience (DevEx) metrics
- Platform as a Product principles

### 1.3 Related Specifications

- [SRP-008 Service Mesh](SRP-008%20Service%20Mesh.md) - Service networking
- GitOps (planned) - GitOps practices
- [SRG-007 SLI SLO SLA](SRG-007%20SLI%20SLO%20SLA.md) - Platform SLOs
- [SRO-001 On-Call & Incident Response](SRO-001%20On-Call%20Incident%20Response.md) - Platform support

## 2. Platform Engineering Fundamentals

### 2.1 What is Platform Engineering?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Developer Experience                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Developer Portal (Backstage)                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │  Software   │  │   Golden    │  │    Self-Service         │  │   │
│  │  │  Catalog    │  │   Paths     │  │    Workflows            │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Internal Developer Platform (IDP)                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │  Infrastructure │  │    Security     │  │     Observability       │ │
│  │  Orchestration  │  │    Controls     │  │     Integration         │ │
│  │  (Crossplane,   │  │  (OPA, Kyverno) │  │  (Prometheus, Grafana)  │ │
│  │   Terraform)    │  │                 │  │                         │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │   CI/CD        │  │   Secrets       │  │     Service Mesh        │ │
│  │   Pipelines    │  │   Management    │  │     (Istio/Linkerd)     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Infrastructure Layer                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │   Kubernetes    │  │  Cloud Provider │  │    Databases            │ │
│  │   Clusters      │  │  (AWS/GCP/Azure)│  │    (RDS, CloudSQL)      │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Platform Engineering vs Traditional DevOps

| Aspect | Traditional DevOps | Platform Engineering |
|--------|-------------------|---------------------|
| **Focus** | Processes and culture | Products and self-service |
| **Delivery** | Per-team tooling | Centralized platform |
| **Approach** | "You build it, you run it" | "Golden paths with guardrails" |
| **Cognitive Load** | High (full stack ownership) | Reduced (abstracted complexity) |
| **Scaling** | Linear (per team) | Exponential (platform leverage) |
| **Standardization** | Varies by team | Consistent across organization |

### 2.3 Platform Team Structure

```yaml
# Platform Team Organization
platform_team:
  structure:
    platform_product_manager:
      responsibilities:
        - Define platform roadmap
        - Gather developer feedback
        - Prioritize features
        - Measure platform adoption

    platform_engineers:
      responsibilities:
        - Build platform components
        - Maintain infrastructure
        - Create golden paths
        - Write documentation
      skills:
        - Kubernetes
        - Infrastructure as Code
        - CI/CD pipelines
        - Cloud providers

    developer_advocates:
      responsibilities:
        - Onboard teams to platform
        - Create tutorials and guides
        - Collect feedback
        - Run office hours

  operating_model:
    support_model: "tiered"
    tiers:
      tier_0: "Self-service documentation and templates"
      tier_1: "Slack channel support (async)"
      tier_2: "Office hours (scheduled)"
      tier_3: "Dedicated engagement (complex cases)"

    sla:
      tier_1_response: "4 hours"
      tier_2_response: "24 hours"
      critical_issues: "1 hour"
```

## 3. Developer Portal (Backstage)

### 3.1 Backstage Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Backstage                                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │  Software   │ │  TechDocs   │ │   Search    │ │  Plugins  │ │
│  │  Catalog    │ │             │ │             │ │           │ │
│  └─────────────┘ └─────────────┘ └─────────────┘ └───────────┘ │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    Core API / Backend                        ││
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────────┐ ││
│  │  │ Catalog  │ │ Scaffolder│ │ Auth     │ │ Permissions   │ ││
│  │  │ API      │ │ API       │ │ API      │ │ API           │ ││
│  │  └──────────┘ └──────────┘ └──────────┘ └────────────────┘ ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   GitHub/GitLab │  │   Kubernetes    │  │   PagerDuty     │
│   (SCM)         │  │   (Runtime)     │  │   (On-call)     │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

### 3.2 Backstage Deployment

```yaml
# Backstage Kubernetes Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backstage
  namespace: platform
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backstage
  template:
    metadata:
      labels:
        app: backstage
    spec:
      serviceAccountName: backstage
      containers:
      - name: backstage
        image: backstage/backstage:latest
        ports:
        - containerPort: 7007
        env:
        - name: POSTGRES_HOST
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: postgres-host
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: postgres-user
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: postgres-password
        - name: GITHUB_TOKEN
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: github-token
        - name: AUTH_GITHUB_CLIENT_ID
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: github-client-id
        - name: AUTH_GITHUB_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: backstage-secrets
              key: github-client-secret
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        readinessProbe:
          httpGet:
            path: /healthcheck
            port: 7007
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /healthcheck
            port: 7007
          initialDelaySeconds: 60
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: backstage
  namespace: platform
spec:
  selector:
    app: backstage
  ports:
  - port: 80
    targetPort: 7007
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: backstage
  namespace: platform
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - backstage.company.com
    secretName: backstage-tls
  rules:
  - host: backstage.company.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: backstage
            port:
              number: 80
```

### 3.3 Backstage Configuration

```typescript
// app-config.yaml equivalent in TypeScript
// packages/backend/src/index.ts

import { createBackend } from '@backstage/backend-defaults';

const backend = createBackend();

// Core plugins
backend.add(import('@backstage/plugin-app-backend/alpha'));
backend.add(import('@backstage/plugin-catalog-backend/alpha'));
backend.add(import('@backstage/plugin-scaffolder-backend/alpha'));
backend.add(import('@backstage/plugin-techdocs-backend/alpha'));
backend.add(import('@backstage/plugin-search-backend/alpha'));

// Auth
backend.add(import('@backstage/plugin-auth-backend'));
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));

// Kubernetes integration
backend.add(import('@backstage/plugin-kubernetes-backend/alpha'));

// GitHub integration
backend.add(import('@backstage/plugin-catalog-backend-module-github/alpha'));

backend.start();
```

```yaml
# app-config.yaml
app:
  title: Internal Developer Portal
  baseUrl: https://backstage.company.com

organization:
  name: MyCompany

backend:
  baseUrl: https://backstage.company.com
  listen:
    port: 7007
  database:
    client: pg
    connection:
      host: ${POSTGRES_HOST}
      port: 5432
      user: ${POSTGRES_USER}
      password: ${POSTGRES_PASSWORD}

integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}

auth:
  environment: production
  providers:
    github:
      production:
        clientId: ${AUTH_GITHUB_CLIENT_ID}
        clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
        signIn:
          resolvers:
            - resolver: usernameMatchingUserEntityName

catalog:
  import:
    entityFilename: catalog-info.yaml
    pullRequestBranchName: backstage-integration
  rules:
    - allow: [Component, System, API, Resource, Location, Template, User, Group]
  locations:
    # GitHub organization discovery
    - type: github-discovery
      target: https://github.com/mycompany
    # Templates location
    - type: url
      target: https://github.com/mycompany/backstage-templates/blob/main/all-templates.yaml

kubernetes:
  serviceLocatorMethod:
    type: multiTenant
  clusterLocatorMethods:
    - type: config
      clusters:
        - name: production
          url: https://k8s-prod.company.com
          authProvider: serviceAccount
          serviceAccountToken: ${K8S_PROD_TOKEN}
        - name: staging
          url: https://k8s-staging.company.com
          authProvider: serviceAccount
          serviceAccountToken: ${K8S_STAGING_TOKEN}

techdocs:
  builder: external
  generator:
    runIn: docker
  publisher:
    type: awsS3
    awsS3:
      bucketName: backstage-techdocs
      region: us-east-1
```

## 4. Software Catalog

### 4.1 Catalog Entity Model

```yaml
# System Definition
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: payment-system
  description: Handles all payment processing
  annotations:
    backstage.io/techdocs-ref: dir:.
  tags:
    - payments
    - critical
  links:
    - url: https://wiki.company.com/payments
      title: Payment System Wiki
spec:
  owner: team-payments
  domain: commerce

---
# Component (Service) Definition
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: Core payment processing service
  annotations:
    backstage.io/techdocs-ref: dir:.
    github.com/project-slug: mycompany/payment-service
    pagerduty.com/integration-key: ${PAGERDUTY_INTEGRATION_KEY}
    prometheus.io/alert-name: payment-service-alerts
    backstage.io/kubernetes-id: payment-service
    backstage.io/kubernetes-namespace: payments
    backstage.io/kubernetes-label-selector: app=payment-service
  tags:
    - go
    - grpc
    - tier-1
  links:
    - url: https://grafana.company.com/d/payment-service
      title: Grafana Dashboard
      icon: dashboard
    - url: https://runbooks.company.com/payment-service
      title: Runbook
      icon: docs
spec:
  type: service
  lifecycle: production
  owner: team-payments
  system: payment-system
  dependsOn:
    - resource:default/postgres-payments
    - component:default/fraud-detection-service
  providesApis:
    - payment-api

---
# API Definition
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payment-api
  description: Payment processing API
  tags:
    - rest
    - openapi
spec:
  type: openapi
  lifecycle: production
  owner: team-payments
  system: payment-system
  definition:
    $text: https://github.com/mycompany/payment-service/blob/main/api/openapi.yaml

---
# Resource (Database) Definition
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: postgres-payments
  description: PostgreSQL database for payment data
  annotations:
    backstage.io/managed-by-location: url:https://github.com/mycompany/infrastructure/blob/main/databases/payments.yaml
spec:
  type: database
  owner: team-platform
  system: payment-system
  dependencyOf:
    - component:default/payment-service

---
# Team (Group) Definition
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: team-payments
  description: Payment processing team
spec:
  type: team
  profile:
    displayName: Payment Team
    email: payments-team@company.com
    picture: https://avatars.company.com/teams/payments.png
  parent: engineering
  children: []
  members:
    - john.doe
    - jane.smith
    - bob.wilson

---
# Domain Definition
apiVersion: backstage.io/v1alpha1
kind: Domain
metadata:
  name: commerce
  description: All commerce-related systems
spec:
  owner: commerce-leadership
```

### 4.2 Catalog Processor for Auto-Discovery

```typescript
// Custom catalog processor for auto-discovery
import {
  CatalogProcessor,
  CatalogProcessorEmit,
  processingResult,
} from '@backstage/plugin-catalog-node';
import { LocationSpec } from '@backstage/plugin-catalog-common';
import { Entity } from '@backstage/catalog-model';

export class KubernetesServiceDiscoveryProcessor implements CatalogProcessor {
  getProcessorName(): string {
    return 'KubernetesServiceDiscoveryProcessor';
  }

  async readLocation(
    location: LocationSpec,
    _optional: boolean,
    emit: CatalogProcessorEmit,
  ): Promise<boolean> {
    if (location.type !== 'kubernetes-service-discovery') {
      return false;
    }

    // Discover services from Kubernetes
    const services = await this.discoverKubernetesServices(location.target);

    for (const service of services) {
      const entity: Entity = {
        apiVersion: 'backstage.io/v1alpha1',
        kind: 'Component',
        metadata: {
          name: service.name,
          namespace: 'default',
          description: `Auto-discovered from Kubernetes: ${service.namespace}/${service.name}`,
          annotations: {
            'backstage.io/kubernetes-id': service.name,
            'backstage.io/kubernetes-namespace': service.namespace,
            'backstage.io/managed-by-location': `kubernetes-service-discovery:${location.target}`,
          },
          labels: service.labels,
        },
        spec: {
          type: 'service',
          lifecycle: this.inferLifecycle(service.namespace),
          owner: this.inferOwner(service.labels),
        },
      };

      emit(processingResult.entity(location, entity));
    }

    return true;
  }

  private inferLifecycle(namespace: string): string {
    if (namespace.includes('prod')) return 'production';
    if (namespace.includes('staging')) return 'experimental';
    return 'development';
  }

  private inferOwner(labels: Record<string, string>): string {
    return labels['team'] || labels['owner'] || 'team-platform';
  }

  private async discoverKubernetesServices(cluster: string) {
    // Implementation to query Kubernetes API
    // Returns array of service metadata
    return [];
  }
}
```

### 4.3 Catalog Scorecard

```yaml
# Scorecard rules for catalog entities
scorecards:
  production-readiness:
    title: Production Readiness
    description: Checks if a component is ready for production
    checks:
      - id: has-owner
        title: Has Owner
        description: Component must have an owner assigned
        rule:
          factRef: metadata.owner
          operator: isNotEmpty
        weight: 10

      - id: has-description
        title: Has Description
        description: Component must have a description
        rule:
          factRef: metadata.description
          operator: minLength
          value: 20
        weight: 5

      - id: has-techdocs
        title: Has TechDocs
        description: Component must have documentation
        rule:
          factRef: metadata.annotations['backstage.io/techdocs-ref']
          operator: isNotEmpty
        weight: 10

      - id: has-pagerduty
        title: Has PagerDuty Integration
        description: Production components must have PagerDuty configured
        rule:
          factRef: metadata.annotations['pagerduty.com/integration-key']
          operator: isNotEmpty
        weight: 15
        condition:
          factRef: spec.lifecycle
          operator: equals
          value: production

      - id: has-runbook
        title: Has Runbook
        description: Production components must have a runbook
        rule:
          factRef: metadata.links
          operator: contains
          field: title
          value: Runbook
        weight: 15

      - id: has-dashboard
        title: Has Monitoring Dashboard
        description: Component must have a monitoring dashboard
        rule:
          factRef: metadata.links
          operator: contains
          field: title
          value: Dashboard
        weight: 10

      - id: ci-cd-configured
        title: CI/CD Configured
        description: Component must have CI/CD pipeline
        rule:
          factRef: metadata.annotations['github.com/project-slug']
          operator: isNotEmpty
        weight: 10

  security-compliance:
    title: Security Compliance
    description: Security requirements for components
    checks:
      - id: has-security-contact
        title: Has Security Contact
        rule:
          factRef: metadata.annotations['security.company.com/contact']
          operator: isNotEmpty
        weight: 20

      - id: vulnerability-scanning
        title: Vulnerability Scanning Enabled
        rule:
          factRef: metadata.annotations['security.company.com/vuln-scanning']
          operator: equals
          value: "enabled"
        weight: 25
```

## 5. Service Templates (Scaffolder)

### 5.1 Golden Path Template Structure

```
templates/
├── backend-service/
│   ├── template.yaml           # Template definition
│   ├── skeleton/               # Files to scaffold
│   │   ├── catalog-info.yaml
│   │   ├── Dockerfile
│   │   ├── Makefile
│   │   ├── .github/
│   │   │   └── workflows/
│   │   │       └── ci.yaml
│   │   ├── kubernetes/
│   │   │   ├── deployment.yaml
│   │   │   └── service.yaml
│   │   ├── src/
│   │   │   └── main.go
│   │   └── docs/
│   │       └── index.md
│   └── docs/
│       └── index.md
├── frontend-app/
│   └── ...
├── data-pipeline/
│   └── ...
└── all-templates.yaml          # Template index
```

### 5.2 Backend Service Template

```yaml
# templates/backend-service/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: backend-service-template
  title: Backend Service (Go)
  description: Create a new Go backend service with all platform integrations
  tags:
    - go
    - backend
    - recommended
spec:
  owner: team-platform
  type: service

  parameters:
    - title: Service Information
      required:
        - name
        - description
        - owner
      properties:
        name:
          title: Service Name
          type: string
          description: Name of the service (lowercase, hyphens allowed)
          pattern: '^[a-z][a-z0-9-]*[a-z0-9]$'
          ui:autofocus: true
        description:
          title: Description
          type: string
          description: A brief description of what this service does
        owner:
          title: Owner
          type: string
          description: Team that owns this service
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group
              spec.type: team

    - title: Technical Configuration
      required:
        - system
        - tier
      properties:
        system:
          title: System
          type: string
          description: System this service belongs to
          ui:field: EntityPicker
          ui:options:
            catalogFilter:
              kind: System
        tier:
          title: Service Tier
          type: string
          description: Criticality tier (affects SLO requirements)
          enum:
            - tier-1
            - tier-2
            - tier-3
          enumNames:
            - 'Tier 1 (99.99% SLO, critical)'
            - 'Tier 2 (99.9% SLO, important)'
            - 'Tier 3 (99% SLO, standard)'
          default: tier-2
        hasDatabase:
          title: Requires Database
          type: boolean
          default: false
        databaseType:
          title: Database Type
          type: string
          enum:
            - postgresql
            - mysql
            - mongodb
          ui:widget: select
          ui:options:
            visible:
              hasDatabase: true

    - title: Repository Configuration
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com
            allowedOwners:
              - mycompany

  steps:
    # Fetch base template
    - id: fetch-base
      name: Fetch Base Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          description: ${{ parameters.description }}
          owner: ${{ parameters.owner }}
          system: ${{ parameters.system }}
          tier: ${{ parameters.tier }}
          hasDatabase: ${{ parameters.hasDatabase }}
          databaseType: ${{ parameters.databaseType }}

    # Create GitHub repository
    - id: publish
      name: Publish to GitHub
      action: publish:github
      input:
        allowedHosts: ['github.com']
        description: ${{ parameters.description }}
        repoUrl: ${{ parameters.repoUrl }}
        defaultBranch: main
        protectDefaultBranch: true
        requireCodeOwnerReviews: true
        requiredStatusCheckContexts:
          - ci/build
          - ci/test
          - security/scan
        repoVisibility: internal
        topics:
          - ${{ parameters.tier }}
          - backend
          - go

    # Register in catalog
    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

    # Provision infrastructure
    - id: provision-infra
      name: Provision Infrastructure
      action: http:backstage:request
      input:
        method: POST
        path: /api/crossplane/provision
        headers:
          Content-Type: application/json
        body:
          serviceName: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          tier: ${{ parameters.tier }}
          hasDatabase: ${{ parameters.hasDatabase }}
          databaseType: ${{ parameters.databaseType }}

    # Create PagerDuty service
    - id: create-pagerduty
      name: Create PagerDuty Service
      action: pagerduty:service:create
      input:
        name: ${{ parameters.name }}
        description: ${{ parameters.description }}
        escalationPolicyId: ${{ parameters.tier === 'tier-1' ? 'P1_POLICY' : 'STANDARD_POLICY' }}

  output:
    links:
      - title: Repository
        url: ${{ steps['publish'].output.remoteUrl }}
      - title: Open in Catalog
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}
      - title: PagerDuty Service
        url: ${{ steps['create-pagerduty'].output.serviceUrl }}
```

### 5.3 Template Skeleton Files

```yaml
# skeleton/catalog-info.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{ values.name }}
  description: ${{ values.description }}
  annotations:
    github.com/project-slug: mycompany/${{ values.name }}
    backstage.io/techdocs-ref: dir:.
    backstage.io/kubernetes-id: ${{ values.name }}
    backstage.io/kubernetes-namespace: ${{ values.name }}
    pagerduty.com/integration-key: ${PAGERDUTY_KEY}
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
  tags:
    - go
    - ${{ values.tier }}
  links:
    - url: https://grafana.company.com/d/${{ values.name }}
      title: Grafana Dashboard
      icon: dashboard
    - url: https://runbooks.company.com/${{ values.name }}
      title: Runbook
      icon: docs
spec:
  type: service
  lifecycle: production
  owner: ${{ values.owner }}
  system: ${{ values.system }}
  {%- if values.hasDatabase %}
  dependsOn:
    - resource:default/${{ values.name }}-db
  {%- endif %}
  providesApis:
    - ${{ values.name }}-api
```

```dockerfile
# skeleton/Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main ./cmd/server

FROM gcr.io/distroless/static:nonroot

COPY --from=builder /app/main /main

USER nonroot:nonroot
EXPOSE 8080 9090

ENTRYPOINT ["/main"]
```

```go
// skeleton/src/main.go
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/prometheus/client_golang/prometheus/promhttp"
)

func main() {
	logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
		Level: slog.LevelInfo,
	}))
	slog.SetDefault(logger)

	// Health and metrics server
	healthMux := http.NewServeMux()
	healthMux.HandleFunc("/healthz", healthHandler)
	healthMux.HandleFunc("/readyz", readyHandler)
	healthMux.Handle("/metrics", promhttp.Handler())

	healthServer := &http.Server{
		Addr:    ":9090",
		Handler: healthMux,
	}

	// Main application server
	appMux := http.NewServeMux()
	appMux.HandleFunc("/", rootHandler)

	appServer := &http.Server{
		Addr:    ":8080",
		Handler: appMux,
	}

	// Start servers
	go func() {
		slog.Info("starting health server", "addr", ":9090")
		if err := healthServer.ListenAndServe(); err != http.ErrServerClosed {
			slog.Error("health server error", "error", err)
		}
	}()

	go func() {
		slog.Info("starting application server", "addr", ":8080")
		if err := appServer.ListenAndServe(); err != http.ErrServerClosed {
			slog.Error("application server error", "error", err)
		}
	}()

	// Graceful shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	slog.Info("shutting down servers")
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := appServer.Shutdown(ctx); err != nil {
		slog.Error("app server shutdown error", "error", err)
	}
	if err := healthServer.Shutdown(ctx); err != nil {
		slog.Error("health server shutdown error", "error", err)
	}
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ok"))
}

func readyHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ready"))
}

func rootHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.Write([]byte(`{"service": "${{ values.name }}", "status": "ok"}`))
}
```

```yaml
# skeleton/.github/workflows/ci.yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ "{{" }} github.repository {{ "}}" }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Test
        run: go test -v -race -coverprofile=coverage.out ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.out

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v3
        with:
          version: latest

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'

  build:
    needs: [test, lint, security]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ "{{" }} env.REGISTRY {{ "}}" }}
          username: ${{ "{{" }} github.actor {{ "}}" }}
          password: ${{ "{{" }} secrets.GITHUB_TOKEN {{ "}}" }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ "{{" }} github.event_name != 'pull_request' {{ "}}" }}
          tags: |
            ${{ "{{" }} env.REGISTRY {{ "}}" }}/${{ "{{" }} env.IMAGE_NAME {{ "}}" }}:${{ "{{" }} github.sha {{ "}}" }}
            ${{ "{{" }} env.REGISTRY {{ "}}" }}/${{ "{{" }} env.IMAGE_NAME {{ "}}" }}:latest
```

## 6. Self-Service Infrastructure

### 6.1 Crossplane for Infrastructure Provisioning

```yaml
# Crossplane CompositeResourceDefinition
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xserviceinfras.platform.company.com
spec:
  group: platform.company.com
  names:
    kind: XServiceInfra
    plural: xserviceinfras
  claimNames:
    kind: ServiceInfra
    plural: serviceinfras
  versions:
    - name: v1alpha1
      served: true
      referenceable: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                serviceName:
                  type: string
                  description: Name of the service
                tier:
                  type: string
                  enum: [tier-1, tier-2, tier-3]
                  description: Service tier for resource sizing
                database:
                  type: object
                  properties:
                    enabled:
                      type: boolean
                    type:
                      type: string
                      enum: [postgresql, mysql]
                    size:
                      type: string
                      enum: [small, medium, large]
              required:
                - serviceName
                - tier

---
# Crossplane Composition
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: service-infra-aws
  labels:
    provider: aws
spec:
  compositeTypeRef:
    apiVersion: platform.company.com/v1alpha1
    kind: XServiceInfra

  patchSets:
    - name: common-tags
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.serviceName
          toFieldPath: spec.forProvider.tags[0].value
        - type: FromCompositeFieldPath
          fromFieldPath: spec.tier
          toFieldPath: spec.forProvider.tags[1].value

  resources:
    # Kubernetes Namespace
    - name: namespace
      base:
        apiVersion: kubernetes.crossplane.io/v1alpha1
        kind: Object
        spec:
          forProvider:
            manifest:
              apiVersion: v1
              kind: Namespace
              metadata:
                labels:
                  istio-injection: enabled
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.serviceName
          toFieldPath: spec.forProvider.manifest.metadata.name

    # RDS Database (if enabled)
    - name: database
      base:
        apiVersion: database.aws.crossplane.io/v1beta1
        kind: RDSInstance
        spec:
          forProvider:
            region: us-east-1
            dbInstanceClass: db.t3.micro
            engine: postgres
            engineVersion: "15"
            masterUsername: admin
            allocatedStorage: 20
            skipFinalSnapshotBeforeDeletion: true
            publiclyAccessible: false
            vpcSecurityGroupIds:
              - sg-xxxxxxxx
            dbSubnetGroupName: default
          writeConnectionSecretToRef:
            namespace: crossplane-system
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.serviceName
          toFieldPath: metadata.name
          transforms:
            - type: string
              string:
                fmt: "%s-db"
        - type: FromCompositeFieldPath
          fromFieldPath: spec.serviceName
          toFieldPath: spec.writeConnectionSecretToRef.name
          transforms:
            - type: string
              string:
                fmt: "%s-db-conn"
        # Size based on tier
        - type: FromCompositeFieldPath
          fromFieldPath: spec.tier
          toFieldPath: spec.forProvider.dbInstanceClass
          transforms:
            - type: map
              map:
                tier-1: db.r6g.large
                tier-2: db.t3.medium
                tier-3: db.t3.micro

    # ElastiCache Redis
    - name: redis
      base:
        apiVersion: cache.aws.crossplane.io/v1beta1
        kind: ReplicationGroup
        spec:
          forProvider:
            region: us-east-1
            engine: redis
            cacheNodeType: cache.t3.micro
            numCacheClusters: 2
            automaticFailoverEnabled: true
            atRestEncryptionEnabled: true
            transitEncryptionEnabled: true
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: spec.serviceName
          toFieldPath: metadata.name
          transforms:
            - type: string
              string:
                fmt: "%s-redis"
```

### 6.2 Self-Service API

```go
// Platform Self-Service API
package platform

import (
	"context"
	"encoding/json"
	"net/http"

	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"sigs.k8s.io/controller-runtime/pkg/client"
)

type ProvisionRequest struct {
	ServiceName string `json:"serviceName"`
	Owner       string `json:"owner"`
	Tier        string `json:"tier"`
	HasDatabase bool   `json:"hasDatabase"`
	DatabaseType string `json:"databaseType,omitempty"`
}

type PlatformAPI struct {
	k8sClient client.Client
}

func (api *PlatformAPI) ProvisionInfrastructure(w http.ResponseWriter, r *http.Request) {
	var req ProvisionRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	// Validate request
	if err := api.validateRequest(req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	// Create ServiceInfra claim
	claim := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "platform.company.com/v1alpha1",
			"kind":       "ServiceInfra",
			"metadata": map[string]interface{}{
				"name":      req.ServiceName,
				"namespace": "platform-claims",
				"labels": map[string]interface{}{
					"owner": req.Owner,
					"tier":  req.Tier,
				},
			},
			"spec": map[string]interface{}{
				"serviceName": req.ServiceName,
				"tier":        req.Tier,
				"database": map[string]interface{}{
					"enabled": req.HasDatabase,
					"type":    req.DatabaseType,
					"size":    api.sizeFromTier(req.Tier),
				},
			},
		},
	}

	if err := api.k8sClient.Create(context.Background(), claim); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	// Return success response
	response := map[string]interface{}{
		"status":      "provisioning",
		"serviceName": req.ServiceName,
		"message":     "Infrastructure provisioning started. Check status in Backstage.",
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

func (api *PlatformAPI) GetInfrastructureStatus(w http.ResponseWriter, r *http.Request) {
	serviceName := r.URL.Query().Get("serviceName")

	claim := &unstructured.Unstructured{}
	claim.SetGroupVersionKind(schema.GroupVersionKind{
		Group:   "platform.company.com",
		Version: "v1alpha1",
		Kind:    "ServiceInfra",
	})

	err := api.k8sClient.Get(context.Background(), client.ObjectKey{
		Namespace: "platform-claims",
		Name:      serviceName,
	}, claim)

	if err != nil {
		http.Error(w, err.Error(), http.StatusNotFound)
		return
	}

	// Extract status
	status, _, _ := unstructured.NestedMap(claim.Object, "status")

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(status)
}

func (api *PlatformAPI) sizeFromTier(tier string) string {
	switch tier {
	case "tier-1":
		return "large"
	case "tier-2":
		return "medium"
	default:
		return "small"
	}
}

func (api *PlatformAPI) validateRequest(req ProvisionRequest) error {
	// Add validation logic
	return nil
}
```

## 7. Golden Paths

### 7.1 Golden Path Definition

```yaml
# Golden paths configuration
golden_paths:
  backend_service:
    name: "Backend Service (Go)"
    description: "Standard path for Go backend services"
    template: backend-service-template
    requirements:
      - Structured JSON logging
      - Prometheus metrics
      - Health endpoints (/healthz, /readyz)
      - Graceful shutdown
      - OpenTelemetry tracing
    includes:
      - CI/CD pipeline (GitHub Actions)
      - Kubernetes manifests
      - Dockerfile (distroless)
      - TechDocs documentation
      - PagerDuty integration
    guardrails:
      - Security scanning (Trivy)
      - Dependency updates (Dependabot)
      - Code coverage > 70%
      - No critical vulnerabilities

  frontend_app:
    name: "Frontend Application (React)"
    description: "Standard path for React frontend applications"
    template: frontend-app-template
    requirements:
      - TypeScript
      - Component testing (React Testing Library)
      - E2E testing (Playwright)
      - Performance budgets
    includes:
      - Vite build configuration
      - Storybook
      - CI/CD pipeline
      - CDN deployment (CloudFront)

  data_pipeline:
    name: "Data Pipeline (Airflow)"
    description: "Standard path for data pipelines"
    template: data-pipeline-template
    requirements:
      - Idempotent DAGs
      - Data quality checks
      - Lineage tracking
    includes:
      - Airflow DAG structure
      - Great Expectations integration
      - Data catalog registration
```

### 7.2 Path Compliance Checker

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from enum import Enum

class ComplianceStatus(Enum):
    COMPLIANT = "compliant"
    NON_COMPLIANT = "non_compliant"
    PARTIAL = "partial"
    NOT_APPLICABLE = "not_applicable"

@dataclass
class ComplianceCheck:
    name: str
    description: str
    status: ComplianceStatus
    details: Optional[str] = None
    remediation: Optional[str] = None

@dataclass
class GoldenPathCompliance:
    service_name: str
    path_name: str
    overall_status: ComplianceStatus
    checks: List[ComplianceCheck]
    score: float  # 0-100

class GoldenPathChecker:
    def __init__(self, catalog_client, github_client, k8s_client):
        self.catalog = catalog_client
        self.github = github_client
        self.k8s = k8s_client

    async def check_compliance(
        self,
        service_name: str,
        path_name: str
    ) -> GoldenPathCompliance:
        """Check if a service complies with its golden path"""

        checks = []

        # Get service metadata from catalog
        service = await self.catalog.get_component(service_name)
        repo_slug = service.metadata.annotations.get('github.com/project-slug')

        # Check 1: Repository structure
        checks.append(await self._check_repo_structure(repo_slug, path_name))

        # Check 2: CI/CD pipeline
        checks.append(await self._check_cicd_pipeline(repo_slug))

        # Check 3: Documentation
        checks.append(await self._check_documentation(service))

        # Check 4: Monitoring
        checks.append(await self._check_monitoring(service))

        # Check 5: Security scanning
        checks.append(await self._check_security_scanning(repo_slug))

        # Check 6: Kubernetes configuration
        checks.append(await self._check_k8s_config(service_name))

        # Calculate overall status and score
        compliant_count = sum(1 for c in checks if c.status == ComplianceStatus.COMPLIANT)
        total_checks = len(checks)
        score = (compliant_count / total_checks) * 100

        if score == 100:
            overall_status = ComplianceStatus.COMPLIANT
        elif score >= 70:
            overall_status = ComplianceStatus.PARTIAL
        else:
            overall_status = ComplianceStatus.NON_COMPLIANT

        return GoldenPathCompliance(
            service_name=service_name,
            path_name=path_name,
            overall_status=overall_status,
            checks=checks,
            score=score
        )

    async def _check_repo_structure(
        self,
        repo_slug: str,
        path_name: str
    ) -> ComplianceCheck:
        """Check if repo follows golden path structure"""

        required_files = {
            'backend_service': [
                'Dockerfile',
                'Makefile',
                'go.mod',
                'catalog-info.yaml',
                '.github/workflows/ci.yaml',
                'docs/index.md'
            ],
            'frontend_app': [
                'Dockerfile',
                'package.json',
                'vite.config.ts',
                'catalog-info.yaml',
                '.github/workflows/ci.yaml'
            ]
        }

        required = required_files.get(path_name, [])
        missing = []

        for file_path in required:
            exists = await self.github.file_exists(repo_slug, file_path)
            if not exists:
                missing.append(file_path)

        if not missing:
            return ComplianceCheck(
                name="Repository Structure",
                description="All required files present",
                status=ComplianceStatus.COMPLIANT
            )
        else:
            return ComplianceCheck(
                name="Repository Structure",
                description=f"Missing files: {', '.join(missing)}",
                status=ComplianceStatus.NON_COMPLIANT,
                remediation=f"Add missing files: {', '.join(missing)}"
            )

    async def _check_cicd_pipeline(self, repo_slug: str) -> ComplianceCheck:
        """Check CI/CD pipeline configuration"""

        # Check if workflows exist
        workflows = await self.github.list_workflows(repo_slug)

        required_jobs = ['test', 'lint', 'security', 'build']
        found_jobs = set()

        for workflow in workflows:
            content = await self.github.get_workflow_content(repo_slug, workflow.path)
            for job in required_jobs:
                if job in content.get('jobs', {}):
                    found_jobs.add(job)

        missing_jobs = set(required_jobs) - found_jobs

        if not missing_jobs:
            return ComplianceCheck(
                name="CI/CD Pipeline",
                description="All required jobs configured",
                status=ComplianceStatus.COMPLIANT
            )
        else:
            return ComplianceCheck(
                name="CI/CD Pipeline",
                description=f"Missing jobs: {', '.join(missing_jobs)}",
                status=ComplianceStatus.PARTIAL,
                remediation=f"Add missing CI jobs: {', '.join(missing_jobs)}"
            )

    async def _check_monitoring(self, service) -> ComplianceCheck:
        """Check monitoring configuration"""

        annotations = service.metadata.annotations

        checks_passed = []
        checks_failed = []

        # Check Prometheus scraping
        if annotations.get('prometheus.io/scrape') == 'true':
            checks_passed.append('Prometheus scraping')
        else:
            checks_failed.append('Prometheus scraping not configured')

        # Check dashboard link
        has_dashboard = any(
            link.title == 'Grafana Dashboard'
            for link in service.metadata.links
        )
        if has_dashboard:
            checks_passed.append('Grafana dashboard')
        else:
            checks_failed.append('No Grafana dashboard link')

        # Check PagerDuty
        if annotations.get('pagerduty.com/integration-key'):
            checks_passed.append('PagerDuty integration')
        else:
            checks_failed.append('No PagerDuty integration')

        if not checks_failed:
            return ComplianceCheck(
                name="Monitoring",
                description="All monitoring configured",
                status=ComplianceStatus.COMPLIANT
            )
        elif checks_passed:
            return ComplianceCheck(
                name="Monitoring",
                description=f"Partial: {', '.join(checks_failed)}",
                status=ComplianceStatus.PARTIAL,
                remediation=f"Configure: {', '.join(checks_failed)}"
            )
        else:
            return ComplianceCheck(
                name="Monitoring",
                description="No monitoring configured",
                status=ComplianceStatus.NON_COMPLIANT,
                remediation="Set up Prometheus, Grafana, and PagerDuty"
            )
```

## 8. Developer Experience Metrics

### 8.1 DORA Metrics Integration

```yaml
# DORA Metrics Collection
dora_metrics:
  deployment_frequency:
    description: "How often code is deployed to production"
    targets:
      elite: "Multiple deploys per day"
      high: "Between once per day and once per week"
      medium: "Between once per week and once per month"
      low: "Between once per month and once every six months"
    collection:
      source: github_deployments
      query: |
        SELECT
          repo_name,
          COUNT(*) as deployment_count,
          COUNT(*) / 7.0 as deploys_per_day
        FROM deployments
        WHERE environment = 'production'
          AND created_at > NOW() - INTERVAL '7 days'
        GROUP BY repo_name

  lead_time_for_changes:
    description: "Time from code commit to production deployment"
    targets:
      elite: "Less than one hour"
      high: "Between one day and one week"
      medium: "Between one week and one month"
      low: "More than one month"
    collection:
      source: github_pull_requests
      query: |
        SELECT
          repo_name,
          AVG(EXTRACT(EPOCH FROM (deployed_at - created_at)) / 3600) as avg_lead_time_hours
        FROM pull_requests pr
        JOIN deployments d ON pr.merge_commit_sha = d.sha
        WHERE d.environment = 'production'
          AND pr.merged_at > NOW() - INTERVAL '30 days'
        GROUP BY repo_name

  change_failure_rate:
    description: "Percentage of deployments causing production failures"
    targets:
      elite: "0-15%"
      high: "16-30%"
      medium: "31-45%"
      low: "46-60%"
    collection:
      source: pagerduty_incidents
      query: |
        SELECT
          service_name,
          COUNT(DISTINCT d.id) as total_deployments,
          COUNT(DISTINCT i.id) as failed_deployments,
          (COUNT(DISTINCT i.id)::float / COUNT(DISTINCT d.id)) * 100 as failure_rate
        FROM deployments d
        LEFT JOIN incidents i ON i.triggered_by_deployment_id = d.id
        WHERE d.created_at > NOW() - INTERVAL '30 days'
        GROUP BY service_name

  mean_time_to_recovery:
    description: "Time to restore service after a failure"
    targets:
      elite: "Less than one hour"
      high: "Less than one day"
      medium: "Less than one week"
      low: "More than one week"
    collection:
      source: pagerduty_incidents
      query: |
        SELECT
          service_name,
          AVG(EXTRACT(EPOCH FROM (resolved_at - created_at)) / 60) as avg_mttr_minutes
        FROM incidents
        WHERE created_at > NOW() - INTERVAL '30 days'
          AND resolved_at IS NOT NULL
        GROUP BY service_name
```

### 8.2 Platform Metrics Dashboard

```json
{
  "dashboard": {
    "title": "Platform Engineering Metrics",
    "panels": [
      {
        "title": "Platform Adoption",
        "type": "stat",
        "gridPos": {"x": 0, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "count(backstage_catalog_entities{kind='Component'})",
            "legendFormat": "Total Services"
          }
        ]
      },
      {
        "title": "Services Using Golden Paths",
        "type": "gauge",
        "gridPos": {"x": 6, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(backstage_golden_path_compliant{status='compliant'}) / count(backstage_catalog_entities{kind='Component'}) * 100",
            "legendFormat": "Adoption %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "max": 100,
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "yellow", "value": 50},
                {"color": "green", "value": 80}
              ]
            }
          }
        }
      },
      {
        "title": "Deployment Frequency",
        "type": "timeseries",
        "gridPos": {"x": 0, "y": 4, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum(increase(deployments_total{env='production'}[1d]))",
            "legendFormat": "Daily Deployments"
          }
        ]
      },
      {
        "title": "Lead Time for Changes (Hours)",
        "type": "timeseries",
        "gridPos": {"x": 12, "y": 4, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "histogram_quantile(0.5, sum(rate(lead_time_seconds_bucket[1d])) by (le)) / 3600",
            "legendFormat": "P50"
          },
          {
            "expr": "histogram_quantile(0.95, sum(rate(lead_time_seconds_bucket[1d])) by (le)) / 3600",
            "legendFormat": "P95"
          }
        ]
      },
      {
        "title": "Template Usage",
        "type": "piechart",
        "gridPos": {"x": 0, "y": 12, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "sum by (template) (backstage_scaffolder_task_count)",
            "legendFormat": "{{template}}"
          }
        ]
      },
      {
        "title": "Self-Service Requests",
        "type": "timeseries",
        "gridPos": {"x": 8, "y": 12, "w": 16, "h": 8},
        "targets": [
          {
            "expr": "sum(increase(platform_self_service_requests_total[1h])) by (request_type)",
            "legendFormat": "{{request_type}}"
          }
        ]
      },
      {
        "title": "Developer Satisfaction (NPS)",
        "type": "gauge",
        "gridPos": {"x": 0, "y": 20, "w": 6, "h": 6},
        "targets": [
          {
            "expr": "platform_developer_nps_score",
            "legendFormat": "NPS"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "min": -100,
            "max": 100,
            "thresholds": {
              "steps": [
                {"color": "red", "value": -100},
                {"color": "yellow", "value": 0},
                {"color": "green", "value": 50}
              ]
            }
          }
        }
      },
      {
        "title": "Time Saved by Platform (Hours/Week)",
        "type": "stat",
        "gridPos": {"x": 6, "y": 20, "w": 6, "h": 6},
        "targets": [
          {
            "expr": "sum(platform_time_saved_hours_total)",
            "legendFormat": "Hours Saved"
          }
        ]
      }
    ]
  }
}
```

### 8.3 Developer Survey Integration

```python
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime
import statistics

@dataclass
class SurveyQuestion:
    id: str
    text: str
    category: str  # satisfaction, usability, features
    scale: str  # nps, likert, open_text

@dataclass
class SurveyResponse:
    question_id: str
    score: Optional[int]
    text: Optional[str]
    timestamp: datetime
    team: str

class DeveloperSurveyAnalyzer:
    def __init__(self, survey_client, metrics_exporter):
        self.survey = survey_client
        self.metrics = metrics_exporter

    async def analyze_platform_satisfaction(self) -> dict:
        """Analyze developer satisfaction with the platform"""

        # Fetch recent survey responses
        responses = await self.survey.get_responses(
            survey_id="platform-satisfaction",
            since=datetime.now() - timedelta(days=30)
        )

        # Calculate NPS
        nps_responses = [r for r in responses if r.question_id == "nps"]
        nps_score = self._calculate_nps(nps_responses)

        # Calculate satisfaction by category
        categories = {}
        for category in ["usability", "documentation", "support", "features"]:
            cat_responses = [r for r in responses if r.question_id.startswith(category)]
            if cat_responses:
                categories[category] = statistics.mean([r.score for r in cat_responses])

        # Extract themes from open-text responses
        text_responses = [r.text for r in responses if r.text]
        themes = self._extract_themes(text_responses)

        # Export metrics
        self.metrics.set_gauge("platform_developer_nps_score", nps_score)
        for cat, score in categories.items():
            self.metrics.set_gauge(
                "platform_satisfaction_score",
                score,
                labels={"category": cat}
            )

        return {
            "nps_score": nps_score,
            "category_scores": categories,
            "top_themes": themes,
            "response_count": len(responses),
            "response_rate": len(responses) / self._get_developer_count()
        }

    def _calculate_nps(self, responses: List[SurveyResponse]) -> float:
        """Calculate Net Promoter Score"""
        if not responses:
            return 0.0

        promoters = sum(1 for r in responses if r.score >= 9)
        detractors = sum(1 for r in responses if r.score <= 6)
        total = len(responses)

        return ((promoters - detractors) / total) * 100

    def _extract_themes(self, texts: List[str]) -> List[dict]:
        """Extract common themes from text responses"""
        # Simplified theme extraction
        # In practice, use NLP libraries like spaCy or transformers
        themes = {}
        keywords = {
            "documentation": ["docs", "documentation", "readme", "guide"],
            "speed": ["slow", "fast", "performance", "quick"],
            "support": ["help", "support", "question", "stuck"],
            "templates": ["template", "scaffold", "boilerplate"],
        }

        for text in texts:
            text_lower = text.lower()
            for theme, words in keywords.items():
                if any(word in text_lower for word in words):
                    themes[theme] = themes.get(theme, 0) + 1

        return sorted(
            [{"theme": k, "count": v} for k, v in themes.items()],
            key=lambda x: x["count"],
            reverse=True
        )[:5]
```

## 9. Implementation Checklist

### 9.1 Phase 1: Foundation (Weeks 1-4)

- [ ] Set up Backstage instance
  - [ ] Deploy to Kubernetes
  - [ ] Configure authentication (GitHub/OIDC)
  - [ ] Set up PostgreSQL database
- [ ] Create initial Software Catalog
  - [ ] Define entity schemas
  - [ ] Import existing services
  - [ ] Set up auto-discovery
- [ ] Create first golden path template
  - [ ] Backend service template
  - [ ] CI/CD pipeline template
  - [ ] Documentation template

### 9.2 Phase 2: Self-Service (Weeks 5-8)

- [ ] Deploy Crossplane for infrastructure
  - [ ] Configure AWS/GCP providers
  - [ ] Create CompositeResourceDefinitions
  - [ ] Build Compositions for common patterns
- [ ] Integrate self-service workflows
  - [ ] Database provisioning
  - [ ] Namespace creation
  - [ ] Secret management
- [ ] Add plugins to Backstage
  - [ ] Kubernetes plugin
  - [ ] PagerDuty plugin
  - [ ] Cost insights plugin

### 9.3 Phase 3: Optimization (Weeks 9-12)

- [ ] Implement compliance checking
  - [ ] Golden path compliance
  - [ ] Security compliance
  - [ ] Documentation compliance
- [ ] Set up metrics collection
  - [ ] DORA metrics
  - [ ] Platform usage metrics
  - [ ] Developer satisfaction surveys
- [ ] Create platform SLOs
  - [ ] Backstage availability
  - [ ] Template execution time
  - [ ] Self-service request latency

## 10. Related Specifications

| Specification | Relationship |
|--------------|--------------|
| [SRP-008 Service Mesh](SRP-008%20Service%20Mesh.md) | Service networking for platform |
| GitOps (planned) | GitOps integration |
| [SRG-007 SLI SLO SLA](SRG-007%20SLI%20SLO%20SLA.md) | Platform SLOs |
| [SRO-001 On-Call & Incident Response](SRO-001%20On-Call%20Incident%20Response.md) | Platform support model |
| [SRG-011 Feature Flags](SRG-011%20Feature%20Flags.md) | Feature management in templates |

---

**Document History:**

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-01-15 | Platform Team | Initial specification |
