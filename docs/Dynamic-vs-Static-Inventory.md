# Dynamic vs. static inventory
The first decision to make is when using the playbook in this repository to deploy Telegraf agents to nodes is whether you would like to manage the inventory for your deployment using the dynamic inventory scripts that are provided in the [common-utils](../common-utils) submodule for AWS and OpenStack environments or whether you are managing your inventory statically. Since the static inventory use case is the simplest, we'll start our discussion there. Then we'll move on to a discussion of how to control the deployment of Telegraf agents in both AWS and OpenStack environments by tagging the VMs in those environments so that the dynamic inventory scripts from the [common-utils](../common-utils) submodule can be used to dynamically construct the lists of nodes that should be targeted by a given `ansible-playbook` run.

## Managing deployments with static inventory files
Let's assume that we are planning to deploy Telegraf agents to three nodes using the playbook in this repository and we are planning on using a [static inventory file](https://docs.ansible.com/ansible/intro_inventory.html) to control that deployment. In this example (and others like it in this document) we'll assume that we're going to construct a single static inventory file (an INI-like formatted file containing the list of hosts that are targeted by any given playbook run), and then pass that file into the `ansible-playbook` command using the `-i, --inventory-file` command-line flag.

In our discussion of the various deployment scenarios supported by this playbook, we show that deployments of Telegraf agents to nodes actually require an associated (assumed to be external) Kafka cluster that those agents will report the logs/metrics that they gather to. For purposes of this discussion, we will focus only on the inventory information associated with the nodes we are deploying our Telegraf agents to, not on the inventory information associated with the Kafka cluster. So, returning to our example, the inventory file associated with our three target nodes might look something like this:

```bash
$ cat combined-inventory
# example combined inventory file for clustered deployment

192.168.34.8 ansible_ssh_host=192.168.34.8 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.9 ansible_ssh_host=192.168.34.9 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.10 ansible_ssh_host=192.168.34.10 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.18 ansible_ssh_host=192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host=192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host=192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

[kafka]
192.168.34.8
192.168.34.9
192.168.34.10

[telegraf]
192.168.34.18
192.168.34.19
192.168.34.20

```

As you can see, our static inventory file consists of a list of the hosts targeted by our deployment followed by two host groups (the `telegraf` and `kafka` host groups). For each host in this file, we provide a list of the parameters that Ansible will need to connect to that host (INI-file style) as a set of `name=value` pairs. In this example, we've defined the following values for each of the entries in our static inventory file:

* **`ansible_ssh_host`**: the hostname/address that Ansible should use to connect to that host; if not specified, the same hostname/address listed at the start of the line for that entry for that host will be used (in this example we are using IP addresses). This parameter can be important when there are multiple network interfaces on each host and only one of them (an admin network, for example) allows for SSH access
* **`ansible_ssh_port`**: the port that Ansible should use when connecting to the host via SSH; if not specified, Ansible will attempt to connect using the default SSH port (port 22)
* **`ansible_ssh_user`**: the username that Ansible should use when connecting to the host via SSH; if not specified, then the value of the global `ansible_user` variable will be used (this variable can be set as an extra variable in the `ansible-playbook` command if the same username is used for all hosts targeted by the playbook run)
* **`ansible_ssh_private_key_file`**: the private key file that should be used when connecting to the host via SSH; if not specified, then the value of the global `ansible_ssh_private_key_file` variable will be used instead (this variable can be set as an extra variable in the `ansible-playbook` command if the same private key is used for all hosts targeted by the playbook run)

With this static inventory file built, it's a relatively simple matter to provision Telegraf agents to those nodes using a single `ansible-playbook` command. Examples of these commands are shown [here](Deployment-Scenarios.md).

## Managing deployments using dynamic inventory scripts
For both AWS and OpenStack environments, the playbook in this repository supports the use of a [dynamic inventory](https://docs.ansible.com/ansible/intro_dynamic_inventory.html) to control deployments to those environments. This is accomplished by making use of the [build-app-host-groups](../common-roles/build-app-host-groups) common role, which builds the host groups that are needed for the playbook run by filtering the hosts in the AWS or OpenStack environment, based on the tags that are assigned them and the tags that were included in the `ansible-playbook` command. This provides an attractive alternative to building static inventory files to control the deployment process in these environments, since the meta-data needed to determine which nodes should be targeted by a given playbook run is readily available in the framework itself.

The process of using the [build-app-host-groups](../common-roles/build-app-host-groups) common role to control a playbook run starts out by tagging the VMs that are going to be the target of a particular deployment with the following tags:

* **Tenant**: the tenant that is using the nodes being targeted for Telegraf deployment. As an example, you might have used a value of `labs` for this tag for nodes that are being used by the Labs department in your organization)
* **Project**: this tag will be given the value of the project that is using the nodes that are being targeted for Telegraf deployment; for example we might have used the value `projectx` for this tag for nodes that are being used for ProjectX
* **Domain**: this tag is used to separate out the various domains that might be defined by the project team to control their deployments. For example, nodes might be separated out based on whether they are being used for `development`, `test`, `preprod`, or `production`. Each of these environments would be tagged appropriately based on their intended use.
* **Application**: this tag is used to identify the application tag of the virtual machines that we are targeting in our playbook run (`zookeeper` for Zookeeper nodes, `solr` for Fusion nodes, etc.); this playbook targets nodes with an `Application` tag of `telegraf` by default, but we highly encourage users to change the tag targeted by a given playbook run to reflect the tags of the target nodes for that deployment (more on this below)

It should be noted here that this playbook assumes that an associated Kafka cluster has already been deployed that the Telegraf agents we are deploying can report the metrics/logs that they will gather to. In the dynamic inventory use case, the nodes that make up this Kafka cluster must be tagged with the same `Tenant`, `Project`, and `Domain` tag values that are used by the nodes we are targeting in a given playbook run. Obviously, these Kafka instances would have been tagged with an `Application` tag of `kafka`, but if the remaining tags do not match then the playbook will not be able to find the associated Kafka cluster and the Telegraf agents we are deploying will fail to start.

Once these tags have been assigned to the VMs that will be targeted by a given Telegraf deployment, it is a relatively simple process to define values for the tags that should be targeted by the `ansible-playbook` command (as extra variables or in a *local variables file*, for example). Specifically, the following variables must be defined when using this dynamic inventory to manage a playbook run:

* **tenant**
* **project**
* **domain**
* **application**

Note that while the `application` value defaults to `telegraf` in this playbook (the default value defined at the top of the [vars/telegraf.yml](../vars/telegraf.yml) file), this value should be redefined as part of any given `ansible-playbook` command (or in a or in a *local variables file*) to reflect the `Application` tag value that was used for the machines being targeted by that playbook run.

With these values assigned as part of the `ansible-playbook` command (or in a or in a *local variables file*), the [build-app-host-groups](../common-roles/build-app-host-groups) role that is used by this playbook can correctly identify the nodes in the AWS or OpenStack environment that are targeted by a given playbook run and then use the resulting (dynamically constructed) lists of Telegraf and Kafka nodes to control the deployment and configuration of Telgraf agents to those target nodes.
