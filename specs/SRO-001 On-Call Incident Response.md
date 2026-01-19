# SRO-001 On-Call & Incident Response

On-Call & Incident Management is the practice of organizing 24/7 support for services, rapid incident response, and their effective resolution to minimize impact on users.

---

## On-Call Organization

### Team Structure

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
      shift_duration: "12_hours"  # or 24 hours
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

### On-Call Schedule

**Weekly Rotation:**
```
Week 1: Alice (Primary), Bob (Secondary)
Week 2: Charlie (Primary), David (Secondary)
Week 3: Eve (Primary), Frank (Secondary)
Week 4: Grace (Primary), Henry (Secondary)
```

**Daily Rotation:**
```
Monday: Alice (Day), Bob (Night)
Tuesday: Charlie (Day), David (Night)
Wednesday: Eve (Day), Frank (Night)
Thursday: Grace (Day), Henry (Night)
Friday: Ivan (Day), Judy (Night)
Saturday: Kevin (Day), Laura (Night)
Sunday: Mike (Day), Nancy (Night)
```

**Tools:**
- PagerDuty
- VictorOps / Splunk On-Call
- Opsgenie
- Grafana OnCall

---

## On-Call Process

### Before Shift

```bash
#!/bin/bash
# pre-shift-checklist.sh

echo "📋 Pre-Shift Checklist"

# 1. Check work environment
Check laptop, internet, phone
Verify VPN access
Test access to monitoring systems

# 2. Review recent incidents
# - Review last 7 days of alerts
# - Check unresolved issues
# - Review trending problems

# 3. Sync with previous on-call
# - Pass context
# - Check action items
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
echo "\n=== Recent Alerts (Last Hour) ==="
curl -s https://datadog.com/api/alerts | jq '.alerts[] | select(.date>")'

# System Health
echo "\n=== System Health ==="
curl -s https://grafana.com/api/dashboards/health | jq '.'

# Escalation Policy
echo "\n=== Current On-Call ==="
curl -s https://pagerduty.com/api/oncalls | jq '.oncalls[].user.summary'
```

### After Shift

```bash
#!/bin/bash
# post-shift-handover.sh

echo "📤 Shift Handover Summary"
echo "Date: $(date)"

# Incident Summary
echo "\n=== Incidents During Shift ==="
echo "Total: 3"
echo "Sev1: 0"
echo "Sev2: 1 (Payment gateway latency)"
echo "Sev3: 2 (Database backup delay, Cache refresh)"

# Action Items
echo "\n=== Action Items ==="
echo "1. Fix database connection pooling (Jira TICKET-123)"
echo "2. Update runbook for cache issues (assigned: Bob)"
echo "3. Review payment service alerts (trending)"

# Notes for next shift
echo "\n=== Notes for Next On-Call ==="
echo "- Database connections spiking around 15:00 UTC"
echo "- New deployment scheduled for tomorrow"
echo "- Increased latency in EU region"
```

---

## Incident Severity Levels

### Severity 1 (Sev1) - Critical

```yaml
sev1:
  definition: "Complete service unavailability for most users"
  examples:
    - website_down: "Website is completely not working"
    - payment_system_down: "Unable to process payments"
    - data_loss: "Data loss"
    - major_security_incident: "System breach"

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
  definition: "Significant service degradation or important feature unavailability"
  examples:
    - partial_outage: "50% of requests fail"
    - severe_performance: "P99 latency > 5000ms"
    - backup_failures: "Backup failures"
    - degraded_functionality: "Key features not working"

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
  definition: "Minimal user impact or workaround available"
  examples:
    - minor_degradation: "10% of requests slower"
    - non_critical_functionality: "'Nice to have' feature"
    - scheduled_task_failure: "Background task errors"
    - monitoring_gap: "Data not being collected"

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
  definition: "Cosmetic or informational issues, no user impact"
  examples:
    - cosmetic_issues: "Wrong color in UI"
    - doc_updates: "Documentation inaccuracy"
    - low_priority_alerts: "Monitoring warnings"
    - questions: "Technical questions"

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

## Incident Lifecycle

```
Alert/Detection → Triage → Response → Investigation → Resolution → Post-Incident
```

### 1. Detection (Detection)

```python
class IncidentDetection:
    def __init__(self, monitoring_tools):
        self.monitoring = monitoring_tools

    def detect_incident(self):
        """Detect incidents from different sources"""

        # From monitoring (Prometheus, Datadog)
        alerts = self.monitoring.get_active_alerts(
            severity=['warning', 'critical']
        )

        # From logs (ERROR, FATAL)
        log_errors = self.monitoring.get_log_spikes(
            level=['error', 'fatal'],
            threshold=100  # errors per minute
        )

        # From user complaints
        user_reports = self.monitoring.get_user_complaints(
            source: ['support_tickets', 'twitter', 'status_page']
        )

        # Correlation - merge related problems
        incidents = self.correlate_signals(
            alerts + log_errors + user_reports
        )

        return incidents
```

### 2. Triage (Assessment and Classification)

```python
class IncidentTriage:
    def triage_incident(self, incident):
        """Assess incident and determine severity"""

        # Impact assessment
        impact = self.calculate_impact(
            affected_users=incident.affected_users,
            affected_requests=incident.error_rate,
            functionality=incident.affected_components
        )

        # Severity classification
        if impact == 'total_outage' or incident.security_breach:
            severity = 1
        elif impact == 'significant_degradation' or incident.revenue_impact:
            severity = 2
        elif impact == 'minor_degradation':
            severity = 3
        else:
            severity = 4

        # Workflow based on severity
        self.start_incident_workflow(severity, incident)

        return {
            'severity': severity,
            'impact': impact,
            'incident_id': self.generate_id()
        }
```

### 3. Response (Reaction)

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

### 4. Investigation (Investigation)

```python
class IncidentInvestigation:
    def investigate(self, incident):
        """Investigate incident"""
        steps = []

        # Collect data
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

        # Analysis
        possible_causes = self.analyze_patterns(steps)

        # Testing hypotheses
        for cause in possible_causes:
            verified = self.test_hypothesis(cause, incident)
            if verified:
                incident.root_cause = cause
                break

        return incident
```

### 5. Resolution (Solution)

```bash
#!/bin/bash
# incident-resolution.sh

echo "🔧 Incident Resolution Process"

# 1. Test the fix
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

# 3. Monitor after deployment
echo "3. Monitoring after fix..."
./monitor-deployment.sh --service=$AFFECTED_SERVICE --duration=15m

# 4. Verify recovery
echo "4. Verifying recovery..."
./verify-recovery.sh --incident=$INCIDENT_ID

# 5. Communication
echo "5. Notifying stakeholders..."
./notify.sh --incident=$INCIDENT_ID --message="resolved"
```

### 6. Post-Incident (Post-Incident Analysis)

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

## Tools

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
        """Create incident in all systems"""

        # Create Slack channel
        channel = self.slack.create_channel(
            name=f"inc-{incident.id}",
            purpose=f"Incident: {incident.title}"
        )

        # Create PagerDuty incident
        pd_incident = self.pagerduty.create_incident(
            title=incident.title,
            urgency=incident.severity,
            assignee=incident.commander
        )

        # Create Status Page incident
        if incident.severity <= 2:
            sp_incident = self.status_page.create_incident(
                name=incident.title,
                impact=incident.impact_level,
                components=incident.affected_components
            )

        # Log in system
        return IncidentRecord(
            id=incident.id,
            slack_channel=channel,
            pagerduty_id=pd_incident.id,
            status_page_id=sp_incident.id if sp_incident else None
        )
```

---

## On-Call and Incident Metrics

```python
ON_CALL_METRICS = {
    # Time to detection
    'incident_detection_time': 'Time from problem to alert',

    # Time to response
    'incident_acknowledgement_time': 'Time to acknowledge',
    'on_call_response_time': 'On-call response time',

    # Time to resolution
    'incident_resolution_time': 'Time to full resolution',
    'incident_mitigation_time': 'Time to impact mitigation (MTTM)',

    # Performance
    'incident_count_by_severity': 'Incident count by severity',
    'on_call_alerts_per_shift': 'Alerts per shift',
    'false_positive_rate': 'False positive rate',

    # Workload
    'on_call_interruptions': 'Interruptions per shift',
    'after_hours_pages': 'Pages outside business hours',
    'weekend_incidents': 'Weekend incidents',

    # Quality
    'postmortems_completed': 'Postmortems completed',
    'action_items_completed': 'Action items completed',
    'recurring_incidents': 'Recurring incidents'
}
```

---

*On-Call & Incident Management - 24/7 support and incident management practice*
