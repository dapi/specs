# SRS-009 Blocking Timeouts

When interacting, it is necessary to provide configurable timeouts on both the service provider side and the service consumer side. Default timeout values should be provided.

Using a timeout on the service provider side allows timely notification to the service consumer that request execution is taking too long, and there is no point in waiting for it to complete.

Using a timeout on the service consumer side allows regulating the response waiting time for a request. Fallback logic for getting a response can be used in those cases where the response result loses its relevance due to excessive waiting time.

# Environment variables for timeout management

* `OPEN_TIMEOUT`
* `READ_TIMEOUT`

# Additionally

See also [SRS-013 Retry Pattern](SRS-013%20Retry%20Pattern.md)
