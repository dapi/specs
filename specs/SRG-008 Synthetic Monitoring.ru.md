# SRG-008 Synthetic Monitoring (Синтетический мониторинг)

Synthetic Monitoring - это метод мониторинга, который использует автоматизированные скрипты для эмуляции поведения пользователей и проверки работоспособности системы с внешней точки зрения.

---

## Чем отличается от других типов мониторинга

| Тип | Как работает | Где запускается | Что видит | Примеры |
|-----|--------------|-----------------|-----------|---------|
| **Synthetic** | Эмулирует пользователя | Извне (разные локации) | Пользовательский путь полностью | Login → Browse → Checkout |
| **Real User** | Собирает метрики реальных юзеров | В браузерах юзеров | Реальное пользовательское поведение | Page Load Time, Core Web Vitals |
| **Application** | Метрики из приложения | Внутри приложения | Производительность кода | HTTP Duration, Error Rates |
| **Infrastructure** | Метрики системы | На хостах | Ресурсы системы | CPU, Memory, Disk |

---

## Типы Synthetic Checks

### 1. HTTP/HTTPS Checks (Проверки HTTP)

Простые HTTP запросы для проверки доступности и производительности endpoints.

```bash
# Пример с curl
#!/bin/bash
# check-health.sh

START_TIME=$(date +%s%3N)
RESPONSE=$(curl -s -w "%{http_code} %{time_total}" https://api.example.com/health)
END_TIME=$(date +%s%3N)

HTTP_CODE=$(echo $RESPONSE | cut -d' ' -f1)
RESPONSE_TIME=$(echo $RESPONSE | cut -d' ' -f2)
DURATION=$((END_TIME - START_TIME))

if [ "$HTTP_CODE" = "200" ]; then
  echo "✅ Health check passed (HTTP $HTTP_CODE) in ${RESPONSE_TIME}s"
  exit 0
else
  echo "❌ Health check failed (HTTP $HTTP_CODE) in ${RESPONSE_TIME}s"
  exit 1
fi
```

**Что проверять:**
- HTTP status codes (200, 4xx, 5xx)
- Response time (P50, P95, P99)
- SSL certificate validity
- Content verification (проверка наличия ключевых слов)
- HTTP headers

```python
# Пример на Python
import requests
import time
import ssl

class HTTPHealthCheck:
    def __init__(self, url):
        self.url = url

    def check(self):
        start = time.time()
        request_info = {
            'timestamp': time.time(),
            'url': self.url,
            'method': 'GET'
        }

        try:
            response = requests.get(
                self.url,
                timeout=10,
                verify=True,
                allow_redirects=True
            )
            duration = time.time() - start

            result = {
                **request_info,
                'status_code': response.status_code,
                'response_time': duration,
                'ssl_valid': self._verify_ssl(),
                'content_verified': 'status' in response.text,
                'result': 'success' if response.status_code == 200 else 'failure'
            }

            return result

        except Exception as e:
            duration = time.time() - start
            return {
                **request_info,
                'status_code': None,
                'response_time': duration,
                'error': str(e),
                'result': 'failure'
            }

    def _verify_ssl(self):
        """Проверка SSL сертификата"""
        try:
            hostname = self.url.split('//')[1].split('/')[0]
            ctx = ssl.create_default_context()
            with ctx.wrap_socket(None, server_hostname=hostname) as s:
                s.connect((hostname, 443))
                cert = s.getpeercert()
                return True
        except:
            return False
```

### 2. SSL Certificate Checks (Проверка сертификатов)

```python
import ssl
import socket
from datetime import datetime, timedelta

class SSLChecker:
    def __init__(self, hostname):
        self.hostname = hostname

    def check_certificate(self):
        """Проверяет SSL сертификат"""
        try:
            context = ssl.create_default_context()
            with socket.create_connection((self.hostname, 443), timeout=10) as sock:
                with context.wrap_socket(sock, server_hostname=self.hostname) as ssock:
                    cert = ssock.getpeercert()

                    # Извлекаем даты
                    not_before = datetime.strptime(cert['notBefore'], '%b %d %H:%M:%S %Y %Z')
                    not_after = datetime.strptime(cert['notAfter'], '%b %d %H:%M:%S %Y %Z')
                    now = datetime.now()

                    # Проверяем валидность
                    days_remaining = (not_after - now).days
                    is_valid = not_before <= now <= not_after

                    return {
                        'hostname': self.hostname,
                        'valid': is_valid,
                        'days_remaining': days_remaining,
                        'issuer': dict(cert['issuer']),
                        'subject': dict(cert['subject']),
                        'warning': days_remaining < 30
                    }

        except Exception as e:
            return {
                'hostname': self.hostname,
                'valid': False,
                'error': str(e)
            }

# Использование
checker = SSLChecker('api.example.com')
result = checker.check_certificate()
print(f"Certificate valid: {result['valid']}")
print(f"Days remaining: {result['days_remaining']}")
```

### 3. TCP Checks (Проверки TCP соединения)

```python
import socket
import time

class TCPHealthCheck:
    def __init__(self, host, port, timeout=5):
        self.host = host
        self.port = port
        self.timeout = timeout

    def check(self):
        start = time.time()

        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(self.timeout)

            result = sock.connect_ex((self.host, self.port))
            duration = time.time() - start
            sock.close()

            return {
                'timestamp': time.time(),
                'host': self.host,
                'port': self.port,
                'result': 'success' if result == 0 else 'failure',
                'response_time': duration,
                'error_code': result if result != 0 else None
            }

        except Exception as e:
            duration = time.time() - start
            return {
                'timestamp': time.time(),
                'host': self.host,
                'port': self.port,
                'result': 'failure',
                'response_time': duration,
                'error': str(e)
            }
```

### 4. DNS Checks (Проверки DNS)

```python
import dns.resolver
import time

class DNSChecker:
    def __init__(self, domain):
        self.domain = domain

    def check(self):
        start = time.time()
        checks = {}

        try:
            # Проверка A записи
            answers = dns.resolver.resolve(self.domain, 'A')
            checks['A'] = {'status': 'ok', 'records': [str(r) for r in answers]}
        except Exception as e:
            checks['A'] = {'status': 'error', 'error': str(e)}

        try:
            # Проверка MX записи
            answers = dns.resolver.resolve(self.domain, 'MX')
            checks['MX'] = {'status': 'ok', 'records': [(str(r.preference), str(r.exchange)) for r in answers]}
        except Exception as e:
            checks['MX'] = {'status': 'error', 'error': str(e)}

        try:
            # Проверка TXT записи (SPF)
            answers = dns.resolver.resolve(self.domain, 'TXT')
            checks['TXT'] = {'status': 'ok', 'records': [str(r) for r in answers]}
        except Exception as e:
            checks['TXT'] = {'status': 'error', 'error': str(e)}

        duration = time.time() - start

        return {
            'timestamp': time.time(),
            'domain': self.domain,
            'checks': checks,
            'duration': duration,
            'result': 'success' if checks['A']['status'] == 'ok' else 'failure'
        }
```

### 5. Браузерные Checks (с Selenium/Puppeteer)

**Puppeteer (JavaScript/Node.js):**
```javascript
const puppeteer = require('puppeteer');

async function runBrowserCheck() {
  const startTime = Date.now();
  const metrics = {
    timestamp: new Date().toISOString(),
    url: 'https://example.com',
    type: 'browser_check'
  };

  let browser;
  try {
    browser = await puppeteer.launch({
      headless: true,
      args: ['--no-sandbox', '--disable-setuid-sandbox']
    });

    const page = await browser.newPage();

    // Отслеживание метрик
    await page.metrics().then(metrics => {
      metrics.pageMetrics = metrics;
    });

    // Отслеживание performance
    const performance = await page.evaluate(() => {
      return JSON.parse(JSON.stringify(window.performance.timing));
    });

    // Навигация и ожидание
    await page.goto('https://example.com', {
      waitUntil: 'networkidle2',
      timeout: 30000
    });

    // Проверка конкретных элементов
    const loginButton = await page.$('#login-button');
    const isButtonVisible = await page.evaluate(button => {
      const style = window.getComputedStyle(button);
      return style.display !== 'none' && style.visibility !== 'hidden';
    }, loginButton);

    // Ввод данных
    await page.type('#username', 'test@example.com');
    await page.type('#password', 'password123');

    // Клик и навигация
    await Promise.all([
      page.waitForNavigation(),
      page.click('#login-button')
    ]);

    // Скриншот
    await page.screenshot({ path: 'screenshot.png' });

    const duration = Date.now() - startTime;

    return {
      ...metrics,
      result: 'success',
      duration_ms: duration,
      page_metrics: metrics.pageMetrics,
      performance_timing: performance,
      elements_found: {
        login_button: isButtonVisible
      }
    };

  } catch (error) {
    return {
      ...metrics,
      result: 'failure',
      error: error.message
    };
  } finally {
    if (browser) {
      await browser.close();
    }
  }
}

runBrowserCheck().then(result => console.log(result));
```

**Selenium (Python):**
```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time

class BrowserCheck:
    def __init__(self, url):
        self.url = url

    def run_check(self):
        options = webdriver.ChromeOptions()
        options.add_argument('--headless')
        options.add_argument('--no-sandbox')

        driver = webdriver.Chrome(options=options)
        metrics = {'url': self.url, 'timestamp': time.time()}

        try:
            start = time.time()
            driver.get(self.url)

            # Ожидание загрузки
            wait = WebDriverWait(driver, 10)
            wait.until(EC.presence_of_element_located((By.TAG_NAME, "body")))

            # Проверка заголовка
            title = driver.title

            # Screenshot
            driver.save_screenshot('screenshot.png')

            # Проверка корзины (пример для e-commerce)
            try:
                add_to_cart_btn = driver.find_element(By.ID, "add-to-cart")
                add_to_cart_btn.click()

                # Ожидание обновления корзины
                wait.until(EC.text_to_be_present_in_element(
                    (By.ID, "cart-count"), "1"
                ))

                cart_check = True
            except:
                cart_check = False

            duration = time.time() - start

            return {
                **metrics,
                'result': 'success',
                'duration_ms': duration * 1000,
                'title': title,
                'element_checks': {
                    'cart_functionality': cart_check
                }
            }

        except Exception as e:
            return {
                **metrics,
                'result': 'failure',
                'error': str(e)
            }
        finally:
            driver.quit()
```

### 6. Multi-step API Checks (сценарии из нескольких шагов)

```python
class MultiStepAPICheck:
    def __init__(self, base_url):
        self.base_url = base_url
        self.session = requests.Session()
        self.metrics = []

    def check_user_journey(self):
        """Проверка полного пользовательского пути"""
        start = time.time()

        # Шаг 1: Login
        result1 = self._login()
        if not result1['success']:
            return self._fail('login_failed', result1['error'])
        self.metrics.append(('login', result1['duration']))

        # Шаг 2: Get user profile
        result2 = self._get_profile()
        if not result2['success']:
            return self._fail('profile_failed', result2['error'])
        self.metrics.append(('profile', result2['duration']))

        # Шаг 3: Create order (e-commerce)
        result3 = self._create_order()
        if not result3['success']:
            return self._fail('order_failed', result3['error'])
        self.metrics.append(('create_order', result3['duration']))

        # Шаг 4: Payment
        result4 = self._process_payment()
        if not result4['success']:
            return self._fail('payment_failed', result4['error'])
        self.metrics.append(('payment', result4['duration']))

        return self._success()

    def _login(self):
        """Login в систему"""
        start = time.time()
        try:
            response = self.session.post(
                f"{self.base_url}/auth/login",
                json={
                    "email": "test@example.com",
                    "password": "test_password"
                },
                timeout=10
            )

            duration = time.time() - start

            if response.status_code == 200:
                self.auth_token = response.json()['token']
                return {'success': True, 'duration': duration}
            else:
                return {'success': False, 'error': f"HTTP {response.status_code}"}

        except Exception as e:
            return {'success': False, 'error': str(e), 'duration': time.time() - start}

    def _get_profile(self):
        """Получение профиля пользователя"""
        start = time.time()
        try:
            response = self.session.get(
                f"{self.base_url}/api/profile",
                headers={'Authorization': f"Bearer {self.auth_token}"},
                timeout=10
            )

            duration = time.time() - start
            return {
                'success': response.status_code == 200,
                'duration': duration
            }
        except Exception as e:
            return {'success': False, 'error': str(e), 'duration': time.time() - start}

    def _create_order(self):
        """Создание заказа"""
        start = time.time()
        try:
            response = self.session.post(
                f"{self.base_url}/api/orders",
                headers={'Authorization': f"Bearer {self.auth_token}"},
                json={
                    "items": [
                        {"product_id": "prod_1", "quantity": 2}
                    ]
                },
                timeout=10
            )

            duration = time.time() - start

            if response.status_code == 201:
                self.order_id = response.json()['order_id']

            return {
                'success': response.status_code == 201,
                'duration': duration
            }
        except Exception as e:
            return {'success': False, 'error': str(e), 'duration': time.time() - start}

    def _process_payment(self):
        """Обработка платежа"""
        start = time.time()
        try:
            response = self.session.post(
                f"{self.base_url}/api/payments",
                headers={'Authorization': f"Bearer {self.auth_token}"},
                json={
                    "order_id": self.order_id,
                    "payment_method": "card_test",
                    "amount": 99.99
                },
                timeout=10
            )

            duration = time.time() - start
            return {
                'success': response.status_code == 200,
                'duration': duration
            }
        except Exception as e:
            return {'success': False, 'error': str(e), 'duration': time.time() - start}

    def _success(self):
        total_time = sum(d for _, d in self.metrics)

        return {
            'result': 'success',
            'total_duration': total_time,
            'steps': self.metrics,
            'timestamp': time.time()
        }

    def _fail(self, step, error):
        total_time = sum(d for _, d in self.metrics)

        return {
            'result': 'failure',
            'failed_step': step,
            'error': error,
            'completed_steps': self.metrics,
            'timestamp': time.time()
        }

# Использование
checker = MultiStepAPICheck('https://api.example.com')
result = checker.check_user_journey()
print(json.dumps(result, indent=2))
```

---

## Места запуска probes

### Из разных географических локаций

```python
import requests

LOCATIONS = {
    'us-east': 'https://probes.us-east.provider.com/check',
    'us-west': 'https://probes.us-west.provider.com/check',
    'eu-west': 'https://probes.eu-west.provider.com/check',
    'asia-pacific': 'https://probes.apac.provider.com/check',
}

def check_from_all_locations(url):
    """Проверка с разных локаций"""
    results = {}

    for location, probe_url in LOCATIONS.items():
        try:
            response = requests.post(probe_url, json={
                'target_url': url,
                'type': 'http'
            }, timeout=10)

            results[location] = response.json()
        except Exception as e:
            results[location] = {'error': str(e)}

    return results
```

---

## Частота проверок

```yaml
# Проверки каждые 5 минут (baseline)
basic_monitoring:
  http_health:
    interval: 5m
    locations: [us-east, eu-west]

  ssl_certificates:
    interval: 1h
    alert_before_days: 30

  dns_records:
    interval: 30m

# Проверки каждую минуту (critical services)
critical_monitoring:
  payment_gateway:
    interval: 1m
    locations: [us-east, us-central, us-west, eu-west, apac]

  auth_service:
    interval: 1m
    locations: [multi-region]

# Проверки раз в час (non-critical)
best_effort_monitoring:
  analytics_api:
    interval: 1h
    locations: [us-east]
```

---

## Инструменты

### Купленные решения

**Datadog Synthetic Monitoring:**
```yaml
# datadog-synthetics.yml
tests:
  - name: "API Health Check"
    type: api
    request:
      method: GET
      url: https://api.example.com/health
    assertions:
      - operator: isIn
        target: "200"
        type: statusCode
      - operator: lessThan
        target: 1000
        type: responseTime
    locations:
      - aws:us-east-1
      - aws:eu-west-1
    options:
      min_failure_duration: 300
      min_location_failed: 2
      tick_every: 300  # every 5 minutes
```

**Pingdom:**
```xml
<!-- pingdom-check.xml -->
<check>
    <name>Website Availability</name>
    <host>example.com</host>
    <type>http</type>
    <resolution>5</resolution>
    <paused>false</paused>
    <contactids>
        <contact>12345</contact>
    </contactids>
</check>
```

**New Relic Synthetics:**
```javascript
// scripted_browser.js
var assert = require('assert');

$browser.get('https://example.com').then(function() {
    var expected = "Welcome to Example";
    return $browser.findElement($driver.By.tagName('h1')).getText();
}).then(function(text) {
    assert.equal(text, expected, 'Page title does not match');
}).then(function() {
    return $browser.findElement($driver.By.id('login-button')).click();
}).then(function() {
    return $browser.waitForAndFindElement($driver.By.id('dashboard'), 10000);
});
```

### Open Source

**Blackbox Exporter (Prometheus):**
```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: [200, 201]
      method: GET
      fail_if_not_ssl: true
      fail_if_ssl_not_valid: true

  tcp_connect:
    prober: tcp
    timeout: 5s
    tcp:
      query_response:
        - expect: "^SSH-2.0-"
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://api.example.com
        - https://www.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter:9115
      - source_labels: [__param_target]
        target_label: instance
```

**Selenium Grid:**
```yaml
# docker-compose.yml
version: '3'
services:
  selenium_hub:
    image: selenium/hub
    ports:
      - "4444:4444"

  chrome_node:
    image: selenium/node-chrome
    depends_on:
      - selenium_hub
    environment:
      - HUB_HOST=selenium_hub
      - HUB_PORT=4444
```

---

## Метрики synthetic мониторинга

```python
SYNTHETIC_METRICS = {
    # Успешность
    'synthetic_check_success': 'Должная доля успешных проверок',
    'synthetic_check_duration': 'Время выполнения проверки',
    'synthetic_check_failures_total': 'Количество неудач',

    # Сетевые метрики
    'synthetic_dns_lookup_time': 'Время DNS разрешения',
    'synthetic_tcp_connect_time': 'Время TCP подключения',
    'synthetic_tls_handshake_time': 'Время TLS handshake',
    'synthetic_time_to_first_byte': 'Время до первого байта',
    'synthetic_content_download_time': 'Время загрузки контента',

    # HTTP метрики
    'synthetic_http_status_code': 'HTTP status code',
    'synthetic_http_content_length': 'Размер контента',
    'synthetic_http_response_time': 'Время ответа',

    # SSL метрики
    'synthetic_ssl_days_until_expiry': 'Дней до истечения SSL',
    'synthetic_ssl_cert_valid': 'Сертификат валиден',

    # Местоположение
    'synthetic_check_location': 'Локация проверки',
    'synthetic_check_by_location': 'Проверка по локации',
}
```

---

## Алертинг

```yaml
# Алерты в Prometheus
alerting:
  - alert: SyntheticCheckFailureRate
    expr: |
      rate(synthetic_check_failures_total[5m])
      /
      rate(synthetic_check_total[5m]) > 0.05
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Synthetic check failure rate high"
      description: "{{ $value | humanizePercentage }} of checks are failing"

  - alert: SSLCertificateExpiring
    expr: synthetic_ssl_days_until_expiry < 30
    for: 1h
    labels:
      severity: warning
    annotations:
      summary: "SSL certificate expiring soon"
      description: "Certificate expires in {{ $value }} days"

  - alert: SyntheticCheckSlow
    expr: synthetic_check_duration > 5
    for: 10m
    labels:
      severity: info
    annotations:
      summary: "Synthetic check is slow"
      description: "Check duration is {{ $value }}s"
```

---

*Synthetic Monitoring - метод мониторинга через эмуляцию пользовательского поведения*
