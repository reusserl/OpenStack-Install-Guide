==========================================================
  OpenStack Havana on Debian Wheezy Install Guide
==========================================================

:Version: 0.03
:Source: https://github.com/reusserl/OpenStack-Install-Guide
:Keywords: OpenStack, Havana, Debian, Wheezy, Multi, Node, Neutron, Nova, Keystone, Glance, Horizon, Cinder, OpenVSwitch, KVM

Authors
==========

L\. Reusser 

Contributors
=============



Table of Contents
==================

::

  0. What is it?
  1. Requirements
  2. Controller Node
  3. Network Node
  4. Compute Node
  5. Your first VM
  6. Licensing
  7. Contacts
  8. Credits
  9. To do

0. What is it?
==============

OpenStack Havana on Debian Wheezy Install Guide is an easy and tested way to create your own OpenStack platform. 

Status: Beta


1. Requirements
====================

For a three node setup, you need at least three nodes ;-) For testing purpose, you can also use just one node.

:Node Role: NICs
:Control Node: eth0 (10.10.10.51), eth1 (192.168.100.51)
:Network Node: eth0 (10.10.10.52), eth1 (10.20.20.52), eth2 (192.168.100.52)
:Compute Node: eth0 (10.10.10.53), eth1 (10.20.20.53)


.. image:: OpenStack_Setup_Multi-Node_Multi-Nova_X01.01.jpg

2. Controller Node
===================

2.1. Preparing Debian
----------------------

* Install a Debian Wheezy (64 Bit) minimal system and su to root::

   sudo su

* Add Havana repositories::

   echo "deb http://havana.pkgs.enovance.com/debian havana main
   deb http://archive.gplhost.com/debian havana-backports main" > /etc/apt/sources.list.d/openstack.list
   # add gplhost key
   apt-get install -y gplhost-archive-keyring
   # add enovance key
   wget -qO - http://havana.pkgs.enovance.com/debian/dists/havana/pubkey.gpg | apt-key add -
   apt-key list


* Update your system::

   apt-get update && apt-get dist-upgrade -y

* DNS Resolution::

   echo 'domain yourdomain.ch
   nameserver 8.8.8.8
   nameserver 8.8.4.4
   ' > /etc/resolv.conf

* Eternal Command History::

   echo '
   HISTSIZE=1000000
   HISTFILESIZE=9999999
   HISTTIMEFORMAT=(%d.%m.%Y) %H:%M
   export HISTSIZE HISTFILESIZE HISTTIMEFORMAT
   ' >> /root/.bashrc

* Remove Exim and install Postfix::

   apt-get install postfix -y
   dpkg --purge exim4 exim4-base exim4-config exim4-daemon-light

* Activate mail forwarding::

   echo "root: youremail@yourdomain.ch" >>/etc/aliases
   newaliases
   /etc/init.d/postfix reload

* Remove unneeded software::

   dpkg --purge nfs-common rpcbind

* Install some useful tools::

   apt-get install ethtool tcpdump

* Activate VLAN’s::

   apt-get install vlan
   echo "
   # vlan's
   8021q
   " >> /etc/modules
   modprobe 8021q

* Enable injection

   To enable key-file, network & metadata injection into instances images::
   
   echo "nbd max_part=65" >> /etc/modules
   modprobe nbd max_part=65

* Install ntp service::

   apt-get install -y ntp

* Check if ntp is working::

   ntpq -pn
   
   remote                refid      st t when poll reach   delay   offset  jitter
   62.12.173.11    .PPS.            1 u  350 1024  377    4.725   -0.835   0.351
   195.141.190.190 212.161.179.138  2 u  418 1024  377    4.451   -0.971   0.696
   130.60.204.10   130.60.159.8     3 u  584 1024  377    5.139    6.358   2.058

   The reach column should show some non zero values after a minute.

2.2. Networking
----------------

* Only one NIC should be reachable from the internet. This is where the customers will access the webinterface horizon. In our case, this is eth1. But it could also be a bond interface (bond0) or a vlan (vla100) or both bond0.100. So edit /etc/network/interfaces accordingly::

   #Not public available (used for OpenStack management and iscsi)
   auto eth0
   iface eth0 inet static
   address 10.10.10.51
   netmask 255.255.255.0

   #For Exposing OpenStack API over the internet
   auto eth1
   iface eth1 inet static
   address 192.168.100.51
   netmask 255.255.255.0
   gateway 192.168.100.1
   dns-nameservers 8.8.8.8

If you want to use your controller node as network node as well, you have to add an interface for your floating (public) ip addresses. If your floating ip addresses are on the same subnet as your OpenStack API (Horizon), you could use this interface.

* Restart the networking service::

   nohup sh -c "/etc/init.d/networking stop; sleep 2; /etc/init.d/networking start"

2.3. MySQL & RabbitMQ
-----------------------

* Install MySQL::

   apt-get install -y mysql-server python-mysqldb
   # enter a good mysql root password
   mysql_secure_installation

* Configure mysql to accept all incoming requests on our management network::

   sed -i 's/127.0.0.1/10.10.10.51/g' /etc/mysql/my.cnf
   service mysql restart
   netstat -tulpen| grep 3306

2.4. RabbitMQ
-------------------

* Install RabbitMQ::

   apt-get install -y rabbitmq-server 

* Create these databases::

   mysql -u root -p

   #mysql>
   
   #Keystone
   CREATE DATABASE keystone;
   GRANT ALL ON keystone.* TO 'keystoneUser'@'%' IDENTIFIED BY 'keystonePass357';
   
   #Glance
   CREATE DATABASE glance;
   GRANT ALL ON glance.* TO 'glanceUser'@'%' IDENTIFIED BY 'glancePass357';

   #Neutron
   CREATE DATABASE neutron;
   GRANT ALL ON neutron.* TO 'neutronUser'@'%' IDENTIFIED BY 'neutronPass357';

   #Nova
   CREATE DATABASE nova;
   GRANT ALL ON nova.* TO 'novaUser'@'%' IDENTIFIED BY 'novaPass357';      

   #Cinder
   CREATE DATABASE cinder;
   GRANT ALL ON cinder.* TO 'cinderUser'@'%' IDENTIFIED BY 'cinderPass357';

   quit;
 
2.5. Others
-------------------

* Install other services::

   apt-get install -y bridge-utils

* Enable IP_Forwarding::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf

   # To save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

2.6. Keystone
-------------------

* Start by the keystone packages::

   apt-get install -y keystone
   # Set up database for Keystone: No
   # Password: keystonePass357  (Service Token)
   # Register administration tenants: No
   # Register keystone endpoints: No

* Adapt the connection attribute in the /etc/keystone/keystone.conf to the new database::

   # vi /etc/keystone/keystone.conf
   connection = mysql://keystoneUser:keystonePass357@10.10.10.51/keystone

* Restart the identity service then synchronize the database::

   service keystone restart
   keystone-manage db_sync

* Get admin_token from config file::

   export OS_SERVICE_TOKEN=`grep ^admin_token /etc/keystone/keystone.conf | cut -d" " -f3`

* Fill up the keystone database using the two scripts available in the `Scripts folder <https://github.com/reusserl/OpenStack-Install-Guide/tree/KeystoneScripts>`_ of this git repository::

   #Modify the **HOST_IP** and **EXT_HOST_IP** variables before executing the scripts
   
   wget https://raw.github.com/reusserl/OpenStack-Install-Guide/master/KeystoneScripts/keystone_basic.sh
   wget https://raw.github.com/reusserl/OpenStack-Install-Guide/master/KeystoneScripts/keystone_endpoints_basic.sh

   chmod +x keystone_basic.sh keystone_endpoints_basic.sh

   vi keystone_basic.sh
   HOST_IP=10.10.10.51
   ADMIN_PASSWORD=${ADMIN_PASSWORD:-SuperAdminPass123}
   SERVICE_PASSWORD=${SERVICE_PASSWORD:-servicePass123}

   vi keystone_endpoints_basic.sh
   # Host address
   HOST_IP=10.10.10.51
   EXT_HOST_IP=192.168.100.51
   # MySQL definitions
   MYSQL_USER=keystoneUser
   MYSQL_DATABASE=keystone
   MYSQL_HOST=$HOST_IP
   MYSQL_PASSWORD=keystonePass55!!
   # Keystone definitions
   KEYSTONE_REGION=RegionOne

   ./keystone_basic.sh
   ./keystone_endpoints_basic.sh

* Modify bashrc file so we won't be bothered later::

   echo "export OS_TENANT_NAME=admin
   export OS_USERNAME=admin
   export OS_PASSWORD='SuperAdminPass123'
   export OS_AUTH_URL=http://192.168.100.51:5000/v2.0/" >> ~/.bashrc

   source ~/.bashrc

* To test Keystone, we use a simple CLI command::

   keystone user-list

2.7. Glance
-------------------

* We Move now to Glance installation::

   apt-get install -y glance
   # Set up a database for Glance? No
   # IP address of your RabbitMQ host: localhost
   # Username for connection to the RabbitMQ server: guest
   # Password for connection to the RabbitMQ server: guest
   # Pipeline flavor: keystone
   # Authentication server hostname: 127.0.0.1
   # Authentication server password: keystonePass357
   # Register Glance in the Keystone endpoint catalog? No


* Update /etc/glance/glance-api-paste.ini with::
   
   # vi /etc/glance/glance-api-paste.ini
   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   delay_auth_decision = true
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = servicePass123

* Update the /etc/glance/glance-registry-paste.ini with::

   # vi /etc/glance/glance-registry-paste.ini
   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = glance
   admin_password = servicePass123

* Update /etc/glance/glance-api.conf with::

   # vi /etc/glance/glance-api.conf
   sql_connection = mysql://glanceUser:glancePass357@10.10.10.51/glance

* And::

   [paste_deploy]
   flavor = keystone
   
* Update the /etc/glance/glance-registry.conf with::

   # vi /etc/glance/glance-registry.conf
   sql_connection = mysql://glanceUser:glancePass357@10.10.10.51/glance

* And::

   [paste_deploy]
   flavor = keystone

* Restart the glance-api and glance-registry services::

   service glance-api restart; service glance-registry restart

* Synchronize the glance database::

   glance-manage db_sync

* To test Glance, upload the cirros cloud image directly from the internet::

   glance image-create --name myFirstImage --is-public true --container-format bare --disk-format qcow2 --location http://download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img

* Now list the image to see what you have just uploaded::

   glance image-list

2.8. Neutron
-------------------

* Install the Neutron server and the OpenVSwitch package collection::

   apt-get install -y neutron-server
   
   Set up a database for Neutron? No
   IP address of your RabbitMQ host: 10.10.10.51
   Username for connection to the RabbitMQ server: guest
   Password for connection to the RabbitMQ server: guest
   Authentication server hostname: 10.10.10.51
   Authentication server password: servicePass123
   Neutron plugin: OpenVSwitch
   Type of network to allocate for tenant networks: gre
   Enable tunneling? yes
   Tunnel id ranges: 1:1000
   Local IP address of this hypervisor: 10.10.10.51
   Register Neutron in the Keystone endpoint catalog? no
   

* Edit and check the OVS plugin configuration file /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://neutronUser:neutronPass357@10.10.10.51/neutron

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   enable_tunneling = True
   local_ip = 10.10.10.51

   #Firewall driver for realizing neutron security group function
   [SECURITYGROUP]
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
   
   
   You can use the following command to check your config::
   grep -v ^$ /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini | grep -v ^#

* Edit /etc/neutron/api-paste.ini ::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = servicePass123

* Update the /etc/neutron/neutron.conf::

   [DEFAULT]
   rabbit_host = 147.87.239.108
   rabbit_password = guest
   rabbit_userid = guest

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = servicePass123
   signing_dir = /var/lib/neutron/keystone-signing
   
   [database]
   connection = mysql://neutronUser:neutronPass357@10.10.10.51/neutron

* Restart the neutron server::

   service neutron-server restart
   # and check the log
   less /var/log/neutron/neutron-server.log
   
   You can ignore the following error message::
   ERROR neutron.common.legacy [-] Skipping unknown group key: firewall_driver

2.9. Nova
------------------

* Start by installing nova components::

   apt-get install -y nova-api nova-cert novnc nova-consoleauth nova-scheduler nova-novncproxy nova-doc nova-conductor
   Set up a database for Nova? no
   IP address of your RabbitMQ host: 10.10.10.51
   Username for connection to the RabbitMQ server: guest
   Password for connection to the RabbitMQ server: guest
   Authentication server hostname: 10.10.10.51
   Authentication server password: servicePass123
   Start nova services at boot? yes
   API to activate: [ ] ec2, [ ] osapi_compute, [ ] metadata 
   Value for my_ip: 10.10.10.51
   Register Nova in the keystone endpoint catalog? No
   
	
* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = servicePass123
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

* Modify the /etc/nova/nova.conf like this::

   [DEFAULT] 
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=10.10.10.51
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass357@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.51
   vncserver_listen=0.0.0.0

   # Network settings
   network_api_class=nova.network.neutronv2.api.API
   neutron_url=http://10.10.10.51:9696
   neutron_auth_strategy=keystone
   neutron_admin_tenant_name=service
   neutron_admin_username=neutron
   neutron_admin_password=servicePass123
   neutron_admin_auth_url=http://10.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want neutron + Nova Security groups
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=neutron
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   #-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver

   #Metadata
   service_neutron_metadata_proxy = True
   neutron_metadata_proxy_shared_secret = helloOpenStack123

   # Compute #
   #compute_driver=libvirt.LibvirtDriver

   # Cinder #
   #volume_api_class=nova.volume.cinder.API
   #osapi_volume_listen_port=5900

* Synchronize your database::

   nova-manage db sync

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done; cd -

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list

2.10. Cinder
--------------

* Install the required packages::

   apt-get install -y cinder-api cinder-scheduler cinder-volume iscsitarget open-iscsi iscsitarget-dkms
   Set up a database for Cinder? No
   IP address of your RabbitMQ host: 10.10.10.51
   Username for connection to the RabbitMQ server: guest
   Password for connection to the RabbitMQ server: guest
   Authentication server hostname: 10.10.10.51
   Authentication server password: servicePass123
   Start cinter services at boot? yes
   Cinder volume group: cinder-volumes
   Register Cinder in the Keystone endpoint catalog? No

* Configure the iscsi services::

   sed -i 's/false/true/g' /etc/default/iscsitarget

* Restart the services::
   
   service iscsitarget start
   service open-iscsi start

* Configure /etc/cinder/api-paste.ini like the following::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   service_protocol = http
   service_host = 192.168.100.51
   service_port = 5000
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = cinder
   admin_password = servicePass123
   signing_dir = /var/lib/cinder

* Edit the /etc/cinder/cinder.conf to::

   [DEFAULT]
   rootwrap_config=/etc/cinder/rootwrap.conf
   sql_connection = mysql://cinderUser:cinderPass@10.10.10.51/cinder
   api_paste_config = /etc/cinder/api-paste.ini
   iscsi_helper=ietadm
   volume_name_template = volume-%s
   volume_group = cinder-volumes
   verbose = True
   auth_strategy = keystone
   iscsi_ip_address=10.10.10.51

* Then, synchronize your database::

   cinder-manage db sync

* Finally, don't forget to create a volumegroup and name it cinder-volumes

  TODO: volume groups should be created before this step

* Restart the cinder services::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i restart; done; cd -

* Verify if cinder services are running::

   cd /etc/init.d/; for i in $( ls cinder-* ); do sudo service $i status; done

2.11. Horizon
--------------

* To install horizon, proceed like this ::

   apt-get install -y openstack-dashboard memcached python-argparse
   Activate Dashboard and disable default VirtualHost? yes
   Should the Dashboard use HTTPS? yes


* Check OpenStack Dashboard at https://192.168.100.51 We can login with the admin / admin_pass


3. Network Node
================

3.1. Preparing the Node
------------------------

See chapter 2.1. Preparing Debian

* Enable IP_Forwarding::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   
   # To save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

3.2.Networking
---------------

* 3 NICs must be present::
   
   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 10.10.10.52
   netmask 255.255.255.0

   # VM Configuration
   auto eth1
   iface eth1 inet static
   address 10.20.20.52
   netmask 255.255.255.0

   # VM internet Access
   auto eth2
   iface eth2 inet static
   address 192.168.100.52
   netmask 255.255.255.0

3.4. OpenVSwitch (Part1)
-------------------------

* Install the openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* Create the bridges::

   #br-int will be used for VM integration	
   ovs-vsctl add-br br-int

   #br-ex is used to make to VM accessible from the internet
   ovs-vsctl add-br br-ex
   
   ovs-vsctl show

3.5. Neutron
------------------

* Install the Neutron openvswitch agent, l3 agent and dhcp agent::

   apt-get -y install neutron-plugin-openvswitch-agent neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent
   Name of the region to be used by the metadata server: RegionOne

* Edit /etc/neutron/api-paste.ini::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = servicePass123

* Edit the OVS plugin configuration file /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://neutronUser:neutronPass@10.10.10.51/neutron

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 10.20.20.52
   enable_tunneling = True

   #Firewall driver for realizing neutron security group function
   [SECURITYGROUP]
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* Update /etc/neutron/metadata_agent.ini::
   
   # The neutron user information for accessing the neutron API.
   auth_url = http://10.10.10.51:35357/v2.0
   auth_region = RegionOne
   admin_tenant_name = service
   admin_user = neutron
   admin_password = servicePass123

   # IP address used by Nova metadata server
   nova_metadata_ip = 10.10.10.51

   # TCP Port used by Nova metadata server
   nova_metadata_port = 8775

   metadata_proxy_shared_secret = helloOpenStack123

* Make sure that your rabbitMQ IP in /etc/neutron/neutron.conf is set to the controller node::

   rabbit_host = 10.10.10.51
   notification_driver = neutron.openstack.common.notifier.rabbit_notifier
   allow_overlapping_ips = True

   #And update the keystone_authtoken section

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = servicePass123
   signing_dir = /var/lib/neutron/keystone-signing
   
* Edit dhcp_agent.ini::

   interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver

* Edit /etc/sudoers.d/neutron_sudoers to give it full access like this (This is unfortunatly mandatory) ::

   visudo -f /etc/sudoers.d/neutron_sudoers
   
   #Modify the neutron user
   neutron ALL=NOPASSWD: ALL

* Restart all the services::

   cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i restart; done; cd -

3.4. OpenVSwitch (Part2)
-------------------------
* Edit the eth2 in /etc/network/interfaces to become like this::

   # VM internet Access
   auto eth2
   iface eth2 inet manual
      up ifconfig $IFACE 0.0.0.0 up
      up ip link set $IFACE promisc on
      down ip link set $IFACE promisc off
      down ifconfig $IFACE down

* Add the eth2 to the br-ex::

   #Internet connectivity will be lost after this step but this won't affect OpenStack's work
   ovs-vsctl add-port br-ex eth2

   Now check if the port eth1.950 was added to br-ex:
   ovs-vsctl show
   
   If you want to get internet connection back, you can assign the eth2 IP address to the br-ex interface manually::
   ifconfig br-ex 192.168.100.51 netmask 255.255.255.0 up
   
   To keep it permanent, add the following lines to /etc/rc.local::
   /sbin/ifconfig br-ex up
   /sbin/ip addr add 192.168.100.51/24 dev br-ex
   # optional # /sbin/ip route add default via 192.168.100.1
   
   Configuring this address in /etc/network/interfaces will not work, because openvswitch creates the br-ex interface and not brctl.

4. Compute Node
=========================

4.1. Preparing the Node
------------------------

See chapter 2.1. Preparing Debian

* Enable IP_Forwarding::

   sed -i 's/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/' /etc/sysctl.conf
   
   # To save you from rebooting, perform the following
   sysctl net.ipv4.ip_forward=1

4.2.Networking
---------------

* Perform the following::
   
   # OpenStack management
   auto eth0
   iface eth0 inet static
   address 10.10.10.53
   netmask 255.255.255.0

   # VM Configuration
   auto eth1
   iface eth1 inet static
   address 10.20.20.53
   netmask 255.255.255.0

4.3 KVM
------------------

* Install kvm::

   apt-get install -y kvm libvirt-bin pm-utils

* Edit the cgroup_device_acl array in the /etc/libvirt/qemu.conf file to::

   cgroup_device_acl = [
   "/dev/null", "/dev/full", "/dev/zero",
   "/dev/random", "/dev/urandom",
   "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
   "/dev/rtc", "/dev/hpet","/dev/net/tun"
   ]

* Delete default virtual bridge ::

   virsh net-destroy default
   virsh net-undefine default

* Enable live migration by updating /etc/libvirt/libvirtd.conf file::

   listen_tls = 0
   listen_tcp = 1
   auth_tcp = "none"

* Edit /etc/default/libvirt-bin file ::

   libvirtd_opts="-d -l"

* Restart the libvirt service and dbus to load the new values::

    service dbus restart && service libvirt-bin restart

4.4. OpenVSwitch
------------------

* Install the openVSwitch::

   apt-get install -y openvswitch-switch openvswitch-datapath-dkms

* Create the bridges::

   #br-int will be used for VM integration	
   ovs-vsctl add-br br-int

4.5. Neutron
------------------

* Install the Neutron openvswitch agent::

   apt-get -y install neutron-plugin-openvswitch-agent

* Edit the OVS plugin configuration file /etc/neutron/plugins/openvswitch/ovs_neutron_plugin.ini with:: 

   #Under the database section
   [DATABASE]
   sql_connection = mysql://neutronUser:neutronPass@10.10.10.51/neutron

   #Under the OVS section
   [OVS]
   tenant_network_type = gre
   tunnel_id_ranges = 1:1000
   integration_bridge = br-int
   tunnel_bridge = br-tun
   local_ip = 10.20.20.53
   enable_tunneling = True
   
   #Firewall driver for realizing neutron security group function
   [SECURITYGROUP]
   firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

* Make sure that your rabbitMQ IP in /etc/neutron/neutron.conf is set to the controller node::
   
   rabbit_host = 10.10.10.51

   #And update the keystone_authtoken section

   [keystone_authtoken]
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = neutron
   admin_password = servicePass123
   signing_dir = /var/lib/neutron/keystone-signing

* Restart all the services::

   service neutron-plugin-openvswitch-agent restart

4.6. Nova
------------------

* Install nova's required components for the compute node::

   apt-get install -y nova-compute-kvm
   Create or update supermin appliance now? no

* Now modify authtoken section in the /etc/nova/api-paste.ini file to this::

   [filter:authtoken]
   paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
   auth_host = 10.10.10.51
   auth_port = 35357
   auth_protocol = http
   admin_tenant_name = service
   admin_user = nova
   admin_password = servicePass123
   signing_dirname = /tmp/keystone-signing-nova
   # Workaround for https://bugs.launchpad.net/nova/+bug/1154809
   auth_version = v2.0

* Edit /etc/nova/nova-compute.conf file ::
   
   [DEFAULT]
   libvirt_type=kvm
   libvirt_ovs_bridge=br-int
   libvirt_vif_type=ethernet
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   libvirt_use_virtio_for_bridges=True

* Modify the /etc/nova/nova.conf like this::

   [DEFAULT] 
   logdir=/var/log/nova
   state_path=/var/lib/nova
   lock_path=/run/lock/nova
   verbose=True
   api_paste_config=/etc/nova/api-paste.ini
   compute_scheduler_driver=nova.scheduler.simple.SimpleScheduler
   rabbit_host=10.10.10.51
   nova_url=http://10.10.10.51:8774/v1.1/
   sql_connection=mysql://novaUser:novaPass@10.10.10.51/nova
   root_helper=sudo nova-rootwrap /etc/nova/rootwrap.conf

   # Auth
   use_deprecated_auth=false
   auth_strategy=keystone

   # Imaging service
   glance_api_servers=10.10.10.51:9292
   image_service=nova.image.glance.GlanceImageService

   # Vnc configuration
   novnc_enabled=true
   novncproxy_base_url=http://192.168.100.51:6080/vnc_auto.html
   novncproxy_port=6080
   vncserver_proxyclient_address=10.10.10.53
   vncserver_listen=0.0.0.0

   # Network settings
   network_api_class=nova.network.neutronv2.api.API
   neutron_url=http://10.10.10.51:9696
   neutron_auth_strategy=keystone
   neutron_admin_tenant_name=service
   neutron_admin_username=neutron
   neutron_admin_password=servicePass123
   neutron_admin_auth_url=http://10.10.10.51:35357/v2.0
   libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtHybridOVSBridgeDriver
   linuxnet_interface_driver=nova.network.linux_net.LinuxOVSInterfaceDriver
   #If you want Neutron + Nova Security groups
   firewall_driver=nova.virt.firewall.NoopFirewallDriver
   security_group_api=neutron
   #If you want Nova Security groups only, comment the two lines above and uncomment line -1-.
   #-1-firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
   
   #Metadata
   service_neutron_metadata_proxy = True
   neutron_metadata_proxy_shared_secret = helloOpenStack123

   # Compute #
   #compute_driver=libvirt.LibvirtDriver

   # Cinder #
   #volume_api_class=nova.volume.cinder.API
   #osapi_volume_listen_port=5900
   #cinder_catalog_info=volume:cinder:internalURL

* Restart nova-* services::

   cd /etc/init.d/; for i in $( ls nova-* ); do sudo service $i restart; done; cd -

* Check for the smiling faces on nova-* services to confirm your installation::

   nova-manage service list


5. Your first VM
================

To start your first VM, we first need to create a new tenant, user and internal network.
HINT: Replace <id> with the id of your command output:

* Create a new tenant ::

   keystone tenant-create --name project_one
   export put_id_of_project_one=<id>

* Create a new user and assign the member role to it in the new tenant (keystone role-list to get the appropriate id)::

   keystone user-create --name=user_one --pass=user_one --tenant-id $put_id_of_project_one --email=user_one@domain.com
   export put_id_of_user_one=<id>
   
* Get ID of member role::

   keystone role-list
   
   +----------------------------------+----------------------+
   |                id                |         name         |
   +----------------------------------+----------------------+
   | 50e331a2a9f2483e80515be0c8a66239 |    KeystoneAdmin     |
   | 8f8e6ff7bb3d407b825f69d325a90fdc | KeystoneServiceAdmin |
   | 594dad7afd8e473db41dfadcfc3c88c1 |        Member        |
   | 9fe2ff9ee4384b1894a90878d3e92bab |       _member_       |
   | 5e155772a7b446e2bc0dc5c884d8eae2 |        admin         |
   +----------------------------------+----------------------+
   
   We write down id of role „Member“.
   export put_id_of_member_role=594dad7afd8e473db41dfadcfc3c88c1

   Assign „Member“ role to new user in project_one:
   keystone user-role-add --tenant-id $put_id_of_project_one --user-id $put_id_of_user_one --role-id $put_id_of_member_role

* Create a new network for the tenant::

   neutron net-create --tenant-id $put_id_of_project_one net_proj_one 

* Create a new subnet inside the new tenant network::

   neutron subnet-create --tenant-id $put_id_of_project_one net_proj_one 50.50.1.0/24 --dns_nameservers list=true 8.8.8.7 8.8.8.8
   export put_subnet_id_here=<id>
   
* Create a router for the new tenant::

   neutron router-create --tenant-id $put_id_of_project_one router_proj_one
   export put_router_proj_one_id_here=<id>
   
* Add the router to the running l3 agent (if it wasn't automatically added)::

   neutron agent-list (to get the l3 agent ID)
   export l3_agent_ID=<id>
   neutron l3-agent-router-add $l3_agent_ID router_proj_one

* Add the router to the subnet::

   neutron router-interface-add $put_router_proj_one_id_here $put_subnet_id_here

* Restart all neutron services::

   cd /etc/init.d/; for i in $( ls neutron-* ); do sudo service $i restart; done; cd -

* Create an external network with the tenant id belonging to the admin tenant (keystone tenant-list to get the appropriate id)::

   keystone tenant-list
   export put_id_of_admin_tenant=<id>
   neutron net-create --tenant-id $put_id_of_admin_tenant ext_net --router:external=True
   export put_id_of_ext_net_here=<id>

* Create a subnet for the floating ips::

   neutron subnet-create --tenant-id $put_id_of_admin_tenant --allocation-pool start=192.168.100.102,end=192.168.100.126 --gateway 192.168.100.1 ext_net 192.168.100.0/24 --enable_dhcp=False

* Set your router's gateway to the external network:: 

   neutron router-gateway-set $put_router_proj_one_id_here $put_id_of_ext_net_here

* Source creds relative to your project one tenant now::

   vi creds_proj_one

   #Paste the following:
   export OS_TENANT_NAME=project_one
   export OS_USERNAME=user_one
   export OS_PASSWORD=user_one
   export OS_AUTH_URL="http://192.168.100.51:5000/v2.0/"

   source creds_proj_one

* Add this security rules to make your VMs pingable::

   nova --no-cache secgroup-add-rule default icmp -1 -1 0.0.0.0/0
   nova --no-cache secgroup-add-rule default tcp 22 22 0.0.0.0/0

* Start by allocating a floating ip to the project one tenant::

   neutron floatingip-create ext_net
   export put_id_floating_ip=<id>

* Start a VM::

   nova --no-cache boot --image myFirstImage --flavor 1 my_first_vm 

* Pick the id of the port corresponding to your VM::

   neutron port-list
   
   +--------------------------------------+-----+-------------------+-----------------------------------------------+
   | id                                   | name| mac_address       | fixed_ips                                     |
   +--------------------------------------+-----+-------------------+-----------------------------------------------+
   | 006d41bc-a3a3-4ff0-aa32-c23d69e6d2c8 |     | fa:16:3e:0b:e7:34 | {"subnet_id": "", "ip_address": "50.50.1.2"}  |
   | a8b2d1d8-3dad-41c9-8041-63f151c3d091 |     | fa:16:3e:fc:99:64 | {"subnet_id": "", "ip_address": "50.50.1.1"}  |
   | f217fea3-217b-45fc-a2d4-bcd394666bfc |     | fa:16:3e:66:49:25 | {"subnet_id": "", "ip_address": "50.50.1.3"}  |
   +--------------------------------------+-----+-------------------+-----------------------------------------------+

   The first entry is our DHCP Server, the second entry is our router and the third entry is our first vm.
   (I removed the subnet id to fit the screen.)
   export put_id_vm_port=<id> # f217fea3-217b-45fc-a2d4-bcd394666bfc in my case.
   
* Associate the floating IP to your VM::

   neutron floatingip-associate $put_id_floating_ip $put_id_vm_port

That's it! ping your VM and enjoy OpenStack.

6. Licensing
============

OpenStack Install Guide is licensed under a Creative Commons Attribution 3.0 Unported License.

To view a copy of this license, visit [ http://creativecommons.org/licenses/by/3.0/deed.en_US ].

7. Contact
===========

L\. Reusser : reusserl@users.noreply.github.com

8. Credits
=================

This work has been based on:

* OpenStack Grizzly Install Guide (Master Branch) [https://github.com/mseknibilel/OpenStack-Grizzly-Install-Guide]
* OpenStack Installation Guide for Debian 7.0 (Wheezy) [http://docs.openstack.org/havana/install-guide/install/apt-debian/content/index.html]

9. To do
========

Your suggestions are always welcomed.





