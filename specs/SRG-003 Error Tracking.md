# SRG-003 Error Tracking

Applications report exceptions to Sentry.

The data source address for Sentry is specified through the environment variable
`SENTRY_DSN`

When sending information to Sentry, the application specifies its version (AppVersion) and
name (Service).

# Business HTTP service

If the application supports the HTTP protocol for business functions, it allows
testing the sending of test messages to sentry through a special request:

```
curl -XPOST http://service/error?key=KEY
```

As a result of this request, a message is sent to Sentry and a response is received with
status `200`.

Or an error message in text form with status `500`.

Where:

* `/error` - path configurable through the `TEST_ERROR_PATH` environment variable. Default
  value `/error`
* `KEY` - key value configurable through the `TEST_ERROR_KEY` environment variable.
  Default value `SRS-009`.

# Service with HTTP metrics

If the application does not handle business HTTP requests, but serves
prometheus metrics, it must support a POST request to the `/error` path on the
prometheus metrics port.
