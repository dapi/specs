# Site Reliability Handbook

:gb: [English version](README.md)

Комплексная коллекция спецификаций, паттернов, руководств и операционных практик для построения надежных, масштабируемых и наблюдаемых программных систем.

## Типы документов

| Префикс | Тип | Описание | Кол-во |
|---------|-----|----------|--------|
| **SRS** | Спецификации | Технические требования, которым должно соответствовать ПО | 20 |
| **SRP** | Паттерны | Архитектурные паттерны и проектные решения | 9 |
| **SRG** | Руководства | Инструкции по реализации и практические гайды | 19 |
| **SRO** | Операции | Операционные практики и процессы | 9 |

**Всего: 57 документов**

---

## SRS - Спецификации (20)

Технические требования, определяющие, что должны реализовывать программные системы.

| ID | Название | Приоритет | Роли |
|----|----------|-----------|------|
| [SRS-001](specs/SRS-001%20Stateless%20Services.ru.md) | Stateless Services | P1 | Developer, Architect |
| [SRS-002](specs/SRS-002%20Environment%20Variables.ru.md) | Environment Variables | P1 | Developer, DevOps |
| [SRS-003](specs/SRS-003%20Application%20Versioning.ru.md) | Application Versioning | P2 | Developer, DevOps |
| [SRS-004](specs/SRS-004%20Expose%20Application%20Version.ru.md) | Expose Application Version | P2 | Developer |
| [SRS-005](specs/SRS-005%20Liveness%20Probes.ru.md) | Liveness Probes | P1 | Developer, SRE |
| [SRS-006](specs/SRS-006%20Liveness%20Probes%20Command.ru.md) | Liveness Probes Command | P2 | Developer, SRE |
| [SRS-007](specs/SRS-007%20Circuit%20Breaker.ru.md) | Circuit Breaker | P1 | Developer, Architect |
| [SRS-008](specs/SRS-008%20Graceful%20Shutdown.ru.md) | Graceful Shutdown | P1 | Developer |
| [SRS-009](specs/SRS-009%20Blocking%20Timeouts.ru.md) | Blocking Timeouts | P1 | Developer |
| [SRS-010](specs/SRS-010%20Request%20Idempotency.ru.md) | Request Idempotency | P1 | Developer, Architect |
| [SRS-011](specs/SRS-011%20Deadline%20Propagation.ru.md) | Deadline Propagation | P2 | Developer, Architect |
| [SRS-012](specs/SRS-012%20Stand-Independent%20Images.ru.md) | Stand-Independent Images | P1 | Developer, DevOps |
| [SRS-013](specs/SRS-013%20Retry%20Pattern.ru.md) | Retry Pattern | P1 | Developer |
| [SRS-014](specs/SRS-014%20Fallback.ru.md) | Fallback | P2 | Developer, Architect |
| [SRS-015](specs/SRS-015%20Bulkhead%20Pattern.ru.md) | Bulkhead Pattern | P2 | Developer, Architect |
| [SRS-016](specs/SRS-016%20Rate%20Limiting.ru.md) | Rate Limiting | P1 | Developer, Architect |
| [SRS-017](specs/SRS-017%20Database%20Connection%20Pooling.ru.md) | Database Connection Pooling | P1 | Developer, DBA |
| [SRS-018](specs/SRS-018%20Secrets%20Management.ru.md) | Secrets Management | P1 | Developer, DevOps, Security |
| [SRS-019](specs/SRS-019%20Service%20Authentication.ru.md) | Service Authentication | P1 | Developer, Security |
| [SRS-020](specs/SRS-020%20Authorization%20Pattern.ru.md) | Authorization Pattern | P1 | Developer, Security |

---

## SRP - Паттерны (9)

Архитектурные паттерны и проверенные проектные решения для типичных задач.

| ID | Название | Приоритет | Роли |
|----|----------|-----------|------|
| [SRP-001](specs/SRP-001%20Jobs%20Management.ru.md) | Jobs Management | P2 | Developer, Architect |
| [SRP-002](specs/SRP-002%20Scaling%20and%20State.ru.md) | Scaling and State | P1 | Architect, SRE |
| [SRP-003](specs/SRP-003%20Load%20Shedding.ru.md) | Load Shedding | P2 | Developer, Architect |
| [SRP-004](specs/SRP-004%20Distributed%20Caching.ru.md) | Distributed Caching | P2 | Developer, Architect |
| [SRP-005](specs/SRP-005%20Load%20Balancing.ru.md) | Load Balancing | P1 | Architect, SRE |
| [SRP-006](specs/SRP-006%20Auto-scaling.ru.md) | Auto-scaling | P2 | SRE, DevOps |
| [SRP-007](specs/SRP-007%20API%20Gateway.ru.md) | API Gateway | P2 | Architect, Developer |
| [SRP-008](specs/SRP-008%20Service%20Mesh.ru.md) | Service Mesh | P3 | Architect, SRE |
| [SRP-009](specs/SRP-009%20Materialized%20Views.ru.md) | Materialized Views & Caching | P2 | Developer, DBA |

---

## SRG - Руководства (19)

Руководства по реализации и практическая документация по конкретным темам.

| ID | Название | Приоритет | Роли |
|----|----------|-----------|------|
| [SRG-001](specs/SRG-001%20Metrics%20Collection.ru.md) | Metrics Collection | P1 | Developer, SRE |
| [SRG-002](specs/SRG-002%20Logging.ru.md) | Logging | P1 | Developer, SRE |
| [SRG-003](specs/SRG-003%20Error%20Tracking.ru.md) | Error Tracking | P1 | Developer, SRE |
| [SRG-004](specs/SRG-004%20Distributed%20Tracing.ru.md) | Distributed Tracing | P1 | Developer, SRE |
| [SRG-005](specs/SRG-005%20Alerting%20Rules.ru.md) | Alerting Rules | P1 | SRE, DevOps |
| [SRG-006](specs/SRG-006%20Audit%20Logging.ru.md) | Audit Logging | P2 | Developer, Security |
| [SRG-007](specs/SRG-007%20SLI%20SLO%20SLA.ru.md) | SLI, SLO, SLA | P1 | SRE, Product |
| [SRG-008](specs/SRG-008%20Synthetic%20Monitoring.ru.md) | Synthetic Monitoring | P2 | SRE |
| [SRG-009](specs/SRG-009%20Database%20Migrations.ru.md) | Database Migrations | P1 | Developer, DBA |
| [SRG-010](specs/SRG-010%20Backup%20Recovery.ru.md) | Backup & Recovery | P1 | DBA, SRE |
| [SRG-011](specs/SRG-011%20Feature%20Flags.ru.md) | Feature Flags | P2 | Developer, Product |
| [SRG-012](specs/SRG-012%20Security%20Monitoring.ru.md) | Security Monitoring | P1 | Security, SRE |
| [SRG-013](specs/SRG-013%20Database%20Replication.ru.md) | Database Replication | P1 | DBA |
| [SRG-014](specs/SRG-014%20Database%20Monitoring.ru.md) | Database Monitoring | P1 | DBA, SRE |
| [SRG-015](specs/SRG-015%20Query%20Optimization.ru.md) | Query Optimization | P2 | Developer, DBA |
| [SRG-016](specs/SRG-016%20Database%20Sharding.ru.md) | Database Sharding | P3 | DBA, Architect |
| [SRG-017](specs/SRG-017%20Table%20Partitioning.ru.md) | Table Partitioning | P2 | DBA |
| [SRG-018](specs/SRG-018%20Read%20Replicas.ru.md) | Read Replicas | P2 | DBA, Architect |
| [SRG-019](specs/SRG-019%20Database%20Performance%20Tuning.ru.md) | Database Performance Tuning | P2 | DBA |

---

## SRO - Операции (9)

Операционные практики, процессы и процедуры для эксплуатации надежных систем.

| ID | Название | Приоритет | Роли |
|----|----------|-----------|------|
| [SRO-001](specs/SRO-001%20On-Call%20Incident%20Response.ru.md) | On-Call & Incident Response | P1 | SRE, DevOps |
| [SRO-002](specs/SRO-002%20Chaos%20Engineering.ru.md) | Chaos Engineering | P2 | SRE |
| [SRO-003](specs/SRO-003%20Cost%20Optimization%20FinOps.ru.md) | Cost Optimization & FinOps | P2 | SRE, Finance |
| [SRO-004](specs/SRO-004%20Multi-Region%20DR.ru.md) | Multi-Region DR | P1 | Architect, SRE |
| [SRO-005](specs/SRO-005%20Capacity%20Planning.ru.md) | Capacity Planning | P2 | SRE, Architect |
| [SRO-006](specs/SRO-006%20Platform%20Engineering.ru.md) | Platform Engineering | P2 | SRE, Architect |
| [SRO-007](specs/SRO-007%20Database%20High%20Availability.ru.md) | Database High Availability | P1 | DBA, SRE |
| [SRO-008](specs/SRO-008%20Database%20Maintenance.ru.md) | Database Maintenance | P1 | DBA |
| [SRO-009](specs/SRO-009%20Database%20Security.ru.md) | Database Security | P1 | DBA, Security |

---

## Уровни приоритета

| Приоритет | Описание | Документов |
|-----------|----------|------------|
| **P1** | Критично - Обязательно реализовать | 32 |
| **P2** | Важно - Следует реализовать | 22 |
| **P3** | Желательно | 3 |

---

## Сводки по ролям

### Сводка для разработчика
Основные спецификации для разработчиков приложений:
- **Надежность**: [SRS-001](specs/SRS-001%20Stateless%20Services.ru.md), [SRS-007](specs/SRS-007%20Circuit%20Breaker.ru.md), [SRS-008](specs/SRS-008%20Graceful%20Shutdown.ru.md), [SRS-009](specs/SRS-009%20Blocking%20Timeouts.ru.md), [SRS-013](specs/SRS-013%20Retry%20Pattern.ru.md)
- **Наблюдаемость**: [SRG-001](specs/SRG-001%20Metrics%20Collection.ru.md), [SRG-002](specs/SRG-002%20Logging.ru.md), [SRG-003](specs/SRG-003%20Error%20Tracking.ru.md), [SRG-004](specs/SRG-004%20Distributed%20Tracing.ru.md)
- **Конфигурация**: [SRS-002](specs/SRS-002%20Environment%20Variables.ru.md), [SRS-003](specs/SRS-003%20Application%20Versioning.ru.md)
- **Безопасность**: [SRS-018](specs/SRS-018%20Secrets%20Management.ru.md), [SRS-019](specs/SRS-019%20Service%20Authentication.ru.md), [SRS-020](specs/SRS-020%20Authorization%20Pattern.ru.md)

### Сводка для архитектора
Ключевые паттерны для системных архитекторов:
- **Отказоустойчивость**: [SRS-007](specs/SRS-007%20Circuit%20Breaker.ru.md), [SRS-015](specs/SRS-015%20Bulkhead%20Pattern.ru.md), [SRS-014](specs/SRS-014%20Fallback.ru.md), [SRP-003](specs/SRP-003%20Load%20Shedding.ru.md)
- **Масштабируемость**: [SRP-002](specs/SRP-002%20Scaling%20and%20State.ru.md), [SRP-005](specs/SRP-005%20Load%20Balancing.ru.md), [SRP-006](specs/SRP-006%20Auto-scaling.ru.md)
- **Интеграция**: [SRP-007](specs/SRP-007%20API%20Gateway.ru.md), [SRP-008](specs/SRP-008%20Service%20Mesh.ru.md)

### Сводка для SRE
Документация по операционной деятельности:
- **Мониторинг**: [SRG-001](specs/SRG-001%20Metrics%20Collection.ru.md), [SRG-005](specs/SRG-005%20Alerting%20Rules.ru.md), [SRG-007](specs/SRG-007%20SLI%20SLO%20SLA.ru.md), [SRG-008](specs/SRG-008%20Synthetic%20Monitoring.ru.md)
- **Управление инцидентами**: [SRO-001](specs/SRO-001%20On-Call%20Incident%20Response.ru.md), [SRO-002](specs/SRO-002%20Chaos%20Engineering.ru.md)
- **Надежность**: [SRO-004](specs/SRO-004%20Multi-Region%20DR.ru.md), [SRO-005](specs/SRO-005%20Capacity%20Planning.ru.md)

### Сводка для DBA
Ресурсы по администрированию баз данных:
- **Производительность**: [SRG-015](specs/SRG-015%20Query%20Optimization.ru.md), [SRG-019](specs/SRG-019%20Database%20Performance%20Tuning.ru.md), [SRP-009](specs/SRP-009%20Materialized%20Views.ru.md)
- **Доступность**: [SRG-013](specs/SRG-013%20Database%20Replication.ru.md), [SRO-007](specs/SRO-007%20Database%20High%20Availability.ru.md), [SRG-018](specs/SRG-018%20Read%20Replicas.ru.md)
- **Операции**: [SRG-009](specs/SRG-009%20Database%20Migrations.ru.md), [SRG-010](specs/SRG-010%20Backup%20Recovery.ru.md), [SRO-008](specs/SRO-008%20Database%20Maintenance.ru.md)
- **Безопасность**: [SRO-009](specs/SRO-009%20Database%20Security.ru.md)
- **Масштабирование**: [SRG-016](specs/SRG-016%20Database%20Sharding.ru.md), [SRG-017](specs/SRG-017%20Table%20Partitioning.ru.md)

---

## Отраслевые стандарты и ресурсы

- [Google SRE Books](https://sre.google/books/)
- [The Twelve-Factor App](https://12factor.net/)
- [Cloud Native Computing Foundation](https://www.cncf.io/)
- [OpenTelemetry](https://opentelemetry.io/)
- [Prometheus](https://prometheus.io/)

---

## Лицензия

Документация предоставляется под лицензией MIT.
