# SRG-011 Feature Flags & Toggles

Feature Flags (флаги функций) — механизм управления функциональностью приложения без деплоя нового кода.

## Зачем нужны Feature Flags?

### Безопасные релизы
- Постепенное включение функций для части пользователей
- Мгновенный откат без деплоя при проблемах
- Снижение риска при выкатке новой функциональности

### Canary Releases
- Выкатка на 1-5% пользователей для проверки стабильности
- Мониторинг метрик перед полным релизом
- Автоматический откат при деградации SLO

### A/B Тестирование
- Сравнение разных вариантов реализации
- Принятие решений на основе данных
- Оптимизация конверсии и UX

### Trunk-Based Development
- Разработка в main/master ветке
- Код новых функций скрыт за флагами
- Уменьшение merge conflicts

## Типы Feature Flags

### 1. Release Flags (Релизные флаги)
Временные флаги для безопасного релиза.

```yaml
# Пример конфигурации
feature_flags:
  new_checkout_flow:
    type: release
    enabled: false
    rollout_percentage: 0
    created_at: "2024-01-15"
    owner: "team-checkout"
    ticket: "PROJ-1234"
```

**Жизненный цикл:**
1. Создание флага (disabled)
2. Включение для разработчиков
3. Постепенный rollout (1% → 10% → 50% → 100%)
4. Удаление флага из кода

### 2. Ops Flags (Операционные флаги)
Флаги для управления поведением в production.

```yaml
feature_flags:
  enable_cache_layer:
    type: ops
    enabled: true
    description: "Включает кэширование на уровне API"

  maintenance_mode:
    type: ops
    enabled: false
    description: "Режим обслуживания"
```

### 3. Experiment Flags (Флаги экспериментов)
Для A/B тестирования и экспериментов.

```yaml
feature_flags:
  checkout_button_color:
    type: experiment
    variants:
      - name: control
        value: "blue"
        weight: 50
      - name: treatment
        value: "green"
        weight: 50
    metrics:
      - conversion_rate
      - bounce_rate
```

### 4. Permission Flags (Флаги доступа)
Управление доступом к функциям.

```yaml
feature_flags:
  admin_dashboard:
    type: permission
    enabled_for:
      - role: admin
      - role: support
    disabled_for:
      - role: user
```

## Стратегии Rollout

### Percentage Rollout (Процентный)

```python
class PercentageRollout:
    def __init__(self, percentage: int):
        self.percentage = percentage

    def is_enabled(self, user_id: str) -> bool:
        # Детерминированный хеш для консистентности
        hash_value = hash(user_id) % 100
        return hash_value < self.percentage
```

**Рекомендуемая стратегия выкатки:**
```
День 1: 1%   - проверка на ошибки
День 2: 5%   - мониторинг метрик
День 3: 25%  - проверка под нагрузкой
День 4: 50%  - половина пользователей
День 5: 100% - полная выкатка
```

### User Targeting (Таргетинг)

```python
class UserTargeting:
    def is_enabled(self, user: User, flag: FeatureFlag) -> bool:
        # Проверка по ID
        if user.id in flag.allowed_user_ids:
            return True

        # Проверка по атрибутам
        if flag.country and user.country in flag.country:
            return True

        # Проверка по сегменту
        if flag.segment and user.segment == flag.segment:
            return True

        # Fallback на процентный rollout
        return self.percentage_check(user.id, flag.percentage)
```

### Ring Deployment

```
Ring 0: Разработчики (internal)     - немедленно
Ring 1: Бета-тестеры (early adopters) - через 1 день
Ring 2: 10% пользователей             - через 3 дня
Ring 3: 50% пользователей             - через 5 дней
Ring 4: 100% пользователей            - через 7 дней
```

## Реализация

### Интерфейс Feature Flag Service

```python
from abc import ABC, abstractmethod
from typing import Optional, Dict, Any

class FeatureFlagService(ABC):
    @abstractmethod
    def is_enabled(
        self,
        flag_name: str,
        user_id: Optional[str] = None,
        attributes: Optional[Dict[str, Any]] = None
    ) -> bool:
        """Проверяет, включен ли флаг для пользователя."""
        pass

    @abstractmethod
    def get_variant(
        self,
        flag_name: str,
        user_id: str
    ) -> str:
        """Возвращает вариант для A/B теста."""
        pass
```

### Пример использования в коде

```python
# Хорошо - проверка в начале функции
def process_checkout(user: User, cart: Cart):
    if feature_flags.is_enabled("new_checkout_flow", user.id):
        return new_checkout_flow(user, cart)
    return legacy_checkout_flow(user, cart)

# Хорошо - декоратор
@feature_flag("async_processing")
def process_order(order: Order):
    # Новая логика
    pass

# Плохо - вложенные проверки
def process_order(order: Order):
    if feature_flags.is_enabled("feature_a"):
        if feature_flags.is_enabled("feature_b"):
            if feature_flags.is_enabled("feature_c"):
                # Сложно поддерживать
                pass
```

### Конфигурация по умолчанию

```yaml
feature_flags:
  defaults:
    # Флаг выключен если сервис недоступен
    fallback_enabled: false

    # Кэширование конфигурации
    cache_ttl_seconds: 60

    # Таймаут запроса к сервису флагов
    timeout_ms: 100

    # Логирование решений
    log_decisions: true
```

## Инструменты

### Сравнение платформ

| Платформа | Тип | A/B тесты | SDK | Цена |
|-----------|-----|-----------|-----|------|
| LaunchDarkly | SaaS | Да | 25+ языков | $$$ |
| Unleash | Open Source | Да | 15+ языков | Free/Paid |
| Flagsmith | Open Source | Да | 10+ языков | Free/Paid |
| Split.io | SaaS | Да | 10+ языков | $$$ |
| ConfigCat | SaaS | Ограничено | 10+ языков | $ |

### Self-hosted решение (Unleash)

```yaml
# docker-compose.yml
version: '3'
services:
  unleash:
    image: unleashorg/unleash-server:latest
    ports:
      - "4242:4242"
    environment:
      DATABASE_URL: postgres://user:pass@db/unleash

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: unleash
```

## Мониторинг и метрики

### Ключевые метрики

```python
# Prometheus метрики
feature_flag_evaluations_total = Counter(
    'feature_flag_evaluations_total',
    'Total feature flag evaluations',
    ['flag_name', 'result']
)

feature_flag_evaluation_duration = Histogram(
    'feature_flag_evaluation_duration_seconds',
    'Time to evaluate feature flag',
    ['flag_name']
)

# При каждой проверке
def evaluate_flag(flag_name: str, user_id: str) -> bool:
    with feature_flag_evaluation_duration.labels(flag_name).time():
        result = _do_evaluate(flag_name, user_id)
        feature_flag_evaluations_total.labels(
            flag_name=flag_name,
            result=str(result)
        ).inc()
        return result
```

### Alerting

```yaml
# Prometheus alerts
groups:
  - name: feature_flags
    rules:
      - alert: FeatureFlagServiceDown
        expr: up{job="feature-flag-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Feature flag service is down"

      - alert: HighFlagEvaluationLatency
        expr: |
          histogram_quantile(0.99,
            rate(feature_flag_evaluation_duration_seconds_bucket[5m])
          ) > 0.1
        for: 5m
        labels:
          severity: warning
```

## Best Practices

### 1. Именование флагов

```python
# Хорошо
"enable_new_checkout_flow"
"experiment_button_color_2024q1"
"ops_enable_redis_cache"

# Плохо
"flag1"
"test"
"new_feature"
```

### 2. Документация флагов

```yaml
feature_flags:
  enable_async_notifications:
    description: "Включает асинхронную отправку уведомлений через очередь"
    owner: "team-notifications"
    created_at: "2024-01-15"
    expected_removal: "2024-03-01"
    ticket: "PROJ-5678"
    dependencies:
      - "rabbitmq"
      - "notification-worker"
```

### 3. Гигиена флагов

```
Правила удаления флагов:
1. Release flags: удалить в течение 2 недель после 100% rollout
2. Experiment flags: удалить после завершения эксперимента
3. Ops flags: документировать и ревьюить каждые 6 месяцев
4. Permission flags: ревьюить при изменении ролевой модели
```

### 4. Тестирование

```python
# Тестирование обоих путей
class TestCheckoutFlow:
    def test_new_checkout_enabled(self, mock_feature_flags):
        mock_feature_flags.is_enabled.return_value = True
        result = process_checkout(user, cart)
        assert result.uses_new_flow

    def test_new_checkout_disabled(self, mock_feature_flags):
        mock_feature_flags.is_enabled.return_value = False
        result = process_checkout(user, cart)
        assert not result.uses_new_flow
```

## Антипаттерны

### 1. Флаг-спагетти
```python
# Плохо - слишком много вложенных флагов
if flag_a:
    if flag_b:
        if flag_c:
            # 2^3 = 8 возможных путей
```

### 2. Бессмертные флаги
```python
# Плохо - флаг существует годами
if feature_flags.is_enabled("new_design"):  # Создан в 2019
    # "Новый" дизайн уже 5 лет
```

### 3. Отсутствие fallback
```python
# Плохо - без обработки ошибок
result = feature_flag_service.is_enabled("flag")

# Хорошо - с fallback
try:
    result = feature_flag_service.is_enabled("flag")
except FeatureFlagServiceError:
    result = False  # Безопасное значение по умолчанию
```

## Ссылки

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly: Feature Flag Best Practices](https://launchdarkly.com/blog/best-practices-for-feature-flags/)
- [Unleash Documentation](https://docs.getunleash.io/)
- [Google: Testing in Production](https://sre.google/sre-book/testing-reliability/)
