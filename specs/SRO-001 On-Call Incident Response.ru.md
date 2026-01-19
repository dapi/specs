# SRO-001 On-Call & Incident Response (Дежурства и управление инцидентами)

On-Call & Incident Management - это практика организации 24/7 поддержки сервисов, быстрого реагирования на инциденты и их эффективного разрешения для минимизации воздействия на пользователей.

---

## On-Call Организация

### Структура команд

```yaml
sre_organization:
  on_call_structure:
    primary_on_call:
      title: "Primary On-Call Engineer"
      responsibilities:
        - first_response_to_alerts
        - initial_incident_triage
        - communication_with_stakeholders
        - escalation_decisions
      shift_duration: "12_hours"  # или 24 часа
      timezone: "primary"

    secondary_on_call:
      title: "Secondary On-Call Engineer"
      responsibilities:
        - escalation_from_primary
        - critical_incidents
        - complex_troubleshooting
        - technical_decisions
      availability: "15_minutes_to_respond"

    incident_commander:
      title: "Incident Commander (for major incidents)"
      responsibilities:
        - overall_incident_coordination
        - stakeholder_communication
        - resource_coordination
        - decision_making
      activated: "for_sev1_and_sev2_incidents"

    support_engineer:
      title: "Support Engineer (optional)"
      responsibilities:
        - l1_l2_support_tasks
        - routine_operations
        - documentation
      escalation_path: "primary -> secondary"
```

### График дежурств

**Weekly Rotation:**
```
Неделя 1: Alice (Primary), Bob (Secondary)
Неделя 2: Charlie (Primary), David (Secondary)
Неделя 3: Eve (Primary), Frank (Secondary)
Неделя 4: Grace (Primary), Henry (Secondary)
```

**Daily Rotation:**
```
Понедельник: Alice (Day), Bob (Night)
Вторник: Charlie (Day), David (Night)
Среда: Eve (Day), Frank (Night)
Четверг: Grace (Day), Henry (Night)
Пятница: Ivan (Day), Judy (Night)
Суббота: Kevin (Day), Laura (Night)
Воскресенье: Mike (Day), Nancy (Night)
```

**Tools:**
- PagerDuty
- VictorOps / Splunk On-Call
- Opsgenie
- Grafana OnCall

---

## Процесс On-Call

### Before Shift

```bash
#!/bin/bash
# pre-shift-checklist.sh

echo "📋 Pre-Shift Checklist"

# 1. Проверить рабочее окружение
Check laptop, internet, phone
Verify VPN access
Test access to monitoring systems

# 2. Review recent incidents
# - Посмотреть последние 7 дней alerts
# - Проверить unresolved issues
# - Review trending problems

# 3. Синхронизация с предыдущим on-call
# - Передать контекст
# - Проверить action items
# - Review runbooks

# 4. Set up notifications
# - Check pager/mobile
# - Verify Slack/email
# - Test escalation chains
```

### During Shift

```bash
#!/bin/bash
# on-call-whats-happening.sh

echo "📊 Current Situation"

# Active Incidents
echo "=== Active Incidents ==="
curl -s https://pagerduty.com/api/incidents | jq '.incidents[] | select(.status=="triggered")'

# Recent Alerts
echo "\\n=== Recent Alerts (Last Hour) ==="
curl -s https://datadog.com/api/alerts | jq '.alerts[] | select(.date>")'

# System Health
echo "\\n=== System Health ==="
curl -s https://grafana.com/api/dashboards/health | jq '.'

# Escalation Policy
echo "\\n=== Current On-Call ==="
curl -s https://pagerduty.com/api/oncalls | jq '.oncalls[].user.summary'
```

### After Shift

```bash
#!/bin/bash
# post-shift-handover.sh

echo "📤 Shift Handover Summary"
echo "Date: $(date)"

# Incident Summary
echo "\\n=== Incidents During Shift ==="
echo "Total: 3"
echo "Sev1: 0"
echo "Sev2: 1 (Payment gateway latency)"
echo "Sev3: 2 (Database backup delay, Cache refresh)")

# Action Items
echo "\\n=== Action Items ==="
echo "1. Fix database connection pooling (Jira TICKET-123)"
echo "2. Update runbook for cache issues (assigned: Bob)"
echo "3. Review payment service alerts (trending)"

# Notes for next shift
echo "\\n=== Notes for Next On-Call ==="
echo "- Database connections spiking around 15:00 UTC"
echo "- New deployment scheduled for tomorrow"
echo "- Increased latency in EU region"
```

---

## Уровни критичности инцидентов

### Severity 1 (Sev1) - Critical

```yaml
sev1:
  definition: "Полная недоступность сервиса для большинства пользователей"
  examples:
    - website_down: "Сайт полностью не работает"
    - payment_system_down: "Невозможно принимать платежи"
    - data_loss: "Потеря данных"
    - major_security_incident: "Взлом системы"

  response:
    response_time: "immediate"  # < 5 minutes
    resolution_target: "4_hours"
    participants:
      - primary_on_call
      - secondary_on_call
      - incident_commander
      - engineering_manager
      - stakeholders

  communication:
    frequency: "30_minutes_to_stakeholders"
    channels: ["incident_management_tool", "slack_war_room", "email", "conference_bridge"]
    escalation: "immediate_vp_eng"

  postmortem:
    required: true
    timeline: "5_business_days"
```

### Severity 2 (Sev2) - Major

```yaml
sev2:
  definition: "Значительная деградация сервиса или недоступность важной функции"
  examples:
    - partial_outage: "50% запросов завершаются с ошибками"
    - severe_performance: "P99 latency > 5000ms"
    - backup_failures: "Ошибки резервного копирования"
    - degraded_functionality: "Ключевые фичи не работают"

  response:
    response_time: "15_minutes"
    resolution_target: "8_hours"
    participants:
      - primary_on_call
      - secondary_on_call
      - incident_commander
      - relevant_team_lead

  communication:
    frequency: "1_hour"
    channels: ["incident_tool", "slack"]
    escalation: "on_slo_breach"

  postmortem:
    required: true
    timeline: "10_business_days"
```

### Severity 3 (Sev3) - Moderate

```yaml
sev3:
  definition: "Минимальное влияние на пользователей или доступно Workaround"
  examples:
    - minor_degradation: "10% запросов медленнее"
    - non_critical_functionality: "Фича категории 'nice to have'"
    - scheduled_task_failure: "Ошибка в фоновой задаче"
    - monitoring_gap: "Данные не собираются"

  response:
    response_time: "30_minutes"
    resolution_target: "24_hours"
    participants:
      - primary_on_call
      - optional: secondary_on_call

  communication:
    frequency: "as_needed"
    channels: ["incident_tool"]
    escalation: "if_not_resolved_in_24h"

  postmortem:
    required: false
    timeline: "optional"
```

### Severity 4 (Sev4) - Low

```yaml
sev4:
  definition: "Косметические или информационные вопросы, без влияния на пользователей"
  examples:
    - cosmetic_issues: "Неправильный цвет в UI"
    - doc_updates: "Неточность в документации"
    - low_priority_alerts: "Warnings из monitoring"
    - questions: "Технические вопросы"

  response:
    response_time: "business_hours"
    resolution_target: "72_hours"
    participants:
      - on_call_as_needed

  communication:
    frequency: "none"
    channels: ["ticket_system"]
    escalation: "rare"

  postmortem:
    required: false
```

---

## Жизненный цикл инцидента

```
Alert/Detection → Triage → Response → Investigation → Resolution → Post-Incident
```

### 1. Detection (Обнаружение)

```python
class IncidentDetection:
    def __init__(self, monitoring_tools):
        self.monitoring = monitoring_tools

    def detect_incident(self):
        """Детектируем инциденты из разных источников"""

        # Из monitoring (Prometheus, Datadog)
        alerts = self.monitoring.get_active_alerts(
            severity=['warning', 'critical']
        )

        # Из логов (ERROR, FATAL)
        log_errors = self.monitoring.get_log_spikes(
            level=['error', 'fatal'],
            threshold=100  # ошибок в минуту
        )

        # Из пользовательских жалоб
        user_reports = self.monitoring.get_user_complaints(
            source: ['support_tickets', 'twitter', 'status_page']
        )

        # Корреляция - объединяем связанные проблемы
        incidents = self.correlate_signals(
            alerts + log_errors + user_reports
        )

        return incidents
```

### 2. Triage (Оценка и классификация)

```python
class IncidentTriage:
    def triage_incident(self, incident):
        """Оцениваем инцидент и определяем severity"""

        # Оценка impact
        impact = self.calculate_impact(
            affected_users=incident.affected_users,
            affected_requests=incident.error_rate,
            functionality=incident.affected_components
        )

        # Классификация severity
        if impact == 'total_outage' or incident.security_breach:
            severity = 1
        elif impact == 'significant_degradation' or incident.revenue_impact:
            severity = 2
        elif impact == 'minor_degradation':
            severity = 3
        else:
            severity = 4

        # Воркфлоу в зависимости от severity
        self.start_incident_workflow(severity, incident)

        return {
            'severity': severity,
            'impact': impact,
            'incident_id': self.generate_id()
        }
```

### 3. Response (Реакция)

```yaml
incident_response:
  declare_incident:
    actions:
      - create_incident_channel: "#inc-2024-001"
      - start_incident_timer: true
      - assign_incident_commander: based_on_severity
      - notify_oncall: primary_and_secondary
      - page_if_sev1_or_sev2: true

  initial_communication:
    template: |
      🚨 INCIDENT {{ incident.id }} - {{ incident.severity }}

      Title: {{ incident.title }}
      Started: {{ incident.started_at }}
      Affected: {{ incident.affected_services }}
      Commander: {{ incident.commander }}
      Channel: {{ incident.slack_channel }}

      https://incidents.example.com/{{ incident.id }}

    recipients:
      - engineering_team_slack
      - oncall_pagerduty
      - stakeholders_if_sev1
      - status_page_if_customer_impact
```

### 4. Investigation (Расследование)

```python
class IncidentInvestigation:
    def investigate(self, incident):
        """Исследуем инцидент"""
        steps = []

        # Собираем данные
        steps.append(self.collect_metrics(
            timeframe: incident.timeframe,
            metrics: ['error_rate', 'latency', 'throughput']
        ))

        steps.append(self.collect_logs(
            services: incident.affected_services,
            level: ['error', 'warn', 'fatal'],
            timeframe: incident.timeframe
        ))

        steps.append(self.collect_traces(
            error_spans: incident.error_spans,
            timeframe: incident.timeframe
        ))

        # Анализ
        possible_causes = self.analyze_patterns(steps)

        # Testing hypotheses
        for cause in possible_causes:
            verified = self.test_hypothesis(cause, incident)
            if verified:
                incident.root_cause = cause
                break

        return incident
```

### 5. Resolution (Решение)

```bash
#!/bin/bash
# incident-resolution.sh

echo "🔧 Incident Resolution Process"

# 1. Тестируем фикс
echo "1. Testing fix in staging..."
./test-fix.sh --environment=staging

if [ $? -eq 0 ]; then
    echo "✅ Fix validated in staging"
else
    echo "❌ Fix failed in staging"
    exit 1
fi

# 2. Deploy fix
echo "2. Deploying fix to production..."
./deploy.sh --service=$AFFECTED_SERVICE --check-only

# 3. Мониторинг после деплоя
echo "3. Monitoring after fix..."
./monitor-deployment.sh --service=$AFFECTED_SERVICE --duration=15m

# 4. Проверка recovery
echo "4. Verifying recovery..."
./verify-recovery.sh --incident=$INCIDENT_ID

# 5. Communication
echo "5. Notifying stakeholders..."
./notify.sh --incident=$INCIDENT_ID --message="resolved"
```

### 6. Post-Incident (Пост-инцидентный анализ)

```markdown
# Incident Report Template

## Summary
- **Incident ID:** INC-2024-001
- **Date:** 2024-01-15 14:30 UTC
- **Duration:** 2 hours 15 minutes
- **Severity:** Sev2
- **Affected:** Payment service

## Timeline
- 14:30: Alert triggered (error rate > 50%)
- 14:32: On-call acknowledged
- 14:35: Incident declared, war room opened
- 14:45: Root cause identified (DB connection pool exhaustion)
- 15:00: Mitigation deployed (increase pool size)
- 16:45: Service fully recovered

## Root Cause
Database connection pool size insufficient after deployment of v2.1

## Impact
- 30% of payment transactions failed
- ~$50,000 revenue impact
- 500+ affected customers

## Lessons Learned
1. Load testing didn't simulate real traffic patterns
2. Connection pool monitoring missing
3. Rollback procedure unclear

## Action Items
- [ ] Add connection pool metrics (Owner: Alice, Due: 2024-01-22)
- [ ] Improve load testing scenarios (Owner: Bob, Due: 2024-01-29)
- [ ] Document rollback procedures (Owner: Charlie, Due: 2024-01-20)
```

---

## Communication

### Incident Communication Matrix

```yaml
communication_channels:
  internal:
    team:
      channel: slack #incident-war-room
      frequency: real_time
      content: technical_details

    company:
      channel: slack #general-updates
      frequency: hourly
      content: high_level_status

  external:
    customers:
      channel: status_page
      frequency: every_30_minutes_sev1
      content: impact_summary

    enterprise_clients:
      channel: email + phone
      frequency: every_15_minutes_sev1
      content: detailed_status

    partners:
      channel: status_page + email
      frequency: hourly
      content: service_impact
```

### Status Page Updates

```markdown
# Status Page Template

## In Progress (14:45 UTC)

🟡 **Performance Degradation - Payment Service**

**Status:** Investigating
**Started:** 14:30 UTC
**Impact:** Some users experiencing slow payment processing
**ETA:** Under investigation

We're currently investigating reports of slow payment processing.
Our engineering team is working to identify the root cause.

Updates will be posted here every 30 minutes.

---

## Update (15:15 UTC)

**Status:** Identified

We've identified the issue: Database connection pool exhaustion.

**Mitigation:** We're increasing connection pool size
**Expected Resolution:** 15:45 UTC

---

## Resolved (16:45 UTC)

**Status:** Fully Resolved

Service is fully recovered. All payment processing back to normal.

**Root Cause:** Database connection pool insufficient after new deployment
**Timeline:** 2 hours 15 minutes
**Next Steps:** Post-incident review scheduled for tomorrow
```

---

## Инструменты

### PagerDuty

```yaml
# escalation_policy.yaml
escalation_policy:
  name: "Critical Service On-Call"
  repeat: 3

  escalation_rules:
    - level: 1
      timeout_minutes: 5
      targets:
        - type: user
          id: primary_oncall

    - level: 2
      timeout_minutes: 10
      targets:
        - type: user
          id: secondary_oncall
        - type: schedule
          id: engineering_team

    - level: 3
      timeout_minutes: 15
      targets:
        - type: user
          id: engineering_manager
        - type: user
          id: vp_engineering
```

### Incident Management Tool

```python
class IncidentManagement:
    def __init__(self, slack, pagerduty, status_page):
        self.slack = slack
        self.pagerduty = pagerduty
        self.status_page = status_page

    def create_incident(self, incident):
        """Создаем инцидент в всех системах"""

        # Создаем Slack channel
        channel = self.slack.create_channel(
            name=f"inc-{incident.id}",
            purpose=f"Incident: {incident.title}"
        )

        # Создаем PagerDuty incident
        pd_incident = self.pagerduty.create_incident(
            title=incident.title,
            urgency=incident.severity,
            assignee=incident.commander
        )

        # Создаем Status Page incident
        if incident.severity <= 2:
            sp_incident = self.status_page.create_incident(
                name=incident.title,
                impact=incident.impact_level,
                components=incident.affected_components
            )

        # Логируем в системе
        return IncidentRecord(
            id=incident.id,
            slack_channel=channel,
            pagerduty_id=pd_incident.id,
            status_page_id=sp_incident.id if sp_incident else None
        )
```

---

## Метрики on-call и инцидентов

```python
ON_CALL_METRICS = {
    # Time to detection
    'incident_detection_time': 'Время от проблемы до алерта',

    # Time to response
    'incident_acknowledgement_time': 'Время до подтверждения',
    'on_call_response_time': 'Время реакции on-call',

    # Time to resolution
    'incident_resolution_time': 'Время до полного решения',
    'incident_mitigation_time': 'Время до минимизации влияния (MTTM)',

    # Производительность
    'incident_count_by_severity': 'Количество инцидентов по severity',
    'on_call_alerts_per_shift': 'Алертов за смену',
    'false_positive_rate': 'Доля false positive алертов',

    # Workload
    'on_call_interruptions': 'Прерываний за смену',
    'after_hours_pages': 'Пейджей вне рабочих часов',
    'weekend_incidents': 'Инцидентов в выходные',

    # Качество
    'postmortems_completed': 'Постмортемов завершено',
    'action_items_completed': 'Action items выполнено',
    'recurring_incidents': 'Повторяющихся инцидентов'
}
```

---

*On-Call & Incident Management - практика 24/7 поддержки и управления инцидентами*
