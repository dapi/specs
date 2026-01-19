# SRS-013 Retry Pattern (Request retry)

Retry technique should be used on the consumer side in case of short-term unavailability of the provider. To use this technique, you must provide a configurable number of attempts and a configurable constant interval between attempts on the consumer side. Using increasing intervals between attempts with a small number of attempts is not recommended. Default parameter values should be provided.

## Implementation of retry for HTTP clients: Retry should be performed under certain conditions:
* Invalid server response code
* Request not formed (internal server error or TCP connection)
* TCP connection errors

Retry should NOT be performed when it is obvious that the request will not be processed, for example, the provider signals that it is overloaded.

<img width="926" alt="Screenshot 2024-12-11 at 16 49 20" src="https://github.com/user-attachments/assets/9b48733a-6bc3-47c6-b16d-c3248485ce83" />

It is recommended to use retry functionality for data read operations (typically GET requests). For data modification operations, retry functionality is recommended only if the service provider guarantees operation idempotency.

## Additionally

Number of retries and intervals should be managed through environment variables.

See also [SRS-009 Blocking Timeouts](SRS-009%20Blocking%20Timeouts.md)
