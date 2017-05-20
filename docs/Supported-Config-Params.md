# Supported configuration parameters
The playbook in the [site.yml](../site.yml) file in this repository pulls in a set of default values for many of the configuration parameters that are needed to deploy Telegraf from the [vars/telegraf.yml](../vars/telegraf.yml) file. The parameters defined in that file define a reasonable set of defaults for a fairly generic Telegraf deployment to one or more target nodes, including defaults for the name of the network interface over which the Telegraf agent should communicate with the Kafka cluster and the list of logs/metrics that should be gathered.

In addition to the defaults defined in the [vars/telegraf.yml](../vars/telegraf.yml) file, there are a larger set of parameters that can be used to either control the deployment of Telegraf to the target nodes during an `ansible-playbook` run or to configure those Telegraf agents once the installation is complete. In this section, we summarize these options, breaking them out into:

* parameters used to control the `ansible-playbook` run
* parameters used during the deployment process itself, and
* parameters used to configure the Telegraf agents once those agents have been installed

Each of these sets of parameters are described in their own section, below.

## Parameters used to control the playbook run
The following parameters can be used to control the `ansible-playbook` run itself, defining things like how Ansible should connect to the nodes involved in the playbook run, which nodes should be targeted, which packages must be installed during the deployment process, and where those packages should be obtained from:

* **`ansible_ssh_private_key_file`**: the location of the private key that should be used when connecting to the target nodes via SSH; this parameter is useful when there is one private key that is used to connect to all of the target nodes in a given playbook run
* **`ansible_user`**: the username that should be used when connecting to the target nodes via SSH; is useful if the same username is used when connecting to all of the target nodes in a given playbook run
* **`telegraf_url`**: the URL that a custom version of the Telegraf executable can be downloaded from; this executable will be used to overwrite the executable that is installed from the standard InfluxDB repository during the playbook run and, as such, could be used to replace that version with a newer (or older?) version during the deployment process
* **`local_telegraf_file`**: the local path on the Ansible host to a custom version of the Telegraf executable; this executable will be uploaded to the target nodes and will overwrite the executable that is installed from the standard InfluxDB repository during the playbook run
* **`cloud`**: if the inventory is being managed dynamically, this parameter is used to indicate the type of target cloud for the deployment (either `aws` or `osp`); this controls how the [build-app-host-groups](../common-roles/build-app-host-groups) common role retrieves the list of target nodes for the deployment
* **`private_key_path`**: used to define the directory where the private keys are maintained when the inventory for the playbook run is being managed dynamically; in these cases, the scripts used to retrieve the dynamic inventory information will return the names of the keys that should be used to access each node, and the playbook will search the directory specified by this parameter to find the corresponding key files. If this value is not specified then the current working directory will be searched for those keys by default
* **`proxy_env`**: a hash map that is used to define the proxy settings to use for downloading distribution files and installing packages; supports the `http_proxy`, `no_proxy`, `proxy_username`, and `proxy_password` fields as part of this hash map
* **`reset_proxy_settings`**: used to reset any HTTP/YUM proxy settings that may have been made in a previous playbook run back to the defaults (no proxy); this is useful when a proxy was incorrectly set in a previous playbook run and the user wants to return to a "no-proxy" setup in the current playbook run
* **`yum_repo_url`**: used to set the URL for a local YUM mirror. This parameter is only used for CentOS-based deployments; when deploying Telegraf to RHEL-based nodes this parameter is silently ignored and the RHEL package repositories defined locally on the node will be used for any packages installed during the deployment process

## Parameters used during the deployment process
These parameters are used to control the deployment process itself; currently this list consists of only one parameter (the `telegraf_package_list`), and that parameter is undefined by default.

* **`telegraf_package_list`**: the list of packages that should be installed on the targeted nodes; typically this parameter is left undefined, but it could be used to install additional packages on all targeted nodes during the playbook run

## Parameters used to configure the Telegraf agents
These parameters are used configure the Telegraf agents themselves during a playbook run, defining things like the interfaces that they should be listening on for requests, the metrics/logs that they should be collecting, and the destinations that they should be reporting those metrics/logs to:

* **`data_iface`**: the name of the interface that the Telegraf agents should use when talking with the Kafka cluster. An interface of this name must exist for the playbook to run successfully, and if unspecified a value of `eth0` is assumed
* **`iface_description_array`**: this parameter can be used in place of the `data_iface` parameter described above, and it provides users with the ability to specify a description of that interface rather than identifying it by name (more on this, below)
* **`log_paths`**: a list of the paths searched for logfiles by the agent that collects and reports logs to Kafka
* **`log_patterns`**: a list of the patterns used by the agent that collects and reports logs to Kafka; these patterns are used to convert the log file entries to JSON formatted messages
* **`log_parse_from_beginning`**: a flag indicating whether or not the agent that collects and reports logs to Kafka should parse the log files that it finds from the beginning; by default this value is set to `true`
* **`input-filters`**: a hash of hashes that describes the inputs that should be gathered by each agent depoyed (the `logs` and `metrics` agents); by default the `logs` agent gathers `logparser` data while the `metrics` agent gathers CPU, Disk, DiskIO, Kernel, Memory, Process, Swap, and System data
* **`output-filters`**: a hash of hashes that describes the outputs for each agent deployed (the `logs` and `metrics` agents); by default both report the meta-data they gatehr to the associated Kafka cluster

## Interface names vs. interface descriptions
For some operating systems on some platforms, it can be difficult (if not impossible) to determine the name of the interface that should be passed into the playbook using the `data_iface` parameter that we described, above. In those situations, the playbook in this repository provides an alternative; specifying that interface using the `iface_description_array` parameter instead.

Put quite simply, the `iface_description_array` lets you specify a description for each of the networks that you are interested in, then retrieve the names of those networks on each machine in a variable that can be used elsewhere in the playbook. To accomplish this, the `iface_description_array` is defined as an array of hashes (one per interface), each of which include the following fields:

* **`type`**: the type of description being provided, currently only the `cidr` type is supported
* **`val`**: a value describing the network in question; since only `cidr` descriptions are currently supported, a CIDR value that looks something like `192.168.34.0/24` should be used for this field
* **`as_var`**: the name of the variable that you would like the interface name returned as

With these values in hand, the playbook will search the available networks on each machine and return a list of the interface names for each network that was described in the `iface_description_array` as the value of the fact named in the `as_var` field for that network's entry. For example, given this description:

```json
    iface_description_array: [
        { as_var: 'data_iface', type: 'cidr', val: '192.168.34.0/24' },
    ]
```

the playbook will return the name of the network that matches the CIDR `192.168.34.0/24` as the value of the `data_iface` fact. This fact can then be used later in the playbook to correctly configure the nodes to talk to the associated Kafka cluster.

It should be noted that if you choose to take this approach when constructing your `ansible-playbook` runs, a matching entry in the `iface_description_array` must be specified for the `data_iface` network, otherwise the default value of `eth0` will be used for this fact (and the playbook run may result in Telegraf agents that are at best misconfigured; if the `eth0` network does not exist then the playbook will fail to run altogether).