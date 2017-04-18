# Example deployment scenarios

There are a two basic deployment scenarios that are supported by this playbook. In the first scenario (shown below) we'll walk through the deployment of a pair of Telegraf agents to one or more nodes using a static inventory file. In the second scenario, we will show how that same deployment could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file.

## Scenario #1: Using static inventory to control a Telegraf deployment
In this scenario, let's assuming that we're deploying our Telegraf agents to a set of three nodes and configuring those Telegraf agents to report the metrics that they gather to a Kafka cluster that also consists of three nodes. Furthermore, let's assume that we're going to be using a static inventory file to control our Telegraf deployment. The static inventory file that we will be using for this example looks like this:

```bash
$ cat test-telegraf-inventory
# example inventory file for a multi-node deployment

192.168.34.18 ansible_ssh_host=192.168.34.18 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.19 ansible_ssh_host=192.168.34.19 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'
192.168.34.20 ansible_ssh_host=192.168.34.20 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/zk_cluster_private_key'

$
```

To correctly configure our Telegraf agents to talk to the Kafka cluster, the playbook will need to connect to the nodes that make up the associated Kafka cluster and collect information from them, and to do so we'll have to pass in the information that Ansible will need to make those connections to the playbook. We do this by passing in a separate inventory file (the `kafka_inventory_file` for the deployment) that contains the inventory information for the members of the Kafka cluster that our Telegraf agents will be reporting their metrics to. For the purposes of this example, let's assume that our `kafka_inventory_file` looks something like this:

```bash
$ cat kafka-inventory
# example inventory file for a clustered deployment

192.168.34.8 ansible_ssh_host=192.168.34.8 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.9 ansible_ssh_host=192.168.34.9 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'
192.168.34.10 ansible_ssh_host=192.168.34.10 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/kafka_cluster_private_key'

$
```

To deploy Telegraf agents to the three nodes in our static inventory file, we'd run a command that looks something like this:

```bash
$ ansible-playbook -i test-telegraf-inventory -e "{ \
      host_inventory: ['192.168.34.18', '192.168.34.19', '192.168.34.20'], \
      inventory_type: static, data_iface: eth0, kafka_inventory_file: './kafka-inventory' \
    }" site.yml
```

You could also combine the inventory files for the Telegraf and Kafka nodes into one file. However, to do so you need to define which nodes belong in which host group (`telegraf` vs. `kafka`) by adding those host groups to the end of the file:

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

and that combined inventory file could then be passed into the `ansible-playbook` command:

```bash
$ ansible-playbook -i combined-inventory -e "{ \
      host_inventory: ['192.168.34.8', '192.168.34.9', '192.168.34.10'], \
      inventory_type: static, data_iface: eth0, kafka_inventory_file: './combined-inventory', \
      telegraf_url: 'http://192.168.34.254/telegraf/linux_amd64/telegraf' \
      yum_repo_url: 'http://192.168.34.254/centos' \
    }" site.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
inventory_type: static
data_iface: eth0
kafka_inventory_file: './combined-inventory'
telegraf_url: 'http://192.168.34.254/telegraf/linux_amd64/telegraf'
yum_repo_url: 'http://192.168.34.254/centos'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-multi-node-deployment-params.yml`, the resulting command would look something like this:

```bash
$ ansible-playbook -i combined-inventory -e "{ \
      host_inventory: ['192.168.34.18', '192.168.34.19', '192.168.34.20'], \
      local_vars_file: 'test-multi-node-deployment-params.yml' \
    }" site.yml
```

Once the playbook run is complete, we can simply SSH into one of the nodes in our Kafka cluster (eg. 192.168.34.9) and use the `kafka-console-consumer` to check and make sure tha the metrics collected by our Telegraf agents are being written to our Kafka cluster by those agents.

## Scenario #2: Using dynamic inventory with Telegraf deployments
In this section we will repeat the multi-node deployment that we just showed in the previous scenario, but we will use the dynamic inventory scripts provided in the [common-utils](../common-utils) submodule to control the deployment of our Telegraf agents to an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the instances in the AWS or OpenStack environment that we will be targeting for deployment with Telegraf agents with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will use whatever their existing `Application` tag happens to be (`zookeeper` for the `zookeeper` nodes we want to monitor, `solr` for the `solr` nodes we want to monitor, etc.
* Have already deployed a Kafka cluster into this environment; this cluster should be tagged with the same  `Tenant`, `Project`, `Domain` tags that will be used when targeting instances for deployment with Telegraf agents (above); obviously the  `Application` for the Kafka cluster nodes would be `kafka`
* Once all of the nodes that we are planning to target with our Telegraf deployment are tagged appropriately, we can run `ansible-playbook` command similar to the `ansible-playbook` command shown in the previous scenario; this will deploy the Telegraf agents to those nodes and configure the Telegraf agents to report the metrics and logs that they gather to the associated Kafka cluster.

In terms of what the commands look like, lets assume for this example that the our target nodes are tagged with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: zookeeper

The `ansible-playbook` command used to deploy the Telegraf agents to our nodes and configure to report their metrics and logs to our Kafka cluster in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/osp/openstack -e "{ \
        host_inventory: 'meta-Application_zookeeper:&meta-Cloud_osp:&meta-Tenant_labs:&meta-Project_projectx:&meta-Domain_preprod', \
        application: zookeeper, cloud: osp, tenant: labs, project: projectx, domain: preprod, inventory_type: dynamic, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0 \
    }" site.yml
```

The playbook uses the tags in this playbook run to identify the nodes that make up the associated Kafka cluster (which must be up and running for this playbook command to work) and the target nodes for the playbook run, installs Telegraf agents on the target nodes, and configures those agents to report the metrics and logs that they gather to the associated Kafka cluster. 

In an AWS environment, the command would look something like this:

```bash
$ ansible-playbook -i common-utils/inventory/aws/ec2 -e "{ \
        host_inventory: 'tag_Application_zookeeper:&tag_Cloud_aws:&tag_Tenant_labs:&tag_Project_projectx:&tag_Domain_preprod', \
        application: zookeeper, cloud: aws, tenant: labs, project: projectx, domain: preprod, inventory_type: dynamic, \
        ansible_user: cloud-user, private_key_path: './keys', data_iface: eth0 \
    }" site.yml
```

As you can see, this command is basically the same command that was shown for the OpenStack use case; it only differs slightly in terms of the name of the inventory script passed in using the `-i, --inventory-file` command-line argument, the value passed in for `Cloud` tag (and the value for the associated `cloud` variable), and the prefix used when specifying the tags that should be matched in the `host_inventory` value (`tag_` instead of `meta-`). In both cases the result would be a set of Telegraf agents deployed to the appropriately tagged nodes and reporting metrics and logs back to the Kafka cluster. The number of nodes targeted by the playbook run is determined (completely) by the tags that were assigned to the VMs in the cloud environment in question.
