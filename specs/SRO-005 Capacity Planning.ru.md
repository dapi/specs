# SRO-005 Capacity Planning

Capacity Planning — процесс определения производственных мощностей, необходимых для удовлетворения текущих и будущих потребностей системы.

## Зачем Capacity Planning?

### Проблемы без планирования

- Неожиданные outage'и при росте нагрузки
- Перерасход бюджета на избыточные ресурсы
- Невозможность масштабирования в нужный момент
- Деградация производительности без понимания причин

### Преимущества Capacity Planning

- **Предсказуемость**: понимание когда потребуется масштабирование
- **Оптимизация затрат**: баланс между производительностью и стоимостью
- **Проактивность**: решение проблем до их возникновения
- **Бизнес-планирование**: поддержка роста бизнеса инфраструктурой

## Основные концепции

### Capacity vs Performance

```
Capacity (Ёмкость):
  "Сколько система может обработать?"
  Измеряется в: requests/sec, users, transactions, storage

Performance (Производительность):
  "Как быстро система обрабатывает?"
  Измеряется в: latency, response time, throughput

Связь:
  При приближении к capacity → performance деградирует

  Пример:
    Capacity: 10,000 RPS
    При 5,000 RPS: latency = 50ms ✅
    При 9,000 RPS: latency = 200ms ⚠️
    При 10,000+ RPS: latency = timeout ❌
```

### Утилизация и Headroom

```yaml
utilization_levels:
  healthy:
    range: "0-60%"
    description: "Нормальная работа с запасом"
    action: "Мониторинг"

  warning:
    range: "60-80%"
    description: "Приближение к пределу"
    action: "Планирование масштабирования"

  critical:
    range: "80-90%"
    description: "Высокий риск деградации"
    action: "Немедленное масштабирование"

  overload:
    range: ">90%"
    description: "Деградация производительности"
    action: "Экстренные меры"

headroom_formula: |
  Headroom = (Capacity - Current Load) / Capacity × 100%

  Пример:
    Capacity = 10,000 RPS
    Current Load = 6,000 RPS
    Headroom = (10,000 - 6,000) / 10,000 × 100% = 40%
```

## Performance Baseline

### Установление Baseline

```python
# Сбор baseline метрик
class BaselineCollector:
    def __init__(self, prometheus_url: str):
        self.prom = PrometheusConnect(url=prometheus_url)

    def collect_baseline(self, service: str, days: int = 30) -> dict:
        """
        Собирает baseline метрики за указанный период.
        """
        metrics = {}

        # Latency percentiles
        for p in [50, 90, 95, 99]:
            query = f'''
                histogram_quantile(0.{p},
                    sum(rate(http_request_duration_seconds_bucket{{
                        service="{service}"
                    }}[5m])) by (le)
                )
            '''
            metrics[f'latency_p{p}'] = self._get_stats(query, days)

        # Throughput
        query = f'sum(rate(http_requests_total{{service="{service}"}}[5m]))'
        metrics['throughput'] = self._get_stats(query, days)

        # Error rate
        query = f'''
            sum(rate(http_requests_total{{service="{service}", status=~"5.."}}[5m]))
            / sum(rate(http_requests_total{{service="{service}"}}[5m]))
        '''
        metrics['error_rate'] = self._get_stats(query, days)

        # Resource utilization
        metrics['cpu_utilization'] = self._get_resource_stats(service, 'cpu', days)
        metrics['memory_utilization'] = self._get_resource_stats(service, 'memory', days)

        return metrics

    def _get_stats(self, query: str, days: int) -> dict:
        """Возвращает статистику по запросу."""
        data = self.prom.custom_query_range(
            query=query,
            start_time=datetime.now() - timedelta(days=days),
            end_time=datetime.now(),
            step='1h'
        )
        values = [float(v[1]) for v in data[0]['values'] if v[1] != 'NaN']

        return {
            'min': min(values),
            'max': max(values),
            'avg': sum(values) / len(values),
            'p50': np.percentile(values, 50),
            'p95': np.percentile(values, 95),
            'p99': np.percentile(values, 99)
        }
```

### Документирование Baseline

```yaml
# baseline.yaml
service: payment-service
collected_at: "2024-01-15"
collection_period_days: 30

baseline_metrics:
  latency:
    p50: 45ms
    p90: 120ms
    p95: 180ms
    p99: 350ms

  throughput:
    avg: 2500 RPS
    peak: 5200 RPS
    peak_time: "Friday 18:00-20:00"

  error_rate:
    avg: 0.05%
    max: 0.3%

  resources:
    cpu:
      avg: 35%
      peak: 72%
    memory:
      avg: 60%
      peak: 78%

capacity_limits:
  max_rps: 8000
  max_concurrent_connections: 5000

scaling_triggers:
  cpu_threshold: 70%
  memory_threshold: 80%
  latency_p99_threshold: 500ms
```

## Load Forecasting

### Методы прогнозирования

#### 1. Линейная экстраполяция

```python
from sklearn.linear_model import LinearRegression
import numpy as np

def linear_forecast(historical_data: list, days_ahead: int) -> float:
    """
    Простая линейная экстраполяция.

    historical_data: [(timestamp, value), ...]
    """
    X = np.array([i for i in range(len(historical_data))]).reshape(-1, 1)
    y = np.array([d[1] for d in historical_data])

    model = LinearRegression()
    model.fit(X, y)

    future_point = len(historical_data) + days_ahead
    forecast = model.predict([[future_point]])[0]

    return {
        'forecast': forecast,
        'growth_rate_per_day': model.coef_[0],
        'days_to_capacity': (capacity_limit - y[-1]) / model.coef_[0]
    }
```

#### 2. Сезонное прогнозирование (Prophet)

```python
from prophet import Prophet
import pandas as pd

def seasonal_forecast(
    historical_data: pd.DataFrame,
    periods: int = 90,
    include_holidays: bool = True
) -> pd.DataFrame:
    """
    Прогнозирование с учётом сезонности.

    historical_data: DataFrame с колонками 'ds' (date) и 'y' (value)
    """
    model = Prophet(
        yearly_seasonality=True,
        weekly_seasonality=True,
        daily_seasonality=True,
        changepoint_prior_scale=0.05
    )

    if include_holidays:
        # Добавление праздников и специальных событий
        model.add_country_holidays(country_name='RU')

        # Добавление бизнес-событий
        special_events = pd.DataFrame({
            'holiday': 'black_friday',
            'ds': pd.to_datetime(['2024-11-29', '2025-11-28']),
            'lower_window': -1,
            'upper_window': 1,
        })
        model.add_seasonality(name='black_friday', period=365, fourier_order=5)

    model.fit(historical_data)

    future = model.make_future_dataframe(periods=periods)
    forecast = model.predict(future)

    return forecast[['ds', 'yhat', 'yhat_lower', 'yhat_upper']]
```

#### 3. ML-based прогнозирование

```python
from sklearn.ensemble import GradientBoostingRegressor
from sklearn.preprocessing import StandardScaler

class MLCapacityForecaster:
    def __init__(self):
        self.model = GradientBoostingRegressor(
            n_estimators=100,
            max_depth=5,
            learning_rate=0.1
        )
        self.scaler = StandardScaler()

    def prepare_features(self, df: pd.DataFrame) -> pd.DataFrame:
        """Создание признаков для ML модели."""
        features = pd.DataFrame()

        # Временные признаки
        features['hour'] = df['timestamp'].dt.hour
        features['day_of_week'] = df['timestamp'].dt.dayofweek
        features['month'] = df['timestamp'].dt.month
        features['is_weekend'] = df['timestamp'].dt.dayofweek >= 5

        # Лаговые признаки
        for lag in [1, 7, 30]:
            features[f'load_lag_{lag}d'] = df['load'].shift(lag * 24)

        # Скользящие средние
        for window in [24, 168, 720]:  # 1 day, 1 week, 1 month
            features[f'load_ma_{window}h'] = df['load'].rolling(window).mean()

        # Бизнес-метрики
        features['active_users'] = df['active_users']
        features['marketing_spend'] = df['marketing_spend']

        return features.dropna()

    def train(self, df: pd.DataFrame):
        features = self.prepare_features(df)
        X = self.scaler.fit_transform(features)
        y = df['load'].iloc[len(df) - len(features):]

        self.model.fit(X, y)

    def forecast(self, future_features: pd.DataFrame) -> np.array:
        X = self.scaler.transform(future_features)
        return self.model.predict(X)
```

### Прогноз роста

```yaml
# growth_forecast.yaml
service: api-gateway
forecast_date: "2024-01-15"

current_state:
  daily_requests: 50M
  peak_rps: 2500
  active_users: 100K

growth_assumptions:
  organic_growth: 5%  # monthly
  marketing_campaigns:
    - date: "2024-03-01"
      expected_increase: 20%
    - date: "2024-11-29"  # Black Friday
      expected_increase: 300%

forecasts:
  3_months:
    daily_requests: 58M
    peak_rps: 2900
    confidence: 90%

  6_months:
    daily_requests: 67M
    peak_rps: 3350
    confidence: 80%

  12_months:
    daily_requests: 90M
    peak_rps: 4500
    confidence: 60%

capacity_planning:
  current_capacity: 5000 RPS
  time_to_80_percent: "4 months"
  recommended_scaling_date: "2024-04-01"
```

## Bottleneck Analysis

### Методология выявления узких мест

```python
class BottleneckAnalyzer:
    def __init__(self, metrics_client):
        self.metrics = metrics_client

    def analyze_system(self, service: str) -> dict:
        """
        Комплексный анализ узких мест системы.
        """
        bottlenecks = []

        # 1. CPU Bottleneck
        cpu_analysis = self.analyze_cpu(service)
        if cpu_analysis['is_bottleneck']:
            bottlenecks.append({
                'type': 'CPU',
                'severity': cpu_analysis['severity'],
                'details': cpu_analysis,
                'recommendation': 'Horizontal scaling or CPU optimization'
            })

        # 2. Memory Bottleneck
        memory_analysis = self.analyze_memory(service)
        if memory_analysis['is_bottleneck']:
            bottlenecks.append({
                'type': 'Memory',
                'severity': memory_analysis['severity'],
                'details': memory_analysis,
                'recommendation': 'Memory optimization or vertical scaling'
            })

        # 3. Database Bottleneck
        db_analysis = self.analyze_database(service)
        if db_analysis['is_bottleneck']:
            bottlenecks.append({
                'type': 'Database',
                'severity': db_analysis['severity'],
                'details': db_analysis,
                'recommendation': db_analysis['recommendation']
            })

        # 4. Network Bottleneck
        network_analysis = self.analyze_network(service)
        if network_analysis['is_bottleneck']:
            bottlenecks.append({
                'type': 'Network',
                'severity': network_analysis['severity'],
                'details': network_analysis,
                'recommendation': 'Network optimization or CDN'
            })

        # 5. External Dependencies
        deps_analysis = self.analyze_dependencies(service)
        for dep in deps_analysis['slow_dependencies']:
            bottlenecks.append({
                'type': 'External Dependency',
                'severity': dep['severity'],
                'details': dep,
                'recommendation': f"Optimize {dep['name']} or add caching"
            })

        return {
            'bottlenecks': sorted(bottlenecks, key=lambda x: x['severity'], reverse=True),
            'primary_bottleneck': bottlenecks[0] if bottlenecks else None,
            'overall_health': self.calculate_health_score(bottlenecks)
        }

    def analyze_database(self, service: str) -> dict:
        """Анализ базы данных как узкого места."""
        metrics = {
            'connection_pool_usage': self.metrics.query(
                f'pg_stat_activity_count{{service="{service}"}}'
                f' / pg_settings_max_connections{{service="{service}"}}'
            ),
            'query_latency_p99': self.metrics.query(
                f'histogram_quantile(0.99, pg_query_duration_seconds_bucket{{service="{service}"}})'
            ),
            'lock_wait_time': self.metrics.query(
                f'pg_stat_activity_wait_event_count{{service="{service}", wait_event_type="Lock"}}'
            ),
            'replication_lag': self.metrics.query(
                f'pg_replication_lag_seconds{{service="{service}"}}'
            )
        }

        is_bottleneck = (
            metrics['connection_pool_usage'] > 0.8 or
            metrics['query_latency_p99'] > 0.1 or
            metrics['lock_wait_time'] > 10
        )

        recommendation = []
        if metrics['connection_pool_usage'] > 0.8:
            recommendation.append('Increase connection pool or add PgBouncer')
        if metrics['query_latency_p99'] > 0.1:
            recommendation.append('Optimize slow queries or add indexes')
        if metrics['lock_wait_time'] > 10:
            recommendation.append('Review transaction isolation or reduce lock contention')

        return {
            'is_bottleneck': is_bottleneck,
            'severity': self.calculate_severity(metrics),
            'metrics': metrics,
            'recommendation': '; '.join(recommendation)
        }
```

### Визуализация узких мест

```yaml
# Grafana dashboard для bottleneck analysis
dashboard:
  title: "Bottleneck Analysis"

  rows:
    - title: "Resource Saturation"
      panels:
        - title: "CPU Saturation"
          type: gauge
          query: |
            avg(rate(container_cpu_usage_seconds_total[5m]))
            / avg(container_spec_cpu_quota / container_spec_cpu_period)
          thresholds:
            - value: 0.6
              color: green
            - value: 0.8
              color: orange
            - value: 0.9
              color: red

        - title: "Memory Saturation"
          type: gauge
          query: |
            container_memory_working_set_bytes
            / container_spec_memory_limit_bytes

        - title: "Connection Pool Saturation"
          type: gauge
          query: |
            pg_stat_activity_count / pg_settings_max_connections

    - title: "Latency Breakdown"
      panels:
        - title: "Request Latency by Component"
          type: stacked_bar
          queries:
            - label: "App Processing"
              query: "app_processing_duration_seconds"
            - label: "Database"
              query: "db_query_duration_seconds"
            - label: "External APIs"
              query: "external_api_duration_seconds"
            - label: "Network"
              query: "network_latency_seconds"
```

## Load Testing

### Стратегии нагрузочного тестирования

```yaml
load_testing_strategies:
  smoke_test:
    description: "Базовая проверка работоспособности"
    duration: "1-5 minutes"
    load: "10% of expected"
    goal: "Verify system works"

  load_test:
    description: "Тестирование при ожидаемой нагрузке"
    duration: "30-60 minutes"
    load: "Expected average load"
    goal: "Verify SLO compliance"

  stress_test:
    description: "Тестирование за пределами нормы"
    duration: "30-60 minutes"
    load: "150-200% of expected"
    goal: "Find breaking point"

  spike_test:
    description: "Тестирование резких скачков"
    pattern: "Sudden increase to 500%+"
    duration: "5-10 minutes spike"
    goal: "Test auto-scaling"

  soak_test:
    description: "Длительное тестирование"
    duration: "4-24 hours"
    load: "80% of capacity"
    goal: "Find memory leaks, degradation"
```

### k6 нагрузочные тесты

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Custom metrics
const errorRate = new Rate('errors');
const latencyP99 = new Trend('latency_p99');

// Test configuration
export const options = {
  scenarios: {
    // Baseline load
    baseline: {
      executor: 'constant-arrival-rate',
      rate: 1000,
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 100,
      maxVUs: 500,
    },

    // Ramp up test
    ramp_up: {
      executor: 'ramping-arrival-rate',
      startRate: 100,
      timeUnit: '1s',
      preAllocatedVUs: 500,
      maxVUs: 2000,
      stages: [
        { duration: '5m', target: 500 },
        { duration: '10m', target: 1000 },
        { duration: '10m', target: 2000 },
        { duration: '5m', target: 3000 },
        { duration: '10m', target: 3000 },  // Hold at peak
        { duration: '5m', target: 0 },
      ],
    },

    // Spike test
    spike: {
      executor: 'ramping-arrival-rate',
      startRate: 500,
      timeUnit: '1s',
      preAllocatedVUs: 1000,
      maxVUs: 5000,
      stages: [
        { duration: '2m', target: 500 },
        { duration: '30s', target: 5000 },  // Spike!
        { duration: '5m', target: 5000 },
        { duration: '30s', target: 500 },
      ],
    },
  },

  thresholds: {
    http_req_duration: ['p(99)<500'],  // 99% requests < 500ms
    errors: ['rate<0.01'],              // Error rate < 1%
    http_req_failed: ['rate<0.01'],
  },
};

export default function() {
  const params = {
    headers: {
      'Content-Type': 'application/json',
      'X-Request-ID': `test-${__VU}-${__ITER}`,
    },
    timeout: '10s',
  };

  // Critical user journey
  const endpoints = [
    { method: 'GET', url: '/api/products', weight: 40 },
    { method: 'GET', url: '/api/products/123', weight: 30 },
    { method: 'POST', url: '/api/cart', weight: 20 },
    { method: 'POST', url: '/api/orders', weight: 10 },
  ];

  // Weighted random selection
  const endpoint = selectWeighted(endpoints);

  const response = http.request(endpoint.method, `${__ENV.BASE_URL}${endpoint.url}`, null, params);

  // Record metrics
  check(response, {
    'status is 2xx': (r) => r.status >= 200 && r.status < 300,
    'latency < 500ms': (r) => r.timings.duration < 500,
  });

  errorRate.add(response.status >= 400);
  latencyP99.add(response.timings.duration);

  sleep(Math.random() * 2);  // Random think time
}

function selectWeighted(items) {
  const totalWeight = items.reduce((sum, item) => sum + item.weight, 0);
  let random = Math.random() * totalWeight;

  for (const item of items) {
    random -= item.weight;
    if (random <= 0) return item;
  }
  return items[0];
}
```

### Locust для Python

```python
# locustfile.py
from locust import HttpUser, task, between, events
from locust.runners import MasterRunner
import logging

class APIUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        """Выполняется при старте каждого пользователя."""
        # Аутентификация
        response = self.client.post("/auth/login", json={
            "username": "test_user",
            "password": "test_password"
        })
        self.token = response.json().get("token")
        self.client.headers = {"Authorization": f"Bearer {self.token}"}

    @task(40)
    def browse_products(self):
        """40% трафика - просмотр каталога."""
        self.client.get("/api/products")

    @task(30)
    def view_product(self):
        """30% трафика - просмотр товара."""
        product_id = random.randint(1, 10000)
        self.client.get(f"/api/products/{product_id}")

    @task(20)
    def add_to_cart(self):
        """20% трафика - добавление в корзину."""
        self.client.post("/api/cart", json={
            "product_id": random.randint(1, 10000),
            "quantity": random.randint(1, 5)
        })

    @task(10)
    def checkout(self):
        """10% трафика - оформление заказа."""
        self.client.post("/api/orders", json={
            "payment_method": "card",
            "delivery_address": "Test Address"
        })


# Custom metrics collection
@events.request.add_listener
def on_request(request_type, name, response_time, response_length, exception, **kwargs):
    if exception:
        logging.error(f"Request failed: {name}, {exception}")

    # Send to Prometheus/Grafana
    if response_time > 500:
        logging.warning(f"Slow request: {name}, {response_time}ms")


# Capacity test configuration
@events.init.add_listener
def on_locust_init(environment, **kwargs):
    if isinstance(environment.runner, MasterRunner):
        print("Running as master")

        # Define test stages
        environment.runner.register_message('capacity_test', start_capacity_test)


def start_capacity_test(environment, msg, **kwargs):
    """Автоматический capacity test."""
    stages = [
        (100, 60),    # 100 users for 60s
        (500, 120),   # 500 users for 120s
        (1000, 180),  # 1000 users for 180s
        (2000, 300),  # 2000 users for 300s
    ]

    for users, duration in stages:
        environment.runner.start(users, spawn_rate=10)
        time.sleep(duration)

        # Check if system is healthy
        if environment.runner.stats.total.fail_ratio > 0.05:
            print(f"System failing at {users} users")
            break
```

## Capacity Planning Process

### Процесс планирования

```yaml
capacity_planning_process:
  frequency: "quarterly"
  participants:
    - SRE Team
    - Product Managers
    - Finance
    - Engineering Leads

  steps:
    1_data_collection:
      duration: "1 week"
      activities:
        - "Collect historical metrics"
        - "Document current capacity"
        - "Identify growth trends"

    2_demand_forecasting:
      duration: "1 week"
      activities:
        - "Analyze business roadmap"
        - "Review marketing plans"
        - "Apply forecasting models"
        - "Account for seasonality"

    3_gap_analysis:
      duration: "2-3 days"
      activities:
        - "Compare forecast vs capacity"
        - "Identify bottlenecks"
        - "Assess risk levels"

    4_planning:
      duration: "1 week"
      activities:
        - "Design scaling solutions"
        - "Estimate costs"
        - "Create timeline"
        - "Define milestones"

    5_review_approval:
      duration: "2-3 days"
      activities:
        - "Present to stakeholders"
        - "Get budget approval"
        - "Document decisions"
```

### Capacity Planning Document

```yaml
# capacity-plan-q2-2024.yaml
document:
  title: "Q2 2024 Capacity Plan"
  version: "1.0"
  date: "2024-03-15"
  author: "SRE Team"
  reviewers: ["CTO", "VP Engineering", "Finance Director"]

executive_summary: |
  Based on projected 40% growth in Q2, we need to scale compute by 50%
  and database by 30%. Total additional monthly cost: $45,000.
  Risk of capacity issues without scaling: High (85% probability).

current_state:
  services:
    api_gateway:
      current_capacity: 5000 RPS
      current_utilization: 65%
      bottleneck: "CPU"

    user_service:
      current_capacity: 3000 RPS
      current_utilization: 70%
      bottleneck: "Database connections"

    order_service:
      current_capacity: 1000 RPS
      current_utilization: 55%
      bottleneck: "External payment API"

demand_forecast:
  methodology: "Prophet + business input"
  confidence_level: 80%

  projections:
    end_of_q2:
      daily_active_users: 150K  # +50%
      peak_rps: 7500  # +50%
      daily_transactions: 25K  # +40%

    assumptions:
      - "Marketing campaign in April (+20% users)"
      - "New product launch in May"
      - "Normal organic growth 5%/month"

capacity_gaps:
  critical:
    - service: "api_gateway"
      gap: "2500 RPS"
      risk: "Service degradation during peak"
      timeline: "April 15"

  warning:
    - service: "user_service"
      gap: "500 RPS"
      risk: "Slow authentication during peak"
      timeline: "May 1"

scaling_plan:
  immediate_actions:
    - action: "Scale API Gateway to 10 replicas"
      cost: "+$5,000/month"
      timeline: "April 1"
      owner: "Platform Team"

    - action: "Add read replicas for User DB"
      cost: "+$3,000/month"
      timeline: "April 10"
      owner: "Database Team"

  q2_actions:
    - action: "Migrate to larger DB instance"
      cost: "+$10,000/month"
      timeline: "May 1"

    - action: "Implement caching layer"
      cost: "+$2,000/month + 2 weeks engineering"
      timeline: "May 15"

    - action: "Optimize payment API integration"
      cost: "2 weeks engineering"
      timeline: "April-May"

budget:
  current_monthly: $120,000
  projected_monthly: $165,000
  increase: $45,000 (+37.5%)

  breakdown:
    compute: +$25,000
    database: +$13,000
    networking: +$5,000
    monitoring: +$2,000

risks:
  - risk: "Forecast underestimates growth"
    probability: "Medium"
    mitigation: "Build 30% headroom"

  - risk: "New feature delays scaling"
    probability: "Low"
    mitigation: "Decouple scaling from features"

approval:
  status: "Pending"
  required_by: "2024-03-25"
  approvers:
    - name: "CTO"
      decision: ""
    - name: "Finance Director"
      decision: ""
```

## Мониторинг и алерты

### Capacity метрики

```yaml
# Prometheus recording rules
groups:
  - name: capacity_metrics
    interval: 1m
    rules:
      # Capacity utilization
      - record: service:capacity_utilization:ratio
        expr: |
          sum(rate(http_requests_total[5m])) by (service)
          / on(service) service:capacity_limit:rps

      # Time to capacity exhaustion
      - record: service:time_to_capacity_exhaustion:days
        expr: |
          (service:capacity_limit:rps - sum(rate(http_requests_total[5m])) by (service))
          / (
            predict_linear(sum(rate(http_requests_total[5m])) by (service)[30d:1d], 0)
            - sum(rate(http_requests_total[5m])) by (service)
          ) / 86400

      # Resource headroom
      - record: service:cpu_headroom:ratio
        expr: |
          1 - (
            sum(rate(container_cpu_usage_seconds_total[5m])) by (service)
            / sum(container_spec_cpu_quota / container_spec_cpu_period) by (service)
          )
```

### Алерты

```yaml
groups:
  - name: capacity_alerts
    rules:
      - alert: CapacityUtilizationHigh
        expr: service:capacity_utilization:ratio > 0.8
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Service {{ $labels.service }} at {{ $value | humanizePercentage }} capacity"
          runbook: "https://wiki/runbooks/capacity-high"

      - alert: CapacityUtilizationCritical
        expr: service:capacity_utilization:ratio > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.service }} at critical capacity"
          action: "Immediate scaling required"

      - alert: CapacityExhaustionSoon
        expr: service:time_to_capacity_exhaustion:days < 14
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Service {{ $labels.service }} will exhaust capacity in {{ $value }} days"
          action: "Plan scaling within 2 weeks"

      - alert: HeadroomInsufficient
        expr: service:cpu_headroom:ratio < 0.2
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Service {{ $labels.service }} has only {{ $value | humanizePercentage }} CPU headroom"
```

### Dashboard

```yaml
# Grafana Capacity Planning Dashboard
dashboard:
  title: "Capacity Planning Overview"
  refresh: "1m"

  variables:
    - name: service
      type: query
      query: "label_values(http_requests_total, service)"

  rows:
    - title: "Capacity Overview"
      panels:
        - title: "Current Capacity Utilization"
          type: gauge
          query: "service:capacity_utilization:ratio{service='$service'}"
          thresholds: [0.6, 0.8, 0.9]

        - title: "Days Until Capacity Exhaustion"
          type: stat
          query: "service:time_to_capacity_exhaustion:days{service='$service'}"
          colorMode: "background"
          thresholds: [7, 14, 30]

    - title: "Trends"
      panels:
        - title: "Load Growth Trend"
          type: timeseries
          queries:
            - expr: "sum(rate(http_requests_total{service='$service'}[1h]))"
              legendFormat: "Current"
            - expr: "predict_linear(sum(rate(http_requests_total{service='$service'}[1h]))[30d:1d], 86400*30)"
              legendFormat: "30d Forecast"

        - title: "Capacity vs Load"
          type: timeseries
          queries:
            - expr: "service:capacity_limit:rps{service='$service'}"
              legendFormat: "Capacity"
            - expr: "sum(rate(http_requests_total{service='$service'}[5m]))"
              legendFormat: "Current Load"
```

## Best Practices

### 1. Регулярный пересмотр

```yaml
review_schedule:
  weekly:
    - "Check capacity utilization trends"
    - "Review alerts triggered"

  monthly:
    - "Update baseline metrics"
    - "Review growth predictions accuracy"
    - "Adjust forecasting models"

  quarterly:
    - "Full capacity planning cycle"
    - "Budget review"
    - "Architecture review for bottlenecks"
```

### 2. Автоматизация

```python
# Автоматический capacity report
class CapacityReporter:
    def generate_weekly_report(self) -> dict:
        report = {
            'generated_at': datetime.now().isoformat(),
            'services': {}
        }

        for service in self.get_services():
            report['services'][service] = {
                'utilization': self.get_utilization(service),
                'trend': self.calculate_trend(service),
                'days_to_capacity': self.predict_exhaustion(service),
                'recommendations': self.get_recommendations(service),
                'status': self.determine_status(service)
            }

        # Send to Slack
        self.send_slack_report(report)

        # Create Jira tickets for warnings
        for service, data in report['services'].items():
            if data['status'] in ['warning', 'critical']:
                self.create_jira_ticket(service, data)

        return report
```

### 3. Buffer Planning

```yaml
buffer_strategy:
  minimum_headroom: 30%

  adjustments:
    # Increase buffer for critical services
    tier_1_services:
      headroom: 40%
      reason: "Zero tolerance for degradation"

    # Special events
    marketing_campaigns:
      additional_buffer: 50%
      duration: "campaign + 1 week"

    # Seasonal
    black_friday:
      additional_buffer: 200%
      duration: "1 week"
```

## Anti-patterns

### 1. Reactive scaling
```
❌ "Масштабируемся когда падаем"
✅ Проактивное планирование с headroom
```

### 2. Over-provisioning
```
❌ "Запас 300% на всякий случай"
✅ Data-driven sizing с разумным buffer
```

### 3. Ignoring business context
```
❌ "Линейная экстраполяция достаточна"
✅ Учёт бизнес-событий, сезонности, маркетинга
```

### 4. One-time planning
```
❌ "Спланировали год назад"
✅ Регулярный пересмотр и адаптация
```

## Checklist

### Baseline
- [ ] Собраны метрики производительности за 30+ дней
- [ ] Задокументированы текущие лимиты capacity
- [ ] Определены узкие места
- [ ] Установлены пороговые значения

### Forecasting
- [ ] Выбрана методология прогнозирования
- [ ] Учтены бизнес-события
- [ ] Учтена сезонность
- [ ] Определены confidence intervals

### Testing
- [ ] Проведён baseline load test
- [ ] Проведён stress test
- [ ] Определена точка отказа
- [ ] Протестировано auto-scaling

### Process
- [ ] Определён процесс capacity planning
- [ ] Назначены ответственные
- [ ] Настроен мониторинг
- [ ] Автоматизированы отчёты

## Ссылки

- [Google SRE Book - Capacity Planning](https://sre.google/sre-book/software-engineering-in-sre/)
- [AWS Well-Architected - Performance](https://docs.aws.amazon.com/wellarchitected/latest/performance-efficiency-pillar/)
- [k6 Load Testing](https://k6.io/docs/)
- [Locust Documentation](https://docs.locust.io/)
- [Prophet Forecasting](https://facebook.github.io/prophet/)
- [Grafana Capacity Planning](https://grafana.com/blog/2019/10/30/how-to-use-grafana-for-capacity-planning/)
