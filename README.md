# Ansible Plays for Lightbend ConductR

These plays and playbooks provision [Lightbend ConductR](https://conductr.lightbend.com) cluster nodes in AWS EC2 using [Ansible](http://www.ansible.com). ConductR is the project name for Service Orchestration in Lightbend Production Suite.

**This version of ConductR Ansible is compatible with ConductR's Master branch, currently 2.0.x.**
For previous versions, use the corresponding branch, i.e. Conductr-Ansible 1.1.x branch for use with ConductR 1.1.x.

ConductR can be deployed to a simple 'flat' topology or to a production style private/public topology. For those just getting started, evaluating or otherwise wanting the easier option should use the flat approach. Only if you want the segmentation of private agents should you use the 'private-agents' version of the playbooks.

Use create-network-ec2.yml to setup a new VPC and create your cluster in the new VPC. This is the flat topology. You only need to provide your access keys and what region to execute in.
The playbook outputs a vars file for use with the build-cluster-ec.yml. The username and password of 'commercial.properties' and the 'keypair' of generated vars file must be set prior to use.

The playbook build-cluster-ec2.yml launches four nodes running both the core and the agent processes as well as a small template instance for imaging. Be certain to review and customize the vars file before building the cluster.

The create-private-agent-network-ec2.yml playbook also configures a VPC for Production Suite. This is the public/agent path. It launches an admin bastion host for then running the build cluster plays from. This is required to manage private agent nodes that will not have a public ip address. The playbook build-private-agent-cluster-ec2.yml launches three core nodes, three private agents, three public agents and one small template node instances across three availability zones. Core, private agent, public agent and template nodes can be of different AMI, instance and volume size.

## Prerequisites

You'll need the following in order to use these playbooks.

* Lightbend credentials. Obtain for free from [lightbend.com](https://www.lightbend.com/product/conductr/developer). Apply your credentials to `conductr/files/commercial.credentials.template` and save as `conductr/files/commercial.credentials`.
* Access Key and Secret for an AWS Account with permissions to admin EC2.
* Ansible installed on a controller host. The faster the controller host's connection to the chosen EC2 region, the faster nodes will launch. Ssh access to a small to medium instance in the same AWS region as your cluster works well. See Ansible Setup below for further details.
* An AWS Key Pair (PEM file) downloaded to the Ansible controller host.
* A copy of the ConductR and ConductR-Agent deb installation package on the Ansible controller host.
* A copy of this GitHub repo on the Ansible controller host.

## Setup

ConductR is **not** provided by this repository. Visit the [Customer Portal](https://together.lightbend.com/) to download or [Lightbend.com](https://www.lightbend.com/products/conductr) to sign up to evaluate Lightbend Production Suite. Developers should use the [developer sandbox](https://www.lightbend.com/product/conductr/developer) to validate bundle packaging and execution.

Obtain your Lightbend credentials from [Lightbend.com](https://www.lightbend.com/product/conductr/developer) and replace 'username' and 'password' in 'conductr/files/commercial.properties'.

Copy the ConductR deb (conductr_x.y.z-systemd_all.deb) *and* the ConductR-Agent deb (conductr-agent_x.y.z-systemd_all.deb) installation package into the `conductr/files` folder in your local copy of this repo. The installation package will be uploaded from this folder by the ConductR play to each of the EC2 instances for installation. The names and versions in the vars file should be updated whenever changing versions.

Log into the AWS Console and select or generate a key pair in the region you intend to use. You'll need both the path to a local copy of the PEM file and the key pair name used in console to record in your vars file. Specify your 'keypair' in the vars file.

## Running the Plays

From an Ansible enabled shell, export your AWS Key and Secret for use by the plays.

```bash
export AWS_ACCESS_KEY_ID='ABC123'
export AWS_SECRET_ACCESS_KEY='abc123'
```

Disable Ansible host key checking so that the play doesn't need you to accept the first connection to the nodes.

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
```

Runing the create network play without any arguments defaults to executing in the EC2 region of us-east-1.

```bash
ansible-playbook create-network-ec2.yml
```

Optionally specify what [EC2 region](http://docs.aws.amazon.com/general/latest/gr/rande.html#ec2_region) you want to execute in as `EC2_REGION` using a -e key value pair. For example to execute in eu-west-1: 

```bash
ansible-playbook create-network-ec2.yml -e "EC2_REGION=eu-west-1"
```

To create a network with private and public agent in differnet subnets run the `create-private-agent-network' playbook. This playbook requires a keypair setting for launching a bastion host

```bash
ansible-playbook create-private-agent-network-ec2.yml  -e "KEYPAIR={{ keyname }}" --private-key /path/to/{{keypair}}
```

The playbook defaults to availability zones `a`, `b`, and `c`. Change the create-network playbook directly to use other or fewer zones.

The create network playbook produces a vars file in the `vars` folder named `{{EC2_REGION}}_vars.yml` where {{EC2_REGION}} is the region used. You **must** add the name of your key pair to `{{EC2_REGION}}_vars.yml` in order to use it with the build cluster script. Change the "Key Pair Name" of `KEYPAIR: "Key Pair Name"` to that of the key pair name, which may be different than the file name and generally does not end in the .pem file extension.

If you want to execute in a region other than us-east-1, you will also need to change the AMI value for `IMAGE` in your vars file to an Ubuntu image in that region. The AMI listed is the Ubuntu 16.04 LTS HVM EBS boot image published by Canonical for us-east-1. Other versions and types of Ubuntu instances are expected to work. The [Ubuntu AMI Locator](http://cloud-images.ubuntu.com/locator/ec2/) can help you find AMI's for alternative regions and instance types.

We pass both our vars file and EC2 PEM key to our playbook as command line arguments. The VARS_FILE template can be the one created from the create script. There is also a `vars.yml` template you can use instead. The private-key value must be the local path and filename of the keypair that has the key pair name `KEYPAIR` specified in the vars file. For example our key pair may be named `ConductR_Key` in AWS and reside locally as `~/secrets/ConductR.pem`. In which case we would set `KEYPAIR` to `ConductR_Key` and pass `~/secrets/ConductR.pem` as our private-key argument.

All the nodes will be assigned a public ip address so you can ssh into nodes using the specified PEM key with the user name from REMOTE_USER in `vars/main.yml`. 

```bash
ansible-playbook build-cluster-ec2.yml -e "VARS_FILE=vars/{{EC2_REGION}}_vars.yml" --private-key /path/to/{{keypair}}
```

or if you are using the private agent network

```bash
ansible-playbook build-private-agent-cluster-ec2.yml -e "VARS_FILE=vars/{{EC2_REGION}}_vars.yml" --private-key /path/to/{{keypair}}
```

## Accessing cluster applications

If all went well you now have a three node ConductR cluster. The nodes are registered with the ELB. In order to access applications from the internet you must add a listener to the ELB and ensure port access. To expose bundle endpoints to the world you must:
* Add a listener to the ELB. The instance port of the listener will be that of the bundle endpoint.
* Grant the ELB-SG inbound access on the instance port in to the Node-SG
* If using ELB ports other than 80 and 443, allow the world, `0.0.0.0/0`, inbound access on the ELB port in to the ELB-SG.

The Visualizer sample application has been setup as an example. Start at least one instance of Visualizer in the cluster. Now you can access the Visualizer sample application using port 80 of the ELB DNS Name in your browser. You may delete or remap the Visualizer ELB listeners and corresponding security group access as desired.

### Running many services on few ports

For the most part, we only expose ports 80 and 443 to the world. We might have an interal ELB mapping to set of services on a different port, but we generally don't want to have many port mappings to manage. Hostnames and paths can be used to segment traffic between applications.

For example to run a staging instance of an application on the same cluster and service endpoint port, use a [Configuration-Bundle](http://conductr.lightbend.com/docs/2.0.x/BundleConfiguration#Configuration-Bundles) with a staging hostname.

 ```
 components = {
   lightbend-www = {
     endpoints = {
       “www” = {
         bind-protocol = “http”
         bind-port = 0
         service-name = “Lightbend_WWW_Staging”
         services = [“http://mdavis.lightbend.com”]
       }
     }
   }
 }
 ```

### Websockets

Applications using websockets require a TCP listener in the ELB instead of a HTTP/HTTPS listener. In this configuration one must [configure proxy protocol support](http://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/enable-proxy-protocol.html) on the ELB in order to retrieve client connection information.

### Enabling SSL

Add a HTTPS listener to the load balancer in order to access the cluster securely. You will need to upload a X.509 certificate when creating an HTTPS listener if you haven't already. 

### Optional Variables

The vars file templates contain variables for controlling optional features and components.

`VOL_TYPE` and `VOL_SIZE` determine the type and size (GB) of the storage volume attached to each instance. Use `gp2` for General Purpose (SSD) volumes, `io1` for Provisioned IOPS (SSD) volumes, and `standard` for Magnetic volumes.

`ENABLE_DEBUG` defaults to "true." When set to "true," `-Dakka.loglevel=debug` is added to ConductR's `conf/application.ini` to enable ConductR debug level logging. Use "false" to disable.

`ENABLE_ROLES` defaults to "true." When set to "true," `-Dconductr.resource-provider.match-offer-roles=on` is added to ConductR's `conf/application.ini` to enable role matching. Use "false" to disable.

`INSTALL_DOCKER` defaults to "true." When set to "true," the Docker apt repository is used to install lxc-docker for ConductR non-root usage. Use "false" to disable.

`CONDUCTR_PRIVATE_ROLES` is a list and defaults to `web` and `elasticsearch` if not specified. To append additional role, e.g. `GPU` specify the following within the vars file:

```
CONDUCTR_PRIVATE_ROLES:
  - web
  - elasticsearch
  - test
```

`CONDUCTR_PUBLIC_ROLES` is a list and defaults to `haproxy` if not specified. To append additional role, e.g. `public` specify the following within the vars file:

```
CONDUCTR_PUBLIC_ROLES:
  - haproxy
  - public
```

## Ansible Setup

These plays are being developed out of the current master branch of Ansible. They may or may not work with older packaged versions.

Setup Ansible from source using git and pip.

```bash
git clone https://github.com/ansible/ansible.git --recursive
sudo apt-get install python-setuptools autoconf g++ python2.7-dev
sudo apt-get install build-essential libssl-dev libffi-dev python-dev
sudo easy_install pip
sudo pip install paramiko PyYAML Jinja2 httplib2 boto
```

Should `pip` not satisfy requirements, `easy_install` is an alternative python installer. Example: `sudo python -m easy_install pyyaml`.

Create a hosts file for Ansible.

```bash
sudo mkdir /etc/ansible
echo -e "[local]\n127.0.0.1" | sudo tee -a /etc/ansible/hosts
```

Configure a shell to use Ansible from source

```bash
cd ansible
source ./hacking/env-setup -q
```

## Network Architecture

The architecture created by create-network-ec2.yml utilizes three Availability Zones (AZs) but can be adjusted to more. The image below uses two AZs for simiplicity.

![alt tag](doc/ConductR-Ansible-EC2-2AZ-Arch.png)


### Private Agent mode
Both the VPC creation and cluster building playbooks have 'private-agent' versions. They are used exactly as above but have more amis, subnets. security groups, etc.
