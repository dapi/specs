# SRG-002 Logging

This document describes the principles of application logging.

1. Applications write their logs to `STDOUT`.
2. The application writes a startup message with its name and version as the first
   line to the main function *BEFORE all initializations, memory allocations and establishing
   connections*. For example: `Starting Blockberry v1.2.3`.
3. The application writes to the log when it is initialized and ready to accept
   requests/perform work: `Initialization completed. Ready to work.`
4. Messages are written to the log in a SINGLE line (avoid multi-line messages).
5. The length of a record should not exceed 10Kb.
6. Add `correlation_id` to messages
7. Split messages larger than 10Kb into parts and search by `correlation_id`
8. Limit log fields to only the most necessary
9. Avoid multi-line messages
10. Do not write passwords and other sensitive information in messages
11. Categorize messages by level (INFO, ERROR, DEBUG, etc.)
12. The application writes to the log about its termination and reason [see SRS-008 Graceful Shutdown](SRS-008%20Graceful%20Shutdown.md)

# Log record formats

Timestamp is written in ISO8601 format with timezone indication.

## Text format

Text string in UTF-8 encoding. Format:

```
 <DATE> <LOG_SEVERITY_SYMBOL> <MESSAGE>
```

Example:

```
2024-12-12T19:31:35+00:00 INFO Starting Application v1.2.3
```

## JSON

Required fields:

* `timestamp`
* `message` - text string with message
* `level` - record level

Any other fields are also allowed.

Example:

```
{"timestamp":"2024-12-12T19:31:35+00:00","message":"Starting Blockberry v1.2.3","level": "INFO", "extra":"Other info"}
```
## Date/Timestamp

Valid date/time values: https://docs.datadoghq.com/logs/log_configuration/parsing/?tab=matchers#parsing-dates

Recommended format: `yyyy-MM-dd'T'HH:mm:ss.SSSZ` example: `2016-11-29T16:21:36.431+0000`

## Severity/Level

Valid values for level
* https://en.wikipedia.org/wiki/Syslog#Severity_level
* https://docs.datadoghq.com/logs/log_configuration/processors/?tab=ui#log-status-remapper

# Best practices

* [Logging best practices in an app or add-on for Splunk Enterprise](https://dev.splunk.com/enterprise/docs/developapps/addsupport/logging/loggingbestpractices/)
* [Logging examples in an app or add-on for Splunk Enterprise](https://dev.splunk.com/enterprise/docs/developapps/addsupport/logging/logginexamples/)
* [Logging Best Practices: The 13 You Should Know](https://www.scalyr.com/blog/the-10-commandments-of-logging/)
