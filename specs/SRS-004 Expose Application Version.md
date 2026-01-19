# SRS-004 Expose Application Version

To control the configuration of running applications in a microservice architecture,
it is necessary to be able to obtain information about their version from the application.

The application reports its version in the following ways:

## 1. Response to GET /version request

Applications respond to an HTTP GET request on all HTTP ports (excluding health
port, including metric ports and all functional ports) with the path `/version` and return:

* Its name in the `name` field
* Its version (AppVersion) in the `version` field
* `deployment_info` - information from the `DEPLOYMENT_INFO` environment variable with
  which the application was started.

Example:

```sh
curl http://localhost/version
{"version":"v1.2.3","deployment_info":"image:tag","name":"blockberry"}
```

## 2. Add HTTP headers to every response (including error responses)

* `X-App-Name`
* `X-App-Version`

For example:

```sh
X-App-Name: myapp
X-App-Version: v0.1.0
```
