# SRS-005 Liveness Probes

The application allows checking its health via HTTP or TCP port.

When performing Liveness/Readiness probes, the application SHOULD NOT open new network connections or make network requests. We probably shouldn't be checking the availability of our dependencies in readiness checks due to the potential for cascading failures.

Startup probe can be smart and complex. Liveness probe should be very simple. Treat Liveness probe as a way to know whether the application needs to be restarted.
Readiness.

The application writes to log when a probe request arrives and the response result, but only at log-level=`debug`

## Types of probes

* Liveness (health) - to know that the container is alive. If this probe fails, the pod will be restarted. Done on path `/live`
* Readiness (ready) - to know when the checked container will be ready to accept network traffic. If this probe fails, traffic will be removed from the pod. Done on path `/health`
* Startup (started) - to determine when the application has started and you can proceed to liveness/readiness checks. Until this probe succeeds, the pod will not proceed through the lifecycle.

## HTTP probes

If the application supports HTTP probes, the port of the responding server is specified via the environment variable `HEALTH_PORT`

* Liveness probe is done on path `/live`. This type of probe is desirable but not mandatory. If the application does not support this endpoint, the scheduler uses `/health`
* Readiness probe is done on path `/health`
* Startup probe is done on path `/startup`. Desirable but not mandatory.

Any response with status 200 is considered successful.

HTTP request must fit within `30ms` and not depend on service load.

## TCP probes

* Port for liveness probe is specified via environment variable `LIVENESS_PORT`
* Port for readiness probe is specified via environment variable `HEALTH_PORT`

Any probe that successfully opened a connection on the required port is considered successful.

## gRPC probes

More details: https://github.com/grpc/grpc/blob/master/doc/health-checking.md

* Port for liveness probe is specified via environment variable `LIVENESS_PORT`
* Port for readiness probe is specified via environment variable `HEALTH_PORT`


# More information

<img src="https://andrewlock.net/content/images/2020/k8s_probes.svg" />

* [SRS-006 Liveness Probes Command](SRS-006%20Liveness%20Probes%20Command.md)

* [What is difference in Liveness Probe, Readiness Probe and Startup Probe in Kubernetes?](https://medium.com/@edu.ukulelekim/what-is-difference-in-liveness-probe-readiness-probe-and-startup-probe-in-kubernetes-e116c4563c13)
* https://faun.pub/the-difference-between-liveness-readiness-and-startup-probes-781bd3141079
* https://andrewlock.net/deploying-asp-net-core-applications-to-kubernetes-part-6-adding-health-checks-with-liveness-readiness-and-startup-probes/

