# Agent Configuration

The playbooks in this repository make use of the `configuration_list` parameter to configure the Telegraf agents that are deployed during a playbook run to collect and report data correctly. This document walks through (in detail) how to construct a `configuration_list` for your playbook run and describes the default `configuration_list` that is defined in the [vars/telegraf.yml](../vars/telegraf.yml) file.

## Entries in the configuration_list

The `configuration_list` is actually a list of dictionary entries, where each dictionary entry in the list contains the information needed to customize the configuration of a given Telegraf agent. The `configuration_list` approach is designed to be easily extensible, but there are some limitations in terms of what has been implemented to date. In the sections that follow we will describe the currently supported parameters, their default values, and any such limitations in some detail. That said, each entry in the `configuration_list` has two parameters that should be defined for that entry, specifically:

* **`type`**: the type of entry that is described by a given entry in the `configuration_list`; a value **must** be defined for this parameter, and there are currently two values that can be specified for the `type` in a given `configuration_list` entry (the `agent` and `measurement_set` types); entries with any other `type` value will be silently ignored during a playbook run
* **`config`**: a dictionary of name/value pairs containing the parameters need to customize the configuration file for the `agent` or `measurement_set` in question; if this parameter is not specified then reasonable defaults will be assumed for the values that could have been set using this dictionary (and the parameters associated with the given `configuration_list` entry will be customized using those default values)

The entries that can be placed into the `config` dictionary for each entry and the basic parameters supported in that `configuration_list` entry will vary somewhat depending on the entry `type`; so with that in mind let's explore the configuration parameters that can be specified for the two entry types that are currently supported in the Telegraf playbooks.

### General agent configuration

In the playbooks in this repository, we assume that there is only one entry in the `configuration_list` with a type of `agent`. The entries of this type can only include the two parameters described above (the `type` parameter, which is set to `agent`, and the `config` dictionary containing the parameters that should be customized for the agent being deployed. In addition, there is currently only one parameter that can be customized using the `agent` configuration list entry:

* **`precision`**: the precision that should be used for the timestamps associated with the measurements collected by the agent. The value associated with this parameter must be a string value consisting of an integer number followed by a suffix that describes the units; examples include strings like `1ns`, `1ms`, and `1s`. If a non-units number is used (`15ns`, for example), then an error may be thrown by the underlying Telegraf agent and behavior of the agent is likely to be different from what you expect, so users should stick with numbers that represent actual units

It should be noted that this `precision` value defaults to `1ns`, so if a value is not specified for this parameter in the `agent` configuration list entry (or if that configuration list entry is left out of the `configuration_list` altogether), the agents that are deployed will be configured to collect data with timestamps reported to the nearest nanosecond.

### Configuring measurement sets

In addition to the `agent` configuration (described above), there **must** be one or more `measurement_set` entries defined in the `configuration_list` value that is passed into the playbook run. As was the case with the `agent` entry, these `measurement_set` entries support both a `type` (which must be set to `measurement_set`) and `config` dictionary, but they also support three additional fields:

* **`name`**: the name of the `measurement_set`; the value of this parameter is assumed to be an alpha-numeric string that is suitable for inclusion in a filename on the target systems and **must** be unique within the `configuration_list` that is passed into the playbook run; if there are two `configuration_list` entries with the same `name`, then the behavior of the deployed agent may be different from what you expect
* **`input_filter`**: a colon (`:`) separated string of the measurements that the agent should gather as part of the `measurement_set` in question; the inputs specified in this parameter must comply with the input plugins that are currently supported by the Telegraf agents that are being deployed (for more information on the input plugins that are currently available in Telegraf, see [this section of the Telegraf README.md](https://github.com/influxdata/telegraf/blob/master/README.md#input-plugins) in the main Telegraf repository at GitHub.com)
* **`output_filter`**: a colon (`:`) separated string of the destinations that the agent should report the data that it gathers to as part of the `measurement_set` in question; the outputs specified in this parameter must comply with the output plugins that are currently supported by the Telegraf agents that are being deployed (for more information on the output plugins that are currently available in Telegraf, see [this section of the Telegraf README.md](https://github.com/influxdata/telegraf/blob/master/README.md#output-plugins) in the main Telegraf repository at GitHub.com)

It should be noted here that while there are a number of output plugins currently available for Telegraf agents, the playbooks in this repository currently only support customization of the configuration of the `kafka` output plugins in the `measurement_set` values that are defined in the `configuration_list` passed into the playbook run. For a `kafka` output plugins, the following paramaeters can be specified in the associated `config` dictionary entry:

* **`topic`**: the name of the topic that the measurements in the named measurement set should be published to by the Telegraf agent; if this value is not specified a default topic value of `telegraf` is used
* **`json_timestamp_units`**: the timestamp units that should be used when reporting JSON output to the Kafka cluster. As is the case with the `precision` value defined for the `agent` entries (see above):
    * the value specified should be an integer followed by a suffix specifying the units
    * if a value is not specified for this parameter, then a default value of `1ns` will be used (so the timestamps in the measurements reported to Kafka will have nanosecond precision), and
    * defining a "non-units" integer (like `15ms`) may result in errors being thrown by (or behavior you don't expect from) the Telegraf agents that are being deployed/configured

Support for customization of the configurations for additional output plugins should be easy to add, but we currently have not done so (since we are mainly interested with pushing the measurements that are gathered by the agents that we are deploying to topics in an associated Kafka cluster in our current deployments).

Similarly, while there are a number of input plugins currently avaialable for use with our Telegraf agents, we only support customization of the `logparser` plugins in the `measurement_set` values that are defined in the `configuration_list` that is passed into the playbook run. For a `logparser` input plugin, the following paramaeters can be specified in the associated `config` dictionary entry:

* **`log_paths`**: a list of the patterns that should be used to find the logfiles that should be parsed and reported back by the agent; for more detail on the format associated with the entries in this array see the [logparser plugin documentation](https://github.com/influxdata/telegraf/blob/master/plugins/inputs/logparser/README.md)
* **`log_parse_from_beginning `**: a flag indicating whether or not the log files found should be parsed from the beginning; can be set to `true` or `false` (and defaults to `false` if not specified)
* **`log_patterns`**: a dictionary containing the patterns that should be used when parsing the log files that are found using the entries in the `log_paths` list; the playbooks in this repository configure the agents that they deploy to use the Grok Parser for the `logparser` input plugins that they customize; so for more information on constructing your own patterns we recommend looking at the [Grok Parser](https://github.com/influxdata/telegraf/blob/master/plugins/inputs/logparser/README.md#grok-parser) section of the `logstash` input plugin's [README.md](https://github.com/influxdata/telegraf/blob/master/plugins/inputs/logparser/README.md) file.

As is the case with the output plugins, support for customization of additional input plugins should be easy to add if such support is needed, but we haven't had need for customization of additional plugins yet in our existing playbook runs.

## An simple example

With this background information in mind, the following code snippet is a fairly complete example of the sort of `configuration_list` you might define to customize the configuration of a set of Telegraf agents such that they report two different measurement sets to two separate topics in an existing Kafka cluster:

```
configuration_list:
  - type: 'agent'
    config:
      precision: 1ns
  - type: 'measurement_set'
    name: 'metrics'
    input_filter: 'cpu:disk:diskio:kernel:mem:processes:swap:system'
    output_filter: 'kafka'
    config:
      kafka:
        topic: metrics
        json_timestamp_units: 1ns
  - type: 'measurement_set'
    name: 'logparser'
    input_filter: 'logparser'
    output_filter: 'kafka'
    config:
      kafka:
        topic: logs
        json_timestamp_units: 1ns
      logparser:
        log_parse_from_beginning: true
        log_paths: ['/var/log/**.log']
        log_patterns:
          CUSTOM_SYSLOG: '%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}'
```

This `configuration_list` defines two measurement sets (the `metrics` and `logparser` measurement sets) and configures the Telegraf agents being deployed to report the measurements made for each of those two measurement sets to separate topics in the associated Kafka cluster (the `metrics` and `logs` topics, respectively).

As you can quite clearly see from this example, the structure of the `config` dictionary defined for each of the defined `measurement_set` entries is that of a dictionary of dictionaries, where each entry contains the configuration parameters for a given plugin (either input or output) by name. So for the `logparser` measurement set there are two entries in the `config` dictionary (one for the `kafka` output plugin and one for the `logparser` input plugin), while for the `metrics` measurement set there is only one entry in the `config` dictionary (for the `kafka` output plugin).

## Summary

Hopefully, the information in this document provides you with enough information to construct your own `configuration_list` entries in order to customize the behavior of the Telegraf agents that you are deploying using the playbooks in this repository. As we mentioned earlier, the framework we are describing here is designed to be flexible and extensible, but still being easy to use. It should be a relatively simple matter to add support for customization of additional input or output plugins over time as the need develops, but to date we have only had the need to customize the two plugins that are mentioned, above. Any additional plugins that are deployed will be setup with whatever default configuration is currently used to configure plugins of tha type in the Telegraf framework.

It should also be noted here that, as is the case with the other parameters passed into our application playbook runs, the `configuration_list` parameter that we are describing here can be left unspecified during a playbook run. In that case, the default value for this parameter that is defined in the [vars/telegraf.yml](../vars/telegraf.yml) file will be used (the default behavior is very similar to the configuration that was described in the `configuration_list` example that was shown, above). If you want to customize the behavior of our Telegraf agents beyond this default configuration, however, you will have to either define a new value for this parameter on the command-line (as an extra variable that is passed into the playbook run, an approach that we do not recommended unless you are really good at writing complex JSON values by hand when constructing your `ansible-playbook` commands), or define this parameter within a local variables file that you then pass into the playbook run (this is our recommended approach if you would like to customize your playbook run and not use the default value defined in the [vars/telegraf.yml](../vars/telegraf.yml) file).