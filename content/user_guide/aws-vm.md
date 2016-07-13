---
layout: bt_wiki
title: Starting an Instance in AWS
category: User Guide
draft: false
weight: 400

---

Amazon web service is the leading cloud provider today and is used by most.

This Use case will guide you step-by-step to getting your very own cloudify VM running on AWS.

The following blueprint holds all the vital information to achieve just that (Bear in mind, you'll have to insert some personal detail to access your account)

## Prerequisites

To use AWS you'll need an AWS account and an IAM user with sufficient permissions.
Because this blueprint uses AWS infrastructure, It needs the AWS plugin.

{{% gsNote title="Prerequisites Installation" %}}
Credentials used will be access_key and secret_key
For the blueprint to run on local you'll need to install the aws plugin.

HOW TO Specified below
{{< /gsWarning >}}

This is our `blueprint.yaml` file:

{{< gsHighlight  yaml  >}}
tosca_definitions_version: cloudify_dsl_1_3

description: >
  This blueprint uses the cloudify Ansible (SSH) plugin to install a
  Cloudify intro Tutorial application. It uses AWS as the infrastructre
  and Uses Ansible to update and deploy the application code

imports:
  - http://www.getcloudify.org/spec/cloudify/3.4/types.yaml
  - http://www.getcloudify.org/spec/aws-plugin/1.4.1/plugin.yaml

inputs:

  aws_access_key_id:
    type: string
    default: 'AK...JHGYSRS'

  aws_secret_access_key:
    type: string
    default: 'M78yyg....'

  ec2_region_name:
    type: string
    default: 'eu-west-1'

  my_server_image_id:
    type: string
    default: 'ami-b265c7c1'

  my_server_instance_type:
    type: string
    default: 'm3.medium'

  use_existing_security_group:
    type: boolean
    default: false

  my_security_group_id:
    type: string
    default: ''

  use_existing_keypair:
    type: boolean
    default: false

  keypair_name:
    type: string
    default: my_keypair

  ssh_key_filename:
    type: string
    default: ~/.ssh/my_keypair.pem

  my_server_id:
    type: string
    default: ''

  use_existing_ip:
    type: boolean
    default: false

  my_server_ip:
    type: string
    default: ''

dsl_definitions:
  aws_config: &AWS_CONFIG
    aws_access_key_id: { get_input: aws_access_key_id }
    aws_secret_access_key: { get_input: aws_secret_access_key }
    ec2_region_name: { get_input: ec2_region_name }

node_templates:

  my_server_ip:
    type: cloudify.aws.nodes.ElasticIP
    properties:
      aws_config: *AWS_CONFIG
      use_external_resource: { get_input: use_existing_ip }
      resource_id: { get_input: my_server_ip }

  my_host:
    type: cloudify.aws.nodes.Instance
    properties:
      aws_config: *AWS_CONFIG
      use_external_resource: { get_input: use_existing_server }
      resource_id: { get_input: my_server_id }
      install_agent: false
      image_id: { get_input: my_server_image_id }
      instance_type: { get_input: my_server_instance_type }
    relationships:
      - target: keypair
        type: cloudify.aws.relationships.instance_connected_to_keypair
      - target: my_server_ip
        type: cloudify.aws.relationships.instance_connected_to_elastic_ip
      - target: my_security_group
        type: cloudify.aws.relationships.instance_connected_to_security_group

  keypair:
    type: cloudify.aws.nodes.KeyPair
    properties:
      aws_config: *AWS_CONFIG
      use_external_resource: { get_input: use_existing_keypair }
      resource_id: { get_input: keypair_name }
      private_key_path: { get_input: ssh_key_filename }

  my_security_group:
    type: cloudify.aws.nodes.SecurityGroup
    properties:
      aws_config: *AWS_CONFIG
      use_external_resource: { get_input: use_existing_security_group }
      resource_id: { get_input: my_security_group_id }
      description: Security group for my_server
      rules:
        - ip_protocol: tcp
          from_port: 22
          to_port: 22
          cidr_ip: 0.0.0.0/0

outputs:

  My_server:
    description: My server running on AWS
    value:
      Active_Server_IP: { get_attribute: [ my_server_ip, aws_resource_id ] }
      keypair_path: { get_property: [ keypair, private_key_path ] }
{{< /gsHighlight >}}

### Blueprints spefic break down

The inputs in this blueprint set the identification for your AWS account and the specifics for the instance type and flavor 

* `aws_access_key_id` & `aws_secret_access_key` is creds for the IAM user in your account.
* `my_server_image_id` is the AMI id that will be used when spawning your instance.
* `my_server_instance_type` The Size\ Type of instnce to be used.

There are additional inputs which can be changed, For a fresh clean installation nothing else is needed.

* `use_existing_keypair` Change to `True` to use an active keypair. specify the name in AWS by changing `keypair_name`
The local path to you pem file is set in `ssh_key_filename`
* `use_existing_ip` Change to `True` to Associate an existing EIP. use `my_server_ip` to specify the IP


# Getting everything to work

Now that we have IAM user credentials ready and have chosen the Type of instance, region where it will be hosted and the AMI from which it will be spawned.
We'll need to update it all into the blueprint file.
Place the file in a directory that will be the working directory for this deployment and name it `blueprint.yaml`
Now go through the commands

## Step-by-step commands to run the blueprint

The following commands will make everything come to life

### $ cfy init

This will initiate the cloudify working environment with the folder you're current located at

{{< gsHighlight  markdown  >}}
$ cfy init
...

Initialization completed successfully

...
{{< /gsHighlight >}}

### $ cfy local install-plugins

To run this blueprint in a "Local" mode, you'' need to install the aws-plugin.
This command will download the plugin and will make it available for cloudify

{{< gsHighlight  markdown  >}}
$ cfy local install-plugins -p blueprint.yaml
...

Collecting https://github.com/cloudify-cosmo/cloudify-aws-plugin/archive/1.4.1.zip (from -r /var/folders/p3/xrjr1c953yv5fnk719ndljnr0000gn/T/requirements_tftykz.txt (line 1))
  Downloading https://github.com/cloudify-cosmo/cloudify-aws-plugin/archive/1.4.1.zip (124kB)
    100% |################################| 126kB 31kB/s
.
.
.
Installing collected packages: boto, cloudify-aws-plugin
  Running setup.py install for cloudify-aws-plugin ... done
Successfully installed boto-2.38.0 cloudify-aws-plugin-1.4.1

...
{{< /gsHighlight >}}

### $ cfy local install --task-retries=9

We are now ready to run the install workflow. This will make everything come to life, Once complete you'll have a AWS instance up and running.

{{< gsHighlight  markdown  >}}
$ cfy local install --task-retries=9
...

Initiated blueprint.yaml
If you make changes to the blueprint, run `cfy local init -p blueprint.yaml` again to apply them
2016-07-12 11:13:24 CFY <local> Starting 'install' workflow execution
.
.
.
2016-07-12 11:28:35 LOG <local> [my_host_8a54b->my_server_ip_807e5|establish] INFO: Associated Elastic IP 52.48.123.105 with instance i-d4753e58.
2016-07-12 11:28:35 CFY <local> [my_host_8a54b->my_server_ip_807e5|establish] Task succeeded 'ec2.elasticip.associate'
2016-07-12 11:28:35 CFY <local> 'install' workflow execution succeeded

...
{{< /gsHighlight >}}


### $ cfy local outputs


{{< gsHighlight  markdown  >}}
$ cfy local outputs
...

{
  "My_server": {
    "Active_Server_IP": "52.48.123.105", 
    "keypair_path": "~/.ssh/my_keypair.pem"
  }
}

...
{{< /gsHighlight >}}

### cfy local uninstall


{{< gsHighlight  markdown  >}}
$ cfy local uninstall --task-retries=9
...

2016-07-12 11:37:54 CFY <local> Starting 'uninstall' workflow execution
2016-07-12 11:37:54 CFY <local> [my_host_8a54b] Stopping node
2016-07-12 11:37:54 CFY <local> [my_host_8a54b.stop] Sending task 'ec2.instance.stop'
.
.
.
2016-07-12 11:38:13 LOG <local> [keypair_0a104.delete] INFO: Deleted key pair: my_keypair.
2016-07-12 11:38:13 CFY <local> [keypair_0a104.delete] Task succeeded 'ec2.keypair.delete'
2016-07-12 11:38:14 CFY <local> 'uninstall' workflow execution succeeded

...
{{< /gsHighlight >}}


# What's Next

Initiate a VM in AWS and execute shell commands on it

