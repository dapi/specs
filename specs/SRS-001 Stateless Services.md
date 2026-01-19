# SRS-001 Stateless Services

## Definition of stateless

Services must be stateless (not store state between requests). This allows:
* Easily scale the service by adding new instances
* Restart containers without data loss
* Distribute load between instances without binding to a specific instance

## What is considered state

State includes:
* User session data
* Incomplete in-memory transactions
* Loaded files waiting for processing
* Cache that cannot be restored
* Temporary data stored on the container disk

## Where to store state

If an application needs state, it should store it in:
* Databases (PostgreSQL, MongoDB, Redis)
* Object storage (S3, MinIO)
* Distributed cache (Redis, Memcached)
* Message brokers (RabbitMQ, Kafka)

## Sessions

For storing user sessions use:
* Redis with TTL
* JWT tokens (stateless sessions)
* Cookies with encrypted data
* DO NOT use in-memory storage on the instance

## Examples of stateful anti-patterns

❌ Incorrect:
```
- Storing user sessions in application memory
- Writing files to container disk
- Using sticky sessions in load balancer
- Storing task queue in memory
```

✅ Correct:
```
- Storing sessions in Redis
- Writing files to S3
- Stateless load balancing
- Using message queues (RabbitMQ, SQS)
```

## Health check for stateless services

During health check, check only the service's ability to process requests:
* CPU/Memory health status
* Database connection presence (optional)
* DO NOT check availability of external dependencies

## Deployment

Stateless services must:
* Start immediately with multiple instances
* Not require special startup order
* Work with any number of replicas
* Scale correctly in both directions

## Additional resources

* [The Twelve-Factor App: Processes](https://12factor.net/processes)
* [Kubernetes Stateless Applications](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)
* [Microservices Pattern: Service Instance per Container](https://microservices.io/patterns/deployment/service-per-container.html)
