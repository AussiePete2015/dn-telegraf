# Deployment via Vagrant
A [Vagrantfile](../Vagrantfile) is included in this repository that can be used to deploy Telegraf agents locally (to one or more VMs hosted under [VirtualBox](https://www.virtualbox.org/)) using [Vagrant](https://www.vagrantup.com/). From the top-level directory of this repository a command like the following will (by default) create a single CentOS 7 virtual machine running under VirtualBox, then deploy a Telegraf agent to that box and configure it to talk to an associated Kafka instance or cluster (we're showing a three-node Kafka cluster in this example):

```bash
$ vagrant -a="192.168.34.88" \
    -k="192.168.34.8,192.168.34.9,192.168.34.10" \
    -i="./kafka_inventory" up
```

Note that the `-a, --addr-list` flag must be used to pass an IP address (or a comma-separated list of IP addresses) into the [Vagrantfile](../Vagrantfile). In the example shown above, we are performing a single-node deployment of Telegraf, when we are performing a multi-node deployment, then we simply need to expand the list of comma-separated values passe in using the `-a, --addr-list` flag:

```bash
$ vagrant -a="192.168.34.88,192.168.34.89,192.168.34.90" \
    -k="192.168.34.8,192.168.34.9,192.168.34.10" \
    -i="./kafka_inventory" up
```

This command will create three nodes, deploy Telegraf agents to all of those nodes, and configure those Telegraf agents to report the metrics and logs information that they gather back to the associated Kafka cluster (which has been passed in using the `-k, --kafka-list` flag). Note that a third argument (the `-i, --kafka-inventory-file` command-line argument) must also be included in these command. That argument is used to pass in a reference to the location of a static inventory file containing the information needed to connect to the nodes in the Kafka cluster (so that the playbook can gather facts about those nodes and configure the Telegraf agents to talk to them correctly). If either of these two arguments are not provided when provisioning Telegraf agents to a node (or nodes), or if the values passed in by either of them are not valid values, then an error will be thrown by the `vagrant` command.

In terms of how it all works, the [Vagrantfile](../Vagrantfile) is written in such a way that the following sequence of events occurs when the `vagrant ... up` command shown above is run:

1. All of the virtual machines targeted by the playbook run (the addresses in the `-a, --addr-list`) are created
1. After all of the nodes have been created, Telegraf agents are deployed all of those nodes and those agents are configured to report their metrics/logs back to the Kafka cluster
1. The `telegraf` service is started on all of the nodes that were just provisioned

Once the playbook run triggered by the [Vagrantfile](../Vagrantfile) is complete, we can simply SSH into one of the nodes in our Kafka cluster (eg. 192.168.38.9) and use the kafka-console-consumer to see if the metrics/logs from our Telegraf agents are being reported back to our Kafka cluster correctly.

So, to recap, by using a single `vagrant ... up` command we were able to quickly deploy Telegraf agents to multiple nodes and configure those agents to report their metrics/logs back to our Kafka cluster. A similar `vagrant ... up` command could be used to deploy Telegraf agents to any number of nodes, provided that a Kafka cluster has already been provisioned that those agents can report to.

## Separating instance creation from provisioning
While the `vagrant up` commands that are shown above can be used to easily deploy Telegraf agents to any number of nodes, the [Vagrantfile](../Vagrantfile) included in this distribution also supports separating out the creation of the virtual machine from the provisioning of that virtual machine using the Ansible playbook contained in this repository's [site.yml](../site.yml) file.

To create a set of virtual machines that we plan on deploying Telegraf agents to without provisioning those machines with Telegraf agents, simply run a command similar to the following:

```bash
$ vagrant -a="192.168.34.88,192.168.34.89,192.168.34.90" \
    up --no-provision
```

This will create a set of three virtual machines with the appropriate IP addresses ("192.168.34.88", "192.168.34.89", and "192.168.34.90"), but will skip the process of provisioning Telegraf agents to those nodes. Note that when you are creating the virtual machines but skipping the provisioning step it is not necessary to provide the list of Kafka nodes using the `-k, --kafka-list` flag, nor is it necessary to provide an inventory file for those Kafka nodes using the `-i, --kafka-inventory-file` flag.

To provision Telegraf agents to the machines that were created above and configure those machines to report their metrics/logs back to an existing Kafka cluster, we simply need to run a command like the following:

```bash
$ vagrant -s="192.168.34.88,192.168.34.89,192.168.34.90" \
    -k="192.168.34.8,192.168.34.9,192.168.34.10" \
    -i="./kafka_inventory" provision
```

That command will attach to the named instances and run the playbook in this repository's [site.yml](../site.yml) file on those node, resulting in the deployment of Telegraf agents to the nodes that were created in the `vagrant ... up --no-provision` command that was shown, above.

## Additional vagrant deployment options
While the commands shown above will install Telegraf agents to one or more nodes with a reasonable, default configuration from a standard location, there are additional command-line parameters that can be used to control the deployment process triggered by a `vagrant ... up` or `vagrant ... provision` command. Here is a complete list of the command-line flags that can be supported by the [Vagrantfile](../Vagrantfile) in this repository:

* **`-a, --addr-list`**: the address list; this is the list of nodes that will be created and provisioned, either by a single `vagrant ... up` command or by a `vagrant ... up --no-provision` command followed by a `vagrant ... provision` command; this command-line flag **must** be provided for almost every `vagrant` command supported by the [Vagrantfile](../Vagrantfile) in this repository
* **`-k, --kafka-list`**: a comma-separated list of the nodes that make up the associated Kafka cluster; this argument **must** be provided for any `vagrant` commands that involve provisioning of Telegraf agents to nodes
* **`-i, --kafka-inventory-file`**: the path to a static inventory file containing the parameters needed to connect to the nodes that make up the associated Kafka cluster; this argument **must** be provided for any `vagrant` commands that involve provisioning of Telegraf agents to nodes
* **`-l, --local-telegraf-file`**: the local path (on the Ansible host) to a `telegraf` executable that should be used in place of the default `telegraf` executable that is installed from the main InfuxDB repository during the playbook run; useful for situations were a newer (or older?) version of Telegraf is needed for a given deployment
* **`-u, --url`**: the URL that can be used to download a `telegraf` executable that should be used in place of the default `telegraf` executable that is installed from the main InfuxDB repository during the playbook run; useful for situations were a newer (or older?) version of Telegraf is needed for a given deployment
* **`-f, --local-vars-file`**: the *local variables file* that should be used when deploying the cluster. A local variables file can be used to maintain the configuration parameter definitions that should be used for a given Telegraf deployment, and values in this file will override any values that are either embedded in the [vars/telegraf.yml](../vars/telegraf.yml) file as defaults or passed into the `ansible-playbook` command as extra variables

As an example of how these options might be used, the following command will download the a newer version of the telegraf executable from a local web server when provisioning the machines created by the `vagrant ... up --no-provision` command shown above:

```bash
$ vagrant -s="192.168.34.88,192.168.34.99,192.168.34.90" \
    -k="192.168.34.8,192.168.34.9,192.168.34.10" \
    -i='/Users/tjmcs/Vagrant/dn-kafka/.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory' \
    -u=http://192.168.34.254/telegraf/linux_amd64/telegraf
    provision
```

While the list of command-line options defined above may not cover all of the customizations that user might want to make when performing production Telegraf deployments, our hope is that the list shown above is more than sufficient for deploying Telegraf agents using Vagrant for local testing purposes.