# Basic deployment scenarios

There are a four basic deployment scenarios that are supported by this repository:

* in the first scenario (the default scenario if the playbook in this repository is used as is), we'll walk through the deployment of a multi-node NGINX cluster in an AWS environment
* in the second scenario, we'll discuss how the configuration used in the first scenario can be modified to perform the same deployment to an OpenStack environment instead
* in the third scenario, we will walk through the process of deploying NGINX in an AWS or OpenStack environment where the target nodes have already been created and tagged. The command used will essentially be the same as those used in the first two scenarios, but some of the parameters from the [vars/nginx.yml](../vars/nginx.yml) and/or [config.yml](../config.yml) files may be ignored. We will discuss this further when describing this third scenario
* finally, in the fourth scenario, we will describe how a static inventory file can be used to control the playbook run, targeting the nodes specified in the `nginx` host group defined in that static inventory file. While this deployment scenario is supported by the playbook in this repository, it is only supported for non-cloud environments (i.e. scenarios where we are not deploying to an AWS or OpenStack environment).

In the first two scenarios, the virtual machines needed for the multi-node NGINX deployment will be created and the OS deployed to each of those instances will be configured using the input variables and configuration files. This configuration process (referred to as the `preflight` process in the playbook run) involves placing each instance on the proper networks, building a filesystem on the attached data volume (if a data volume was defined), and mounting that data volume on the appropriate mount point. Once that preflight process is complete, the playbook then deploys NGINX to each of the target nodes and configures those nodes as a multi-node NGINX cluster.

The default behavior when deploying to either an AWS or OpenStack environment is to control the deployment using a dynamic inventory that is built from the matching nodes in that target environment (i.e. the nodes that have been tagged with tags that match the tag values input as part of the playbook run).

## Scenario #1: node creation and deployment in an AWS environment
In this scenario, we show how a NGINX cluster can be built in an AWS environment where a matching set of nodes does not yet exist. In this scenario, the [provision-nginx.yml](../provision-zookeper.yml) playbook will actually perform several tasks:

* **create the target nodes** and **build the NGINX host group**: since the target nodes do not exist yet, the playbook will use the [aws](../roles/aws) role to create a new set of nodes that will be targeted by the remainder of the playbook run; once these nodes are created, a host group containing the information needed to connect to those nodes is created using the [build-app-host-groups](../roles/build-app-host-groups) role
* **configure those target nodes**: once created, the target nodes must be configured properly based on the input parameters to the playbook run; this can include placing them on the proper subnet (or subnets), creating a data volume if one is defined, adding a filesystem to that data volume, and mounting that data volume in the defined endpoint; all of these tasks are accomplished by calling out to the [preflight](../roles/preflight) role
* **deploying NGINX to the target nodes**: once the target nodes have been created and configured properly, the playbook then deploys NGINX to those target nodes and configures them as a multi-node NGINX cluster

These three sets of tasks are actually handled in a single command, where the tasks outlined above are handled in three separate plays within the [provision-nginx.yml](../provision-zookeper.yml) playbook. The size of the cluster  that will be built is governed by the `node_map` parameter, which has a default value (defined in the [vars/nginx.yml](../vars/nginx.yml) file) that looks like this:

```yaml
node_map:
  - { application: nginx, count: 1 }
```

So, by default, the playbook in this repository will deploy a standalone NGINX instance. Of course there may be multiple instances or clusters in the target environment, so we also need a way to identify which nodes should be targeted by our playbook run. This is accomplished by setting a series of values for the `cloud`, `tenant`, `project`, `dataflow`, `domain`, and `cluster` tags. In the default configuration file used by the [provision-nginx.yml](../provision-zookeper.yml) playbook (the [config.yml](../config.yml) file), these parameters are defined as follows:

```yaml
cloud: aws
tenant: datanexus
project: demo
domain: production
```

the tags shown here **must** be provided with a value for all deployments; the `dataflow` and `cluster` tags (which are not included in the default configuration file) will assume their default values (`none` and `a`, respectively) if a value is not provided during any given playbook run.

In addition to modifying the values for these tags (and providing values for the `dataflow` and/or `cluster` tags), this same configuration file can be used to customize the values for any of the other parameters that are used during the deployment run. In this example, let's assume that we want to customize the configuration in order to:

* set values for the `cluster` and `dataflow` tags used when creating new instances in our AWS environment (to `a` and `pipeline`, respectively)
* customize the `node_map` parameter used during the playbook run to deploy a two node cluster instead of a standalone instance
* provide a virtual IP address that can be bound to the active NGINX instance in the two-node cluster we are deploying; this virtual IP address must be available for use in the target environment
* change the `type` used for the nodes we're deploying in our AWS environment so that `t2.small` nodes are used (the default is to use `t2.micro` nodes)
* define a specific `image` to use when creating new nodes (the `ami-f4533694` image in this example; keep in mind that in an AWS environment, these image IDs are regional, so the value used in one region will not work in another; it should also be noted here that if this value is not specified in an AWS deployment, the playbook will search for an appropriate image to use based on the defined region)
* define non-default sizes for the root and data volumes
* configure the NGINX instances so that they only accept HTTPS requests (any HTTP requests received will be redirected as HTTPS requests)

To make these changes, let's assume that we create a new configuration file, let's call it the `config-aws-custom.yml` file, containing definitions for all of these new parameter values. The resulting file will look something like this:

```yaml
$ cat config-aws-custom.yml
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
cluster: a
dataflow: pipeline
# network configuration used when creating VMs
cidr_block: '10.10.0.0/16'
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# AWS-specific parameters
region: us-west-2
# specify the type and specify an image
type: 't2.micro'
image: ami-f4533694
# Custom NGINX configuration parameters (sets up server to handle
# all requests as HTTPS requests)
nginx_https_only: true
# used for deployments of multi-node NGINX clusters; the IP address here
# must be available for use in the target environment
nginx_virtual_ip: 10.10.2.254
$
```

Notice that we have redefined the values from the default configuration file in this custom configuration file. This is necessary because this file will be used **instead of** the default configuration file.

Now that we've created a custom configuration file for our deployment, we can use that file to create a new (two-node) NGINX cluster in the defined AWS target environment by running a command that looks something like this:

```bash
$ AWS_PROFILE=datanexus_west ansible-playbook -e "{ \
      config_file: 'config-aws-custom.yml' \
    }" provision-nginx.yml
```

This process of modifying the default behavior of the playbook run is discussed further (and in more detail) in [this document](Customizing-the-Deployment.md).

It should also be noted that if there are already a set of tagged nodes running in the target environment that match the set of tags passed into the playbook run, then the configuration parameters associated with creating nodes in the defined cloud (the `node_map`, `type`, `image`, `cidr_block`, `internal_subnet`, `external_subnet`, `root_volume`, and `data_volume`) will be silently ignored and the matching nodes that were found (based on the tags that were passed in) will be targeted for provisioning as a new NGINX cluster rather than creating a new set of nodes. As such, care should be taken when defining the set of tags that are passed into the playbook run (the `tenant`, `project`, `dataflow`, `domain`, and `cluster` tags) to ensure that the combination of tags used provide a unique identifier for the NGINX cluster that is being created in the targeted AWS environment.

As an aside, it should be noted here that the [provision-nginx.yml](../provision-nginx.yml) playbook includes a [shebang](https://en.wikipedia.org/wiki/Shebang_(Unix)) line at the beginning of the playbook file. As such, the playbook can be executed directly (as if it were a shell script instead of a YAML file) rather than using the file as the final argument in an `ansible-playbook` command. This means that the same command that was shown above could be run like this:

```bash
$  AWS_PROFILE=datanexus_west ./provision-nginx.yml -e "{ \
      config_file: 'config-aws-custom.yml' \
    }"
```

This form is available as a replacement for any of the `ansible-playbook` commands that we show in any of our application playbooks. In most of the examples shown in this documentation, the second form will be used rather than the first; which form you choose to is really just a matter of personal preference (since both accomplish the same thing).

## Scenario #2: node creation and deployment in an OpenStack environment
This scenario is basically the same as the previous scenario; the only difference is in the set of parameters that must be defined for this playbook run to succeed. In addition to the parameters that were shown in the custom configuration file we used in our AWS deployment example (above), values must be provided for a few additional parameters that are only needed when creating and configuring the VMs for a new NGINX cluster in an OpenStack environment. Here's an example of what the resulting (OpenStack) custom configuration file might look like:

```yaml
$ cat config-osp-custom.yml
---
# cloud type and VM tags
cloud: osp
tenant: datanexus
project: demo
domain: production
dataflow: pipeline
cluster: a
# define a custom node_map value, increasing the node
# count to two nodes (from the default value of one)
node_map:
  - { application: nginx, count: 2 }
# configuration (internal) and accessible (external) networks
internal_uuid: 849196dc-12e9-4d50-ac52-2e753d03d13f
internal_subnet: 10.10.0.0/24
external_uuid: 92321cab-1a9c-49d6-99a1-62eedbbc2171
external_subnet: 10.10.3.0/24
# the name of the float_pool that the floating IPs assigned to
# the external subnet should be pulled from
float_pool: external
# a few configuration options related to the OpenStack
# environment we are targeting
region: region1
zone: az1
# username used to access the system via SSH
user: cloud-user
# the node type and image to use for our deployment
type: prod.tiny
image: a36db534-5f9d-4a9f-bdd4-8726cc613b4e
# optional; used to define the size of the root and data
# volumes for each node (in gigabytes)
root_volume: 11
data_volume: 40
# the name of the block_pool that the data volumes should be
# allocated from; required if the data_volume parameter is
# defined (as is the case in this example)
block_pool: prod
# finally, well configure the NGINX instance to only accept
# HTTPS requests
nginx_https_only: true
# used for deployments of multi-node AWS clusters; the IP address here
# must be available for use in the target environment
nginx_virtual_ip: 10.10.3.254
```

As you can see, this file differs slightly from the configuration file that was shown previously (in the AWS deployment example that was shown above); specifically, in this example configuration file:

* the `cloud` parameter has been set to `osp` instead of `aws`
* the `region` parameter has been set to a value of `region1`; this is an example only, and the value that you should use for this parameter will vary depending on the specific OpenStack environment you are targeting (talk to your OpenStack administrator to obtain the value you should use for this parameter)
* a `zone` parameter has been added and set to `az1`; this is an example only, and the value that you should use for this parameter will vary depending on the specific OpenStack environment you are targeting (talk to your OpenStack administrator to obtain the value that you should use for this parameter)
* the `user` parameter has been set to a value of `cloud-user`; this is an example only, and the value that you should use for this parameter will vary depending on the specific OpenStack environment you are targeting (talk to your OpenStack administrator to obtain the value that you should use for this parameter)
* the `image` parameter has been set to the UUID of the image that should be used for our deployments; this value will need to be changed to reflect your OpenStack deployment since it must match one of on the images available to you in that environment (it is assumed to point to a CentOS7 or RHEL7 image)
* in addition to the `internal_subnet` and `external_subnet` parameters that we defined in the AWS deployment example (above), a corresponding UUID must be specified that identifies the network that each of those subnets are associated with (the `internal_uuid` and `external_uuid` values, respectively); these UUID values were not necessary for AWS deployments but they are needed for an OpenStack deployment
* a `float_pool` value must correspond to the name of the pool that the floating IP addresses assigned to each of the instances we are creating should be drawn from.
* if a `data_volume` value is provided (as is the case in this example), a corresponding `block_pool` parameter must also be defined that specifies the name of the block pool that those data volumes should be allocated from

It should be noted that while the `zone` parameter shown in this OSP deployment example was not specified in our AWS deployment example, the machines that were created in the AWS deployment example were definitely deployed to a specific availability zone within the VPC targeted by our playbook run. That's because the availability zone being targeted was inferred from the subnet information that was passed into the playbook run in that AWS deployment example. In OpenStack, the modules we are using do not allow us to make that inference, so the availability zone we are targeting must be specified for our playbook run to succeed.

Once that custom configuration file has been created, the command to create the requested set of VMs, configure them on the right networks, attach the requested data volumes, and configure them as a multi-node NGINX cluster is actually quite simple (assuming that you have already configured your local shell so that you have command-line access to the targeted OpenStack environment):

```bash
$  ./provision-nginx.yml -e "{ \
      config_file: 'config-osp-custom.yml' \
    }"
```

This should result in the deployment of a two-node NGINX cluster in the targeted OpenStack region and availability zone (again, provided that an existing set of VMs that matched the input `tenant`, `project`, `dataflow`, `domain`, `application`, and `cluster` tags were not found in the targeted OpenStack environment).

## Scenario #3: deployment to an existing set of AWS or OpenStack nodes
In this section we will repeat the multi-node cluster  deployments that were just shown in the previous two scenarios, but with a small difference. In this scenario,  we will assume that a set of VMs already exists in the targeted AWS or OpenStack environment that we would like to use for our NGINX cluster. This makes the configuration file that we will use much simpler, since the hard work of actually building and configuring the VMs (placing them on the right network, attaching the necessary data volume, etc.) has already been done. The playbook run will be reduced to the following two tasks:

* **building the NGINX host group**: this host group will be built dynamically from the inventory available in the targeted AWS or OpenStack environment, based on the tags that are passed into the playbook run
* **deploying NGINX to the target nodes**: once the matching nodes have been identified, the playbook deploys NGINX to those nodes and configures them as a multi-node NGINX cluster

To accomplish this, the we have to:

* tag the existing VMs that we will be using to build our new NGINX cluster with the appropriate `Cloud`, `Tenant`, `Project`, `Dataflow`, `Domain`, `Application`, and `Cluster` tags
* place those VMs on the appropriate subnets (i.e. the `10.10.1.0/24` and `10.10.2.0/24` subnets) in the appropriate VPC (a VPC with a `10.10.0.0/16` CIDR block) in the `us-west-2` EC2 region
* run a playbook command similar to the commands shown in the previous two scenarios (depending on whether the target environment is an AWS or OpenStack environment); this will deploy NGINX to the target nodes and configure them as a multi-node NGINX cluster.

In terms of what the command looks like, lets assume for this example that we've tagged our target nodes in an AWS with the following VM tags:

* **Cloud**: aws
* **Tenant**: datanexus
* **Project**: demo
* **Dataflow**: none
* **Domain**: production
* **Application**: nginx
* **Cluster**: a

The tags shown in this example have been chosen specifically to match the tags in the default configuration file (the [config.yml](../config.yml) file), which looks like this:

```yaml
$ cat config.yml
## (c) 2017 DataNexus Inc.  All Rights Reserved
#
# Defaults used if a configuration file is not passed into the playbook run
# using the `config_file` extra variable; deploys to an AWS environment
---
# cloud type and VM tags
cloud: aws
tenant: datanexus
project: demo
domain: production
# network configuration used when creating VMs
cidr_block: '10.10.0.0/16'
internal_subnet: '10.10.1.0/24'
external_subnet: '10.10.2.0/24'
# username used to access the system via SSH
user: 'centos'
# AWS-specific parameters
region: us-west-2
# node type and image ID to deploy; note that the image ID is optional for AWS
# deployments but mandatory for OSP deployments; if left undefined in an AWS
# deployment then then the playbook will search for an appropriate image to use
type: 't2.micro'
#image: 'ami-f4533694'
# the default data and root volume sizes
root_volume: 11
data_volume: 20
```

and the network configuration and EC2 region mentioned earlier were chosen to match the network configuration and EC2 region that are defined as default values in this file. If you want to override these values in your own playbook run (to target machines on a different set of subnets, in a different VPC, and/or in a different region) then those parameters should be added to a custom configuration file and that configuration file should be used instead of the default configuration file shown above.

Assuming that we have created our VMs in the `us-west-2` EC2 region in the proper VPC, placed them on the proper subnets within that VPC, and tagged them appropriately, then the command needed to deploy NGINX to those nodes would be quite simple:

```bash
$ AWS_PROFILE=datanexus_west provision-nginx.yml
```

This command would read in the tags from the default configuration file (shown above), find the matching VMs in the `us-west-2` EC2 region, and use those machines as the target nodes for the NGINX deployment (deploying NGINX to those nodes and configuring them as a multi-node NGINX cluster).

The playbook command used to deploy NGINX to a set of target nodes in an OpenStack environment would look quite similar:

```bash
$ provision-nginx.yml -e "{ \
        config_file: config-osp.yml \
    }"        
```

where the configuration file we are passing in would look nearly the same as the default configuration file used in the AWS deployment:

```yaml
$ cat config-osp.yml
---
# cloud type and VM tags
cloud: osp
tenant: datanexus
project: demo
domain: production
user: cloud-user
# used for deployments of multi-node AWS clusters; the IP address here
# must be available for use in the target environment
nginx_virtual_ip: 10.10.2.254
# sets the path to the directory containing the private keys needed
# to access the target nodes
private_key_path: '/tmp/keys'
```

As you can see, this file doesn't differ significantly from the default configuration file that was shown previously, other than:

* changing the value of the `cloud` parameter to `osp` from `aws`
* modifying the default value for the `user` parameter from `centos` to `cloud-user`; the value shown here is meant to be an example, and the actual value for this parameter may differ in your specific OpenStack deployment (check with your OpenStack administrator to see what value you should be using)
* setting the `private_key_path` to a directory that contains the keys needed to access instances in your OpenStack environment; by default this parameter is set to the playbook directory, so if you keep the keys needed to SSH into the target instances in your OpenStack environment in the top-level directory of the repository this parameter is not necessary

As was the case with the AWS deployment example shown previously, we are assuming here that a set of nodes already exists in the targeted OpenStack environment and that those nodes have been tagged appropriately. The matching set of nodes will then be used to construct the NGINX host group, and that set of nodes will be targeted by the playbook run.

## Scenario #4: using static inventory file to control the playbook run
If you are using this playbook to deploy a multi-node NGINX cluster to a non-cloud environment (i.e. an environment other than an AWS or OpenStack environment), then a static inventory file must be passed into the playbook to control which nodes will be targeted by the playbook run and to define how Ansible should connect to those nodes. In this example, let's assume that we are targeting three nodes in our playbook run and that those nodes are already up and running with the IP addresses 192.168.34.250, 192.168.34.251, and 192.168.34.252. In that case, the static inventory file that we will be using might look something like this:

```bash
$ cat combined-inventory
# example inventory file for a clustered deployment

192.168.34.250 ansible_ssh_host= 192.168.34.250 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/nginx_cluster_private_key'
192.168.34.251 ansible_ssh_host= 192.168.34.251 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/nginx_cluster_private_key'
192.168.34.252 ansible_ssh_host= 192.168.34.252 ansible_ssh_port=22 ansible_ssh_user='cloud-user' ansible_ssh_private_key_file='keys/nginx_cluster_private_key'

[nginx]
192.168.34.250
192.168.34.251
192.168.34.252

$
```

For those not familiar with the structure of a static inventory file in Ansible, you can quite clearly see that it is an INI-style file that describes all of the parameters that are needed to connect to each of the target nodes, as well as any host groups that are used during the playbook run. In this example, we have three nodes that we are targeting, and for each node we have defined the IP address that Ansible should use when making an SSH connection to that host, the port and username that should be used for that SSH connection, and the private key file that should be used to authenticate that user. In addition, at the bottom of the example file shown above we have defined a `nginx` host group containing all of the machines that make up that host group. This is the set of nodes that will be targeted by our playbook run.

To deploy NGINX to the three nodes in this static inventory file and configure them to work together as an cluster, we could run a command that looks something like this:

```bash
$ ansible-playbook -i combined-inventory -e "{ \
      nginx_virtual_ip: 192.168.34.242,
    }" provision-nginx.yml
```

```bash
$ provision-nginx.yml -i test-cluster-inventory -e "{ \
      config_file: /dev/null,  nginx_virtual_ip: '192.168.34.242'
    }"
```

Alternatively, rather than passing in the value for the `nginx_virtual_ip ` parameter and any other parameters needed during our playbook run as extra variables on the command-line, we can make use of the *configuration file* support that is built into our application deployment playbooks and construct a YAML file that looks something like this containing the configuration parameters that are being used for this deployment:

```yaml
---
nginx_virtual_ip: '192.168.34.242'
```

we could then pass that that *configuration file* into the playbook run as an argument to the `ansible-playbook` command; assuming the YAML file shown above was in the current working directory and was named `test-config.yml`, the resulting command would look something like this:

```bash
$ provision-nginx.yml -i test-cluster-inventory -e "{ \
      config_file: 'test-config.yml' \
    }"
```

Once the playbook run is complete, we can browse to the virtual IP address we mentioned previously (`http://192.168.34.242`), and we should see the default NGINX home page displayed in the browser window.

As you can quite clearly see from the output of this command, above, the `ansible-playbook` command that we ran in this example used a static inventory file to control the playbook run, deploying NGINX to all three of the target nodes defined in that static inventory file and configuring them as a multi-node NGINX cluster.
