# dn-telegraf
Playbooks/Roles used to deploy a Telegraf agent to a node

# Installation
To deploy a Telegraf agent to a node using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:

```bash
$ git clone --recursive https://github.com/Datanexus/dn-telegraf
```

That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Using this role to deploy telegraf
The `site.yml` file at the top-level of this repository pulls in a set of default values for the parameters that are needed to deploy a Telegraf agent to a node from the `vars/telgraf.yml` file.  The contents of that file currently look like this:

```yaml
# (c) 2016 DataNexus Inc.  All Rights Reserved
#
# Defaults that are necessary for all deployments of
# telegraf; the kafka_addr and influxdb_addr values
# shown here should definitely be overridden elsewhere
---
application: telegraf
kafka_addr: "127.0.0.1"
influxdb_addr: "127.0.0.1"
input_filter: 'cpu:disk:diskio:kernel:mem:processes:swap:system'
output_filter: 'kafka:influxdb'
```

This default configuration defines default values for all of the parameters needed to deploy an instance of the Telegraf agent to a node, including defining reasonable defaults for the input metrics that it should report, the destinations that it should report those metrics to (Kafka and InfluxDB by default), adn the IP addresses of those instances.  To deploy Telegraf to a node with the IP address "192.168.34.32" using the role in this repository (by default the Confluent Kafka distribution will be used), one would simply run a command that looks like this:

```bash
$ export KAFKA_ADDR="192.168.34.8"
$ export INFLUXDB_ADDR="192.168.34.10"
$ export TELEGRAF_ADDR="192.168.34.32"
$ ansible-playbook site.yml -i -e "{host_inventory: ['${KAFKA_ADDR}'], \
    kafka_addr: ${KAFKA_ADDR}, influxdb_addr: ${INFLUXDB_ADDR}}"
```
Note that in this example we are overriding the default values that have been declared in the `vars/telegraf.yml` file for the `kafka_addr` and the `influxdb_addr`.  Unless the Kafka and InfluxDB instances are running locally (on the same node that you are deploying Telegraf to), all of the variables shown in this example must be defined for the playbook to run successfully.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).  The examples shown above also assume that some (shared-key?) mechanism has been used to provide access to the Kafka host from the Ansible host that the ansible-playbook is being run on (if not, then additional arguments might be required to authenticate with that host from the Ansible host that are not shown in the example `ansible-playbook` commands shown above).

# Deployment via vagrant
This repository also includes a Vagrantfile that can be used to deploy telegraf to a VM using `Vagrant`.  From the top-level directory of this repostory a command like the following will (by default) deploy telegraf to a CentOS 7 virtual machine running under VirtualBox (assuming that both vagrant and VirtualBox are installed locally, of course) and will configure that Telegraf agent to report to the named Kafka and InfluxDB instances:

```bash
$ vagrant -k="192.168.34.8" -i="192.168.34.10" -a="192.168.34.32" up
```

Note that the `-k` (or the corresponding `--kafka-addr`) flag must be used to pass in the IP address of a host that is running an instance of Kafka and the `-i` (or the corresponding `--influxdb-addr`) flag must be used to pass in the IP address of a host running an instance of InfluxDB.  The last flag shown in that command, the `-a` (or `--addr`) flag, is used to define the IP address of a new VM that will be created (and to which the corresponding Telegraf agent will be deployed).

## Additional vagrant deployment options
While the `vagrant up` command that is shown above can be used to easily deploy a Telegraf agent to a node, the Vagrantfile included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's `site.yml` file. To create a virtual machine without provisioning it, simply run a command that looks something like this:

```bash
$ vagrant -k="192.168.34.8" -i="192.168.34.10" -a="192.168.34.32" up --no-provision
```

This will create a virtual machine with the appropriate IP address ("192.168.34.32"), but will skip the process of provisioning that VM with an instance of the Confluent Kafka distribution using the playbook in the `site.yml` file.  To provision that machine with a Confluent Kafka instance, you would simply run the following command:

```bash
$ vagrant -k="192.168.34.8" -i="192.168.34.10" -a="192.168.34.32" provision
```

That command will attach to the named instance (the VM at "192.168.34.32") and run the playbook in this repository's `site.yml` file on that node (resulting in the deployment of a Telegraf agent to that node and the configuration of that node to report the default set of metrics to the named Kafka and InfluxDB instances).