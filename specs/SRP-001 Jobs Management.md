# SRP-001 Jobs Management

`job` - is an application that is started manually or automatically by timer/event and has a limited execution time.

`job` supports the ability to stop (termination by signal) without corrupting data and resume its work from the point where it started.

`job` terminates with status and only with status `exit 0` in case of successful completion.
`job` terminates with status other than `exit 0` if it stops execution WITHOUT completing the work (due to exception, received signal).
This status serves as a signal for the scheduler to retry the execution.
