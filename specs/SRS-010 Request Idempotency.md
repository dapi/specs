# SRS-010 Request Idempotency

## Definition

Idempotency is a property of an operation where multiple executions of the operation with the same parameters have the same effect as a single execution.

Mandatory for:
* GET, HEAD, OPTIONS, TRACE methods (per HTTP standard)
* PUT, DELETE methods (usually idempotent)
* For POST when implementing special logic

## The problem

```
Client sends request → Network failure → Client doesn't receive response
Client retries request → Server processes SECOND time
→ Double charge of funds / data duplication
```

## Idempotency Key

Solution: client generates a unique key for each operation.

```
POST /payments HTTP/1.1
Idempotency-Key: 7ba7c8d5-9c4c-4c8c-bf9e-5d5d5f5f5f5f

{
  "amount": 1000,
  "currency": "USD",
  "account": "12345"
}
```

### Key requirements

* Unique for each operation
* Minimum length of 8 characters, UUID v4 is recommended
* Stored on the client side for retry capability
* Time-to-live: minimum 24 hours, 7 days is recommended

## Server-side implementation

### 1. Key storage

Use with time-to-live:
* Redis with TTL
* Database with expires_at column

```sql
CREATE TABLE idempotency_keys (
  key VARCHAR(255) PRIMARY KEY,
  response_body JSONB,
  response_status INTEGER,
  created_at TIMESTAMP,
  expires_at TIMESTAMP
);

CREATE INDEX ON idempotency_keys (expires_at);
```

### 2. Request processing

```python
def handle_request(request):
    idempotency_key = request.headers.get('Idempotency-Key')

    if idempotency_key:
        # Check if already processed
        cached = storage.get(idempotency_key)

        if cached:
            # Return saved response
            return cached.response_status, cached.response_body

    # Process request
    response = process_business_logic(request)

    if idempotency_key:
        # Save response
        storage.set(idempotency_key, response, ttl=24h)

    return response
```

### 3. HTTP status codes

* **200 OK** - request processed, response in body
* **201 Created** - resource created
* **409 Conflict** - request is already being processed (concurrent request)
* **400 Bad Request** - invalid idempotency key

## Configuration

Via environment variables:
```
IDEMPOTENCY_ENABLED=true
IDEMPOTENCY_KEY_TTL=86400  # in seconds, 24 hours
IDEMPOTENCY_STORAGE=redis  # or database
IDEMPOTENCY_KEY_MIN_LENGTH=8
```

## Metrics

```
idempotency_keys_stored (gauge)
idempotency_hits (counter)      # repeated requests with the same key
idempotency_misses (counter)    # new requests
idempotency_errors (counter)    # errors when working with keys
```

## Boundaries of application

Idempotency is NOT applied to:
* GET method (idempotent per HTTP specification)
* Requests without body
* Health check requests
* WebSocket connections

## HTTP methods

### GET, HEAD, PUT, DELETE
Usually idempotent per HTTP specification. Idempotency-Key is not required.

### POST
Requires implementation with Idempotency-Key for critical operations:
* Payments
* Order creation
* User registration
* Sending messages

### PATCH
Depends on semantics:
```
PATCH /balance {"op": "add", "amount": 10}  # NOT idempotent
PATCH /profile {"name": "John"}               # Idempotent
```

## Concurrent requests

If two requests with the same idempotency key arrive simultaneously:

Option 1 (pessimistic):
```
Second request receives 409 Conflict
First request continues processing
```

Option 2 (optimistic):
```
Second request waits for first to complete
Returns the same result
```

Option 1 is recommended for implementation simplicity.

## Best practices

✅ **Do**
* Generate key on the client before sending
* Use UUID v4
* Validate key format
* Store responses with limited TTL
* Return the same status code
* Document idempotent endpoints

❌ **Don't**
* Trust key generation to the server
* Store responses forever
* Ignore duplicates
* Apply to non-POST requests without necessity
* Allow too short keys

## Example implementation

```python
import uuid
import redis

class IdempotencyHandler:
    def __init__(self, redis_client):
        self.redis = redis_client

    def generate_key(self):
        return str(uuid.uuid4())

    def process(self, key, handler):
        if not self._is_valid_key(key):
            return 400, {"error": "Invalid idempotency key"}

        # Check cache
        cached = self.redis.get(f"idemp:{key}")
        if cached:
            return 200, json.loads(cached)

        # Set lock during processing
        if not self.redis.set(f"idemp:lock:{key}", "1", nx=True, ex=300):
            return 409, {"error": "Request being processed"}

        try:
            result = handler()
            self.redis.setex(f"idemp:{key}", 86400, json.dumps(result))
            return result
        finally:
            self.redis.delete(f"idemp:lock:{key}")

    def _is_valid_key(self, key):
        return len(key) >= 8 and re.match(r'^[a-zA-Z0-9-]+$', key)
```

## Additional resources

* [Stripe: Idempotent Requests](https://stripe.com/docs/api/idempotent_requests)
* [PayPal: API Idempotency](https://developer.paypal.com/docs/api/reference/api-requests/#http-request-headers)
* [HTTP Specification: Idempotent Methods](https://tools.ietf.org/html/rfc7231#section-4.2.2)
