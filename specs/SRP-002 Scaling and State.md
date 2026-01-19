# SRP-002 Scaling and State

Applications can operate in one of two modes, which must be explicitly indicated when putting the service into production:

## Replicable

Service does not store state and can run in multiple instances (allows horizontal scaling and provides fault tolerance)

## Non-replicable

Service CANNOT work in multiple instances mode, therefore when accidentally starting an additional instance you MUST ensure that this does NOT lead to data loss or corruption.

IT IS DESIRABLE that such a service, when starting an additional copy, would detect this on its own (for example, through a semaphore/lock mechanism), report this in the log and terminate its execution with a status other than `0`.
