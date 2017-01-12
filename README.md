# dn-telegraf
Playbooks/Roles used to deploy a Telegraf agent to a node

# Installation
To deploy a Telegraf agent to a node using the `site.yml` playbook in this repository, first clone the contents of this repository to a local directory using a command like the following:
```bash
$ git clone --recursive https://github.com/Datanexus/dn-telegraf
```
That command will pull down the repository and it's submodules (currently the only dependency embedded as a submodule is the dependency on the `https://github.com/Datanexus/common-roles` repository).

# Use
To run the included playbook, change directories to the `dn-telegraf` subdirectory and run a set of commands that look something like the following (the commands shown here will install and configure an agent from the most recent version of the Telegraf from the main InfluxData website, for example, onto a machine with at the IP address "192.168.34.32"):
```bash
$ export KAFKA_ADDR="192.168.34.8"
$ export INFLUXDB_ADDR="192.168.34.10"
$ export TELEGRAF_ADDR="192.168.34.32"
$ export INPUT_FILTER="cpu:disk:diskio:kernel:mem:processes:swap:system"
$ export OUTPUT_FILTER="kafka:influxdb"
$ echo "[all]\n${TELEGRAF_ADDR}" > hosts
$ ansible-playbook site.yml --inventory-file hosts --extra-vars "kafka_addr=${KAFKA_ADDR} \
    influxdb_addr=${INFLUXDB_ADDR} input_filter=${INPUT_FILTER} kafka_topics=${OUTPUT_FILTER}"
```
Note that all of the variables shown in this example must be defined for the playbook to run successfully.

# Assumptions
It is assumed that this playbook will be run on a recent (systemd-based) version of RHEL or CentOS (RHEL-7.x or CentOS-7.x, for example); no support is provided for other distributions (and the `site.xml` playbook will not run successfully).  The examples shown above also assume that some (shared-key?) mechanism has been used to provide access to the Kafka host from the Ansible host that the ansible-playbook is being run on (if not, then additional arguments might be required to authenticate with that host from the Ansible host that are not shown in the example `ansible-playbook` commands shown above).

# Deployment via vagrant
The included Vagrantfile can be used to deploy kafka to a VM using `Vagrant`.  From the top-level directory of this repostory a command like the following will (by default) deploy kafka to a CentOS 7 virtual machine running under VirtualBox (assuming that both vagrant and VirtualBox are installed locally, of course):
```bash
$ VAGRANT_DEFAULT_PROVIDER=virtualbox vagrant -k="192.168.34.8" -i="192.168.34.10" -a="192.168.34.32" up
```
Note that the `-k` (or the corresponding `--kafka-addr`) flag must be used to pass in the IP address of a host that is running an instance of Kafka and the `-i` (or the corresponding `--influxdb-addr`) flag must be used to pass in the IP address of a host running an instance of InfluxDB.  The last flag shown in that command, the `-a` (or `--addr`) flag, is used to define the IP address of a new VM that will be created (and to which the corresponding Telegraf agent will be deployed).
