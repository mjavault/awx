# Integration with Third-Party Log Aggregators

This feature builds in the capability to send detailed logs to several kinds
of 3rd party external log aggregation services. Services connected to this
data feed should be useful in order to gain insights into Tower usage
or technical trends. The data is intended to be
sent in JSON format over a HTTP connection using minimal service-specific
tweaks engineered in a custom handler or via an imported library.

## Loggers

This features introduces several new loggers which are intended to
deliver a large amount of information in a predictable structured format,
following the same structure as one would expect if obtaining the data
from the API. These data loggers are the following.

 - awx.analytics.job_status
     - Summaries of status changes for jobs, project updates, inventory
       updates, and others
 - awx.analytics.job_events
     - Data returned from the Ansible callback module
 - awx.analytics.activity_stream
     - Record of changes to the objects within the Ansible Tower app
 - awx.analytics.system_tracking
     - Data gathered by Ansible scan modules ran by scan job templates

These loggers only use log-level of INFO.

Additionally, the standard Tower logs should be deliverable through this
same mechanism. It should be obvious to the user how to enable to disable
each of these 5 sources of data without manipulating a complex dictionary
in their local settings file, as well as adjust the log-level consumed
from the standard Tower logs.

## Supported Services

Currently committed to support:

 - Splunk
 - Elastic Stack / ELK Stack / Elastic Cloud

Under consideration for testing:

 - Sumo Logic
 - Datadog
 - Loggly
 - Red Hat Common Logging via logstash connector

### Elastic Search Instructions

In the development environment, the server can be started up with the
log aggregation services attached via the Makefile targets. This starts
up the 3 associated services of Logstash, Elastic Search, and Kibana
as their own separate containers individually.

In addition to running these services, it establishes connections to the
tower_tools containers as needed. This is derived from the docker-elk
project. (https://github.com/deviantony/docker-elk)

```bash
# Start a single server with links
make docker-compose-elk
# Start the HA cluster with links
make docker-compose-cluster-elk
```

Kibana is the visualization service, and it can be accessed in a web browser
by going to `{server address}:5601`.

If you were to start from scratch, standing up your own version the elastic
stack, then the only change you should need is to add the following lines
to the logstash `logstash.conf` file.

```
filter {
	json {
		source => "message"
	}
}
```

#### Debugging and Pitfalls

Backward-incompatible changes were introduced with Elastic 5.0.0, and
customers may need different configurations depending on what
versions they are using.

# Log Message Schema

Common schema for all loggers:

| Field | Information |
|-----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| cluster_host_id | (string) unique identifier of the host within the Tower cluster |
| level | (choice of DEBUG, INFO, WARNING, ERROR, etc.) Standard python log level, roughly reflecting the significance of the event All of the data loggers as a part of this feature use INFO level, but the other Tower logs will use different levels as appropriate |
| logger_name | (string) Name of the logger we use in the settings, for example, "awx.analytics.activity_stream" |
| @timestamp | (datetime) Time of log |
| path | (string) File path in code where the log was generated |

## Activity Stream Schema

| Field             | Information                                                                                                             |
|-------------------|-------------------------------------------------------------------------------------------------------------------------|
| (common)          | this uses all the fields common to all loggers listed above                                                             |
| actor             | (string) username of the user who took the action documented in the log |
| changes           | (string) unique identifier of the host within the Tower cluster                                                         |
| operation         | (choice of several options) the basic category of the changed logged in the activity stream, for instance, "associate". |
| object1           | (string) Information about the primary object being operated on, consistent with what we show in the activity stream    |
| object2           | (string) if applicable, the second object involved in the action                                                        |

## Job Event Schema

This logger echoes the data being saved into job events, except when they
would otherwise conflict with expected standard fields from the logger,
in which case the fields are named differently.
Notably, the field `host` on the job_event model is given as `event_host`.
There is also a sub-dictionary field `event_data` within the payload,
which will contain different fields depending on the specifics of the
Ansible event.

This logger also includes the common fields.

## Scan / Fact / System Tracking Data Schema

These contain a detailed dictionary-type field either services,
packages, or files.

| Field        | Information                                                                                                                                                                                                       |
|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| (common)     | this uses all the fields common to all loggers listed above                                                                                                                                                       |
| services     | (dict, optional) For services scans, this field is included and has keys based on the name of the service NOTE: Periods are disallowed by elastic search in names, and are replaced with "_" by our log formatter |
| packages     | (dict, optional) Included for log messages from package scans                                                                                                                                                     |
| files        | (dict, optional) Included for log messages from file scans                                                                                                                                                        |
| host         | (str) name of host scan applies to                                                                                                                                                                                |
| inventory_id | (int) inventory id host is inside of  


## Job Status Changes

This is a intended to be a lower-volume source of information about
changes in job states compared to job events, and also intended to
capture changes to types of unified jobs other than job template based
jobs.

In addition to common fields, these logs include fields present on
the job model.

## Tower Logs

In addition to the common fields, this will contain a `msg` field with
the log message. Errors contain a separate `traceback` field.

# Configuring Inside of Tower

Parameters needed in order to configure the connection to the log
aggregation service will include most of the following for all
supported services:

 - Host
 - Port
 - some kind of token
 - enabling sending logs, and selecting which loggers to send
 - use fully qualified domain name (fqdn) or not
 - flag to use HTTPS or not

Some settings for the log handler will not be exposed to the user via
this mechanism. In particular, threading (enabled), and connection type
(designed for HTTP/HTTPS).

Parameters for the items listed above should be configurable through
the Configure-Tower-in-Tower interface.


# Acceptance Criteria Notes

Connection: Testers need to replicate the documented steps for setting up
and connecting with a destination log aggregation service, if that is
an officially supported service. That will involve 1) configuring the
settings, as documented, 2) taking some action in Tower that causes a log
message from each type of data logger to be sent and 3) verifying that
the content is present in the log aggregation service.

Schema: After the connection steps are completed, a tester will need to create
an index. We need to confirm that no errors are thrown in this process.
It also needs to be confirmed that the schema is consistent with the
documentation. In the case of Splunk, we need basic confirmation that
the data is compatible with the existing app schema.

Tower logs: Formatting of Traceback message is a known issue in several
open-source log handlers, so we should confirm that server errors result
in the log aggregator receiving a well-formatted multi-line string
with the traceback message.

Log messages should be sent outside of the
request-response cycle. For example, loggly examples use
`requests_futures.sessions.FuturesSession`, which does some
threading work to fire the message without interfering with other
operations. A timeout on the part of the log aggregation service should
not cause Tower operations to hang.
