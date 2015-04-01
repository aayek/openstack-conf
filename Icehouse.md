OpenStack Icehouse Installation - Multi Node
============================================

This document is based on [the OpenStack Official Documentation](http://docs.openstack.org/icehouse/install-guide/install/apt/content/index.html) for Icehouse and the work of [Chaima Ghribi](https://github.com/ChaimaGhribi)

| __Version__ | _1.0_ |
|---------|-----|
| __Author__ | Francesco Pantano |
| __License__ | Apache License Version 2.0 |


**Table of Contents**

- [OpenStack Icehouse Installation - Multi Node](#)
- [Openstack Components](#)
	- [Keystone](#)
	- [Glance](#)
	- [Nova](#)
	- [Neutron](#)
	- [Swift](#)
- [Qemu+KVM Multiple Networks using TAP networking Interfaces](#)
- [Basic Architecture and Network Configuration](#)
	- [Configure Controller node](#)
	- [Configure Network node](#)
	- [Configure Compute node](#)
- [Installing Components](#)
- [Controller Node](#)
	- [Install the supporting services (MySQL and RabbitMQ)](#)
	- [Install the Identity Service (Keystone)](#)
	- [Install the image Service (Glance)](#)
	- [Install the compute Service (Nova)](#)
	- [Install the network Service (Neutron)](#)
	- [Install the dashboard Service (Horizon)](#)
- [Network Node](#)
- [Compute Node](#)
- [A Docker Example](#)
- [License](#)

Openstack Components
====================
Basic Services:

* Keystone
* Glance
* Nova
* Neutron
* Swift
* Horizon

Supporting Services:

* Mysql
* RabbitMQ

We start this tutorial giving some definition about the main openstack components.

##Keystone

Keystone is the OpenStack identity service: it manages user databases and OpenStack service catalogs
and their API endpoints. It supports multiple authentication mechanisms, such as username-and-password, token-based systems and AWS-style logins.

The main Keystone's Components are:

+ __Users__: digital representations of a person, system, or service that uses OpenStack cloud
services. Keystone ensures that incoming requests are coming from a valid login user that can be
assigned resource-access tokens. Users are assigned to a particular tenant with specific role.

+ __Tenant__: Groups can be mapped to customers,projects or organizations.

+ __Role__: includes a set of assigned rights and privileges for performing a specific set of
operations for a given user. A user token issued by Keystone includes a list of that user's roles. Services then
determine how to interpret those roles.

+ __Credentials__: include username and password, username and API key, or an authentication token.

+ __Authentication__: credentials are initially a username and password or a username and API key. In
response to the credentials, the identity service issues an authentication token that the user must
provides for subsequent requests.

+ __Token__: arbitrary bit of text used to access resources. Each token describes accessible resources. 

+ __Service__: such as Compute (Nova), Object Storage (Swift), or Image Service (Glance),
provides one or more endpoints through which users can access resources and perform operations.

+ __Endpoint__: network-accessible address, usually described by URL, from which services are accessed.


![](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/KeystoneFlow.png)

#Glance

Glance provides discovery, registration and delivery services for disk and server images.
It has the ability to copy (or snapshot) a server image and then to store it promptly. Stored images
then can be used as templates to get new servers up and running quickly, and can also be used to
store and catalog unlimited backups.
Virtual-machine images can be stored in various locations, including simple filesystems and
object-storage systems such as OpenStack Object Storage (code named Swift).

Glance's main components:

* __Glance-api__: API calls for image discovery, retrieval and storage.

* __Glance-registry__: Stores, processes, and retrieves metadata for images (size, type, etc..)

* __Database__: Stores image metadata and supports many backends, including MySQL, SQlite and MongoDB.

* __Storage repository__: Supports normal file systems, RADOS block devices, Amazon S3 and HTTP for
image storage.


##Nova

Nova is built on a messaging architecture and all of its components can typically be run on several
servers. This architecture allows the components to communicate through a message queue. Deferred
objects are used to avoid blocking while a component waits in the message queue for a response.

The main Nova's components are:

* __DB__: stores most of the build-time and run-time state for a cloud infrastructure. This includes the
  instance types that are available for use, instances in use, networks available and projects.

* __Web Dashboard__: External component to communicate with the API

* __API__: Component that uses the queue or http to communicate with other components and to receive http requests

* __Auth Manager__: A python class used by all components to communicate with the backend DB or LDAP;so
  it exposes authorized APIs usage for users, projects, and roles and communicates with OpenStack Keystone

* __Object Store__: It is a simple HTTP-based object-based storage, usually replaced by Glance

* __Scheduler__: Allocates hosts to the appropriate VMs; therefore, it take a virtual machine instance request from
  the queue and determines where it should run (specifically, which compute server host it should
  run on)

* __Network__: Responsible for IP forwarding, bridges and vlans; it accepts networking tasks from the
  queue and then performs tasks to manipulate the network (such as setting up bridging interfaces or
  changing iptables rules)

* __Compute__: Controls the communication between the hypervisor and VMs

![](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/Nova_arch.png)


##Neutron

Neutron, a network service for OpenStack, is a pluggable, scalable and API-driven system for managing
networks and IP addresses. It also provides a variety of network services ranging from L3 forwarding
and NAT to load balancing, edge firewalls and IPSEC VPN.

Neutron manages software-defined networking and can be configured for advanced virtual network
topologies, such as per-tenant private networks and others. Its object abstractions include
networks, subnets and routers. Each has functionality that mimics its physical counterpart: networks
contain subnets, and routers route traffic between different subnets and networks.


Neutron configuration provides two types of networks:

* __External Network__: accessible outside openstack installation by anyone

* __Internal Network__: SDN connected directly to VMs: only VMs can access this network; ip addresses on
  an external network are allocated to ports on internal network, which allows entities outside the
  network to access VM using external IP. On these networks administrators can define firewall rules
  in groups to block and unlock ports or to allow or not a particular traffic type.


![](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/neutron.png)


#Swift

Swift is the OpenStack Object Storage Service provides a cost effective, scale-out, redundant,
scalable and fully-distributed API-accessible storage platform that can be integrated directly into
applications or used for backup, archiving and data retention. It’s equivalent to Amazon's S3.
The nodes in Swift can be broadly classified in two categories:

* __Proxy Node__: This is a public node. It handles all the http request for various Swift
	operations like uploading, managing and modifying metadata. We can setup multiple proxy nodes
	and then load balance them using a standard load balancer.

* __Storage Node__: This node actually stores data. It is recommended to make this node private,
		only accessible via proxy node but not directly. Other than storage service, this node also
		houses container service and account service which are used for managing mapping of
		containers and accounts respectively.


Qemu+KVM Multiple Networks using TAP networking Interfaces
=

First of all we have to configure the host, becouse the guest needs to share the network
configuration using the virtualization method proposed by the TAP mechanism.
Suppose we want two network interface: the first is a NAT "hand made" and the second is a private
vlan between two or more VMs.
So, we have to work in the host adding two bridges interfaces and binding them to the relative TAP
virtual devices:

```
brctl addbr br0
ip tuntap add tap0 mode tap
ifconfig tap0 promisc
brctl addif br0 tap0
dnsmasq --interface=br0 --bind-interfaces --dhcp-range=172.20.0.2,172.20.255.254
```

This is the first lan obtained by a bridging operation; if you want to share the internet access from
the host, you only have to add some iptables rules to enable traffic on the external network (and
allow the incoming one which is dropped by default).
In this way, you can obtain a NAT mode over a _hostonly_ configuration.

Specifically, we should perform these commands:

```
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -A POSTROUTING -o eth0 -J MASQUERADE
iptables -I FORWARD 1 -i tap0 -J ACCEPT
iptables -I FORWARD 1 -o tap0 -m state RELATED, ESTABLISHED -J ACCEPT
```

After that, in order to create a second vlan we just follow the above described approach and perform
the following commands:

```
brctl addbr br1
ip tuntap add tap1 mode tap
ifconfig tap1 promisc
brctl addif br1 tap1
```
Unlike before, an important advice is to set up the network parameters manually: in this way we can
control and better manage the private VLANs between VMs:

```
ifconfig br1 up
ifconfig br1 192.168.56.1 netmask 255.255.255.0
```
Finally, after this configuration phase, we can run the VM using the network devices configured before:

```
qemu-system-x86_64 --enable-kvm -net nic -net bridge,br=br0 -net nic, vlan=1 -net bridge,
vlan=1,br=br1 -hda <img\_path> -m 2048 -cpu host
```
Using this method, we can create all the network configurations needed to the components of the Openstack architecture, adding and removing devices according to your needs.


Basic Architecture and Network Configuration
============================================

This installation guide cover the step-by-step process of installing Openstack on Ubuntu 14.04.
For our environment we use the Icehouse release and the work is focused on a multi-node architecture.
In particular we have three node types:

* Controller Node;
* Network Node;
* Compute node;

The controller node is the base for the entire installation process: it provide management services
like keystone and Horizon.
The Network Node runs networking services and is responsible for virtual network provisioning and
for connecting virtual machines to external networks.
Finally, the Compute Node provide the virtual machine instances in OpenStack.
Let's say we have these three components for a given network: the following picture depicts how from
a networking point of view these components interacts to each others.

![](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/topology.png)

So, in this scenario we have these network settings:

-   **Management Network** (10.0.0.0/24): A network segment used for   administration, not accessible to the public Internet.
-   **VM Traffic Network** (10.0.1.0/24): This network is used as    internal network for traffic between virtual machines in OpenStack,
    and between the virtual machines and the network nodes that provide L3 routes out to the public network.
-   **Public Network** (192.168.100.0/24): This network is connected to the controller nodes so users can access the OpenStack interfaces,and connected to the network nodes to provide VMs with public routable traffic functionality.


Configure Controller node
=========================

The controller node has two Network Interfaces: eth0 is internal (used for connectivity for OpenStack nodes) and eth1 is external.

$ sudo -s
$ echo controller > /etc/hostname

$ vim /etc/hosts: (this file looks like as the follow)


        #controller
        10.0.0.11       controller

        #network
        10.0.0.21       network

        # compute
        10.0.0.31       compute1


$ vim /etc/network/interfaces

        # The management network interface
          auto eth0
          iface eth0 inet static
          address 10.0.0.11
          netmask 255.255.255.0

        # The public network interface
          auto eth1
          iface eth1 inet static
          address 192.168.100.11
          netmask 255.255.255.0
          gateway 192.168.100.1
          dns-nameservers 8.8.8.8

Finally, restart the network:

		$ ifdown eth0 && ifup eth0
		$ ifdown eth1 && ifup eth1

Configure Network node
======================

The network node has three network Interfaces: eth0 for management use, eth1 for connectivity between VMs and eth2 for external connectivity.

$ sudo -s
$ echo network  > /etc/hostname

$ vim /etc/hosts: (this file looks like as the follow)


        #controller
        10.0.0.11       controller

        #network
        10.0.0.21       network

        # compute
        10.0.0.31       compute1



$ vim /etc/network/interfaces

        # The management network interface
          auto eth0
          iface eth0 inet static
          address 10.0.0.21
          netmask 255.255.255.0

        # VM traffic interface
          auto eth1
          iface eth1 inet static
          address 10.0.1.21
          netmask 255.255.255.0

        # The public network interface
          auto eth2
          iface eth2 inet static
          address 192.168.100.21
          netmask 255.255.255.0
          gateway 192.168.100.1
          dns-nameservers 8.8.8.8

Finally, restart the network:

		$ ifdown eth0 && ifup eth0
		$ ifdown eth1 && ifup eth1
		$ ifdown eth2 && ifup eth2

Configure Compute node
======================

The network node has two network Interfaces: eth0 for management use and
eth1 for connectivity between VMs.

$ sudo -s
$ echo compute1  > /etc/hostname

$ vim /etc/hosts: (this file looks like as the follow)


        #controller
        10.0.0.11       controller

        #network
        10.0.0.21       network

        # compute
        10.0.0.31       compute1


$ vim /etc/network/interfaces

        # The management network interface
          auto eth0
          iface eth0 inet static
          address 10.0.0.31
          netmask 255.255.255.0

        # VM traffic interface
          auto eth1
          iface eth1 inet static
          address 10.0.1.31
          netmask 255.255.255.0

Finally, restart the network:

		$ ifdown eth0 && ifup eth0
		$ ifdown eth1 && ifup eth1


Now we can verify that the network work fine by performing a ping from each node to the others.

		# ping a site on the internet:
		ping duckduckgo.com

	    #ping the management interface on the network node:
	    ping network

	    # ping the management interface on the compute node:
	    ping compute1

		# ping the management interface on the controller node:
        ping controller
	


Installing Components
=====================

Controller Node
---------------

This node has the the basic services:

* keystone
* Glance 
* Nova,
* Neutron 
* Horizon 

and also the supporting services such as MySql database, message broker (RabbitMQ), and NTP.


![](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/controller.png)


### Install the supporting services (MySQL and RabbitMQ)

-   Update and Upgrade your System:

        apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade

-   Install NTP service (Network Time Protocol):

        apt-get install -y ntp

-   Install MySQL:

        apt-get install -y mysql-server python-mysqldb

-   Set the bind-address key to the management IP address of the
    controller node:

        vi /etc/mysql/my.cnf
        bind-address = 10.0.0.11

-   Under the [mysqld] section, set the following keys to enable InnoDB,
    UTF-8 character set, and UTF-8 collation by default:

        vi /etc/mysql/my.cnf
        [mysqld]
        default-storage-engine = innodb
        innodb_file_per_table
        collation-server = utf8_general_ci
        init-connect = 'SET NAMES utf8'
        character-set-server = utf8

-   Restart the MySQL service:

        service mysql restart

-   Delete the anonymous users that are created when the database is first started:

        mysql_install_db
        mysql_secure_installation

-   Install RabbitMQ (Message Queue):

        apt-get install -y rabbitmq-server

### Install the Identity Service (Keystone)

-   Install keystone packages:

        apt-get install -y keystone

-   Create a MySQL database for keystone:

        mysql -u root -p

        CREATE DATABASE keystone;
        GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
        GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';

        exit;

-   Remove Keystone SQLite database:

        rm /var/lib/keystone/keystone.db

-   Edit /etc/keystone/keystone.conf:

        vi /etc/keystone/keystone.conf

    > [database] replace connection =
    > sqlite:////var/lib/keystone/keystone.db by connection =
    > mysql://keystone:<KEYSTONE_DBPASS@controller/keystone>
    >
    > [DEFAULT] admin\_token=ADMIN log\_dir=/var/log/keystone

-   Restart the identity service then synchronize the database:

        service keystone restart
        keystone-manage db_sync

-   Check synchronization:

        mysql -u root -p keystone
        show TABLES;

-   Define users, tenants, and roles:

        export OS_SERVICE_TOKEN=ADMIN
        export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0

        #Create an administrative user
        keystone user-create --name=admin --pass=admin_pass --email=admin@domain.com
        keystone role-create --name=admin
        keystone tenant-create --name=admin --description="Admin Tenant"
        keystone user-role-add --user=admin --tenant=admin --role=admin
        keystone user-role-add --user=admin --role=_member_ --tenant=admin

        #Create a normal user
        keystone user-create --name=demo --pass=demo_pass --email=demo@domain.com
        keystone tenant-create --name=demo --description="Demo Tenant"
        keystone user-role-add --user=demo --role=_member_ --tenant=demo

        #Create a service tenant
        keystone tenant-create --name=service --description="Service Tenant"

-   Define services and API endpoints:

        keystone service-create --name=keystone --type=identity --description="OpenStack Identity"

        keystone endpoint-create \
        --service-id=$(keystone service-list | awk '/ identity / {print $2}') \
        --publicurl=http://192.168.100.11:5000/v2.0 \
        --internalurl=http://controller:5000/v2.0 \
        --adminurl=http://controller:35357/v2.0

-   Create a simple credential file:

        vi creds
        #Paste the following: 
        export OS_TENANT_NAME=admin
        export OS_USERNAME=admin
        export OS_PASSWORD=admin_pass
        export OS_AUTH_URL="http://192.168.100.11:5000/v2.0/"

        vi admin_creds
        #Paste the following: 
        export OS_USERNAME=admin
        export OS_PASSWORD=admin_pass
        export OS_TENANT_NAME=admin
        export OS_AUTH_URL=http://controller:35357/v2.0

-   Test Keystone:

        #clear the values in the OS_SERVICE_TOKEN and OS_SERVICE_ENDPOINT environment variables        
         unset OS_SERVICE_TOKEN OS_SERVICE_ENDPOINT

        #Request a authentication token     
        keystone --os-username=admin --os-password=admin_pass --os-auth-url=http://controller:35357/v2.0 token-get

        # Load credential admin file
        source admin_creds

        keystone token-get

        # Load credential file:
        source creds

        keystone user-list
        keystone user-role-list --user admin --tenant admin

### Install the image Service (Glance)

-   Install Glance packages:

        apt-get install -y glance python-glanceclient

-   Create a MySQL database for Glance:

        mysql -u root -p

        CREATE DATABASE glance;
        GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
        GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';

        exit;

-   Configure service user and role:

        keystone user-create --name=glance --pass=service_pass --email=glance@domain.com
        keystone user-role-add --user=glance --tenant=service --role=admin

-   Register the service and create the endpoint:

        keystone service-create --name=glance --type=image --description="OpenStack Image Service"
        keystone endpoint-create \
        --service-id=$(keystone service-list | awk '/ image / {print $2}') \
        --publicurl=http://192.168.100.11:9292 \
        --internalurl=http://controller:9292 \
        --adminurl=http://controller:9292

-   Update /etc/glance/glance-api.conf:

        vi /etc/glance/glance-api.conf

        [database]
        replace sqlite_db = /var/lib/glance/glance.sqlite with
        connection = mysql://glance:GLANCE_DBPASS@controller/glance

        [DEFAULT]
        rpc_backend = rabbit
        rabbit_host = controller

        [keystone_authtoken]
        auth_uri = http://controller:5000
        auth_host = controller
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = glance
        admin_password = service_pass

        [paste_deploy]
        flavor = keystone

-   Update /etc/glance/glance-registry.conf:

        vi /etc/glance/glance-registry.conf

        [database]
        replace sqlite_db = /var/lib/glance/glance.sqlite with:
        connection = mysql://glance:GLANCE_DBPASS@controller/glance

        [keystone_authtoken]
        auth_uri = http://controller:5000
        auth_host = controller
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = glance
        admin_password = service_pass

        [paste_deploy]
        flavor = keystone

-   Restart the glance-api and glance-registry services:

        service glance-api restart; service glance-registry restart

-   Synchronize the glance database:

        glance-manage db_sync

-   Test Glance, upload the cirros cloud image:

        source creds
        glance image-create --name "cirros-0.3.2-x86_64" --is-public true \
        --container-format bare --disk-format qcow2 \
        --location http://cdn.download.cirros-cloud.net/0.3.2/cirros-0.3.2-x86_64-disk.img

-   List Images:

        glance image-list

### Install the compute Service (Nova)

-   Install nova packages:

        apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth \
        nova-novncproxy nova-scheduler python-novaclient

-   Create a Mysql database for Nova:

        mysql -u root -p

        CREATE DATABASE nova;
        GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
        GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';

        exit;

-   Configure service user and role:

        keystone user-create --name=nova --pass=service_pass --email=nova@domain.com
        keystone user-role-add --user=nova --tenant=service --role=admin

-   Register the service and create the endpoint:

        keystone service-create --name=nova --type=compute --description="OpenStack Compute"
        keystone endpoint-create \
        --service-id=$(keystone service-list | awk '/ compute / {print $2}') \
        --publicurl=http://192.168.100.11:8774/v2/%\(tenant_id\)s \
        --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
        --adminurl=http://controller:8774/v2/%\(tenant_id\)s

-   Edit the /etc/nova/nova.conf:

        vi /etc/nova/nova.conf

        [database]
        connection = mysql://nova:NOVA_DBPASS@controller/nova

        [DEFAULT]
        rpc_backend = rabbit
        rabbit_host = controller
        my_ip = 10.0.0.11
        vncserver_listen = 10.0.0.11
        vncserver_proxyclient_address = 10.0.0.11
        auth_strategy = keystone

        [keystone_authtoken]
        auth_uri = http://controller:5000
        auth_host = controller
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = nova
        admin_password = service_pass

-   Remove Nova SQLite database:

        rm /var/lib/nova/nova.sqlite

-   Synchronize your database:

        nova-manage db sync

-   Restart nova-\* services:

        service nova-api restart
        service nova-cert restart
        service nova-conductor restart
        service nova-consoleauth restart
        service nova-novncproxy restart
        service nova-scheduler restart

-   Check Nova is running. The :-) icons indicate that everything is ok
    !:

        nova-manage service list

-   To verify your configuration, list available images:

        source creds
        nova image-list

### Install the network Service (Neutron)

-   Install the Neutron server and the OpenVSwitch packages:

        apt-get install -y neutron-server neutron-plugin-ml2

-   Create a MySql database for Neutron:

        mysql -u root -p

        CREATE DATABASE neutron;
        GRANT ALL PRIVILEGES ON neutron.* TO neutron@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
        GRANT ALL PRIVILEGES ON neutron.* TO neutron@'%' IDENTIFIED BY 'NEUTRON_DBPASS';

        exit;

-   Configure service user and role:

        keystone user-create --name=neutron --pass=service_pass --email=neutron@domain.com
        keystone user-role-add --user=neutron --tenant=service --role=admin

-   Register the service and create the endpoint:

        keystone service-create --name=neutron --type=network --description="OpenStack Networking"

        keystone endpoint-create \
        --service-id=$(keystone service-list | awk '/ network / {print $2}') \
        --publicurl=http://192.168.100.11:9696 \
        --internalurl=http://controller:9696 \
        --adminurl=http://controller:9696 

-   Update /etc/neutron/neutron.conf:

        vi /etc/neutron/neutron.conf

        [database]
        replace connection = sqlite:////var/lib/neutron/neutron.sqlite with
        connection = mysql://neutron:NEUTRON_DBPASS@controller/neutron

        [DEFAULT]
        replace  core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin with
        core_plugin = ml2
        service_plugins = router
        allow_overlapping_ips = True

        auth_strategy = keystone
        rpc_backend = neutron.openstack.common.rpc.impl_kombu
        rabbit_host = controller

        notify_nova_on_port_status_changes = True
        notify_nova_on_port_data_changes = True
        nova_url = http://controller:8774/v2
        nova_admin_username = nova
        # Replace the SERVICE_TENANT_ID with the output of this command (keystone tenant-list | awk '/ service / { print $2 }')
        nova_admin_tenant_id = SERVICE_TENANT_ID
        nova_admin_password = service_pass
        nova_admin_auth_url = http://controller:35357/v2.0

        [keystone_authtoken]
        auth_uri = http://controller:5000
        auth_host = controller
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = neutron
        admin_password = service_pass

-   Configure the Modular Layer 2 (ML2) plug-in:

        vi /etc/neutron/plugins/ml2/ml2_conf.ini

        [ml2]
        type_drivers = gre
        tenant_network_types = gre
        mechanism_drivers = openvswitch

        [ml2_type_gre]
        tunnel_id_ranges = 1:1000

        [securitygroup]
        firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
        enable_security_group = True

-   Configure Compute to use Networking:

        add in /etc/nova/nova.conf

        vi /etc/nova/nova.conf

        [DEFAULT]
        network_api_class=nova.network.neutronv2.api.API
        neutron_url=http://controller:9696
        neutron_auth_strategy=keystone
        neutron_admin_tenant_name=service
        neutron_admin_username=neutron
        neutron_admin_password=service_pass
        neutron_admin_auth_url=http://controller:35357/v2.0
        libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
        linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
        firewall_driver=nova.virt.firewall.NoopFirewallDriver
        security_group_api=neutron

-   Restart the Compute services:

        service nova-api restart
        service nova-scheduler restart
        service nova-conductor restart

-   Restart the Networking service:

        service neutron-server restart

### Install the dashboard Service (Horizon)

-   Install the required packages:

        apt-get install -y apache2 memcached libapache2-mod-wsgi openstack-dashboard

-   You can remove the openstack-dashboard-ubuntu-theme package:

        apt-get remove -y --purge openstack-dashboard-ubuntu-theme

-   Edit /etc/openstack-dashboard/local\_settings.py:

        vi /etc/openstack-dashboard/local_settings.py
        ALLOWED_HOSTS = ['localhost', '192.168.100.11']
        OPENSTACK_HOST = "controller"

-   Reload Apache and memcached:

        service apache2 restart; service memcached restart

-   Note:

        If you have this error: apache2: Could not reliably determine the server's fully qualified domain name, using 127.0.1.1. 
        Set the 'ServerName' directive  globally to suppress this message”

        Solution: Edit /etc/apache2/apache2.conf

        vi /etc/apache2/apache2.conf
        Add the following new line end of file:
        ServerName localhost

-   Reload Apache and memcached:

        service apache2 restart; service memcached restart

-   Check OpenStack Dashboard at <http://192.168.100.11/horizon>. login
    admin/admin\_pass


![](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/horizon.png)


Network Node
============

Now, let's move to second step!

The network node runs the Networking plug-in and different agents (see
the Figure below).

![image](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/network.png)

-   Update and Upgrade your System:

        apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade

-   Install NTP service:

        apt-get install -y ntp

-   Set your network node to follow up your conroller node:

        sed -i 's/server ntp.ubuntu.com/server controller/g' /etc/ntp.conf

-   Restart NTP service:

        service ntp restart

-   Install other services:

        apt-get install -y vlan bridge-utils

-   Edit /etc/sysctl.conf to contain the following:

        vi /etc/sysctl.conf
        net.ipv4.ip_forward=1
        net.ipv4.conf.all.rp_filter=0
        net.ipv4.conf.default.rp_filter=0

-   Implement the changes:

        sysctl -p

-   Install the Networking components:

        apt-get install -y neutron-plugin-ml2 neutron-plugin-openvswitch-agent dnsmasq neutron-l3-agent neutron-dhcp-agent

-   Update /etc/neutron/neutron.conf:

        vi /etc/neutron/neutron.conf

        [DEFAULT]
        auth_strategy = keystone
        rpc_backend = neutron.openstack.common.rpc.impl_kombu
        rabbit_host = controller
        replace  core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin with
        core_plugin = ml2
        service_plugins = router
        allow_overlapping_ips = True

        [keystone_authtoken]
        auth_uri = http://controller:5000
        auth_host = controller
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = neutron
        admin_password = service_pass

-   Edit the /etc/neutron/l3\_agent.ini:

        vi /etc/neutron/l3_agent.ini

        [DEFAULT]
        interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
        use_namespaces = True

-   Edit the /etc/neutron/dhcp\_agent.ini:

        vi /etc/neutron/dhcp_agent.ini

        [DEFAULT]
        interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
        dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
        use_namespaces = True

-   Edit the /etc/neutron/metadata\_agent.ini:

        vi /etc/neutron/metadata_agent.ini

        [DEFAULT]
        auth_url = http://controller:5000/v2.0
        auth_region = regionOne

        admin_tenant_name = service
        admin_user = neutron
        admin_password = service_pass
        nova_metadata_ip = controller
        metadata_proxy_shared_secret = helloOpenStack

-   Note: On the controller node:

        edit the /etc/nova/nova.conf file

        vi /etc/nova/nova.conf

        [DEFAULT]
        service_neutron_metadata_proxy = true
        neutron_metadata_proxy_shared_secret = helloOpenStack

        service nova-api restart

-   Edit the /etc/neutron/plugins/ml2/ml2\_conf.ini:

        vi /etc/neutron/plugins/ml2/ml2_conf.ini

        [ml2]
        type_drivers = gre
        tenant_network_types = gre
        mechanism_drivers = openvswitch

        [ml2_type_gre]
        tunnel_id_ranges = 1:1000

        [ovs]
        local_ip = 10.0.1.21
        tunnel_type = gre
        enable_tunneling = True

        [securitygroup]
        firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
        enable_security_group = True

-   Restart openVSwitch:

        service openvswitch-switch restart

-   Create the bridges:

        #br-int will be used for VM integration
        ovs-vsctl add-br br-int

        #br-ex is used to make to VM accessible from the internet
        ovs-vsctl add-br br-ex

-   Add the eth2 to the br-ex:

        #Internet connectivity will be lost after this step but this won't affect OpenStack's work
        ovs-vsctl add-port br-ex eth2

-   Edit /etc/network/interfaces:

        vi /etc/network/interfaces

        # The public network interface
        auto eth2
        iface eth2 inet manual
        up ifconfig $IFACE 0.0.0.0 up
        up ip link set $IFACE promisc on
        down ip link set $IFACE promisc off
        down ifconfig $IFACE down

        auto br-ex
        iface br-ex inet static
        address 192.168.100.21
        netmask 255.255.255.0
        gateway 192.168.100.1
        dns-nameservers 8.8.8.8

-   Restart network:

        ifdown eth2 && ifup eth2

        ifdown br-ex && ifup br-ex

-   Restart all neutron services:

        service neutron-plugin-openvswitch-agent restart
        service neutron-dhcp-agent restart
        service neutron-l3-agent restart
        service neutron-metadata-agent restart
        service dnsmasq restart

-   Check status:

        service neutron-plugin-openvswitch-agent status
        service neutron-dhcp-agent status
        service neutron-l3-agent status
        service neutron-metadata-agent status
        service dnsmasq status

-   Create a simple credential file:

        vi creds
        #Paste the following:
        export OS_TENANT_NAME=admin
        export OS_USERNAME=admin
        export OS_PASSWORD=admin_pass
        export OS_AUTH_URL="http://192.168.100.11:5000/v2.0/"

-   Check Neutron agents:

        source creds
        neutron agent-list

Compute Node
============

Finally, let's install the services on the compute node!

It uses KVM as hypervisor and runs nova-compute, the Networking plug-in
and layer 2 agent.

![](https://raw.githubusercontent.com/frances-co/openstack-conf/master/imgs/compute.png)

-   Update and Upgrade your System:

        apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade

-   Install ntp service:

        apt-get install -y ntp

-   Set the compute node to follow up your conroller node:

        sed -i 's/server ntp.ubuntu.com/server controller/g' /etc/ntp.conf

-   Restart NTP service:

        service ntp restart

-   Check that your hardware supports virtualization:

        apt-get install -y cpu-checker
        kvm-ok

-   Install and configure kvm:

        apt-get install -y kvm libvirt-bin pm-utils

-   Install the Compute packages:

        apt-get install -y nova-compute-kvm python-guestfs

-   Make the current kernel readable:

        dpkg-statoverride  --update --add root root 0644 /boot/vmlinuz-$(uname -r)

-   Enable this override for all future kernel updates, create the file
    /etc/kernel/postinst.d/statoverride containing:

        vi /etc/kernel/postinst.d/statoverride
        #!/bin/sh
        version="$1"
        # passing the kernel version is required
        [ -z "${version}" ] && exit 0
        dpkg-statoverride --update --add root root 0644 /boot/vmlinuz-${version}

-   Make the file executable:

        chmod +x /etc/kernel/postinst.d/statoverride

-   Modify the /etc/nova/nova.conf like this:

        vi /etc/nova/nova.conf
        [DEFAULT]
        auth_strategy = keystone
        rpc_backend = rabbit
        rabbit_host = controller
        my_ip = 10.0.0.31
        vnc_enabled = True
        vncserver_listen = 0.0.0.0
        vncserver_proxyclient_address = 10.0.0.31
        novncproxy_base_url = http://192.168.100.11:6080/vnc_auto.html
        glance_host = controller
        vif_plugging_is_fatal=false
        vif_plugging_timeout=0

        [database]
        connection = mysql://nova:NOVA_DBPASS@controller/nova

        [keystone_authtoken]
        auth_uri = http://controller:5000
        auth_host = controller
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = nova
        admin_password = service_pass

-   Delete /var/lib/nova/nova.sqlite file:

        rm /var/lib/nova/nova.sqlite

-   Restart nova-compute services:

        service nova-compute restart

-   Edit /etc/sysctl.conf to contain the following:

        vi /etc/sysctl.conf
        net.ipv4.ip_forward=1
        net.ipv4.conf.all.rp_filter=0
        net.ipv4.conf.default.rp_filter=0

-   Implement the changes:

        sysctl -p

-   Install the Networking components:

        apt-get install -y neutron-common neutron-plugin-ml2 neutron-plugin-openvswitch-agent

-   Update /etc/neutron/neutron.conf:

        vi /etc/neutron/neutron.conf

        [DEFAULT]
        auth_strategy = keystone
        replace  core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin with
        core_plugin = ml2
        service_plugins = router
        allow_overlapping_ips = True

        rpc_backend = neutron.openstack.common.rpc.impl_kombu
        rabbit_host = controller

        [keystone_authtoken]
        auth_uri = http://controller:5000
        auth_host = controller
        auth_port = 35357
        auth_protocol = http
        admin_tenant_name = service
        admin_user = neutron
        admin_password = service_pass

-   Configure the Modular Layer 2 (ML2) plug-in:

        vi /etc/neutron/plugins/ml2/ml2_conf.ini

        [ml2]
        type_drivers = gre
        tenant_network_types = gre
        mechanism_drivers = openvswitch

        [ml2_type_gre]
        tunnel_id_ranges = 1:1000

        [ovs]
        local_ip = 10.0.1.31
        tunnel_type = gre
        enable_tunneling = True

        [securitygroup]
        firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
        enable_security_group = True

-   Restart the OVS service:

        service openvswitch-switch restart

-   Create the bridges:

        #br-int will be used for VM integration
        ovs-vsctl add-br br-int

-   Edit /etc/nova/nova.conf:

        vi /etc/nova/nova.conf

        [DEFAULT]
        network_api_class = nova.network.neutronv2.api.API
        neutron_url = http://controller:9696
        neutron_auth_strategy = keystone
        neutron_admin_tenant_name = service
        neutron_admin_username = neutron
        neutron_admin_password = service_pass
        neutron_admin_auth_url = http://controller:35357/v2.0
        linuxnet_interface_driver = nova.network.linux_net.LinuxOVSInterfaceDriver
        firewall_driver = nova.virt.firewall.NoopFirewallDriver
        security_group_api = neutron

-   Edit /etc/nova/nova-compute.conf with the correct hypervisor type
    (set to qemu if using virtualbox for example, kvm is default) :

        vi /etc/nova/nova-compute.conf

        [DEFAULT]
        compute_driver=libvirt.LibvirtDriver
        [libvirt]
        virt_type=qemu

-   Restart nova-compute services:

        service nova-compute restart

-   Restart the Open vSwitch (OVS) agent:

        service neutron-plugin-openvswitch-agent restart

-   Check Nova is running. The :-) icons indicate that everything is ok
    !:

        nova-manage service list



A Docker Example
----------------
TODO...

License
-------


Licensed under the Apache License, Version 2.0 (the "License"); you may
not use this file except in compliance with the License. You may obtain a copy of the License at:

    http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
