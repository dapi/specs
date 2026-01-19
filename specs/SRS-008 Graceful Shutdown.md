# SRS-008 Graceful Shutdown

To guarantee data consistency, resource management and system reliability, a service must be able to shut down gracefully.

1. Stop accepting new requests: the service must stop accepting new incoming traffic, but continue processing current requests.
2. Complete processing of active requests: complete all current tasks, including transactions, database writes, or communication with other services.
3. Notification to other services: write to log about the reasons for termination (SIGINT signal or others).
4. Resource release: properly release resources such as file descriptors, database connections, and network connections.

Learn more:
* https://www.geeksforgeeks.org/graceful-shutdown-in-distributed-systems-and-microservices/
* https://www.linkedin.com/pulse/mastering-graceful-shutdown-distributed-systems-jainal-gosaliya-oliye/
