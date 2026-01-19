# SRO-006 Platform Engineering

| Поле | Значение |
|------|----------|
| **Статус** | APPROVED |
| **Версия** | 1.0 |
| **Приоритет** | P2 |
| **Сложность** | Высокая |
| **Целевая аудитория** | Platform Engineers, DevOps, SRE, Архитекторы |

## 1. Введение

### 1.1 Назначение

Данная спецификация определяет стандарты построения Internal Developer Platform (IDP), которые обеспечивают самообслуживание инфраструктуры, улучшают опыт разработчиков и ускоряют доставку программного обеспечения. Platform Engineering рассматривает платформу как продукт, предоставляя "золотые пути" для типовых задач при сохранении гибкости для сложных случаев.

### 1.2 Область применения

Данная спецификация охватывает:
- Архитектуру Internal Developer Platform
- Порталы разработчика (Backstage, Port, Cortex)
- Каталог сервисов и шаблоны
- Золотые пути (Golden Paths) и стандартизированные маршруты
- Самообслуживание инфраструктуры
- Метрики Developer Experience (DevEx)
- Принципы Platform as a Product

### 1.3 Связанные спецификации

- [SRP-008 Service Mesh](SRP-008%20Service%20Mesh.ru.md) - Сетевое взаимодействие сервисов
- GitOps (запланирован) - Практики GitOps
- [SRG-007 SLI SLO SLA](SRG-007%20SLI%20SLO%20SLA.ru.md) - SLO платформы
- [SRO-001 On-Call & Incident Response](SRO-001%20On-Call%20Incident%20Response.ru.md) - Поддержка платформы

## 2. Основы Platform Engineering

### 2.1 Что такое Platform Engineering?

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Developer Experience                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Портал разработчика (Backstage)              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │  Каталог    │  │  Золотые    │  │    Self-Service         │  │   │
│  │  │  сервисов   │  │   пути      │  │    Workflows            │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Internal Developer Platform (IDP)                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │  Оркестрация    │  │    Контроли     │  │     Observability       │ │
│  │  инфраструктуры │  │  безопасности   │  │     интеграция          │ │
│  │  (Crossplane,   │  │  (OPA, Kyverno) │  │  (Prometheus, Grafana)  │ │
│  │   Terraform)    │  │                 │  │                         │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │   CI/CD        │  │   Управление    │  │     Service Mesh        │ │
│  │   пайплайны    │  │   секретами     │  │     (Istio/Linkerd)     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Инфраструктурный слой                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐ │
│  │   Kubernetes    │  │  Облачный       │  │    Базы данных          │ │
│  │   кластеры      │  │  провайдер      │  │    (RDS, CloudSQL)      │ │
│  │                 │  │  (AWS/GCP/Azure)│  │                         │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Platform Engineering vs Traditional DevOps

| Аспект | Традиционный DevOps | Platform Engineering |
|--------|---------------------|---------------------|
| **Фокус** | Процессы и культура | Продукты и самообслуживание |
| **Доставка** | Инструменты для каждой команды | Централизованная платформа |
| **Подход** | "You build it, you run it" | "Золотые пути с ограждениями" |
| **Когнитивная нагрузка** | Высокая (полное владение стеком) | Сниженная (абстрагированная сложность) |
| **Масштабирование** | Линейное (на команду) | Экспоненциальное (рычаг платформы) |
| **Стандартизация** | Варьируется по командам | Единообразная по организации |

### 2.3 Структура команды платформы

```yaml
# Организация Platform Team
platform_team:
  structure:
    platform_product_manager:
      responsibilities:
        - Определение roadmap платформы
        - Сбор обратной связи разработчиков
        - Приоритизация фич
        - Измерение adoption платформы

    platform_engineers:
      responsibilities:
        - Разработка компонентов платформы
        - Поддержка инфраструктуры
        - Создание золотых путей
        - Написание документации
      skills:
        - Kubernetes
        - Infrastructure as Code
        - CI/CD пайплайны
        - Облачные провайдеры

    developer_advocates:
      responsibilities:
        - Онбординг команд на платформу
        - Создание туториалов и гайдов
        - Сбор обратной связи
        - Проведение office hours

  operating_model:
    support_model: "tiered"
    tiers:
      tier_0: "Self-service документация и шаблоны"
      tier_1: "Поддержка в Slack (асинхронно)"
      tier_2: "Office hours (по расписанию)"
      tier_3: "Выделенное сопровождение (сложные случаи)"

    sla:
      tier_1_response: "4 часа"
      tier_2_response: "24 часа"
      critical_issues: "1 час"
```

## 3. Портал разработчика (Backstage)

### 3.1 Архитектура Backstage

```
┌─────────────────────────────────────────────────────────────────┐
│                         Backstage                                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌───────────┐ │
│  │  Software   │ │  TechDocs   │ │   Поиск     │ │  Плагины  │ │
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

### 3.2 Развертывание Backstage

```yaml
# Kubernetes Deployment для Backstage
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

### 3.3 Конфигурация Backstage

```typescript
// app-config.yaml эквивалент на TypeScript
// packages/backend/src/index.ts

import { createBackend } from '@backstage/backend-defaults';

const backend = createBackend();

// Основные плагины
backend.add(import('@backstage/plugin-app-backend/alpha'));
backend.add(import('@backstage/plugin-catalog-backend/alpha'));
backend.add(import('@backstage/plugin-scaffolder-backend/alpha'));
backend.add(import('@backstage/plugin-techdocs-backend/alpha'));
backend.add(import('@backstage/plugin-search-backend/alpha'));

// Аутентификация
backend.add(import('@backstage/plugin-auth-backend'));
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));

// Интеграция с Kubernetes
backend.add(import('@backstage/plugin-kubernetes-backend/alpha'));

// Интеграция с GitHub
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
    # Обнаружение GitHub организации
    - type: github-discovery
      target: https://github.com/mycompany
    # Расположение шаблонов
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

## 4. Каталог сервисов

### 4.1 Модель сущностей каталога

```yaml
# Определение системы
apiVersion: backstage.io/v1alpha1
kind: System
metadata:
  name: payment-system
  description: Обрабатывает все платежи
  annotations:
    backstage.io/techdocs-ref: dir:.
  tags:
    - payments
    - critical
  links:
    - url: https://wiki.company.com/payments
      title: Wiki платежной системы
spec:
  owner: team-payments
  domain: commerce

---
# Определение компонента (сервиса)
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: payment-service
  description: Основной сервис обработки платежей
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
      title: Дашборд Grafana
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
# Определение API
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: payment-api
  description: API обработки платежей
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
# Определение ресурса (базы данных)
apiVersion: backstage.io/v1alpha1
kind: Resource
metadata:
  name: postgres-payments
  description: PostgreSQL база данных для платежных данных
  annotations:
    backstage.io/managed-by-location: url:https://github.com/mycompany/infrastructure/blob/main/databases/payments.yaml
spec:
  type: database
  owner: team-platform
  system: payment-system
  dependencyOf:
    - component:default/payment-service

---
# Определение команды (группы)
apiVersion: backstage.io/v1alpha1
kind: Group
metadata:
  name: team-payments
  description: Команда обработки платежей
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
# Определение домена
apiVersion: backstage.io/v1alpha1
kind: Domain
metadata:
  name: commerce
  description: Все системы связанные с коммерцией
spec:
  owner: commerce-leadership
```

### 4.2 Процессор каталога для автообнаружения

```typescript
// Кастомный процессор каталога для автообнаружения
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

    // Обнаружение сервисов из Kubernetes
    const services = await this.discoverKubernetesServices(location.target);

    for (const service of services) {
      const entity: Entity = {
        apiVersion: 'backstage.io/v1alpha1',
        kind: 'Component',
        metadata: {
          name: service.name,
          namespace: 'default',
          description: `Автообнаружено из Kubernetes: ${service.namespace}/${service.name}`,
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
    // Реализация запроса к Kubernetes API
    // Возвращает массив метаданных сервисов
    return [];
  }
}
```

### 4.3 Scorecard каталога

```yaml
# Правила scorecard для сущностей каталога
scorecards:
  production-readiness:
    title: Готовность к продакшену
    description: Проверяет готовность компонента к продакшену
    checks:
      - id: has-owner
        title: Есть владелец
        description: У компонента должен быть назначен владелец
        rule:
          factRef: metadata.owner
          operator: isNotEmpty
        weight: 10

      - id: has-description
        title: Есть описание
        description: У компонента должно быть описание
        rule:
          factRef: metadata.description
          operator: minLength
          value: 20
        weight: 5

      - id: has-techdocs
        title: Есть TechDocs
        description: У компонента должна быть документация
        rule:
          factRef: metadata.annotations['backstage.io/techdocs-ref']
          operator: isNotEmpty
        weight: 10

      - id: has-pagerduty
        title: Есть интеграция с PagerDuty
        description: Продакшен компоненты должны иметь настроенный PagerDuty
        rule:
          factRef: metadata.annotations['pagerduty.com/integration-key']
          operator: isNotEmpty
        weight: 15
        condition:
          factRef: spec.lifecycle
          operator: equals
          value: production

      - id: has-runbook
        title: Есть Runbook
        description: Продакшен компоненты должны иметь runbook
        rule:
          factRef: metadata.links
          operator: contains
          field: title
          value: Runbook
        weight: 15

      - id: has-dashboard
        title: Есть дашборд мониторинга
        description: У компонента должен быть дашборд мониторинга
        rule:
          factRef: metadata.links
          operator: contains
          field: title
          value: Dashboard
        weight: 10

      - id: ci-cd-configured
        title: Настроен CI/CD
        description: У компонента должен быть CI/CD пайплайн
        rule:
          factRef: metadata.annotations['github.com/project-slug']
          operator: isNotEmpty
        weight: 10

  security-compliance:
    title: Соответствие безопасности
    description: Требования безопасности для компонентов
    checks:
      - id: has-security-contact
        title: Есть контакт по безопасности
        rule:
          factRef: metadata.annotations['security.company.com/contact']
          operator: isNotEmpty
        weight: 20

      - id: vulnerability-scanning
        title: Сканирование уязвимостей включено
        rule:
          factRef: metadata.annotations['security.company.com/vuln-scanning']
          operator: equals
          value: "enabled"
        weight: 25
```

## 5. Шаблоны сервисов (Scaffolder)

### 5.1 Структура шаблона Golden Path

```
templates/
├── backend-service/
│   ├── template.yaml           # Определение шаблона
│   ├── skeleton/               # Файлы для scaffolding
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
└── all-templates.yaml          # Индекс шаблонов
```

### 5.2 Шаблон Backend Service

```yaml
# templates/backend-service/template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: backend-service-template
  title: Backend Service (Go)
  description: Создание нового Go backend сервиса со всеми интеграциями платформы
  tags:
    - go
    - backend
    - recommended
spec:
  owner: team-platform
  type: service

  parameters:
    - title: Информация о сервисе
      required:
        - name
        - description
        - owner
      properties:
        name:
          title: Название сервиса
          type: string
          description: Название сервиса (строчные буквы, дефисы разрешены)
          pattern: '^[a-z][a-z0-9-]*[a-z0-9]$'
          ui:autofocus: true
        description:
          title: Описание
          type: string
          description: Краткое описание что делает этот сервис
        owner:
          title: Владелец
          type: string
          description: Команда-владелец этого сервиса
          ui:field: OwnerPicker
          ui:options:
            catalogFilter:
              kind: Group
              spec.type: team

    - title: Техническая конфигурация
      required:
        - system
        - tier
      properties:
        system:
          title: Система
          type: string
          description: Система к которой принадлежит сервис
          ui:field: EntityPicker
          ui:options:
            catalogFilter:
              kind: System
        tier:
          title: Уровень сервиса
          type: string
          description: Уровень критичности (влияет на требования SLO)
          enum:
            - tier-1
            - tier-2
            - tier-3
          enumNames:
            - 'Tier 1 (99.99% SLO, критичный)'
            - 'Tier 2 (99.9% SLO, важный)'
            - 'Tier 3 (99% SLO, стандартный)'
          default: tier-2
        hasDatabase:
          title: Требуется база данных
          type: boolean
          default: false
        databaseType:
          title: Тип базы данных
          type: string
          enum:
            - postgresql
            - mysql
            - mongodb
          ui:widget: select
          ui:options:
            visible:
              hasDatabase: true

    - title: Конфигурация репозитория
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Расположение репозитория
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com
            allowedOwners:
              - mycompany

  steps:
    # Получение базового шаблона
    - id: fetch-base
      name: Получение базового шаблона
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

    # Создание репозитория GitHub
    - id: publish
      name: Публикация в GitHub
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

    # Регистрация в каталоге
    - id: register
      name: Регистрация в каталоге
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps['publish'].output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'

    # Провижионинг инфраструктуры
    - id: provision-infra
      name: Провижионинг инфраструктуры
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

    # Создание сервиса PagerDuty
    - id: create-pagerduty
      name: Создание сервиса PagerDuty
      action: pagerduty:service:create
      input:
        name: ${{ parameters.name }}
        description: ${{ parameters.description }}
        escalationPolicyId: ${{ parameters.tier === 'tier-1' ? 'P1_POLICY' : 'STANDARD_POLICY' }}

  output:
    links:
      - title: Репозиторий
        url: ${{ steps['publish'].output.remoteUrl }}
      - title: Открыть в каталоге
        icon: catalog
        entityRef: ${{ steps['register'].output.entityRef }}
      - title: Сервис PagerDuty
        url: ${{ steps['create-pagerduty'].output.serviceUrl }}
```

### 5.3 Файлы скелета шаблона

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
      title: Дашборд Grafana
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

	// Запуск серверов
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

## 6. Самообслуживание инфраструктуры

### 6.1 Crossplane для провижионинга инфраструктуры

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
                  description: Название сервиса
                tier:
                  type: string
                  enum: [tier-1, tier-2, tier-3]
                  description: Уровень сервиса для размера ресурсов
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

    # RDS Database (если включена)
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
        # Размер на основе tier
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
	ServiceName  string `json:"serviceName"`
	Owner        string `json:"owner"`
	Tier         string `json:"tier"`
	HasDatabase  bool   `json:"hasDatabase"`
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

	// Валидация запроса
	if err := api.validateRequest(req); err != nil {
		http.Error(w, err.Error(), http.StatusBadRequest)
		return
	}

	// Создание ServiceInfra claim
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

	// Возврат успешного ответа
	response := map[string]interface{}{
		"status":      "provisioning",
		"serviceName": req.ServiceName,
		"message":     "Провижионинг инфраструктуры запущен. Проверьте статус в Backstage.",
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

	// Извлечение статуса
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
	// Логика валидации
	return nil
}
```

## 7. Золотые пути (Golden Paths)

### 7.1 Определение Golden Path

```yaml
# Конфигурация золотых путей
golden_paths:
  backend_service:
    name: "Backend Service (Go)"
    description: "Стандартный путь для Go backend сервисов"
    template: backend-service-template
    requirements:
      - Структурированное JSON логирование
      - Метрики Prometheus
      - Health endpoints (/healthz, /readyz)
      - Graceful shutdown
      - OpenTelemetry трейсинг
    includes:
      - CI/CD пайплайн (GitHub Actions)
      - Kubernetes манифесты
      - Dockerfile (distroless)
      - TechDocs документация
      - Интеграция с PagerDuty
    guardrails:
      - Сканирование безопасности (Trivy)
      - Обновления зависимостей (Dependabot)
      - Code coverage > 70%
      - Нет критических уязвимостей

  frontend_app:
    name: "Frontend Application (React)"
    description: "Стандартный путь для React frontend приложений"
    template: frontend-app-template
    requirements:
      - TypeScript
      - Компонентное тестирование (React Testing Library)
      - E2E тестирование (Playwright)
      - Бюджеты производительности
    includes:
      - Конфигурация сборки Vite
      - Storybook
      - CI/CD пайплайн
      - Деплой на CDN (CloudFront)

  data_pipeline:
    name: "Data Pipeline (Airflow)"
    description: "Стандартный путь для data пайплайнов"
    template: data-pipeline-template
    requirements:
      - Идемпотентные DAG
      - Проверки качества данных
      - Отслеживание lineage
    includes:
      - Структура Airflow DAG
      - Интеграция Great Expectations
      - Регистрация в каталоге данных
```

### 7.2 Проверка соответствия путям

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
        """Проверка соответствия сервиса золотому пути"""

        checks = []

        # Получение метаданных сервиса из каталога
        service = await self.catalog.get_component(service_name)
        repo_slug = service.metadata.annotations.get('github.com/project-slug')

        # Проверка 1: Структура репозитория
        checks.append(await self._check_repo_structure(repo_slug, path_name))

        # Проверка 2: CI/CD пайплайн
        checks.append(await self._check_cicd_pipeline(repo_slug))

        # Проверка 3: Документация
        checks.append(await self._check_documentation(service))

        # Проверка 4: Мониторинг
        checks.append(await self._check_monitoring(service))

        # Проверка 5: Сканирование безопасности
        checks.append(await self._check_security_scanning(repo_slug))

        # Проверка 6: Kubernetes конфигурация
        checks.append(await self._check_k8s_config(service_name))

        # Расчет общего статуса и оценки
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
        """Проверка соответствия структуры репозитория золотому пути"""

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
                name="Структура репозитория",
                description="Все необходимые файлы присутствуют",
                status=ComplianceStatus.COMPLIANT
            )
        else:
            return ComplianceCheck(
                name="Структура репозитория",
                description=f"Отсутствуют файлы: {', '.join(missing)}",
                status=ComplianceStatus.NON_COMPLIANT,
                remediation=f"Добавьте недостающие файлы: {', '.join(missing)}"
            )

    async def _check_cicd_pipeline(self, repo_slug: str) -> ComplianceCheck:
        """Проверка конфигурации CI/CD пайплайна"""

        # Проверка наличия workflows
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
                name="CI/CD пайплайн",
                description="Все необходимые jobs настроены",
                status=ComplianceStatus.COMPLIANT
            )
        else:
            return ComplianceCheck(
                name="CI/CD пайплайн",
                description=f"Отсутствуют jobs: {', '.join(missing_jobs)}",
                status=ComplianceStatus.PARTIAL,
                remediation=f"Добавьте недостающие CI jobs: {', '.join(missing_jobs)}"
            )

    async def _check_monitoring(self, service) -> ComplianceCheck:
        """Проверка конфигурации мониторинга"""

        annotations = service.metadata.annotations

        checks_passed = []
        checks_failed = []

        # Проверка Prometheus scraping
        if annotations.get('prometheus.io/scrape') == 'true':
            checks_passed.append('Prometheus scraping')
        else:
            checks_failed.append('Prometheus scraping не настроен')

        # Проверка ссылки на дашборд
        has_dashboard = any(
            link.title == 'Grafana Dashboard'
            for link in service.metadata.links
        )
        if has_dashboard:
            checks_passed.append('Grafana дашборд')
        else:
            checks_failed.append('Нет ссылки на Grafana дашборд')

        # Проверка PagerDuty
        if annotations.get('pagerduty.com/integration-key'):
            checks_passed.append('Интеграция PagerDuty')
        else:
            checks_failed.append('Нет интеграции с PagerDuty')

        if not checks_failed:
            return ComplianceCheck(
                name="Мониторинг",
                description="Весь мониторинг настроен",
                status=ComplianceStatus.COMPLIANT
            )
        elif checks_passed:
            return ComplianceCheck(
                name="Мониторинг",
                description=f"Частично: {', '.join(checks_failed)}",
                status=ComplianceStatus.PARTIAL,
                remediation=f"Настройте: {', '.join(checks_failed)}"
            )
        else:
            return ComplianceCheck(
                name="Мониторинг",
                description="Мониторинг не настроен",
                status=ComplianceStatus.NON_COMPLIANT,
                remediation="Настройте Prometheus, Grafana и PagerDuty"
            )
```

## 8. Метрики Developer Experience

### 8.1 Интеграция метрик DORA

```yaml
# Сбор метрик DORA
dora_metrics:
  deployment_frequency:
    description: "Как часто код деплоится в production"
    targets:
      elite: "Несколько деплоев в день"
      high: "От раза в день до раза в неделю"
      medium: "От раза в неделю до раза в месяц"
      low: "От раза в месяц до раза в полгода"
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
    description: "Время от коммита до деплоя в production"
    targets:
      elite: "Менее часа"
      high: "От дня до недели"
      medium: "От недели до месяца"
      low: "Более месяца"
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
    description: "Процент деплоев вызывающих сбои в production"
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
    description: "Время восстановления сервиса после сбоя"
    targets:
      elite: "Менее часа"
      high: "Менее дня"
      medium: "Менее недели"
      low: "Более недели"
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

### 8.2 Дашборд метрик платформы

```json
{
  "dashboard": {
    "title": "Метрики Platform Engineering",
    "panels": [
      {
        "title": "Adoption платформы",
        "type": "stat",
        "gridPos": {"x": 0, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "count(backstage_catalog_entities{kind='Component'})",
            "legendFormat": "Всего сервисов"
          }
        ]
      },
      {
        "title": "Сервисы использующие Golden Paths",
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
        "title": "Частота деплоев",
        "type": "timeseries",
        "gridPos": {"x": 0, "y": 4, "w": 12, "h": 8},
        "targets": [
          {
            "expr": "sum(increase(deployments_total{env='production'}[1d]))",
            "legendFormat": "Ежедневные деплои"
          }
        ]
      },
      {
        "title": "Lead Time для изменений (часы)",
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
        "title": "Использование шаблонов",
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
        "title": "Self-Service запросы",
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
        "title": "Удовлетворенность разработчиков (NPS)",
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
        "title": "Сэкономлено времени платформой (часы/неделя)",
        "type": "stat",
        "gridPos": {"x": 6, "y": 20, "w": 6, "h": 6},
        "targets": [
          {
            "expr": "sum(platform_time_saved_hours_total)",
            "legendFormat": "Сэкономлено часов"
          }
        ]
      }
    ]
  }
}
```

### 8.3 Интеграция опросов разработчиков

```python
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime, timedelta
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
        """Анализ удовлетворенности разработчиков платформой"""

        # Получение недавних ответов
        responses = await self.survey.get_responses(
            survey_id="platform-satisfaction",
            since=datetime.now() - timedelta(days=30)
        )

        # Расчет NPS
        nps_responses = [r for r in responses if r.question_id == "nps"]
        nps_score = self._calculate_nps(nps_responses)

        # Расчет удовлетворенности по категориям
        categories = {}
        for category in ["usability", "documentation", "support", "features"]:
            cat_responses = [r for r in responses if r.question_id.startswith(category)]
            if cat_responses:
                categories[category] = statistics.mean([r.score for r in cat_responses])

        # Извлечение тем из текстовых ответов
        text_responses = [r.text for r in responses if r.text]
        themes = self._extract_themes(text_responses)

        # Экспорт метрик
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
        """Расчет Net Promoter Score"""
        if not responses:
            return 0.0

        promoters = sum(1 for r in responses if r.score >= 9)
        detractors = sum(1 for r in responses if r.score <= 6)
        total = len(responses)

        return ((promoters - detractors) / total) * 100

    def _extract_themes(self, texts: List[str]) -> List[dict]:
        """Извлечение общих тем из текстовых ответов"""
        # Упрощенное извлечение тем
        # На практике используйте NLP библиотеки типа spaCy или transformers
        themes = {}
        keywords = {
            "documentation": ["docs", "documentation", "readme", "guide", "документация"],
            "speed": ["slow", "fast", "performance", "quick", "медленно", "быстро"],
            "support": ["help", "support", "question", "stuck", "помощь", "поддержка"],
            "templates": ["template", "scaffold", "boilerplate", "шаблон"],
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

## 9. Чеклист внедрения

### 9.1 Фаза 1: Основа

- [ ] Развертывание Backstage
  - [ ] Деплой в Kubernetes
  - [ ] Настройка аутентификации (GitHub/OIDC)
  - [ ] Настройка базы данных PostgreSQL
- [ ] Создание начального Software Catalog
  - [ ] Определение схем сущностей
  - [ ] Импорт существующих сервисов
  - [ ] Настройка автообнаружения
- [ ] Создание первого golden path шаблона
  - [ ] Шаблон backend сервиса
  - [ ] Шаблон CI/CD пайплайна
  - [ ] Шаблон документации

### 9.2 Фаза 2: Самообслуживание

- [ ] Развертывание Crossplane для инфраструктуры
  - [ ] Настройка провайдеров AWS/GCP
  - [ ] Создание CompositeResourceDefinitions
  - [ ] Построение Compositions для типовых паттернов
- [ ] Интеграция self-service workflows
  - [ ] Провижионинг баз данных
  - [ ] Создание namespace
  - [ ] Управление секретами
- [ ] Добавление плагинов в Backstage
  - [ ] Плагин Kubernetes
  - [ ] Плагин PagerDuty
  - [ ] Плагин Cost insights

### 9.3 Фаза 3: Оптимизация

- [ ] Внедрение проверок соответствия
  - [ ] Соответствие golden path
  - [ ] Соответствие безопасности
  - [ ] Соответствие документации
- [ ] Настройка сбора метрик
  - [ ] Метрики DORA
  - [ ] Метрики использования платформы
  - [ ] Опросы удовлетворенности разработчиков
- [ ] Создание SLO платформы
  - [ ] Доступность Backstage
  - [ ] Время выполнения шаблонов
  - [ ] Задержка self-service запросов

## 10. Связанные спецификации

| Спецификация | Связь |
|--------------|-------|
| [SRP-008 Service Mesh](SRP-008%20Service%20Mesh.ru.md) | Сетевое взаимодействие для платформы |
| GitOps (запланирован) | Интеграция GitOps |
| [SRG-007 SLI SLO SLA](SRG-007%20SLI%20SLO%20SLA.ru.md) | SLO платформы |
| [SRO-001 On-Call & Incident Response](SRO-001%20On-Call%20Incident%20Response.ru.md) | Модель поддержки платформы |
| [SRG-011 Feature Flags](SRG-011%20Feature%20Flags.ru.md) | Управление фичами в шаблонах |

---

**История документа:**

| Версия | Дата | Автор | Изменения |
|--------|------|-------|-----------|
| 1.0 | 2024-01-15 | Platform Team | Начальная спецификация |
