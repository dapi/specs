# SRS-020 Authorization Pattern (Контроль доступа)

## Определение

Authorization - это процесс определения, имеет ли аутентифицированный пользователь/сервис право на выполнение конкретного действия или доступ к ресурсу.

## Authentication vs Authorization

```
Authentication (аутентификация) = Кто вы?
  ↓ Проверка подлинности
Token is valid, user is authenticated
  ↓
Authorization (авторизация) = Что вам разрешено делать?
  ↓ Проверка прав
User can read orders, but cannot delete them
```

## Паттерны авторизации

### 1. Role-Based Access Control (RBAC)

Access based on roles assigned to users.

```yaml
# Роли
roles:
  - admin
  - manager
  - user
  - guest

# Разрешения
permissions:
  - orders:create
  - orders:read
  - orders:update
  - orders:delete
  - users:read
  - users:update

# Связи роли-разрешения
role_permissions:
  admin: ["*"]  # Все разрешения
  manager: ["orders:*", "users:read"]
  user: ["orders:create", "orders:read", "users:read"]
  guest: ["orders:read"]
```

Реализация:
```python
def has_permission(user, resource, action):
    # Get user's roles
    roles = get_user_roles(user.id)

    # Check each role
    for role in roles:
        permissions = get_role_permissions(role)

        # Check exact permission
        if f"{resource}:{action}" in permissions:
            return True

        # Check wildcard
        if f"{resource}:*" in permissions:
            return True

        # Admin has all permissions
        if "*" in permissions:
            return True

    return False

# Usage
if not has_permission(current_user, "orders", "delete"):
    raise Forbidden("You don't have permission to delete orders")

delete_order(order_id)
```

Плюсы:
* Простота понимания
* Легко масштабируется
* Широко поддерживается

Минусы:
* Не гибкий (роли статичны)
* Проблема role explosion
* Не учитывает контекст

### 2. Attribute-Based Access Control (ABAC)

Access based on attributes of user, resource, environment.

```python
# User attributes
user = {
    "id": 123,
    "roles": ["user"],
    "department": "sales",
    "location": "US",
    "clearance": "confidential"
}

# Resource attributes
order = {
    "id": 456,
    "owner_id": 123,
    "department": "sales",
    "status": "draft",
    "classification": "internal"
}

# Environment attributes
env = {
    "time": "14:30",
    "day": "monday",
    "location": "office",
    "device": "company_laptop"
}

# Policies
policies = [
    {
        "rule": "owners_can_edit",
        "condition": "user.id == resource.owner_id and resource.status == 'draft'",
        "effect": "allow",
        "actions": ["update", "delete"]
    },
    {
        "rule": "same_department",
        "condition": "user.department == resource.department",
        "effect": "allow",
        "actions": ["read"]
    },
    {
        "rule": "high_clearance_required",
        "condition": "resource.classification == 'confidential' and user.clearance >= 'confidential'",
        "effect": "allow",
        "actions": ["read"]
    }
]

# Evaluate policies
def evaluate_policies(user, resource, action, env):
    for policy in policies:
        if action not in policy["actions"]:
            continue

        # Evaluate condition
        condition = policy["condition"]
        if eval_condition(condition, user, resource, env):
            return policy["effect"] == "allow"

    return False  # Deny by default
```

Плюсы:
* Очень гибкий
* Учитывает контекст
* Fine-grained control

Минусы:
* Сложная реализация
* Медленная оценка политик
* Сложно отлаживать
* Нужен PDP (Policy Decision Point)

### 3. Policy-Based Access Control (PBAC)

Similar to ABAC but policies are managed centrally.

```javascript
// Central policy service
const policies = [
  {
    id: "policy-1",
    description: "Managers can approve orders up to $10k",
    subject: { role: "manager" },
    resource: { type: "order" },
    action: "approve",
    condition: "resource.amount <= 10000",
    effect: "allow"
  },
  {
    id: "policy-2",
    description: "Directors can approve any order",
    subject: { role: "director" },
    resource: { type: "order" },
    action: "approve",
    effect: "allow"
  }
];

// Check access via Policy Decision Point
async function checkAccess(user, resource, action) {
  const request = {
    subject: user.attributes,
    resource: resource.attributes,
    action: action
  };

  const response = await fetch('https://pdp.example.com/v1/authorize', {
    method: 'POST',
    body: JSON.stringify(request),
    headers: { 'Content-Type': 'application/json' }
  });

  return response.json().allowed;
}
```

### 4. Access Control Lists (ACL)

Simple list of users/groups and their permissions.

```
Order #456 ACL:
- user:123 (owner) - read, write, delete
- user:789 (manager) - read, write
- group:sales - read
- user:999 (accounting) - read
```

Реализация:
```python
def check_acl(user, resource, action):
    # Get ACL for resource
    acl = get_acl(resource.id)

    # Check if user has permission
    for entry in acl:
        if entry.principal == user.id or entry.principal in user.groups:
            if action in entry.permissions:
                return True

    return False
```

Плюсы:
* Простая реализация
* Понятная модель
* Подходит для файлов, документов

Минусы:
* Не масштабируется (N × M матрица)
* Сложно администрировать
* Нет наследования

## Реализация в приложении

### Middleware pattern

```python
from functools import wraps

def require_permission(resource, action):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            user = get_current_user()

            if not check_permission(user, resource, action):
                raise Forbidden(
                    f"User {user.id} lacks {action} permission on {resource}"
                )

            return f(*args, **kwargs)
        return decorated_function
    return decorator

# Usage
@app.route("/orders/<order_id>", methods=["DELETE"])
@require_permission(resource="orders", action="delete")
def delete_order(order_id):
    # User already authorized
    return Order.delete(order_id)
```

### Decorator pattern (Java)

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RequirePermission {
    String resource();
    String action();
}

@Aspect
@Component
public class AuthorizationAspect {

    @Around("@annotation(requirePermission)")
    public Object checkPermission(ProceedingJoinPoint pjp,
                                  RequirePermission requirePermission) throws Throwable {

        User user = getCurrentUser();
        String resource = requirePermission.resource();
        String action = requirePermission.action();

        if (!permissionService.hasPermission(user, resource, action)) {
            throw new ForbiddenException(
                String.format("User %s lacks %s permission on %s",
                    user.getId(), action, resource)
            );
        }

        return pjp.proceed();
    }
}

// Usage
@RestController
public class OrderController {

    @DeleteMapping("/orders/{id}")
    @RequirePermission(resource = "orders", action = "delete")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

## Роли и разрешения

### Standard roles

```yaml
# admin
permissions:
  - "*"  # All permissions

# manager
permissions:
  - "orders:*"
  - "users:read"
  - "reports:*"

# user
permissions:
  - "orders:create"
  - "orders:read"
  - "orders:update"
  - "profile:read"
  - "profile:update"

# guest
permissions:
  - "orders:read"
  - "public:*"
```

### Fine-grained permissions

```
Resource: order
Actions:
  - create
  - read
  - read:own        # See only own orders
  - read:department # See orders from department
  - update
  - update:own      # Edit only own orders
  - update:status   # Change status (for processors)
  - delete
  - delete:own
  - approve         # Approve orders
  - cancel
```

## Service-to-service authorization

```python
# Service A calls Service B

# Service A
async def call_service_b():
    token = create_service_token(
        issuer="service-a",
        permissions=["users:read", "orders:read"]
    )

    response = await requests.get(
        "http://service-b/api/users",
        headers={"Authorization": f"Bearer {token}"}
    )

# Service B
@app.get("/api/users")
@require_service_permission(resource="users", action="read")
def get_users():
    return User.query.all()
```

## Multi-tenancy

```python
# Organization-based access
user = {
    "id": 123,
    "organization_id": 456,
    "role": "user"
}

def check_tenant_access(user, resource):
    # User can only access resources from their organization
    return resource.organization_id == user.organization_id

def get_orders(user):
    # Automatically filter by tenant
    return Order.query.filter_by(organization_id=user.organization_id).all()
```

## Конфигурация

```
# Authorization
AUTHORIZATION_ENABLED=true
AUTHORIZATION_MODEL=rbac  # rbac, abac, pbac

# JWT
AUTHORIZATION_JWT_CLAIM=permissions  # JWT claim with permissions
AUTHORIZATION_JWT_SCOPE_CLAIM=scope  # OAuth2 scopes

# Cache
AUTHORIZATION_CACHE_ENABLED=true
AUTHORIZATION_CACHE_TTL=300

# Policy refresh
AUTHORIZATION_POLICY_REFRESH_INTERVAL=60

# Debug
AUTHORIZATION_LOG_DECISIONS=true
AUTHORIZATION_LOG_DENIES_ONLY=false
```

## Testing authorization

```python
import pytest

def test_admin_can_delete_order():
    admin = User(roles=["admin"])
    order = Order(id=123, owner_id=999)

    assert check_permission(admin, order, "delete") is True

def test_user_can_delete_own_order():
    user = User(id=123, roles=["user"])
    order = Order(id=456, owner_id=123)

    assert check_permission(user, order, "delete") is True

def test_user_cannot_delete_other_order():
    user = User(id=123, roles=["user"])
    order = Order(id=456, owner_id=999)

    assert check_permission(user, order, "delete") is False

def test_manager_can_approve_within_limit():
    manager = User(roles=["manager"])
    order = Order(amount=5000)

    assert check_permission(manager, order, "approve") is True

def test_manager_cannot_approve_over_limit():
    manager = User(roles=["manager"])
    order = Order(amount=15000)

    assert check_permission(manager, order, "approve") is False
```

## Best practices

✅ **Делать**
* Deny by default (whitelist over blacklist)
* Use least privilege principle
* Test authorization logic
* Log access decisions (especially denies)
* Cache permission checks
* Centralize authorization logic
* Use standard protocols (OAuth2, OIDC)
* Regular permission audits
* Document roles and permissions
* Separate authentication from authorization

❌ **Не делать**
* Hardcode permissions
* Mix authz with business logic
* Skip authorization on internal calls
* Use only client-side authorization
* Ignore authorization errors
* Use broad permissions
* Forget about service-to-service authz
* Skip authorization on "trusted" services

## Дополнительные ресурсы

* [OAuth 2.0 Authorization](https://oauth.net/2/)
* [OpenID Connect](https://openid.net/connect/)
* [RBAC vs ABAC](https://www.okta.com/blog/2020/09/rbac-vs-abac/)
* [Open Policy Agent](https://www.openpolicyagent.org/)
* [Cerbos](https://cerbos.dev/)
* [Oso](https://www.osohq.com/)
