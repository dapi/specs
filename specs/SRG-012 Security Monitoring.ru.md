# SRG-012 Мониторинг безопасности

| Поле | Значение |
|------|----------|
| **Статус** | APPROVED |
| **Версия** | 1.0 |
| **Приоритет** | P1 |
| **Сложность** | High |
| **Целевая аудитория** | Security Engineers, SRE, DevOps, Архитекторы |

## 1. Введение

### 1.1 Назначение

Данная спецификация определяет стандарты мониторинга безопасности в распределённых системах. Мониторинг безопасности обеспечивает видимость событий, связанных с безопасностью, позволяет обнаруживать угрозы и поддерживает реагирование на инциденты и требования соответствия нормативам.

### 1.2 Область применения

Спецификация охватывает:
- Архитектуру наблюдаемости безопасности
- Паттерны интеграции с SIEM
- Логирование безопасности и аудиторские следы
- Обнаружение угроз и аномалий
- Мониторинг безопасности во время выполнения
- Мониторинг соответствия нормативам
- Интеграцию с реагированием на инциденты

### 1.3 Связанные спецификации

- [SRG-002 Logging](SRG-002%20Logging.ru.md) - Общие стандарты логирования
- [SRG-003 Error Tracking](SRG-003%20Error%20Tracking.ru.md) - Мониторинг ошибок
- [SRG-005 Alerting Rules](SRG-005%20Alerting%20Rules.ru.md) - Управление алертами
- [SRO-004 Multi-Region DR](SRO-004%20Multi-Region%20DR.ru.md) - Аварийное восстановление

## 2. Архитектура наблюдаемости безопасности

### 2.1 Уровни мониторинга безопасности

```
┌─────────────────────────────────────────────────────────────────┐
│               Центр управления безопасностью (SOC)              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │    SIEM      │  │   SOAR       │  │  Threat Intelligence │  │
│  │  (Splunk,    │  │  (Автомат.   │  │   (Фиды, IOC)        │  │
│  │   Elastic)   │  │   ответ)     │  │                      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
└─────────┼─────────────────┼─────────────────────┼──────────────┘
          │                 │                     │
          ▼                 ▼                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Security Data Lake                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ События      │  │  Аудит       │  │  Сетевые потоки      │  │
│  │ безопасности │  │  логи        │  │  (Flow Data)         │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          ▲                 ▲                     ▲
          │                 │                     │
┌─────────┴─────────────────┴─────────────────────┴──────────────┐
│                    Уровень сбора данных                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Host-агенты  │  │  Сетевые     │  │  Инструментирование  │  │
│  │ (Falco,      │  │  сенсоры     │  │  приложений          │  │
│  │  OSSEC)      │  │  (Zeek,      │  │  (OWASP, WAF)        │  │
│  │              │  │   Suricata)  │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
          ▲                 ▲                     ▲
          │                 │                     │
┌─────────┴─────────────────┴─────────────────────┴──────────────┐
│                    Инфраструктура                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Kubernetes   │  │  Облачные    │  │  Приложения          │  │
│  │ узлы         │  │  сервисы     │  │  и сервисы           │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Эшелонированная защита (Defense in Depth)

| Уровень | Фокус мониторинга | Инструменты |
|---------|------------------|-------------|
| **Периметр** | DDoS, WAF события, логи файрвола | Cloudflare, AWS WAF, ModSecurity |
| **Сеть** | Анализ трафика, IDS/IPS | Zeek, Suricata, VPC Flow Logs |
| **Хост** | Системные вызовы, целостность файлов | Falco, OSSEC, Wazuh |
| **Контейнер** | Поведение во время выполнения, сканирование образов | Falco, Trivy, Aqua |
| **Приложение** | События аутентификации, злоупотребление API | Custom logging, OWASP ZAP |
| **Данные** | Паттерны доступа, шифрование | Database audit, KMS logs |

## 3. Стандарты логирования безопасности

### 3.1 Категории событий безопасности

```yaml
# Таксономия событий безопасности
security_events:
  authentication:  # Аутентификация
    - login_success         # Успешный вход
    - login_failure         # Неудачный вход
    - logout                # Выход
    - session_timeout       # Истечение сессии
    - mfa_challenge         # Запрос MFA
    - mfa_success           # Успешный MFA
    - mfa_failure           # Неудачный MFA
    - password_change       # Смена пароля
    - password_reset        # Сброс пароля

  authorization:  # Авторизация
    - access_granted        # Доступ предоставлен
    - access_denied         # Доступ запрещён
    - privilege_escalation  # Повышение привилегий
    - role_change           # Смена роли
    - permission_change     # Изменение разрешений

  data_access:  # Доступ к данным
    - data_read             # Чтение данных
    - data_write            # Запись данных
    - data_delete           # Удаление данных
    - data_export           # Экспорт данных
    - bulk_operation        # Массовая операция

  system:  # Системные события
    - service_start         # Запуск сервиса
    - service_stop          # Остановка сервиса
    - config_change         # Изменение конфигурации
    - secret_access         # Доступ к секретам
    - certificate_operation # Операции с сертификатами

  network:  # Сетевые события
    - connection_established  # Соединение установлено
    - connection_blocked      # Соединение заблокировано
    - dns_query               # DNS-запрос
    - tls_error               # Ошибка TLS

  threat:  # Угрозы
    - malware_detected      # Обнаружено вредоносное ПО
    - intrusion_attempt     # Попытка вторжения
    - anomaly_detected      # Обнаружена аномалия
    - policy_violation      # Нарушение политики
```

### 3.2 Формат логов безопасности

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "event_type": "authentication",
  "event_name": "login_failure",
  "severity": "warning",
  "source": {
    "service": "auth-service",
    "instance": "auth-service-7b9f5d4c8-x2k4m",
    "version": "1.2.3"
  },
  "actor": {
    "user_id": "user-123",
    "username": "john.doe@example.com",
    "session_id": "sess-abc123",
    "ip_address": "192.168.1.100",
    "user_agent": "Mozilla/5.0...",
    "geo_location": {
      "country": "US",
      "city": "New York",
      "coordinates": [40.7128, -74.0060]
    }
  },
  "target": {
    "resource_type": "user_account",
    "resource_id": "user-123",
    "action": "authenticate"
  },
  "outcome": {
    "status": "failure",
    "reason": "invalid_password",
    "attempt_count": 3
  },
  "context": {
    "request_id": "req-xyz789",
    "trace_id": "trace-abc123",
    "client_fingerprint": "fp-12345"
  },
  "risk_score": 75,
  "tags": ["brute_force_candidate", "unusual_location"]
}
```

### 3.3 Реализация аудиторского следа

```go
package audit

import (
    "context"
    "encoding/json"
    "time"
)

type AuditEvent struct {
    ID            string                 `json:"id"`
    Timestamp     time.Time              `json:"timestamp"`
    EventType     string                 `json:"event_type"`
    EventName     string                 `json:"event_name"`
    Severity      string                 `json:"severity"`
    Actor         Actor                  `json:"actor"`
    Target        Target                 `json:"target"`
    Outcome       Outcome                `json:"outcome"`
    Changes       []Change               `json:"changes,omitempty"`
    Context       map[string]interface{} `json:"context"`
    Metadata      map[string]interface{} `json:"metadata"`
}

type Actor struct {
    UserID      string            `json:"user_id"`
    Username    string            `json:"username"`
    Role        string            `json:"role"`
    IPAddress   string            `json:"ip_address"`
    UserAgent   string            `json:"user_agent"`
    SessionID   string            `json:"session_id"`
    ServiceName string            `json:"service_name,omitempty"`
    Attributes  map[string]string `json:"attributes,omitempty"`
}

type Target struct {
    ResourceType string `json:"resource_type"`
    ResourceID   string `json:"resource_id"`
    ResourceName string `json:"resource_name,omitempty"`
    Action       string `json:"action"`
}

type Outcome struct {
    Status     string `json:"status"` // success, failure, partial
    Reason     string `json:"reason,omitempty"`
    ErrorCode  string `json:"error_code,omitempty"`
}

type Change struct {
    Field    string      `json:"field"`
    OldValue interface{} `json:"old_value"`
    NewValue interface{} `json:"new_value"`
}

type AuditLogger interface {
    Log(ctx context.Context, event AuditEvent) error
    Query(ctx context.Context, filter AuditFilter) ([]AuditEvent, error)
}

// ImmutableAuditLogger обеспечивает невозможность модификации аудит-логов
type ImmutableAuditLogger struct {
    writer    AuditWriter
    hasher    HashChain
    encryptor Encryptor
}

func (l *ImmutableAuditLogger) Log(ctx context.Context, event AuditEvent) error {
    // Добавляем временную метку и уникальный ID
    event.ID = generateUUID()
    event.Timestamp = time.Now().UTC()

    // Сериализуем событие
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }

    // Добавляем в цепочку хешей для обеспечения целостности
    hash := l.hasher.AddBlock(data)

    // Шифруем чувствительные поля
    encryptedData, err := l.encryptor.Encrypt(data)
    if err != nil {
        return err
    }

    // Записываем в неизменяемое хранилище
    return l.writer.Write(ctx, AuditRecord{
        Data:     encryptedData,
        Hash:     hash,
        PrevHash: l.hasher.PreviousHash(),
    })
}

// HashChain обеспечивает криптографическую проверку целостности
type HashChain struct {
    prevHash []byte
}

func (h *HashChain) AddBlock(data []byte) []byte {
    hash := sha256.Sum256(append(h.prevHash, data...))
    h.prevHash = hash[:]
    return hash[:]
}

func (h *HashChain) Verify(records []AuditRecord) bool {
    var prevHash []byte
    for _, record := range records {
        expectedHash := sha256.Sum256(append(prevHash, record.Data...))
        if !bytes.Equal(expectedHash[:], record.Hash) {
            return false
        }
        prevHash = record.Hash
    }
    return true
}
```

### 3.4 Обогащение событий безопасности

```python
from dataclasses import dataclass
from typing import Optional, Dict, List
import geoip2.database
import hashlib

@dataclass
class EnrichedSecurityEvent:
    original_event: Dict
    geo_data: Optional[Dict]
    threat_intel: Optional[Dict]
    user_context: Optional[Dict]
    risk_score: int
    tags: List[str]

class SecurityEventEnricher:
    def __init__(
        self,
        geoip_db_path: str,
        threat_intel_client,
        user_service_client
    ):
        self.geoip_reader = geoip2.database.Reader(geoip_db_path)
        self.threat_intel = threat_intel_client
        self.user_service = user_service_client

    async def enrich(self, event: Dict) -> EnrichedSecurityEvent:
        """Обогащение события безопасности дополнительным контекстом"""

        tags = []
        risk_score = 0

        # Гео-обогащение
        geo_data = None
        ip_address = event.get('actor', {}).get('ip_address')
        if ip_address:
            geo_data = self._enrich_geo(ip_address)
            if geo_data and geo_data.get('is_anonymous_proxy'):
                tags.append('anonymous_proxy')
                risk_score += 30

        # Обогащение данными threat intelligence
        threat_intel = await self._enrich_threat_intel(event)
        if threat_intel:
            if threat_intel.get('is_known_bad'):
                tags.append('known_threat_actor')
                risk_score += 50
            if threat_intel.get('ioc_matches'):
                tags.extend(threat_intel['ioc_matches'])
                risk_score += 40

        # Обогащение контекстом пользователя
        user_context = None
        user_id = event.get('actor', {}).get('user_id')
        if user_id:
            user_context = await self._enrich_user_context(user_id, event)
            if user_context:
                if user_context.get('unusual_location'):
                    tags.append('unusual_location')
                    risk_score += 20
                if user_context.get('unusual_time'):
                    tags.append('unusual_time')
                    risk_score += 15
                if user_context.get('new_device'):
                    tags.append('new_device')
                    risk_score += 10

        # Расчёт риска по типу события
        event_risk = self._calculate_event_risk(event)
        risk_score += event_risk

        return EnrichedSecurityEvent(
            original_event=event,
            geo_data=geo_data,
            threat_intel=threat_intel,
            user_context=user_context,
            risk_score=min(risk_score, 100),
            tags=tags
        )

    def _enrich_geo(self, ip_address: str) -> Optional[Dict]:
        """Обогащение данными геолокации"""
        try:
            response = self.geoip_reader.city(ip_address)
            return {
                'country': response.country.iso_code,
                'city': response.city.name,
                'latitude': response.location.latitude,
                'longitude': response.location.longitude,
                'is_anonymous_proxy': response.traits.is_anonymous_proxy,
                'is_tor_exit_node': response.traits.is_tor_exit_node
            }
        except Exception:
            return None

    def _calculate_event_risk(self, event: Dict) -> int:
        """Расчёт оценки риска по типу события"""
        risk_weights = {
            ('authentication', 'login_failure'): 10,
            ('authentication', 'mfa_failure'): 15,
            ('authorization', 'access_denied'): 10,
            ('authorization', 'privilege_escalation'): 40,
            ('data_access', 'bulk_operation'): 25,
            ('data_access', 'data_export'): 20,
            ('system', 'config_change'): 15,
            ('system', 'secret_access'): 20,
            ('threat', 'malware_detected'): 50,
            ('threat', 'intrusion_attempt'): 45,
        }

        event_key = (event.get('event_type'), event.get('event_name'))
        return risk_weights.get(event_key, 5)
```

## 4. Интеграция с SIEM

### 4.1 Архитектура SIEM

```yaml
# Конфигурация Elastic SIEM
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: security-siem
spec:
  version: 8.11.0
  nodeSets:
  - name: hot
    count: 3
    config:
      node.roles: ["master", "data_hot", "ingest"]
      xpack.security.enabled: true
      xpack.security.transport.ssl.enabled: true
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          resources:
            requests:
              memory: 8Gi
              cpu: 2
            limits:
              memory: 16Gi
              cpu: 4
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 500Gi
        storageClassName: fast-ssd
  - name: warm
    count: 2
    config:
      node.roles: ["data_warm"]
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 2Ti
        storageClassName: standard
```

### 4.2 Конфигурация сбора логов

```yaml
# Filebeat для сбора логов безопасности
apiVersion: beat.k8s.elastic.co/v1beta1
kind: Beat
metadata:
  name: security-filebeat
spec:
  type: filebeat
  version: 8.11.0
  elasticsearchRef:
    name: security-siem
  config:
    filebeat.autodiscover:
      providers:
      - type: kubernetes
        node: ${NODE_NAME}
        hints.enabled: true
        hints.default_config:
          type: container
          paths:
            - /var/log/containers/*${data.kubernetes.container.id}.log

    filebeat.inputs:
    - type: log
      enabled: true
      paths:
        - /var/log/audit/audit.log
      tags: ["linux_audit"]
      processors:
        - decode_json_fields:
            fields: ["message"]
            target: "audit"

    - type: log
      enabled: true
      paths:
        - /var/log/falco/*.log
      tags: ["falco"]
      processors:
        - decode_json_fields:
            fields: ["message"]
            target: "falco"

    processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
        - logs_path:
            logs_path: "/var/log/containers/"
    - add_cloud_metadata: ~
    - add_host_metadata: ~

    output.elasticsearch:
      hosts: ["https://security-siem-es-http:9200"]
      index: "security-logs-%{+yyyy.MM.dd}"

    setup.ilm:
      enabled: true
      rollover_alias: "security-logs"
      pattern: "{now/d}-000001"
      policy_name: "security-logs-policy"
```

### 4.3 Правила обнаружения

```yaml
# Правила обнаружения Elastic Security
# Правило: Попытка brute force аутентификации
---
name: "Попытка brute force аутентификации"
description: "Обнаружение множественных неудачных попыток входа с одного источника"
risk_score: 73
severity: high
type: threshold
index:
  - security-logs-*
language: kuery
query: |
  event_type:authentication AND event_name:login_failure
threshold:
  field:
    - actor.ip_address
    - actor.username
  value: 5
  cardinality:
    field: actor.username
    value: 1
from: now-5m
interval: 1m
actions:
  - action_type: .slack
    params:
      message: |
        🚨 Обнаружена Brute Force атака
        IP источника: {{context.rule.threshold_field_0}}
        Целевой пользователь: {{context.rule.threshold_field_1}}
        Попыток: {{state.signals_count}}
tags:
  - authentication
  - brute_force
  - attack

---
# Правило: Попытка повышения привилегий
name: "Попытка повышения привилегий"
description: "Обнаружение несанкционированного повышения привилегий"
risk_score: 85
severity: critical
type: eql
index:
  - security-logs-*
language: eql
query: |
  sequence by actor.user_id with maxspan=1h
    [authorization where event_name == "role_change"]
    [data_access where event_name == "bulk_operation" or event_name == "data_export"]
actions:
  - action_type: .pagerduty
    params:
      severity: critical
      summary: "Обнаружено повышение привилегий с последующим доступом к данным"

---
# Правило: Аномальный объём экспорта данных
name: "Аномальный объём экспорта данных"
description: "Обнаружение необычно большого экспорта данных"
risk_score: 65
severity: medium
type: machine_learning
machine_learning_job_id: security_data_export_anomaly
anomaly_threshold: 75
```

## 5. Мониторинг безопасности во время выполнения

### 5.1 Конфигурация Falco

```yaml
# Falco DaemonSet для runtime security в Kubernetes
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      serviceAccountName: falco
      hostNetwork: true
      hostPID: true
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      containers:
      - name: falco
        image: falcosecurity/falco:0.36.2
        securityContext:
          privileged: true
        args:
          - /usr/bin/falco
          - --cri=/run/containerd/containerd.sock
          - -K=/var/run/secrets/kubernetes.io/serviceaccount/token
          - -k=https://kubernetes.default
          - --k8s-node=${FALCO_K8S_NODE}
          - -pk
        env:
        - name: FALCO_K8S_NODE
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - mountPath: /run/containerd/containerd.sock
          name: containerd-socket
          readOnly: true
        - mountPath: /host/dev
          name: dev-fs
          readOnly: true
        - mountPath: /host/proc
          name: proc-fs
          readOnly: true
        - mountPath: /etc/falco
          name: falco-config
      volumes:
      - name: containerd-socket
        hostPath:
          path: /run/containerd/containerd.sock
      - name: dev-fs
        hostPath:
          path: /dev
      - name: proc-fs
        hostPath:
          path: /proc
      - name: falco-config
        configMap:
          name: falco-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-config
  namespace: security
data:
  custom_rules.yaml: |
    # Кастомные правила Falco

    - rule: Доступ к чувствительным файлам
      desc: Обнаружение доступа к чувствительным файлам
      condition: >
        open_read and
        (fd.name startswith /etc/shadow or
         fd.name startswith /etc/passwd or
         fd.name startswith /root/.ssh/ or
         fd.name contains id_rsa or
         fd.name contains .pem) and
        not proc.name in (sshd, sudo, su, passwd)
      output: >
        Доступ к чувствительному файлу
        (user=%user.name command=%proc.cmdline file=%fd.name container=%container.id
        image=%container.image.repository k8s.pod=%k8s.pod.name k8s.ns=%k8s.ns.name)
      priority: WARNING
      tags: [filesystem, sensitive]

    - rule: Shell запущен в контейнере
      desc: Обнаружение запуска shell в контейнере
      condition: >
        spawned_process and
        container and
        proc.name in (bash, sh, zsh, ash, dash) and
        proc.pname != entrypoint.sh and
        not container.image.repository in (
          docker.io/library/busybox,
          gcr.io/distroless/debug
        )
      output: >
        Shell запущен в контейнере
        (user=%user.name shell=%proc.name parent=%proc.pname
        container=%container.id image=%container.image.repository
        k8s.pod=%k8s.pod.name k8s.ns=%k8s.ns.name)
      priority: NOTICE
      tags: [container, shell]

    - rule: Криптомайнинг активность
      desc: Обнаружение потенциального криптомайнинга
      condition: >
        spawned_process and
        (proc.name in (xmrig, minerd, minergate, stratum) or
         proc.cmdline contains "stratum+tcp" or
         proc.cmdline contains "pool.minergate")
      output: >
        Обнаружен криптомайнинг
        (user=%user.name command=%proc.cmdline container=%container.id
        image=%container.image.repository)
      priority: CRITICAL
      tags: [cryptomining, malware]

    - rule: Исходящее соединение на подозрительный порт
      desc: Обнаружение исходящих соединений на известные вредоносные порты
      condition: >
        outbound and
        fd.sport in (4444, 5555, 6666, 9999, 31337) and
        not container.image.repository in (security-scanner)
      output: >
        Подозрительное исходящее соединение
        (user=%user.name command=%proc.cmdline connection=%fd.name
        container=%container.id)
      priority: WARNING
      tags: [network, suspicious]

    - rule: Доступ к Kubernetes секретам
      desc: Обнаружение доступа к Kubernetes секретам из неожиданных процессов
      condition: >
        open_read and
        fd.name startswith /var/run/secrets/kubernetes.io and
        not proc.name in (kubectl, kube-proxy, kubelet)
      output: >
        Доступ к Kubernetes секрету
        (user=%user.name command=%proc.cmdline file=%fd.name
        container=%container.id k8s.pod=%k8s.pod.name)
      priority: WARNING
      tags: [kubernetes, secrets]
```

### 5.2 Falco Sidekick для маршрутизации алертов

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: falco-sidekick
  namespace: security
spec:
  replicas: 2
  selector:
    matchLabels:
      app: falco-sidekick
  template:
    metadata:
      labels:
        app: falco-sidekick
    spec:
      containers:
      - name: falco-sidekick
        image: falcosecurity/falcosidekick:2.28.0
        ports:
        - containerPort: 2801
        env:
        - name: SLACK_WEBHOOKURL
          valueFrom:
            secretKeyRef:
              name: falco-sidekick-secrets
              key: slack-webhook
        - name: SLACK_MINIMUMPRIORITY
          value: "warning"
        - name: ELASTICSEARCH_HOSTPORT
          value: "https://security-siem-es-http:9200"
        - name: ELASTICSEARCH_INDEX
          value: "falco"
        - name: PAGERDUTY_ROUTINGKEY
          valueFrom:
            secretKeyRef:
              name: falco-sidekick-secrets
              key: pagerduty-key
        - name: PAGERDUTY_MINIMUMPRIORITY
          value: "critical"
        - name: PROMETHEUS_ENABLED
          value: "true"
```

## 6. Мониторинг сетевой безопасности

### 6.1 Мониторинг Network Policies

```go
package netmon

import (
    "context"
    "time"

    networkingv1 "k8s.io/api/networking/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/client-go/kubernetes"
)

type NetworkPolicyAuditor struct {
    client     kubernetes.Interface
    auditLog   AuditLogger
}

func (a *NetworkPolicyAuditor) AuditPolicies(ctx context.Context) error {
    // Получаем все network policies
    policies, err := a.client.NetworkingV1().
        NetworkPolicies("").
        List(ctx, metav1.ListOptions{})
    if err != nil {
        return err
    }

    for _, policy := range policies.Items {
        // Проверяем на слишком разрешительные политики
        findings := a.analyzePolicyRisks(&policy)
        if len(findings) > 0 {
            a.auditLog.Log(ctx, AuditEvent{
                EventType: "network_policy_audit",
                EventName: "risky_policy_detected",
                Severity:  "warning",
                Target: Target{
                    ResourceType: "NetworkPolicy",
                    ResourceID:   string(policy.UID),
                    ResourceName: policy.Name,
                },
                Metadata: map[string]interface{}{
                    "namespace": policy.Namespace,
                    "findings":  findings,
                },
            })
        }
    }

    return nil
}

func (a *NetworkPolicyAuditor) analyzePolicyRisks(
    policy *networkingv1.NetworkPolicy,
) []string {
    var findings []string

    // Проверка на allow-all ingress
    for _, ingress := range policy.Spec.Ingress {
        if len(ingress.From) == 0 {
            findings = append(findings, "allow_all_ingress")
        }
        for _, from := range ingress.From {
            if from.IPBlock != nil && from.IPBlock.CIDR == "0.0.0.0/0" {
                findings = append(findings, "allow_internet_ingress")
            }
        }
    }

    // Проверка на allow-all egress
    for _, egress := range policy.Spec.Egress {
        if len(egress.To) == 0 {
            findings = append(findings, "allow_all_egress")
        }
    }

    return findings
}

// FlowLogAnalyzer - анализ VPC Flow Logs
type FlowLogAnalyzer struct {
    storage    FlowLogStorage
    detector   AnomalyDetector
    alerter    Alerter
}

type FlowLog struct {
    Timestamp       time.Time `json:"timestamp"`
    SourceIP        string    `json:"src_ip"`
    DestinationIP   string    `json:"dst_ip"`
    SourcePort      int       `json:"src_port"`
    DestinationPort int       `json:"dst_port"`
    Protocol        string    `json:"protocol"`
    Bytes           int64     `json:"bytes"`
    Packets         int64     `json:"packets"`
    Action          string    `json:"action"` // ACCEPT, REJECT
    Direction       string    `json:"direction"`
}

func (a *FlowLogAnalyzer) AnalyzeFlows(ctx context.Context, flows []FlowLog) error {
    // Обнаружение сканирования портов
    portScanners := a.detectPortScanning(flows)
    for ip, scanDetails := range portScanners {
        a.alerter.Alert(ctx, Alert{
            Severity: "high",
            Title:    "Обнаружено сканирование портов",
            Message:  fmt.Sprintf("IP %s просканировал %d уникальных портов", ip, scanDetails.UniquePortCount),
            Metadata: scanDetails,
        })
    }

    // Обнаружение эксфильтрации данных
    exfilCandidates := a.detectDataExfiltration(flows)
    for _, candidate := range exfilCandidates {
        a.alerter.Alert(ctx, Alert{
            Severity: "critical",
            Title:    "Потенциальная эксфильтрация данных",
            Message:  fmt.Sprintf("Необычный исходящий объём данных: %d MB в %s",
                candidate.BytesMB, candidate.DestinationIP),
            Metadata: candidate,
        })
    }

    return nil
}
```

## 7. Обнаружение угроз и реагирование

### 7.1 Конвейер обнаружения угроз

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from enum import Enum
import asyncio

class ThreatSeverity(Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

@dataclass
class DetectedThreat:
    threat_id: str
    timestamp: str
    severity: ThreatSeverity
    threat_type: str
    description: str
    indicators: List[Dict]
    affected_assets: List[str]
    evidence: List[Dict]
    recommended_actions: List[str]
    mitre_attack_techniques: List[str]

class ThreatDetectionEngine:
    def __init__(
        self,
        rule_engine,
        ml_detector,
        threat_intel_client,
        correlation_engine
    ):
        self.rule_engine = rule_engine
        self.ml_detector = ml_detector
        self.threat_intel = threat_intel_client
        self.correlation_engine = correlation_engine

    async def analyze_events(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """Анализ событий через множественные методы обнаружения"""

        threats = []

        # Запуск методов обнаружения параллельно
        results = await asyncio.gather(
            self._rule_based_detection(events),
            self._ml_based_detection(events),
            self._threat_intel_matching(events),
            self._behavioral_analysis(events)
        )

        # Объединение и дедупликация результатов
        for result_set in results:
            threats.extend(result_set)

        # Корреляция угроз
        correlated = await self.correlation_engine.correlate(threats)

        return correlated

    async def _rule_based_detection(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """Применение правил SIGMA/YARA"""

        threats = []

        for event in events:
            matches = self.rule_engine.match(event)
            for match in matches:
                threats.append(DetectedThreat(
                    threat_id=generate_threat_id(),
                    timestamp=event['timestamp'],
                    severity=ThreatSeverity(match.severity),
                    threat_type=match.threat_type,
                    description=match.description,
                    indicators=[],
                    affected_assets=[event.get('source', {}).get('service')],
                    evidence=[event],
                    recommended_actions=match.recommended_actions,
                    mitre_attack_techniques=match.mitre_techniques
                ))

        return threats

    async def _behavioral_analysis(
        self,
        events: List[Dict]
    ) -> List[DetectedThreat]:
        """Обнаружение угроз на основе поведенческих паттернов"""

        threats = []

        # Группировка событий по пользователю/сущности
        entities = self._group_by_entity(events)

        for entity_id, entity_events in entities.items():
            # Проверка на паттерны атак
            patterns = self._detect_attack_patterns(entity_events)

            for pattern in patterns:
                threats.append(DetectedThreat(
                    threat_id=generate_threat_id(),
                    timestamp=entity_events[-1]['timestamp'],
                    severity=ThreatSeverity(pattern.severity),
                    threat_type=pattern.attack_type,
                    description=pattern.description,
                    indicators=[],
                    affected_assets=[entity_id],
                    evidence=entity_events,
                    recommended_actions=pattern.recommended_actions,
                    mitre_attack_techniques=pattern.mitre_techniques
                ))

        return threats

    def _detect_attack_patterns(
        self,
        events: List[Dict]
    ) -> List:
        """Обнаружение известных паттернов атак"""

        patterns = []

        # Паттерн: Credential stuffing
        login_failures = [e for e in events
                        if e.get('event_name') == 'login_failure']
        if len(login_failures) > 10:
            unique_usernames = set(e.get('actor', {}).get('username')
                                  for e in login_failures)
            if len(unique_usernames) > 5:
                patterns.append(AttackPattern(
                    attack_type="credential_stuffing",
                    severity="high",
                    description="Множественные неудачные входы с разными именами пользователей",
                    mitre_techniques=["T1110.004"],
                    recommended_actions=[
                        "Включить rate limiting",
                        "Внедрить CAPTCHA",
                        "Заблокировать IP источника"
                    ]
                ))

        # Паттерн: Цепочка повышения привилегий
        role_changes = [e for e in events
                       if e.get('event_name') == 'role_change']
        sensitive_access = [e for e in events
                          if e.get('event_name') in ['secret_access', 'data_export']]
        if role_changes and sensitive_access:
            patterns.append(AttackPattern(
                attack_type="privilege_escalation_chain",
                severity="critical",
                description="Смена роли с последующим доступом к чувствительным данным",
                mitre_techniques=["T1078", "T1548"],
                recommended_actions=[
                    "Отозвать повышенные привилегии",
                    "Провести аудит доступа к ресурсам",
                    "Сбросить учётные данные пользователя"
                ]
            ))

        return patterns
```

### 7.2 Автоматизированное реагирование на инциденты

```yaml
# SOAR Playbook: Реагирование на Brute Force
name: brute_force_response
description: Автоматизированное реагирование на brute force атаки
triggers:
  - alert_type: brute_force_authentication
    severity: [high, critical]

steps:
  - name: gather_context
    type: enrichment
    actions:
      - get_user_details:
          user_id: "{{alert.actor.user_id}}"
      - get_ip_reputation:
          ip: "{{alert.actor.ip_address}}"
      - get_recent_events:
          user_id: "{{alert.actor.user_id}}"
          timeframe: 24h

  - name: assess_risk
    type: decision
    conditions:
      - if: "{{ip_reputation.is_malicious}} == true"
        action: block_ip_immediately
      - if: "{{recent_events.failed_logins}} > 20"
        action: lock_account
      - if: "{{user_details.is_privileged}} == true"
        action: escalate_to_security
      - else:
        action: standard_response

  - name: block_ip_immediately
    type: action
    actions:
      - add_to_blocklist:
          ip: "{{alert.actor.ip_address}}"
          duration: 24h
          reason: "Brute force атака - вредоносный IP"
      - notify:
          channel: security-alerts
          message: "Заблокирован вредоносный IP {{alert.actor.ip_address}}"

  - name: lock_account
    type: action
    actions:
      - disable_user:
          user_id: "{{alert.actor.user_id}}"
          reason: "Автоматическая блокировка - обнаружен brute force"
      - notify_user:
          user_id: "{{alert.actor.user_id}}"
          template: account_locked_security
      - create_ticket:
          type: security_incident
          priority: high

  - name: escalate_to_security
    type: action
    actions:
      - page_oncall:
          team: security
          message: "Привилегированный аккаунт под brute force атакой"
      - create_war_room:
          name: "Incident-{{alert.id}}"
          invite: [security, sre]

  - name: document_incident
    type: action
    run: always
    actions:
      - create_incident_report:
          alert: "{{alert}}"
          context: "{{gather_context}}"
          actions_taken: "{{steps_executed}}"
```

## 8. Метрики безопасности и дашборды

### 8.1 KPI безопасности

```yaml
# Определение метрик безопасности
security_metrics:
  detection:  # Обнаружение
    - name: mean_time_to_detect
      description: "Среднее время от возникновения угрозы до обнаружения"
      target: "< 5 минут"
      calculation: "avg(detection_time - event_time)"

    - name: detection_coverage
      description: "Процент захваченных событий безопасности"
      target: "> 99%"
      calculation: "captured_events / total_events * 100"

    - name: false_positive_rate
      description: "Процент ложноположительных алертов"
      target: "< 5%"
      calculation: "false_positive_alerts / total_alerts * 100"

  response:  # Реагирование
    - name: mean_time_to_respond
      description: "Среднее время от обнаружения до начала реагирования"
      target: "< 15 минут"
      calculation: "avg(response_time - detection_time)"

    - name: mean_time_to_contain
      description: "Среднее время до локализации активных угроз"
      target: "< 1 час"
      calculation: "avg(containment_time - detection_time)"

    - name: mean_time_to_remediate
      description: "Среднее время до полного устранения инцидента"
      target: "< 24 часа"
      calculation: "avg(remediation_time - detection_time)"

  coverage:  # Покрытие
    - name: asset_monitoring_coverage
      description: "Процент активов с мониторингом безопасности"
      target: "100%"

    - name: vulnerability_scan_coverage
      description: "Процент активов, просканированных на уязвимости"
      target: "> 95%"

  compliance:  # Соответствие нормативам
    - name: policy_compliance_rate
      description: "Процент систем, соответствующих политике безопасности"
      target: "> 98%"

    - name: patch_compliance_rate
      description: "Процент систем с актуальными патчами"
      target: "> 95%"
```

### 8.2 Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Центр операций безопасности (SOC)",
    "tags": ["security", "soc"],
    "panels": [
      {
        "title": "Обзор алертов безопасности",
        "type": "stat",
        "gridPos": {"x": 0, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(increase(security_alerts_total[24h]))",
            "legendFormat": "Всего алертов (24ч)"
          }
        ]
      },
      {
        "title": "Критические алерты",
        "type": "stat",
        "gridPos": {"x": 6, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "sum(increase(security_alerts_total{severity=\"critical\"}[24h]))",
            "legendFormat": "Критические"
          }
        ]
      },
      {
        "title": "MTTD (Среднее время обнаружения)",
        "type": "gauge",
        "gridPos": {"x": 12, "y": 0, "w": 6, "h": 4},
        "targets": [
          {
            "expr": "avg(security_detection_time_seconds)",
            "legendFormat": "MTTD"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "max": 600,
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 180},
                {"color": "red", "value": 300}
              ]
            }
          }
        }
      },
      {
        "title": "Алерты по типу",
        "type": "timeseries",
        "gridPos": {"x": 0, "y": 4, "w": 16, "h": 8},
        "targets": [
          {
            "expr": "sum by (alert_type) (rate(security_alerts_total[5m]))",
            "legendFormat": "{{alert_type}}"
          }
        ]
      },
      {
        "title": "Оценка соответствия требованиям",
        "type": "gauge",
        "gridPos": {"x": 16, "y": 4, "w": 8, "h": 8},
        "targets": [
          {
            "expr": "security_compliance_score",
            "legendFormat": "Compliance"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "min": 0,
            "max": 100,
            "thresholds": {
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 80},
                {"color": "green", "value": 95}
              ]
            }
          }
        }
      }
    ]
  }
}
```

## 9. Мониторинг соответствия нормативам

### 9.1 Маппинг фреймворков соответствия

```yaml
# Маппинг контролей соответствия
compliance_frameworks:
  SOC2:
    security_monitoring_controls:
      CC6.1:
        description: "Логические и физические контроли доступа"
        monitoring_requirements:
          - authentication_logging
          - access_control_audit
          - session_monitoring
        evidence:
          - "Логи аутентификации с пользователем, временем, результатом"
          - "Аудиторский след изменений контроля доступа"
          - "Логи длительности и активности сессий"

      CC6.6:
        description: "Мониторинг событий безопасности"
        monitoring_requirements:
          - security_event_collection
          - alert_generation
          - incident_tracking

      CC7.2:
        description: "Мониторинг системы"
        monitoring_requirements:
          - performance_monitoring
          - availability_monitoring
          - anomaly_detection

  PCI_DSS:
    security_monitoring_controls:
      "10.1":
        description: "Аудиторские следы для системных компонентов"
        monitoring_requirements:
          - user_action_logging
          - admin_action_logging
          - system_event_logging

      "10.2":
        description: "Автоматизированные аудиторские следы"
        monitoring_requirements:
          - individual_user_access
          - actions_by_privileged_users
          - access_to_audit_trails
          - invalid_access_attempts

      "10.6":
        description: "Ежедневный просмотр логов"
        monitoring_requirements:
          - daily_log_review_process
          - automated_log_analysis
          - exception_reporting

  GDPR:
    security_monitoring_controls:
      article_32:
        description: "Безопасность обработки"
        monitoring_requirements:
          - access_logging
          - data_access_monitoring
          - breach_detection

      article_33:
        description: "Уведомление о нарушении"
        monitoring_requirements:
          - breach_detection_capability
          - notification_workflow
          - documentation_process
```

### 9.2 Автоматизация соответствия

```python
from dataclasses import dataclass
from typing import List, Dict, Optional
from datetime import datetime, timedelta

@dataclass
class ComplianceCheckResult:
    control_id: str
    status: str  # compliant, non_compliant, partial, not_applicable
    evidence: List[Dict]
    findings: List[str]
    last_checked: datetime
    next_review: datetime

class ComplianceMonitor:
    def __init__(
        self,
        siem_client,
        audit_client,
        evidence_storage
    ):
        self.siem = siem_client
        self.audit = audit_client
        self.evidence = evidence_storage

    async def check_compliance(
        self,
        framework: str,
        control_ids: Optional[List[str]] = None
    ) -> List[ComplianceCheckResult]:
        """Запуск проверок соответствия для указанных контролей"""

        controls = self._get_controls(framework, control_ids)
        results = []

        for control in controls:
            result = await self._check_control(control)
            results.append(result)

            # Сохранение доказательств
            await self.evidence.store(
                control_id=control.control_id,
                check_time=datetime.utcnow(),
                result=result
            )

        return results

    async def generate_compliance_report(
        self,
        framework: str,
        period_start: datetime,
        period_end: datetime
    ) -> Dict:
        """Генерация отчёта о соответствии для аудиторов"""

        results = await self.check_compliance(framework)

        # Расчёт общей оценки соответствия
        compliant_count = sum(1 for r in results if r.status == "compliant")
        total_count = len(results)
        compliance_score = (compliant_count / total_count) * 100

        return {
            "framework": framework,
            "period": {
                "start": period_start.isoformat(),
                "end": period_end.isoformat()
            },
            "overall_score": compliance_score,
            "summary": {
                "compliant": compliant_count,
                "non_compliant": sum(1 for r in results if r.status == "non_compliant"),
                "partial": sum(1 for r in results if r.status == "partial"),
                "total": total_count
            },
            "generated_at": datetime.utcnow().isoformat()
        }
```

## 10. Чек-лист внедрения

### 10.1 Фаза 1: Фундамент (Недели 1-4)

- [ ] Развернуть инфраструктуру логирования безопасности
  - [ ] Настроить структурированный формат логов безопасности
  - [ ] Реализовать аудиторский след с проверкой целостности
  - [ ] Настроить отправку логов в SIEM
- [ ] Реализовать базовое обнаружение угроз
  - [ ] Развернуть Falco для runtime security
  - [ ] Настроить мониторинг аутентификации
  - [ ] Настроить правила обнаружения brute force
- [ ] Создать дашборды безопасности
  - [ ] Дашборд обзора алертов
  - [ ] Дашборд мониторинга аутентификации

### 10.2 Фаза 2: Обнаружение (Недели 5-8)

- [ ] Улучшить возможности обнаружения угроз
  - [ ] Реализовать ML-обнаружение аномалий
  - [ ] Интегрировать фиды threat intelligence
  - [ ] Настроить поведенческий анализ
- [ ] Развернуть мониторинг сетевой безопасности
  - [ ] Анализ VPC flow logs
  - [ ] Аудит network policies
- [ ] Реализовать автоматическое обогащение
  - [ ] GeoIP обогащение
  - [ ] Контекст пользователя
  - [ ] Скоринг рисков

### 10.3 Фаза 3: Реагирование (Недели 9-12)

- [ ] Реализовать возможности SOAR
  - [ ] Плейбуки автоматического реагирования
  - [ ] Интеграция с тикетингом инцидентов
  - [ ] Автоматизация коммуникаций
- [ ] Развернуть мониторинг соответствия
  - [ ] Конфигурация маппинга контролей
  - [ ] Автоматические проверки соответствия
  - [ ] Автоматизация сбора доказательств
- [ ] Создать процедуры операций безопасности
  - [ ] Runbook-и реагирования на инциденты
  - [ ] Процедуры эскалации
  - [ ] Процесс пост-инцидентного анализа

## 11. Связанные спецификации

| Спецификация | Связь |
|--------------|-------|
| [SRG-002 Logging](SRG-002%20Logging.ru.md) | Основа для логирования событий безопасности |
| [SRG-003 Error Tracking](SRG-003%20Error%20Tracking.ru.md) | Интеграция с мониторингом ошибок |
| [SRG-005 Alerting Rules](SRG-005%20Alerting%20Rules.ru.md) | Маршрутизация и эскалация алертов безопасности |
| [SRO-004 Multi-Region DR](SRO-004%20Multi-Region%20DR.ru.md) | DR для инфраструктуры безопасности |
| [SRO-002 Chaos Engineering](SRO-002%20Chaos%20Engineering.ru.md) | Хаос-тестирование безопасности |

---

**История документа:**

| Версия | Дата | Автор | Изменения |
|--------|------|-------|-----------|
| 1.0 | 2024-01-15 | Команда безопасности | Начальная спецификация |
