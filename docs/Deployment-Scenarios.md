# Example deployment scenarios

There are a two basic deployment scenarios that are supported by this playbook. In the first scenario (shown below) we'll walk through the deployment of a pair of Telegraf agents to one or more nodes using a static inventory file. In the second scenario, we will show how that same deployment could be performed using the dynamic inventory scripts for both AWS and OpenStack instead of a static inventory file.

## Scenario #1: Using static inventory to control a Telegraf deployment
In this scenario, let's assuming that we're deploying our Telegraf agents to a set of three nodes and configuring those Telegraf agents to report the metrics that they gather to a Kafka cluster that also consists of three nodes. Furthermore, let's assume that we're going to be using a static inventory file to control our Telegraf deployment. The static inventory file that we will be using for this example looks like this:

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

To deploy Telegraf agents to the three nodes in our static inventory file, we'd run a command that looks something like this:

```bash
$ ansible-playbook -i combined-inventory -e "{ \
      data_iface: eth0, \
      telegraf_url: 'http://192.168.34.254/telegraf/linux_amd64/telegraf' \
      yum_repo_url: 'http://192.168.34.254/centos' \
    }" provision-telegraf.yml
```

Alternatively, rather than passing all of those arguments in on the command-line as extra variables, we can make use of the *local variables file* support that is built into this playbook and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
data_iface: eth0
telegraf_url: 'http://192.168.34.254/telegraf/linux_amd64/telegraf'
yum_repo_url: 'http://192.168.34.254/centos'
```

and then we can pass in the *local variables file* as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-multi-node-deployment-params.yml`, the resulting command would look something like this:

```bash
$ ansible-playbook -i combined-inventory -e "{ \
      local_vars_file: 'test-multi-node-deployment-params.yml' \
    }" provision-telegraf.yml
```

As an aside, it should be noted here that the [provision-telegraf.yml](../provision-telegraf.yml) playbook includes a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the beginning of the playbook file. As such, the playbook can be executed directly as a shell script (rather than using the file as the final input to an `ansible-playbook` command). This means that the command that was shown above could also be run as:

```bash
$ ./provision-telegraf.yml -i test-cluster-inventory -e "{ \
      local_vars_file: 'test-multi-node-deployment-params.yml' \
    }"
```

This form is available as a replacement for any of the `ansible-playbook` commands that we show here; which form you use will likely be a matter of personal preference (since both accomplish the same thing).

Once the playbook run is complete, we can simply SSH into one of the nodes in our Kafka cluster (eg. 192.168.34.9) and use the `kafka-console-consumer` to check and make sure tha the metrics collected by our Telegraf agents are being written to our Kafka cluster by those agents.

## Scenario #2: Using dynamic inventory with Telegraf deployments
In this section we will repeat the multi-node cluster deployment that we just showed in the previous scenario, but we will use the `build-app-host-groups` role that is provided in the [common-roles](../common-roles) submodule to control the deployment of a seto of Telegraf agents to a set of target nodes in an AWS or OpenStack environment rather than relying on a static inventory file.

To accomplish this, the we have to:

* Tag the instances in the AWS or OpenStack environment that we will be targeting for deployment with Telegraf agents with the appropriate `Tenant`, `Project`, `Domain`, and `Application` tags. Note that for all of the nodes we are targeting in our playbook runs, we will use whatever their existing `Application` tag happens to be (`zookeeper` for the `zookeeper` nodes we want to monitor, `solr` for the `solr` nodes we want to monitor, etc.)
* Have already deployed a Kafka cluster into this environment; this cluster should be tagged with the same  `Tenant`, `Project`, `Domain` tags that will be used when targeting instances for deployment with Telegraf agents (above); obviously the  `Application` for the Kafka cluster nodes would be `kafka`
* Once all of the nodes that we are planning to target with our Telegraf deployment are tagged appropriately, we can run `ansible-playbook` command similar to the `ansible-playbook` command shown in the previous scenario; this will deploy the Telegraf agents to those nodes and configure the Telegraf agents to report the metrics and logs that they gather to the associated Kafka cluster.

In terms of what the commands look like, lets assume for this example that the our target nodes are tagged with the following VM tags:

* **Tenant**: labs
* **Project**: projectx
* **Domain**: preprod
* **Application**: zookeeper

The `ansible-playbook` command used to deploy the Telegraf agents to our nodes and configure to report their metrics and logs to our Kafka cluster in an OpenStack environment would then look something like this:

```bash
$ ansible-playbook -e "{ \
        application: zookeeper, cloud: osp, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0 \
    }" provision-telegraf.yml
```

The playbook uses the tags in this playbook run to identify the nodes that make up the associated Kafka cluster (which must be up and running for this playbook command to work) and the target nodes for the playbook run, installs Telegraf agents on the target nodes, and configures those agents to report the metrics and logs that they gather to the associated Kafka cluster. 

In an AWS environment, the command would look something like this:

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
        application: zookeeper, cloud: aws, \
        tenant: labs, project: projectx, domain: preprod, \
        private_key_path: './keys', data_iface: eth0 \
    }" provision-telegraf.yml
```

As you can see, these two commands only differ in terms of the environment variable defined at the beginning of the command-line used to provision to the AWS environment (`AWS_PROFILE=datanexus_west`) and the value defined for the `cloud` variable (`osp` versus `aws`). In both cases the result would be the deployment of a pair of Telegraf agents to the set of with tags that match the input `application`, `tenant`, `project`, and `domain` tags. Those agents would be configured to report their metrics to the matching Kafka cluster. The number of nodes that will have Telegraf agents deployed to them will be determined (completely) by the number of nodes in the OpenStack or AWS environment that have been tagged with a matching set of `application`, `tenant`, `project` and `domain` tags.
