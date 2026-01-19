# SRG-011 Feature Flags & Toggles

Feature Flags (feature toggles) — a mechanism for controlling application functionality without deploying new code.

## Why Feature Flags?

### Safe Releases
- Gradual feature rollout to a subset of users
- Instant rollback without deployment when issues arise
- Reduced risk when releasing new functionality

### Canary Releases
- Rollout to 1-5% of users to verify stability
- Monitor metrics before full release
- Automatic rollback on SLO degradation

### A/B Testing
- Compare different implementation variants
- Data-driven decision making
- Conversion and UX optimization

### Trunk-Based Development
- Development in main/master branch
- New feature code hidden behind flags
- Reduced merge conflicts

## Types of Feature Flags

### 1. Release Flags
Temporary flags for safe releases.

```yaml
# Configuration example
feature_flags:
  new_checkout_flow:
    type: release
    enabled: false
    rollout_percentage: 0
    created_at: "2024-01-15"
    owner: "team-checkout"
    ticket: "PROJ-1234"
```

**Lifecycle:**
1. Flag creation (disabled)
2. Enable for developers
3. Gradual rollout (1% → 10% → 50% → 100%)
4. Remove flag from code

### 2. Ops Flags
Flags for production behavior management.

```yaml
feature_flags:
  enable_cache_layer:
    type: ops
    enabled: true
    description: "Enables caching at API level"

  maintenance_mode:
    type: ops
    enabled: false
    description: "Maintenance mode"
```

### 3. Experiment Flags
For A/B testing and experiments.

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

### 4. Permission Flags
Feature access control.

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

## Rollout Strategies

### Percentage Rollout

```python
class PercentageRollout:
    def __init__(self, percentage: int):
        self.percentage = percentage

    def is_enabled(self, user_id: str) -> bool:
        # Deterministic hash for consistency
        hash_value = hash(user_id) % 100
        return hash_value < self.percentage
```

**Recommended rollout strategy:**
```
Day 1: 1%   - error checking
Day 2: 5%   - metrics monitoring
Day 3: 25%  - load testing
Day 4: 50%  - half of users
Day 5: 100% - full rollout
```

### User Targeting

```python
class UserTargeting:
    def is_enabled(self, user: User, flag: FeatureFlag) -> bool:
        # Check by ID
        if user.id in flag.allowed_user_ids:
            return True

        # Check by attributes
        if flag.country and user.country in flag.country:
            return True

        # Check by segment
        if flag.segment and user.segment == flag.segment:
            return True

        # Fallback to percentage rollout
        return self.percentage_check(user.id, flag.percentage)
```

### Ring Deployment

```
Ring 0: Developers (internal)       - immediately
Ring 1: Beta testers (early adopters) - after 1 day
Ring 2: 10% of users                  - after 3 days
Ring 3: 50% of users                  - after 5 days
Ring 4: 100% of users                 - after 7 days
```

## Implementation

### Feature Flag Service Interface

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
        """Check if flag is enabled for user."""
        pass

    @abstractmethod
    def get_variant(
        self,
        flag_name: str,
        user_id: str
    ) -> str:
        """Returns variant for A/B test."""
        pass
```

### Code Usage Example

```python
# Good - check at function start
def process_checkout(user: User, cart: Cart):
    if feature_flags.is_enabled("new_checkout_flow", user.id):
        return new_checkout_flow(user, cart)
    return legacy_checkout_flow(user, cart)

# Good - decorator
@feature_flag("async_processing")
def process_order(order: Order):
    # New logic
    pass

# Bad - nested checks
def process_order(order: Order):
    if feature_flags.is_enabled("feature_a"):
        if feature_flags.is_enabled("feature_b"):
            if feature_flags.is_enabled("feature_c"):
                # Hard to maintain
                pass
```

### Default Configuration

```yaml
feature_flags:
  defaults:
    # Flag disabled if service unavailable
    fallback_enabled: false

    # Configuration caching
    cache_ttl_seconds: 60

    # Flag service request timeout
    timeout_ms: 100

    # Decision logging
    log_decisions: true
```

## Tools

### Platform Comparison

| Platform | Type | A/B Tests | SDK | Price |
|----------|------|-----------|-----|-------|
| LaunchDarkly | SaaS | Yes | 25+ languages | $$$ |
| Unleash | Open Source | Yes | 15+ languages | Free/Paid |
| Flagsmith | Open Source | Yes | 10+ languages | Free/Paid |
| Split.io | SaaS | Yes | 10+ languages | $$$ |
| ConfigCat | SaaS | Limited | 10+ languages | $ |

### Self-hosted Solution (Unleash)

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

## Monitoring and Metrics

### Key Metrics

```python
# Prometheus metrics
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

# On each check
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

### 1. Flag Naming

```python
# Good
"enable_new_checkout_flow"
"experiment_button_color_2024q1"
"ops_enable_redis_cache"

# Bad
"flag1"
"test"
"new_feature"
```

### 2. Flag Documentation

```yaml
feature_flags:
  enable_async_notifications:
    description: "Enables async notification sending via queue"
    owner: "team-notifications"
    created_at: "2024-01-15"
    expected_removal: "2024-03-01"
    ticket: "PROJ-5678"
    dependencies:
      - "rabbitmq"
      - "notification-worker"
```

### 3. Flag Hygiene

```
Flag removal rules:
1. Release flags: remove within 2 weeks after 100% rollout
2. Experiment flags: remove after experiment completion
3. Ops flags: document and review every 6 months
4. Permission flags: review when role model changes
```

### 4. Testing

```python
# Test both paths
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

## Anti-patterns

### 1. Flag Spaghetti
```python
# Bad - too many nested flags
if flag_a:
    if flag_b:
        if flag_c:
            # 2^3 = 8 possible paths
```

### 2. Immortal Flags
```python
# Bad - flag exists for years
if feature_flags.is_enabled("new_design"):  # Created in 2019
    # "New" design for 5 years now
```

### 3. Missing Fallback
```python
# Bad - no error handling
result = feature_flag_service.is_enabled("flag")

# Good - with fallback
try:
    result = feature_flag_service.is_enabled("flag")
except FeatureFlagServiceError:
    result = False  # Safe default value
```

## References

- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
- [LaunchDarkly: Feature Flag Best Practices](https://launchdarkly.com/blog/best-practices-for-feature-flags/)
- [Unleash Documentation](https://docs.getunleash.io/)
- [Google: Testing in Production](https://sre.google/sre-book/testing-reliability/)
