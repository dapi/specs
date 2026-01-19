# SRS-007 Circuit Breaker

Overload Protection (Circuit Breaker)

The circuit breaker technique must be used to avoid cascading overload of services in the call chain.

On the service provider side, the number of unsuccessful attempts to respond to a request from the consumer is counted. When the number of unsuccessful attempts exceeds a specified threshold value within a specified time interval, the circuit breaker trips and returns an error to the consumer indicating that the service is overloaded, and operations in the service are not called. After some time, a trial deactivation of the circuit breaker is performed and monitoring of attempts to respond to consumer requests continues. When the number of unsuccessful attempts falls below the threshold value, the circuit breaker is deactivated and service operations begin to be called.

Pattern references:
* https://microservices.io/patterns/reliability/circuit-breaker.html
* https://martinfowler.com/bliki/CircuitBreaker.html

The following configurable parameters should be provided:
* sliding window size (number of calls for which the percentage of erroneous calls is calculated)
* trip threshold (percentage ratio of calls resulting in errors to the total number of calls in the sliding window, leading to activation of overload protection)
* time period after which the circuit breaker transitions to the half-open state

Default values should be provided for the parameters.

It is recommended to implement overload protection in product gateways, since they are entry points to a particular product for read operations. For write operations, overload protection can be used after careful analysis of possible side effects (including on the service consumer side).
