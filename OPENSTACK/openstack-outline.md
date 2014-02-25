Deploy OpenStack
All new machines should show as Ready in the MAAS Web UI and the bootstrap node should be  running, we can begin deploying the OpenStack charms.
Define where the services will be allocated

Note: The table at the beginning of this document shows the decided roles as well, this is a reminder before we start deploying roles/services to nodes.

We will use the following setup:
Juju Bootstrap node
Main service: zookeeper
Other services allocated: 
Nova Cloud Controller
Main service: Nova Cloud Controller
Other services allocated: Keystone, OpenStack Dashboard (Horizon)
Swift storage nodes (2-5)
Main service: Swift Storage
Other services allocated: 
Swift storage node 1
Main service: Swift Storage
Other services allocated: Glance
Swift Proxy node
Main service: Swift Proxy
Other services allocated:
Nova Compute nodes (3-6)
Main service: Nova Cloud Compute
Other services:
RabbitMQ node
Main service: RabbitMQ
Other services allocated: MySQL

Note: We will deploy the main services, as described above, to the desired nodes with juju deploy and the other services with jitsu deploy-to

Section
Define where the services will be allocated
Status
PENDING | DONE
Comments
If there are issues or comments worth pointing out include them here, if not leave it blank.

Checkout the OpenStack charms locally
$ mkdir -p $HOME/charms/precise
$ cd $HOME/charms/precise
$ for i in glance keystone nova-cloud-controller nova-compute openstack-dashboard cinder rabbitmq-server swift-storage swift-proxy mysql landscape-client;
do
	charm get $i
done
Create the OpenStack configuration file
Edit /home/ubuntu/charms/openstack.yaml on the maas node adding the following configuration needed for the openstack charms:

keystone:
  openstack-origin: cloud:precise-folsom/updates
  admin-password: openstack
nova-cloud-controller:
  openstack-origin: cloud:precise-folsom/updates
  network-manager: FlatDHCPManager
  bridge-interface: br100
  bridge-ip: 1.1.21.9
  bridge-netmask: 255.255.255.224
  config-flags: ec2_private_dns_show_ip=true,flat_network_bridge=br100,public_interface=eth0
nova-compute:
  openstack-origin: cloud:precise-folsom/updates
  bridge-interface: br100
  bridge-ip: 1.1.21.8
  bridge-netmask: 255.255.255.224
  flat-interface: eth0
  config-flags: ec2_private_dns_show_ip=true,flat_network_bridge=br100,public_interface=eth0
swift-proxy:
  openstack-origin: cloud:precise-folsom/updates
  country: "US"
  state: "NJ"
  locale: "Some locale"
  auth-type: keystone
swift-storage:
  openstack-origin: cloud:precise-folsom/updates
  block-device: sdb
  overwrite: "true"
  # remove overwrite: “true” after deployment so the block device is not erased if you deploy again 
glance:
  openstack-origin: cloud:precise-folsom/updates
cinder:
  openstack-origin: cloud:precise-folsom/updates
  block-device: sdb
  overwrite: "true"
openstack-dashboard:
  openstack-origin: cloud:precise-folsom/updates

Section
Checkout the OpenStack charms locally. Create the OpenStack configuration file
Status
PENDING | DONE
Comments
If there are issues or comments worth pointing out include them here, if not leave it blank.

Deploy the OpenStack Services

Note: juju deploy deploys the service to a MAAS node and juju add-unit scales out the deployed service to additional nodes. By using constraints we can decide what node we deploy to or scale out to.
Note: to watch the output of juju running hooks on each node, open an additional terminal on the maas server and run:
$  juju debug-log
Note: The OpenStack related services need the openstack.yaml files when we deploy them.
Deploy Swift services to swift storage nodes

We can start by deploying Swift Storage on every node allocated to Swift Storage. From the directory where we have the openstack.yaml file and the charms directory we run the following commands:

$ juju deploy --repository  . --config=openstack.yaml --constraints maas-name=swift-storage-01.customer.com local:precise/swift-storage

$ juju set-constraints maas-name=swift-storage-02.customer.com
$ juju add-unit swift-storage

$ juju set-constraints maas-name=swift-storage-03.customer.com
$ juju add-unit swift-storage

$ juju set-constraints maas-name=swift-storage-04.customer.com
$ juju add-unit swift-storage 

$ juju set-constraints maas-name=swift-storage-05.customer.com
$ juju add-unit swift-storage

Clear the constraints:
$ juju set-constraints maas-name=

Make sure that we don’t overwrite sdc in future deployments:

$ juju set swift-storage overwrite=false
Deploy swift-proxy (Swift API)
$ juju deploy --repository . --config=openstack.yaml --constraints maas-name=swift-api-01.customer.com local:precise/swift-proxy
Deploy Nova Cloud Controller
$ juju deploy --repository . --config=openstack.yaml --constraints maas-name=nova-cloud-controller-01.customer.com local:precise/nova-cloud-controller
Find the ID of the nodes that will allocate each service
If we need to deploy more than one service to a particular node, we will use the jitsu deploy-to command which works with node IDs instead of node names.

With juju status we check the identification number assigned to each node and we will use it to deploy services to the desired nodes. If the output of juju status contains this:

17:
    agent-state: running
    dns-name: nova-cloud-controller-01.customer.com7ba9-11e2-88c2-003048fd7032/
    instance-id: /MAAS/api/1.0/nodes/node-6de875ee-7ba9-11e2-88c2-003048fd7032/
    instance-state: unknown

In this case we will use the node ID 17 to deploy to nova-cloud-controller-01.customer.com the desired services. One at a time, waiting for each to finish.

Note: if the services below have a dedicated machine we will use juju deploy as in the above services, instead jitsu deploy-to
Deploy Glance
$ jitsu deploy-to <machine-id> --repository . --config=openstack.yaml local:precise/glance
Deploy Cinder
$ jitsu deploy-to <machine-id> --repository . --config=openstack.yaml local:precise/cinder
Deploy MySQL
$ jitsu deploy-to <machine-id> --repository . local:precise/mysql
Deploy RabbitMQ Server
$ jitsu deploy-to <machine-id> --repository . local:precise/rabbitmq-server
Deploy Nova Compute
$ jitsu deploy-to <machine-id> --repository . --config=openstack.yaml local:precise/nova-compute
Deploy Keystone
jitsu deploy-to <machine-id> --repository . --config=openstack.yaml local:precise/keystone
Deploy Horizon
jitsu deploy-to <machine-id> --repository . --config=openstack.yaml  local:precise/openstack-dashboard

Section
Deploy the OpenStack Charms
Status
PENDING | DONE
Comments
If there are issues or comments worth pointing out include them here, if not leave it blank.

Add relations between the OpenStack services
Check juju status
Verify that the proper machines are deployed and that there are no errors reported:

$ juju status

Note: It is recommended to verify that each relation is added successfully using juju status  before moving to the next relation. Some relations may involve the same node, such as mysql, and may conflict if configuration is happening on the same node at the same time.
Note: It is a good practice to check the juju log during the creation of the relations:
$ juju debug-log	
Start adding relations between charms:

$ juju add-relation keystone mysql

We wait until the relation is set. After it finishes check it with juju status:

$ juju status mysql
$ juju status keystone

If the relations are set and the services started then we proceed with the rest.

$ juju add-relation nova-cloud-controller mysql
$ juju add-relation nova-cloud-controller rabbitmq-server
$ juju add-relation nova-cloud-controller glance
$ juju add-relation nova-cloud-controller keystone
$ juju add-relation nova-compute mysql
$ juju add-relation nova-compute rabbitmq-server
$ juju add-relation nova-compute glance
$ juju add-relation nova-compute nova-cloud-controller
$ juju add-relation glance mysql
$ juju add-relation glance keystone
$ juju add-relation cinder keystone
$ juju add-relation cinder mysql
$ juju add-relation cinder rabbitmq-server
$ juju add-relation cinder nova-cloud-controller
$ juju add-relation openstack-dashboard keystone
$ juju add-relation swift-proxy swift-storage
$ juju add-relation swift-proxy keystone

Finally, the output of juju status should show the all the relations.

Section
Add relations between the OpenStack services
Status
PENDING | DONE
Comments
If there are issues or comments worth pointing out include them here, if not leave it blank.

Set up Access to Openstack and Start Booting VMs
Once charms are deployed and relations are added, we can configure the private network on each compute node.
Get the Keystone IP Address and admin_token
Get the admin token from  the machine running the keystone service:

$ juju ssh keystone/0
$ sudo grep admin_token /etc/keystone/keystone.conf

Save the admin_token to use it on the next step.
Create an rc file with the OpenStack environment
We create a nova.rc file that will have the required environment to run OpenStack commands. 
First find the Keystone hostname
To find the Keystone hostname we can run juju status keystone:

$ juju status keystone | grep dns-name
2013-02-22 10:27:54,355 INFO Connecting to environment...
2013-02-22 10:27:55,038 INFO Connected to environment.
2013-02-22 10:27:55,490 INFO 'status' command finished successfully
    dns-name: <keystone_hostname>
Now create the rc file
$ echo << EOF > ./nova.rc
export SERVICE_ENDPOINT=http://<keystone_hostname>:35357/v2.0/
export SERVICE_TOKEN=<keystone_admin_token>
export OS_AUTH_URL=http://<keystone_hostname>:35357/v2.0/
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_TENANT_NAME=admin
EOF

Load the rc file and check that the OpenStack environment
Before we do any nova or glance command we will load the file we just created:

$ source ./nova.rc
$ nova endpoints

At this point the output of nova endpoints should show the information of all the available OpenStack endpoints.
Download the Ubuntu Cloud Image
$ mkdir ~/iso
$ cd ~/iso
$ wget http://cloud-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img
Import the Ubuntu Cloud Image into Glance
Note:  glance comes with the package glance-client which may need to be installed where you plan the run the command from

$ apt-get install glance-client
$ glance add name="Precise x86_64" is_public=true container_format=ovf disk_format=qcow2 < precise-server-cloudimg-amd64-disk1.img
Create OpenStack private network
Note:  nova-manage  can be run from the nova-cloud-controller node or any of the nova-compute nodes. To access the node we run the following command:
$ juju ssh nova-cloud-controller/0

$ sudo nova-manage network create --label=private --fixed_range_v4=1.1.21.32/27 --num_networks=1 --network_size=32 --multi_host=T --bridge_interface=eth0 --bridge=br100

To make sure that we have created the network we can now run the following command:

$ sudo nova-manage network list
Create OpenStack public / floating network
$ sudo nova-manage floating create --ip_range=1.1.21.64/26
$ sudo nova-manage floating list
Allow ping and ssh access adding them to the default security group
Note: The following commands are run from a machine where we have the package python-novaclient installed and within a session where we have loaded the above created nova.rc file.

$ nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
$ nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
Create and register the ssh keys in OpenStack
Generate a default keypair
$ ssh-keygen -t rsa -f ~/.ssh/admin-key
Copy the public key into Nova
We will name it admin-key:
Note: In the precise version of python-novaclient the command works with --pub_key instead of --pub-key

$ nova keypair-add --pub-key ~/.ssh/admin-key.pub admin-key

And make sure it’s been successfully created:
$ nova keypair-list
Create our first instance
We created an image with glance before. Now we need the image ID to start our first instance. The ID can be found with this command:

$ nova image-list

Note: we can also use the command glance image-list
Boot the instance:

$ nova boot --flavor=m1.small --image=< image_id_from_glance_index > --key-name admin-key testserver1

Add a floating IP to the new instance
First we allocate a floating IP from the ones we created above:

$ nova floating-ip-create

Then we associate the floating IP obtained above to the new instance:

$ nova add-floating-ip 9363f677-2a80-447b-a606-a5bd4970b8e6  1.1.21.65



Create and attach a Cinder volume to the instance
Note: All these steps can be also done through the Horizon Web UI

We make sure that cinder works by creating a 1GB volume and attaching it to the VM:

$ cinder create --display_name test-cinder1 1

Get the ID of the volume with cinder list:

$ cinder list

Attach it to the VM as vdb

$ nova volume-attach test-server1 bbb5c5c2-a5fd-4fe1-89c2-d16fe91578d4 /dev/vdb

Now we should be able to ssh the VM test-server1 from a server with the private key we created above and see that vdb appears in /proc/partitions

Section
Set up Access to Openstack and Start Booting VMs
Status
PENDING | DONE
Comments
If there are issues or comments worth pointing out include them here, if not leave it blank.


