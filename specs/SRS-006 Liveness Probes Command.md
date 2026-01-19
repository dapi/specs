# SRS-006 Liveness Probes Command

## Motivation

Some workers written in golang are so busy that they don't have time to respond to HTTP/TCP liveness probes.

Solution - use [probes through commands](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command)

This solution is only suitable for workers/jobs and is not used for webservers.

## Solution

A command (script, program, etc.) `/is_live` appears in the application image that returns status `0` if the application is alive, and status `1` if the application is dead.

## Implementation from the application side

### Option 1

1. The application regularly (frequency is determined by the developer and noted in the operation documentation) creates a file, and if the file exists, it updates its modification time.
2. A command (program/script) `/is_live` is added to the image that checks for the presence of the file, deletes it if it exists and returns exit status `0`, if the file does not exist, it returns exit status `1`

### Option 2

1. The application regularly (frequency is determined by the developer and noted in the operation documentation) creates a file, and if the file exists, it updates its modification time.
2. A command (program/script) `/is_live` is added to the image that checks for the presence of the file and returns exit status `0` if the file modification time is not older than ${LIVENESS_TIMEOUT} (default 30) seconds, otherwise it returns exit status `1`
