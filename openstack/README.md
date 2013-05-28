##Deploying OpenStack with Ansible
-----------------------------------

- Requires Ansible 1.2
- Expects CentOS/RHEL 6 hosts (64 bit)
- Kernel Version 2.6.32-358.2.1.el6.x86_64

### A Primer into OpenStack Architecture
-----------------------------------------

![Alt text](/images/os_architecture.png "Arch")

As the above diagram depicts, there are several key components that make up the OpenStack Cloud Infrastructure. A brief description about the services are as follows:


- DashBoard(Horizon): A Web based GUI that provides an interface for end user or Administrator to iteract with the OpenStack cloud infrastructure. 

- Nova: The nova component is primarily responsible to take api requests from endusers/admins to create virtual machines and co-ordinate in creating/delivering them. The nova component consists of several subcomponents/processes which helps Nova to deliver the virtual machines, let's have a brief look them.

 - nova-api: Runs the api server, which accepts and responds to the enduser/admin requests

 - nova-compute: Runs on the compute nodes and interacts with hypervisors to create/terminate virtual machines.

 - nova-volume: Responsible to provide persistant storage to vm's, it is gradually being migrated to Cinder.

 - nova-nework: Responsible for networking requirements like adding firewall rules, creating bridges etc... . This component is also slowly being migrated to Quantum Services.

 - nova-scheduler: Responsible to choose the best compute server where the requested virtual machine should run.

 - nova-console: Responsible for providing the vm's console VNC to end users.
   
- Glance: This service provides the endusers/admins with options to add/remove images to the OpenStack Infrastructure, which would be used to deploy virtual machines.

- Keystone: The keystone is repsonsible for providing identity and authentication service's to all other services that require these services.

- Quantum: Quantum is primarily responsible for providing networks and connectivity to the nova instances.

- Swift: The Swift service provides endusers/admins with a higly scalable and fault tolerant object store.

- Cinder: Provides permanant disk storage services to nova instances.



###Ansible's Example Deployment Diagram
-----------------------------------------

![Alt text](/images/os_dep_diagram.png "diagram")

As the above diagram shows Ansible playbooks which deploy's OpenStack combines all the management processes into a single node (controller node) and the compute service into the other nodes(Compute Nodes). 

The Management services deployed/configured on the controller node are as follows:

- Nova: Uses MySQL to store vm info,states. The communication between several nova components are handled by Apache QPID messaging system.

- Quantum: Uses MySQL as it's backend database and QPID for interprocess communication, The quantum service uses OpenVswitch plugin to provide network services, L2 and L3 service's are taken care by quantum while firewall service is taken care by Nova Network.

- Cinder: Uses MySQL as backend database and provides permanant storage to instaces via lvm and tgtd daemon(ISCSI).

- Keystone: Uses MySQL to store the tenant/user details and provides identity and authorization service to all other components.

- Glance: Uses MySQL as the backend database, and image's are stored on the local filesystem.

- Horizon: Uses memcached to store user session details and provides UI interface to endusers/admins.

The Playbooks also configure's the Compute node's with the following components.

- Nova-compute: The nova compute service is configured to use qemu as the hypervisor and uses libvirt to create/configure/teminate virtual machines.

- Quantum Agent: A quantum agent is also configured on the compute node which uses OpenVswitch to provide L2 services to the running virtual machines.



###Physical Network deployment diagram      
---------------------------------------

![Alt text](/images/os_phy_network.png "phy_net")

The diagram in Fig 1.a shows a typical openstack production network setup which consists of four networks. A management network for management of the servers. A data network which would have all the inter vm  traffic flowing through it, an external network which would faciliate the flow of network traffic destined to an outside network, This network/interface would be attached to the network controller. The fourth network is the api network, which would be used by the endusers to access the api's for cloud operations.

For the sake of simplicity the ansible example deployment simplifies the network topology by deploying two networks, The first network would handle the traffic for management/api and data, while the second network/interface would be for the external access.

###Logical Network deployment diagram
--------------------------------------

![Alt text](/images/os_log_network.png "log_network")

Openstack provides several logical network deployment options like flat networks, provider router with private networks, tenant routers with private networks. The above diagram shows the network deployment carried out by the ansible playbooks, which is reffered to as provider router with private networks.

As it can be seen above all the tenants would get thier own private networks which can be one or many. The provider, in this case the Cloud admin would have a dedicated router which would have an external interface for all outgoing traffic. Incase any of the tenants require external access (internet/other datacenter networks) they could attach thier private network to the router and get external access.

     
###Network under the Hood
------------------------------

![Alt text](/images/os_network_detailed.png "Detailed network")

As described previousely the Ansible example playbooks deploys Openstack with a network topology of  "Provider Router with private networks for tenants". The above diagram give's a brief on the data flow in this network toplogy.

As the first step during deployment the Ansible tasks creates a few OpenVswitch software bridges on the controller and compute nodes. 
They are as follows:

####Controller Node:

- br-int: Known as the integration bridge will have all the interface of the router attached to it.

- br-ext: Known as the external bridge will have the interface's attached to it that has an external ip address, for example when an external network is created and gateway is set, a new interface is created and attached to the br-ext bridge with the gateway ip. Also all floating ip's will have an interface created and attached to this bridge.
Also a physical nic is added to this bridge so that external flows through this physical nic to reach the next hop router.

####Compute Node:

- br-int: This integration bridge will have all the interfaces from virtual machines attached to it.

Apart from the above bridges there will be one more bridge automatically created by Openstack for communication between computenode and network controller node.

- br-tun: The tunnel bridge are created on both the compute node's and the network controller node, they will have two interface attached to it. 

    - A patch port/interface:   This acts like an uplink between the integration bridge and the tunnel bridge, so that all the traffic at the intergration bridge will be forwarded to the tunnel bridge.

    - A GRE Tunnel port/interface: A gre tunnel basically encapsulates all the traffic incoming to it's interface with it's own ip address and delivers it to the opposite endpoint. So in our case all the compute nodes will have a tunnel setup with network controller as it's remote endpoint. So tunnel interface in the br-tun bridge in compute node delivers all the vm traffic coming from the br-int interface to the tunnel endpoint interface in the controller's br-tun bridge.


###A step by step look at what happens during various networks operations:
----------------------------------------------------------------------------

####A private tenant network is created:

When a private network for a tenant is created, the first task done is 

- Create an interface and assign an ip from that range and attach it to the br-int bridge on the controller node, This interface would be used by the dhcp-agent process running on the node to give out dynamic ip address to vm's that get created on this subnet. So as the above diagram shows if the tenant subnet created is 10.0.0.0/24 an ip of 10.0.0.3 would be added to a vnic and added to br-int bridge and this interface would be used by the dnsmasq process to give ip's to vm's in this subnet.

- Also an virtual interface would be created and added to the br-int bridge, this interface would typically be given the first ip in subnet created and would as the default gateway for instances that have an interface attached to this subnet. So if the subnet created is 10.0.0.0/24 an ip of 10.0.0.1 would be given to the router interface which would be attached to the br-int bridge.



####A new vm is created:

As per the above diagram let's consider that a tenant vm is created and running on 10.0.0.0/24 subnet with an ip of 10.0.0.5 on a compute node.

- During the vm creation process the openvswitch agent in the compute node would have attached the virtual nic of the vm to br-int bridge. Also the br-tun bridge would have been set up with a tunnel interface with source as the compute node and the remote endpoint as the network controller.

- Also a patch interface would have been created between the two bridges so that all traffic in the br-int bridge are forwarded to the br-tun interface.

####External network is created and gateway set on router:

As the figure above depicts an external network is used to route the traffic from the internal/private openstack networks to the external network/internet. External network creation expects a physical interface added to the br-ex bridge and that physical interface can transfer packets to next hop router for the external network. As per the above diagram 1.1.1.1 would be an interface on the next hop router for the Openstack controller.

- The first thing openstack does on external network creation is that it creates a virtual interface by the name "qg-xxx" on the br-ex bridge and assigns  the second ip on the subnet to this interface. eg: 1.1.1.2

- Openstack also adds a default route to the controller, which would be the first ip on the subnet. Eg: 1.1.1.1

####Associate Floating IP to a VM:

Floating ip is used to give access to the vm from external networks, also for giving external access to the vm's in the private network.

- When a floating ip is associated with a vm, a virtual interface is created in the br-ex bridge and the public ip is assigned to this interface.

- A Destination NAT is setup in the iptables so that the traffic coming to a particular external ip would be translated to it's corresponding private IP.

- A Source NAT is setup in the iptables so that traffic originating from a particular private ip is natted to it's corresponding external ip so that it can access external network.

 
####A Step by Step look a VM in a private network accesing internet:

As per the above diagram, consider the tenant vm is on 10.0.0.0/24 subnet and has an ip of 10.0.0.5. An external network is created with a subnet of 1.1.1.0/24 and it attached to the external router. A floating ip is also given to the tenant's VM's 1.1.1.10.
So in case of the vm pinging an external ip, the sequece of events would be thus.

- 1) Since the destination is in a external subnet the vm would try to forward the packet to it's default gateway, which in this case would be 10.0.0.1 which is a virtual interface in the br-int bridge on the controller node.

- 2) The packet traverses through the patch interface in the br-int bridge to the br-tun bridge.

- 3) The br-tun bridge in compute node has tunnel endpoint with 192.168.2.42 as source and remote endpoint as 192.168.2.41 which is the data interace of controller node. The packet then gets encapusalted in the gre packets and get transferred to the network controller node.

- 4) On reaching the br-tun bridge in network controller the gre packet is stripped and forwarded to the br-int bridge through the patch interface.

- 5) The 10.0.0.1 interface intercepts the packet and based on the routing rules forwards the packet to appropriate interface. eg: 1.1.1.2

- 6) The snat rule on the postrouting chain of iptables nats the source ip from 10.0.0.5 to 1.1.1.10 and the packet is sent out to next hop router 1.1.1.1

- 7) On recieving a reply the pre-routing chains does a destination nat and the 1.1.1.10 is changed to 10.0.0.5 and follows the same route and reaches the vm.

 
###Deploying OpenStack with Ansible.
--------------------------------------


As discussed above this example deploys OpenStack in a all in one model, where all the management service's are deployed on a single host(Controller Node) and the Hypervisor's (Compute nodes) are deployed on the other nodes. 

####Pre-Requisite's

- Centos: 6.4
- Kernel Version: 2.6.32-358.2.1.el6.x86_64 
- Controller Node: Should have an extra NIC (for external traffic routing) and an extra disk or partition for Cinder Volumes.

Before the playbooks are run please make sure that the following paramters in group_vars/all meets your environment else modify the same or any other that may need modifications.

- quantum_external_interface: eth2 # The nic that would be used for external traffic, make sure there is no ip assigned to it and it is in enabled state.

- cinder_volume_dev: /dev/vdb # An additional disk or a partition that would be used for cinder volumes, Please make sure that it is not mounted or any other volume groups are created on this disk.

- external_subnet_cidr: 1.1.1.0/24 #The subnet that would be used for external network access. If you are just tesiting any subnet should be fine and the vm's should be accesible from the controller using this external ip. If you need Ansible host to access the vm's using this public ip, make sure ansible host has an ip in this segment and that interface is on the same broadcast domain as the second interface on the network controller.

Once the Pre-requisite's are done modify the inventory file 'hosts' to match your environment, here's and example inventory.

        [openstack_controller]
        openstack-controller

        [openstack_compute]
        openstack-compute



Run the playbook to deploy the OpenStack Cloud:

        ansible-playbook -i hosts site.yml




Once the playbooks complete you can check the deployment by logging into the Horizon console http://<controller-ip>/dashboard. The credentials for login would be as defined in groupvars/all. The state of all services can be verified by checking the "system info" tab.

![Alt text](/images/os_sysinfo.png "sysinfo")

###Uploading Image to OpenStack
----------------------------------------




Once the stack is deployed Image can be uploaded to Glance for vm creation by the following command. This uploads a cirros image from a weburl directly to glance.

Note: if your external network is not a valid network, this download of image from internet might not work as OpenStack adds a default route to your controller to the first ip of external network. So please make sure you have the right default route set in your controller.


        ansible-playbook -i hosts playbooks/image.yml -e "image_name=cirros image_url=https://launchpad.net/cirros/trunk/0.3.0/+download/cirros-0.3.0-x86_64-disk.img"




We can verify the image status by checking the "images" tab in Horizon.

![Alt text](/images/os_images.png "Images")



### Create a tenant and thier private network
-------------------------------------------------




The following command shows how to create a tenant and a private network. The following command creates a tenant by the name tenant1 and creates an admin tenant by the name of t1admin and also provisions a private network/subnet with a cidr of 2.2.2.0/24 for thier vm's. 



        ansible-playbook -i hosts playbooks/tenant.yml -e "tenant_name=tenant1 tenant_username=t1admin tenant_password=abc network_name=t1net subnet_name=t1subnet subnet_cidr=2.2.2.0/24 tunnel_id=3"




The status of the tenant and the network can be verified from Horizon. The tenant list would be available in "projects tab"

![Alt text](/images/os_projects.png "projects")




The network topology would be visible in the "project-> Network Topology" tab.

![Alt text](/images/os_networks.png "networks")





###Creating a VM for the tenant.
-----------------------------------





To create a vm for a tenant we could issue the following command. The command creates a new vm for tenant1. The vm is attached to the network created above t1net and also the ansible users public key is injected into the node, so that the vm is ready to be managed by Ansible.



        ansible-playbook -i hosts playbooks/vm.yml -e "tenant_name=tenant1 tenant_username=t1admin tenant_password=abc network_name=t1net vm_name=t1vm flavor_id=6 keypair_name=t1keypair image_name=cirros" 





Once created the vm can be verified by going to the "instances" tab in Horizon"

![Alt text](/images/os_instances.png "Instances")





We can also get the console of the VM from the Horizon UI by clicking on the "console" drop down menu.

![Alt text](/images/os_vnc.png "console")





###Managing OpenStack vm's dynamically from Ansible.
-----------------------------------------------------




There are multiple ways to manage the vm's deployed in OpenStack via Ansible.

- 1) Use the inventory script. While creating the virtual machine pass metadata paramater to the vm specifying the group it should belong to and the inventory script would generate hosts arranged by thier group name.

- 2) If the requirement is to deploy and manage the virtual machines using the same Playbook this can be achieved by writing a play to create virtual machines and add those vm's dynamically to the inventory using 'add_host' module. Later write another play with group name defined in the add_host module and list out the tasks to be carried out on the vm's.







