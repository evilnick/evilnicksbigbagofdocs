Title:Installing OpenStack

#OpenStack

##Intro

OpenStack is a versatile, open source cloud environment equally suited to serving up public, private or hybrid clouds. Canonical is a Platinum Member of the OpenStack foundation and has been involved with the OpenStack project since its inception; the software covered in this document has been developed with the intention of providing a streamlined way to deploy and manage OpenStack installations.

### Scope of this documentation

The OpenStack platform is powerful and its uses diverse. This section of documentation 
is primarily concerned with deploying a 'standard' running OpenStack system using, but not limited to, Canonical components such as MAAS, Juju and Ubuntu. Where appropriate other methods and software will be mentioned.

### Assumptions

1. Use of MAAS
   This document is written to provide instructions on how to deploy OpenStack using MAAS for hardware provisioning. If you are not deploying directly on hardware, this method will still work, with a few alterations, assuming you have a properly configured Juju environment. The main difference will be that you will have to provide different configuration options depending on the network configuration.
   
2. Use of Juju
   This document assumes an up to date, stable release version of Juju.
   
3. Local network configuration
   This document assumes that you have an adequate local network configuration, including separate interfaces for access to the OpenStack cloud. Ideal networks are laid out in the [MAAS][MAAS documentation for OpenStack]

## Planning an installation

Before deploying any services, it is very useful to take stock of the resources available and how they are to be used. OpenStack comprises of a number of interrelated services (Nova, Swift, etc) which each have differing demands in terms of hosts. For example, the Swift service, which provides object storage, has a different requirement than the Nova service, which provides compute resources.

The minimum requirements for each service and recommendations are laid out in the official [oog][OpenStack Operations Guide] which is available (free) in HTML or various downloadable formats. For guidance, the following minimums are recommended for Ubuntu Cloud:

[insert minimum hardware spec]



The recommended composition of nodes for deploying OpenStack with MAAS and Juju is that all nodes in the system should be capable of running *ANY* of the services. This is best practice for the robustness of the system, as since any physical node should fail, another can be repurposed to take its place. This obviously extends to any hardware requirements such as extra network interfaces.

If for reasons of economy or otherwise you choose to use different configurations of hardware, you should note that your ability to overcome hardware failure will be reduced. It will also be necessary to target deployments to specific nodes - see the section in the MAAS documentation on tags [MAAS tags]


###Create the OpenStack configuration file

We will be using Juju charms to deploy the component parts of OpenStack. Each charm encapsulates everything required to set up a particular service. However, the individual services have many configuration options, some of which we will want to change. 

To make this task easier and more reproduceable, we will create a separate configuration file with the relevant options for all the services. This is written in a standard YAML format. 

You can download the [openstack-config.yaml] file we will be using from here. It is also reproduced below:

```
keystone:
  openstack-origin: cloud:trusty-icehouse/updates
  admin-password: openstack
  debug: 'true'
  log-level: DEBUG
nova-cloud-controller:
  openstack-origin: cloud:trusty-icehouse/updates
  network-manager: 'Neutron'
  quantum-security-groups: 'yes'
  neutron-external-network: Public_Network
nova-compute:
  openstack-origin: cloud:trusty-icehouse/updates
  enable-live-migration: 'True'
  migration-auth-type: "none"
  virt-type: kvm
  #virt-type: lxc
  enable-resize: 'True'
quantum-gateway:
  openstack-origin: cloud:trusty-icehouse/updates
  ext-port: 'eth1'
  plugin: ovs
glance:
  openstack-origin: cloud:trusty-icehouse/updates
  ceph-osd-replication-count: 3
openstack-dashboard:
  openstack-origin: cloud:trusty-icehouse/updates
cinder:
  openstack-origin: cloud:trusty-icehouse/updates
  block-device: None
  ceph-osd-replication-count: 3
  overwrite: "true"
  glance-api-version: 2
ceph:
  source: cloud:trusty-icehouse/updates
  fsid: a51ce9ea-35cd-4639-9b5e-668625d3c1d8
  monitor-secret: AQCk5+dR6NRDMRAAKUd3B8SdAD7jLJ5nbzxXXA==
  osd-devices: /dev/sdb
  osd-reformat: 'True'
ceph-radosgw:
  source: cloud:trusty-icehouse/havana
```

For all services, we have configured the openstack-origin to point at the latest stable version, which at the time of writing is `cloud:trusty-icehouse/updates`. Further configuration for each service is explained below:

####keystone
admin password:
    You should set a memorable password here to be able to access OpenStack when it is deployed

debug: 
    It is useful to set this to 'true' initially, to monitor the setup. this will produce more verbose messaging.
   
log-level: 
    Similarly, setting the log-level to DEBUG means that more verbose logs can be generated. These options can be changed once the system is set up and running normally. 

####nova-cloud-controller

cloud-controller:
    'Neutron' - Other options are now depricated.
    
quantum-security-groups: 
    'yes'
    
neutron-external-network: 
    Public_Network - This is an interface we will use for allowing access to the cloud, and will be defined later

####nova-compute
enable-live-migration: 
  We have set this to 'True'
  migration-auth-type: "none"
  virt-type: kvm
  enable-resize: 'True'
####quantum-gateway
ext-port:
    This is where we specify the hardware for the public network. Use 'eth1' or the relevant 
  plugin: ovs
  
  
####glance

  ceph-osd-replication-count: 3

####cinder
  openstack-origin: cloud:trusty-icehouse/updates
  block-device: None
  ceph-osd-replication-count: 3
  overwrite: "true"
  glance-api-version: 2
  
####ceph
  
fsid: 
    The fsid is simply a unique identifier. You can generate a suitable value by running `uuidgen`  which should return a value which looks like: a51ce9ea-35cd-4639-9b5e-668625d3c1d8
    
monitor-secret: 
    The monitor secret is a secret string used to authenticate access. There is advice on how to generate a suitable secure secret at [ceph][the ceph website]. A typical value would be `AQCk5+dR6NRDMRAAKUd3B8SdAD7jLJ5nbzxXXA==`
    
osd-devices: 
    This should point (in order of preference) to a device,partition or filename. In this case we will assume secondary device level storage located at `/dev/sdb`
    
osd-reformat: 
    We will set this to 'True', allowing ceph to reformat the drive on provisioning. 
    
  
##Deploying OpenStack with Juju
Now that the configuration is defined, we can use Juju to deploy and relate the services.

###Initialising Juju
Juju requires a minimal amount of setup. Here we assume it has already been configured to work with your MAAS cluster (see the [juju_install][Juju Install Guide] for more information on this.

Firstly, we need to fetch images and tools that Juju will use:
```
juju sync-tools --debug
```
Then we can create the bootstrap instance:

```
juju bootstrap --upload-tools --debug
```
We use the upload-tools switch to use the local versions of the tools which we just fetched. The debug switch will give verbose output which can be useful. This process may take a few minutes, as Juju is creating an instance and installing the tools. When it has finished, you can check the status of the system with the command:
```
juju status
```
This should return something like:
```
---------- example 
```
### Deploy the OpenStack Charms

Now that the Juju bootstrap node is up and running we can deploy the services required to make our OpenStack installation. To configure these services properly as they are deployed, we will make use of the configuration file we defined earlier, by passing it along with the `--config` switch with each deploy command. Substitute in the name and path of your config file if different.

It is useful but not essential to deploy the services in the order below. It is also highly reccommended to open an additional terminal window and run the command `juju debug-log`. This will output the logs of all the services as they run, and can be useful for troubleshooting.

It is also recommended to run a `juju status` command periodically, to check that each service has been installed and is running properly. If you see any errors, please consult the [troubleshooting][troubleshooting section below].

```
juju deploy --to=0 juju-gui
juju deploy rabbitmq-server
juju deploy mysql
juju deploy --config openstack-config.yaml openstack-dashboard
juju deploy --config openstack-config.yaml keystone
juju deploy --config openstack-config.yaml ceph -n 3
juju deploy --config openstack-config.yaml nova-compute -n 3
juju deploy --config openstack-config.yaml quantum-gateway
juju deploy --config openstack-config.yaml cinder
juju deploy --config openstack-config.yaml nova-cloud-controller
juju deploy --config openstack-config.yaml glance
juju deploy --config openstack-config.yaml ceph-radosgw
```


### Add relations between the OpenStack services

Although the services are now deployed, they are not yet connected together. Each service currently exists in isolation. We use the `juju add-relation`command to make them aware of each other and set up any relevant connections and protocols. This extra configuration is taken care of by the individual charms themselves. 

    
We should start adding relations between charms by setting up the Keystone authorization service and its database, as this will be needed by many of the other connections:

juju add-relation keystone mysql

We wait until the relation is set. After it finishes check it with juju status:

```
juju status mysql
juju status keystone
```

It can take a few moments for this service to settle. Although it is certainly possible to continue adding relations (Juju manages a queue for pending actions) it can be counterproductive in terms of the overall time taken, as many of the relations refer to the same services.
The following relations also need to be made:
```
juju add-relation nova-cloud-controller mysql
juju add-relation nova-cloud-controller rabbitmq-server
juju add-relation nova-cloud-controller glance
juju add-relation nova-cloud-controller keystone
juju add-relation nova-compute mysql
juju add-relation nova-compute rabbitmq-server
juju add-relation nova-compute glance
juju add-relation nova-compute nova-cloud-controller
juju add-relation glance mysql
juju add-relation glance keystone
juju add-relation cinder keystone
juju add-relation cinder mysql
juju add-relation cinder rabbitmq-server
juju add-relation cinder nova-cloud-controller
juju add-relation openstack-dashboard keystone
juju add-relation swift-proxy swift-storage
juju add-relation swift-proxy keystone
```
Finally, the output of juju status should show the all the relations as complete. The OpenStack cloud is now running, but it needs to be populated with some additional components before it is ready for use.




##Preparing OpenStack for use

###Configuring access to Openstack



The configuration data for OpenStack can be fetched by reading the configuration file generated by the Keystone service. You can also copy this information by logging in to the Horizon (OpenStack Dashboard) service and examining the configuration there. However, we actually need only a few bits of information. The following bash script can be run to extract the relevant information:

```
#!/bin/bash

set -e

KEYSTONE_IP=`juju status keystone/0 | grep public-address | awk '{ print $2 }' | xargs host | grep -v alias | awk '{ print $4 }'`
KEYSTONE_ADMIN_TOKEN=`juju ssh keystone/0 "sudo cat /etc/keystone/keystone.conf | grep admin_token" | sed -e '/^M/d' -e 's/.$//' | awk '{ print $3 }'`

echo "Keystone IP: [${KEYSTONE_IP}]"
echo "Keystone Admin Token: [${KEYSTONE_ADMIN_TOKEN}]"

cat << EOF > ./nova.rc
export SERVICE_ENDPOINT=http://${KEYSTONE_IP}:35357/v2.0/
export SERVICE_TOKEN=${KEYSTONE_ADMIN_TOKEN}
export OS_AUTH_URL=http://${KEYSTONE_IP}:35357/v2.0/
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_TENANT_NAME=admin
EOF

juju scp ./nova.rc nova-cloud-controller/0:~
```  
This script extract the required information and then copies the file to the instance running the nova-cloud-controller.
Before we do any nova or glance command we will load the file we just created:

```
$ source ./nova.rc
$ nova endpoints
```

At this point the output of nova endpoints should show the information of all the available OpenStack endpoints.

### Install the Ubuntu Cloud Image

In order for OpenStack to create instances in its cloud, it needs to have access to relevant images
$ mkdir ~/iso
$ cd ~/iso
$ wget http://cloud-images.ubuntu.com/trusty/current/trusty-server-cloudimg-amd64-disk1.img

###Import the Ubuntu Cloud Image into Glance
!!!Note:  glance comes with the package glance-client which may need to be installed where you plan the run the command from

```
apt-get install glance-client
glance add name="Trusty x86_64" is_public=true container_format=ovf disk_format=qcow2 < trusty-server-cloudimg-amd64-disk1.img
```
###Create OpenStack private network
Note:  nova-manage  can be run from the nova-cloud-controller node or any of the nova-compute nodes. To access the node we run the following command:

```
juju ssh nova-cloud-controller/0

sudo nova-manage network create --label=private --fixed_range_v4=1.1.21.32/27 --num_networks=1 --network_size=32 --multi_host=T --bridge_interface=eth0 --bridge=br100
```

To make sure that we have created the network we can now run the following command:

```
sudo nova-manage network list
```

### Create OpenStack public network
```
sudo nova-manage floating create --ip_range=1.1.21.64/26
sudo nova-manage floating list
```
Allow ping and ssh access adding them to the default security group
Note: The following commands are run from a machine where we have the package python-novaclient installed and within a session where we have loaded the above created nova.rc file.

```
nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
```

###Create and register the ssh keys in OpenStack
Generate a default keypair
```
ssh-keygen -t rsa -f ~/.ssh/admin-key
```
###Copy the public key into Nova
We will name it admin-key:
Note: In the precise version of python-novaclient the command works with --pub_key instead of --pub-key

```
nova keypair-add --pub-key ~/.ssh/admin-key.pub admin-key
```
And make sure it’s been successfully created:
```
nova keypair-list
```

###Create a test instance
We created an image with glance before. Now we need the image ID to start our first instance. The ID can be found with this command:
```
nova image-list
```

Note: we can also use the command glance image-list
###Boot the instance:

```
nova boot --flavor=m1.small --image=< image_id_from_glance_index > --key-name admin-key testserver1
```

###Add a floating IP to the new instance
First we allocate a floating IP from the ones we created above:

```
nova floating-ip-create
```

Then we associate the floating IP obtained above to the new instance:

```
nova add-floating-ip 9363f677-2a80-447b-a606-a5bd4970b8e6  1.1.21.65
```


### Create and attach a Cinder volume to the instance
Note: All these steps can be also done through the Horizon Web UI

We make sure that cinder works by creating a 1GB volume and attaching it to the VM:

```
cinder create --display_name test-cinder1 1
```

Get the ID of the volume with cinder list:

```
cinder list
```

Attach it to the VM as vdb

```
nova volume-attach test-server1 bbb5c5c2-a5fd-4fe1-89c2-d16fe91578d4 /dev/vdb
```

Now we should be able to ssh the VM test-server1 from a server with the private key we created above and see that vdb appears in /proc/partitions

[[[[[[[[[[[[[[[[[[[[notes]]]]]]]]]]]]]]]]]]]]]]]]]]]]
slice up using maas tags:

compute  
storage +disk -cpu
admin   
network multiple nics

use tags to deploy 

talk config options.
https://code.launchpad.net/~james-page/charms/bundles/openstack-on-openstack/bundle
[[[[[[[[[[[[[[[[[[[[[[[[[[[]]]]]]]]]]]]]]]]]]]]]]]]]]]]

[troubleshooting]
[oog][http://docs.openstack.org/ops/]
[MAAS tags]
[openstack-config.yaml]
[ceph][http://ceph.com/docs/master/dev/mon-bootstrap/]
