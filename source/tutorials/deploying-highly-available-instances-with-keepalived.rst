####################################################
Deploying Highly Available instances with Keepalived
####################################################

This tutorial assumes you have installed the OpenStack command line tools and
sourced an openrc file, as explained at :ref:`command-line-tools`. We also
assume that you have uploaded a ssh key as explained at
:ref:`uploading-an-ssh-key`.


Introduction
============

In this tutorial you will learn how to deploy a highly available instance pair
using VRRP. This tutorial is largely based from a `blog post`_ by Aaron O'Rosen
with modifications appropriate for Catalysts cloud. Networks and names have
been kept largely compatible with the source material. Additionally information
about configuring ``allowed_address_pairs`` in heat was sourced from this
`post`_.

.. _blog post: http://blog.aaronorosen.com/implementing-high-availability-instances-with-neutron-using-vrrp/

.. _post: https://www.hastexo.com/blogs/syed/2014/08/05/orchestrating-highly-available-load-balancers-openstack-heat


We will be using two different methods to setup this stack. Initially we will
use the ``neutron`` and ``nova``  command line tools to complete the setup
manually. We will then replicate the manual configuration using a ``heat``
template to instantiate the same stack automatically.

Virtual Router Redundancy Protocol
==================================
`VRRP`_ provides hardware redundancy and automatic failover for routers. It
allows specifying a virtual router which maps to two or more physical routers.
Individual VRRP router instances share an IP address but at any time, only one
of the instances is the master (active), the other instances are backups and
will not respond using the virtual address. If the master fails, one of the
backups is elected as the new master and will begin to respond on the virtual
address.

Instances use priorities from 1 (lowest) through 255 (highest), devices running
VRRP dynamically elect master and backup routers based on their respective
priorities. Only the router that is acting as the master sends out VRRP
advertisements at any given point in time. The master router sends
advertisements to backup routers at regular intervals (default 1 second). If a
backup router does not receive an advertisement for a set period, the backup
router with the next highest priority takes over as master and begins
forwarding packets.

VRRP instances communicate using packets with multicast IP address 224.0.0.18
and IP protocol number 112. The protocol is defined in `RFC3768`_.

.. _VRRP: https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol

.. _RFC3768: https://en.wikipedia.org/wiki/Virtual_Router_Redundancy_Protocol

.. note::

 There is an extension to VRRP that uses IPSEC-AH (IP protocol 51) for
 integrity (see http://www.keepalived.org/draft-ietf-vrrp-ipsecah-spec-00.txt).
 This tutorial will demostrate using standard VRRP. See this `article`_ for
 more information on securing VRRP.

.. _article: http://louwrentius.com/configuring-attacking-and-securing-vrrp-on-linux.html


Allowed Address Pairs
=====================

Allowed Address Pairs is a Neutron Extension that extends the port attribute to
enable you to specify arbitrary ``mac_address/ip_address(cidr)`` pairs that are
allowed to pass through a port regardless of the subnet associated with the
network.

Let's double check that this extension is available on the Catalyst Cloud:

.. code-block:: bash

 $ neutron ext-list
 +-----------------------+-----------------------------------------------+
 | alias                 | name                                          |
 +-----------------------+-----------------------------------------------+
 | service-type          | Neutron Service Type Management               |
 | security-group        | security-group                                |
 | l3_agent_scheduler    | L3 Agent Scheduler                            |
 | ext-gw-mode           | Neutron L3 Configurable external gateway mode |
 | binding               | Port Binding                                  |
 | metering              | Neutron Metering                              |
 | agent                 | agent                                         |
 | quotas                | Quota management support                      |
 | dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
 | multi-provider        | Multi Provider Network                        |
 | external-net          | Neutron external network                      |
 | router                | Neutron L3 Router                             |
 | allowed-address-pairs | Allowed Address Pairs                         |
 | vpnaas                | VPN service                                   |
 | extra_dhcp_opt        | Neutron Extra DHCP opts                       |
 | provider              | Provider Network                              |
 | extraroute            | Neutron Extra Route                           |
 +-----------------------+-----------------------------------------------+

As you can see, the Allowed Address Pairs extension is available.

.. _clone-orchestration-repo:

Clone Orchestration Git Repository
==================================

Before we start let's checkout the
https://github.com/catalyst/catalystcloud-orchestration git repository. We will
be using some scripts and heat templates from this repository in this tutorial.

.. code-block:: bash

 $ git clone https://github.com/catalyst/catalystcloud-orchestration.git && ORCHESTRATION_DIR="$(pwd)/catalystcloud-orchestration" && echo $ORCHESTRATION_DIR

Network Setup
=============

Let's create a network called ``vrrp-net`` where we can run our highly
available hosts:

.. code-block:: bash

 $ neutron net-create vrrp-net
 Created a new network:
 +----------------+--------------------------------------+
 | Field          | Value                                |
 +----------------+--------------------------------------+
 | admin_state_up | True                                 |
 | id             | 617ff618-9da6-4c47-ab3f-527fe5413ea8 |
 | name           | vrrp-net                             |
 | shared         | False                                |
 | status         | ACTIVE                               |
 | subnets        |                                      |
 | tenant_id      | 0cb6b9b744594a619b0b7340f424858b     |
 +----------------+--------------------------------------+

Now let's set up a subnet of the network we have just created. We are going to
do this so we can use part of the ``vrrp-net`` as a dynamically assigned pool
of addresses and reserve the rest of the addresses for manual assignment. In
this case the pool addresses are in the range 2-200 while the remainder of the
``/24`` will be statically assigned.

.. code-block:: bash

 $ neutron subnet-create --name vrrp-subnet --allocation-pool \
   start=10.0.0.2,end=10.0.0.200 vrrp-net 10.0.0.0/24
 Created a new subnet:
 +------------------+--------------------------------------------+
 | Field            | Value                                      |
 +------------------+--------------------------------------------+
 | allocation_pools | {"start": "10.0.0.2", "end": "10.0.0.200"} |
 | cidr             | 10.0.0.0/24                                |
 | dns_nameservers  |                                            |
 | enable_dhcp      | True                                       |
 | gateway_ip       | 10.0.0.1                                   |
 | host_routes      |                                            |
 | id               | 7c3ca3d4-70a2-4fdd-be9e-4b6bd1eef537       |
 | ip_version       | 4                                          |
 | name             | vrrp-subnet                                |
 | network_id       | 617ff618-9da6-4c47-ab3f-527fe5413ea8       |
 | tenant_id        | 0cb6b9b744594a619b0b7340f424858b           |
 +------------------+--------------------------------------------+

Next we will create a router, we will give this router an interface on our new
subnet and we will set its gateway as our public network:

.. code-block:: bash

 $ neutron router-create vrrp-router
 Created a new router:
 +-----------------------+--------------------------------------+
 | Field                 | Value                                |
 +-----------------------+--------------------------------------+
 | admin_state_up        | True                                 |
 | external_gateway_info |                                      |
 | id                    | 8e9df7a5-0d5a-4574-bbbe-b4db35616efa |
 | name                  | vrrp-router                          |
 | status                | ACTIVE                               |
 | tenant_id             | 0cb6b9b744594a619b0b7340f424858b     |
 +-----------------------+--------------------------------------+

 $ neutron router-interface-add vrrp-router vrrp-subnet
 Added interface 7e11450c-b605-4931-a304-0d864e205ed2 to router vrrp-router.

 $ neutron router-gateway-set vrrp-router public-net
 Set gateway for router vrrp-router

.. note::

 If you look at the ports created at this point using the ``neutron port-list`` command you will notice three interfaces have been created. The ip 10.0.0.1 is the gateway address while 10.0.0.2 and 10.0.0.3 provide DHCP for this network.


Security Group Setup
====================

Now we will create the ``vrrp-sec-group`` security group with rules to
allow http, ssh and icmp ingres:

.. code-block:: bash

 $ neutron security-group-create vrrp-sec-group
 Created a new security_group:
 +----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | Field                | Value                                                                                                                                                                                                                                                                                                                         |
 +----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
 | description          |                                                                                                                                                                                                                                                                                                                               |
 | id                   | 3d50882c-c8b8-4c39-9758-390593a5774b                                                                                                                                                                                                                                                                                          |
 | name                 | vrrp-sec-group                                                                                                                                                                                                                                                                                                                |
 | security_group_rules | {"remote_group_id": null, "direction": "egress", "remote_ip_prefix": null, "protocol": null, "tenant_id": "0cb6b9b744594a619b0b7340f424858b", "port_range_max": null, "security_group_id": "3d50882c-c8b8-4c39-9758-390593a5774b", "port_range_min": null, "ethertype": "IPv4", "id": "33d9bf4b-03a2-4169-a47d-1116345d9e1d"} |
 |                      | {"remote_group_id": null, "direction": "egress", "remote_ip_prefix": null, "protocol": null, "tenant_id": "0cb6b9b744594a619b0b7340f424858b", "port_range_max": null, "security_group_id": "3d50882c-c8b8-4c39-9758-390593a5774b", "port_range_min": null, "ethertype": "IPv6", "id": "2e192759-871c-449f-ab67-cc9f03ed2f35"} |
 | tenant_id            | 0cb6b9b744594a619b0b7340f424858b                                                                                                                                                                                                                                                                                              |
 +----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

 $ neutron security-group-rule-create --protocol icmp vrrp-sec-group
 Created a new security_group_rule:
 +-------------------+--------------------------------------+
 | Field             | Value                                |
 +-------------------+--------------------------------------+
 | direction         | ingress                              |
 | ethertype         | IPv4                                 |
 | id                | 9ddcc056-0915-4365-a303-a5a1d691c87e |
 | port_range_max    |                                      |
 | port_range_min    |                                      |
 | protocol          | icmp                                 |
 | remote_group_id   |                                      |
 | remote_ip_prefix  |                                      |
 | security_group_id | 3d50882c-c8b8-4c39-9758-390593a5774b |
 | tenant_id         | 0cb6b9b744594a619b0b7340f424858b     |
 +-------------------+--------------------------------------+

 $ neutron security-group-rule-create --protocol tcp --port-range-min 80 --port-range-max 80 vrrp-sec-group
 Created a new security_group_rule:
 +-------------------+--------------------------------------+
 | Field             | Value                                |
 +-------------------+--------------------------------------+
 | direction         | ingress                              |
 | ethertype         | IPv4                                 |
 | id                | 55cbfd57-03c5-4ed8-a760-33453b447669 |
 | port_range_max    | 80                                   |
 | port_range_min    | 80                                   |
 | protocol          | tcp                                  |
 | remote_group_id   |                                      |
 | remote_ip_prefix  |                                      |
 | security_group_id | 3d50882c-c8b8-4c39-9758-390593a5774b |
 | tenant_id         | 0cb6b9b744594a619b0b7340f424858b     |
 +-------------------+--------------------------------------+

 $ neutron security-group-rule-create --protocol tcp --port-range-min 22 --port-range-max 22 vrrp-sec-group
 Created a new security_group_rule:
 +-------------------+--------------------------------------+
 | Field             | Value                                |
 +-------------------+--------------------------------------+
 | direction         | ingress                              |
 | ethertype         | IPv4                                 |
 | id                | e9c0d635-e1bb-498d-8bd2-64e4a4d553c3 |
 | port_range_max    | 22                                   |
 | port_range_min    | 22                                   |
 | protocol          | tcp                                  |
 | remote_group_id   |                                      |
 | remote_ip_prefix  |                                      |
 | security_group_id | 3d50882c-c8b8-4c39-9758-390593a5774b |
 | tenant_id         | 0cb6b9b744594a619b0b7340f424858b     |
 +-------------------+--------------------------------------+

Next we will add a rule to allow our Keepalived instances to communicate with
each other via VRRP broadcasts:

.. code-block:: bash

 $ neutron security-group-rule-create --protocol 112 --remote-group-id vrrp-sec-group vrrp-sec-group
 Created a new security_group_rule:
 +-------------------+--------------------------------------+
 | Field             | Value                                |
 +-------------------+--------------------------------------+
 | direction         | ingress                              |
 | ethertype         | IPv4                                 |
 | id                | 2c10b6fd-5729-480d-a4f8-88fe1286dceb |
 | port_range_max    |                                      |
 | port_range_min    |                                      |
 | protocol          | 112                                  |
 | remote_group_id   | 3d50882c-c8b8-4c39-9758-390593a5774b |
 | remote_ip_prefix  |                                      |
 | security_group_id | 3d50882c-c8b8-4c39-9758-390593a5774b |
 | tenant_id         | 0cb6b9b744594a619b0b7340f424858b     |
 +-------------------+--------------------------------------+

Instance Creation
=================

The next step is to boot two instances where we will run Keepalived and Apache.
We will be using the Ubuntu 14.04 image and ``c1.c1r1`` flavour. We will assign
these instances to the ``vrrp-sec-group`` security group. We will also provide
the name of our SSH key so we can login to these machines via SSH once they are
created:

.. note::
 You will need to substitute the name of your SSH key.

To find the correct IDs you can use the following commands:

.. code-block:: bash

 $ VRRP_IMAGE_ID=$(glance image-show ubuntu-14.04-x86_64 | grep ' id '| awk '{ print $4 }') && echo $VRRP_IMAGE_ID
 9eab2d64-818c-4548-980d-535412d16249

 $ VRRP_FLAVOR_ID=$(nova flavor-list | grep 'c1.c1r1' | awk '{ print $2 }') && echo $VRRP_FLAVOR_ID
 28153197-6690-4485-9dbc-fc24489b0683

 $ VRRP_NET_ID=$(neutron net-show vrrp-net | grep ' id '| awk '{ print $4 }') && echo $VRRP_NET_ID
 617ff618-9da6-4c47-ab3f-527fe5413ea8

 $ nova keypair-list
 +------------------+-------------------------------------------------+
 | Name             | Fingerprint                                     |
 +------------------+-------------------------------------------------+
 | vrrp-demo-key    | 9a:17:a8:1f:48:a4:f4:0d:c8:1b:ee:de:d4:a1:60:0b |
 +------------------+-------------------------------------------------+

We will be passing a script to our instance boot command using the
``--user-data`` flag. This script sets up Keepalived and Apache on our master
and backup instances. This saves us having to execute these commands manually.
This script is located in the git repository you cloned previously at
:ref:`clone-orchestration-repo`.

.. code-block:: bash

 $ cat "$ORCHESTRATION_DIR/hot/ubuntu-14.04/vrrp-basic/vrrp-setup.sh"
 #!/bin/bash

 HOSTNAME=$(hostname)

 if [ "$HOSTNAME" == "vrrp-master" ]; then
     KEEPALIVED_STATE='MASTER'
     KEEPALIVED_PRIORITY=100
 elif [ "$HOSTNAME" == "vrrp-backup" ]; then
     KEEPALIVED_STATE='BACKUP'
     KEEPALIVED_PRIORITY=50
 else
     echo "invalid hostname $HOSTNAME for install script $0";
     exit 1;
 fi

 IP=$(ip addr | grep inet | grep eth0 | grep -v secondary | awk '{ print $2 }' | awk -F'/' '{ print $1 }')

 echo "$IP $HOSTNAME" >> /etc/hosts

 apt-get update
 apt-get -y install keepalived

 echo "vrrp_instance vrrp_group_1 {
     state $KEEPALIVED_STATE
     interface eth0
     virtual_router_id 1
     priority $KEEPALIVED_PRIORITY
     authentication {
         auth_type PASS
         auth_pass password
     }
     virtual_ipaddress {
         10.0.0.201/24 brd 10.0.0.255 dev eth0
     }
 }" > /etc/keepalived/keepalived.conf

 apt-get -y install apache2
 echo "$HOSTNAME" > /var/www/html/index.html
 service keepalived restart

Let's run the boot command (you will need to substitute your SSH key name and
path to the ``vrrp-setup.sh`` script):

.. code-block:: bash

 $ nova boot --image $VRRP_IMAGE_ID --flavor $VRRP_FLAVOR_ID --nic net-id=$VRRP_NET_ID --security_groups \
   vrrp-sec-group --user-data vrrp-setup.sh --key_name vrrp-demo-key vrrp-master

 +--------------------------------------+------------------------------------------------------------+
 | Property                             | Value                                                      |
 +--------------------------------------+------------------------------------------------------------+
 | OS-DCF:diskConfig                    | MANUAL                                                     |
 | OS-EXT-AZ:availability_zone          | nz-por-1a                                                  |
 | OS-EXT-STS:power_state               | 0                                                          |
 | OS-EXT-STS:task_state                | scheduling                                                 |
 | OS-EXT-STS:vm_state                  | building                                                   |
 | OS-SRV-USG:launched_at               | -                                                          |
 | OS-SRV-USG:terminated_at             | -                                                          |
 | accessIPv4                           |                                                            |
 | accessIPv6                           |                                                            |
 | adminPass                            | p7GmoGyK2HDP                                               |
 | config_drive                         |                                                            |
 | created                              | 2015-08-26T03:57:15Z                                       |
 | flavor                               | c1.c1r1 (28153197-6690-4485-9dbc-fc24489b0683)             |
 | hostId                               |                                                            |
 | id                                   | ebd4b72f-6fcf-4e1d-ad7d-507b944f86df                       |
 | image                                | ubuntu-14.04-x86_64 (9eab2d64-818c-4548-980d-535412d16249) |
 | key_name                             | vrrp-demo-key                                              |
 | metadata                             | {}                                                         |
 | name                                 | vrrp-master                                                |
 | os-extended-volumes:volumes_attached | []                                                         |
 | progress                             | 0                                                          |
 | security_groups                      | vrrp-sec-group                                             |
 | status                               | BUILD                                                      |
 | tenant_id                            | 0cb6b9b744594a619b0b7340f424858b                           |
 | updated                              | 2015-08-26T03:57:15Z                                       |
 | user_id                              | 8c1914eda99d406195674864f2846d45                           |
 +--------------------------------------+------------------------------------------------------------+

 $ nova boot --image $VRRP_IMAGE_ID --flavor $VRRP_FLAVOR_ID --nic net-id=$VRRP_NET_ID --security_groups \
   vrrp-sec-group --user-data vrrp-setup.sh --key_name vrrp-demo-key vrrp-backup

Let's check the instances have been created:

.. code-block:: bash

 $ nova list
 +--------------------------------------+-------------+--------+------------+-------------+-------------------+
 | ID                                   | Name        | Status | Task State | Power State | Networks          |
 +--------------------------------------+-------------+--------+------------+-------------+-------------------+
 | ebd4b72f-6fcf-4e1d-ad7d-507b944f86df | vrrp-master | ACTIVE | -          | Running     | vrrp-net=10.0.0.4 |
 | f980dc30-9d3e-4e47-adf5-8f6715be6a8a | vrrp-backup | ACTIVE | -          | Running     | vrrp-net=10.0.0.5 |
 +--------------------------------------+-------------+--------+------------+-------------+-------------------+

Virtual Address Setup
=====================

The next step is to create the IP address that will be used by our virtual
router:

.. code-block:: bash

 $ neutron port-create --fixed-ip ip_address=10.0.0.201 --security-group vrrp-sec-group vrrp-net
 Created a new port:
 +-----------------------+-----------------------------------------------------------------------------------+
 | Field                 | Value                                                                             |
 +-----------------------+-----------------------------------------------------------------------------------+
 | admin_state_up        | True                                                                              |
 | allowed_address_pairs |                                                                                   |
 | binding:vnic_type     | normal                                                                            |
 | device_id             |                                                                                   |
 | device_owner          |                                                                                   |
 | fixed_ips             | {"subnet_id": "7c3ca3d4-70a2-4fdd-be9e-4b6bd1eef537", "ip_address": "10.0.0.201"} |
 | id                    | 40aa1a50-4a96-4103-beaf-89bdb0b49327                                              |
 | mac_address           | fa:16:3e:40:69:5f                                                                 |
 | name                  |                                                                                   |
 | network_id            | 617ff618-9da6-4c47-ab3f-527fe5413ea8                                              |
 | security_groups       | 3d50882c-c8b8-4c39-9758-390593a5774b                                              |
 | status                | DOWN                                                                              |
 | tenant_id             | 0cb6b9b744594a619b0b7340f424858b                                                  |
 +-----------------------+-----------------------------------------------------------------------------------+

Now we need to create a floating IP and point it to our virtual router IP using
its port ID:

.. code-block:: bash

 $ VRRP_VR_PORT_ID=$(neutron port-list | grep '10.0.0.201' | awk '{ print $2 }') && echo $VRRP_VR_PORT_ID
 40aa1a50-4a96-4103-beaf-89bdb0b49327

 $ neutron floatingip-create --port-id=$VRRP_VR_PORT_ID public-net
 Created a new floatingip:
 +---------------------+--------------------------------------+
 | Field               | Value                                |
 +---------------------+--------------------------------------+
 | fixed_ip_address    | 10.0.0.201                           |
 | floating_ip_address | 150.242.40.102                       |
 | floating_network_id | 849ab1e9-7ac5-4618-8801-e6176fbbcf30 |
 | id                  | 1247fd9d-af4b-448b-9635-51b7a71f56ad |
 | port_id             | 40aa1a50-4a96-4103-beaf-89bdb0b49327 |
 | router_id           | 8e9df7a5-0d5a-4574-bbbe-b4db35616efa |
 | status              | DOWN                                 |
 | tenant_id           | 0cb6b9b744594a619b0b7340f424858b     |
 +---------------------+--------------------------------------+

Next up we update the ports associated with each instance to allow the virtual
router IP as an ``allowed-address-pair``. This will allow them to send traffic
using this address.

.. code-block:: bash

 $ VRRP_MASTER_PORT=$(neutron port-list -- --network_id=$VRRP_NET_ID | grep '10.0.0.4' | awk '{ print $2 }') && echo $VRRP_MASTER_PORT
 8f1997e4-fd12-41df-9fb9-d4605e5157d8

 $ VRRP_BACKUP_PORT=$(neutron port-list -- --network_id=$VRRP_NET_ID | grep '10.0.0.5' | awk '{ print $2 }') && echo $VRRP_BACKUP_PORT
 1736183d-8beb-4131-bb60-eb447bcb18f4

 $ neutron port-update $VRRP_MASTER_PORT --allowed_address_pairs list=true type=dict ip_address=10.0.0.201
 Updated port: 8f1997e4-fd12-41df-9fb9-d4605e5157d8

 $ neutron port-update $VRRP_BACKUP_PORT --allowed_address_pairs list=true type=dict ip_address=10.0.0.201
 Updated port: 1736183d-8beb-4131-bb60-eb447bcb18f4

Check that the virtual router address is associated with this port under
``allowed_address_pairs``:

.. code-block:: bash

 $ neutron port-show $VRRP_MASTER_PORT
 +-----------------------+---------------------------------------------------------------------------------+
 | Field                 | Value                                                                           |
 +-----------------------+---------------------------------------------------------------------------------+
 | admin_state_up        | True                                                                            |
 | allowed_address_pairs | {"ip_address": "10.0.0.201", "mac_address": "fa:16:3e:f7:af:bf"}                |
 | binding:vnic_type     | normal                                                                          |
 | device_id             | ebd4b72f-6fcf-4e1d-ad7d-507b944f86df                                            |
 | device_owner          | compute:nz-por-1a                                                               |
 | extra_dhcp_opts       |                                                                                 |
 | fixed_ips             | {"subnet_id": "7c3ca3d4-70a2-4fdd-be9e-4b6bd1eef537", "ip_address": "10.0.0.4"} |
 | id                    | 8f1997e4-fd12-41df-9fb9-d4605e5157d8                                            |
 | mac_address           | fa:16:3e:f7:af:bf                                                               |
 | name                  |                                                                                 |
 | network_id            | 617ff618-9da6-4c47-ab3f-527fe5413ea8                                            |
 | security_groups       | 3d50882c-c8b8-4c39-9758-390593a5774b                                            |
 | status                | ACTIVE                                                                          |
 | tenant_id             | 0cb6b9b744594a619b0b7340f424858b                                                |
 +-----------------------+---------------------------------------------------------------------------------+

We should now have a stack that looks something like this:

.. image:: _static/vrrp-network.png
   :align: center


.. _vrrp-testing:

VRRP Testing
============

We should now have a working VRRP setup so let's try it out! We should be able
to curl the floating IP associated with our virtual router:

.. code-block:: bash

 $ VRRP_FLOATING_IP=$(neutron floatingip-list | grep 10.0.0.201 | awk '{ print $6 }') && echo $VRRP_FLOATING_IP
 150.242.40.121
 $ curl $VRRP_FLOATING_IP
 vrrp-master

As you can see we are hitting the master instance. Let's take down the port the
virtual router address is configured on on the master to test that we failover
to the backup:

.. code-block:: bash

 $ neutron port-update $VRRP_MASTER_PORT --admin_state_up=False
 Updated port: 8f1997e4-fd12-41df-9fb9-d4605e5157d8

Curl again:

.. code-block:: bash

 $ curl $VRRP_FLOATING_IP
 vrrp-backup

.. _instance-access:

Instance Access
===============

If we want to take a closer look at what is happening when we switch between
VRRP hosts we need to SSH to the instances. We won't use the floating IP
associated with our virtual router, as that will be switching between instances
which will make our SSH client unhappy. Consequently, we will assign a floating
IP to each instance for SSH access.

.. code-block:: bash

 $ neutron floatingip-create --port-id=$VRRP_MASTER_PORT public-net
 Created a new floatingip:
 +---------------------+--------------------------------------+
 | Field               | Value                                |
 +---------------------+--------------------------------------+
 | fixed_ip_address    | 10.0.0.4                             |
 | floating_ip_address | 150.242.40.110                       |
 | floating_network_id | 849ab1e9-7ac5-4618-8801-e6176fbbcf30 |
 | id                  | e411608f-7548-45a5-98e5-d1f55b92a350 |
 | port_id             | 8f1997e4-fd12-41df-9fb9-d4605e5157d8 |
 | router_id           | 8e9df7a5-0d5a-4574-bbbe-b4db35616efa |
 | status              | DOWN                                 |
 | tenant_id           | 0cb6b9b744594a619b0b7340f424858b     |
 +---------------------+--------------------------------------+

 $ neutron floatingip-create --port-id=$VRRP_BACKUP_PORT public-net
 Created a new floatingip:
 +---------------------+--------------------------------------+
 | Field               | Value                                |
 +---------------------+--------------------------------------+
 | fixed_ip_address    | 10.0.0.5                             |
 | floating_ip_address | 150.242.40.112                       |
 | floating_network_id | 849ab1e9-7ac5-4618-8801-e6176fbbcf30 |
 | id                  | 72e3d549-b3e8-432d-b8af-f48c32268082 |
 | port_id             | 1736183d-8beb-4131-bb60-eb447bcb18f4 |
 | router_id           | 8e9df7a5-0d5a-4574-bbbe-b4db35616efa |
 | status              | DOWN                                 |
 | tenant_id           | 0cb6b9b744594a619b0b7340f424858b     |
 +---------------------+--------------------------------------+

Now we can SSH to our instances. We will connect using the default ``ubuntu``
user that is configured on Ubuntu cloud images. You will need to substitute the
correct floating IP address.

You can tail syslog in order to see what keepalived is doing. For example, here
we can see the backup instance switch from backup to master state:

.. code-block:: bash

 $ tail -f /var/log/syslog
 Aug 26 05:17:47 vrrp-backup kernel: [ 4807.732605] IPVS: ipvs loaded.
 Aug 26 05:17:47 vrrp-backup Keepalived_vrrp[2980]: Opening file '/etc/keepalived/keepalived.conf'.
 Aug 26 05:17:47 vrrp-backup Keepalived_vrrp[2980]: Configuration is using : 60109 Bytes
 Aug 26 05:17:47 vrrp-backup Keepalived_healthcheckers[2979]: Opening file '/etc/keepalived/keepalived.conf'.
 Aug 26 05:17:47 vrrp-backup Keepalived_healthcheckers[2979]: Configuration is using : 4408 Bytes
 Aug 26 05:17:47 vrrp-backup Keepalived_vrrp[2980]: Using LinkWatch kernel netlink reflector...
 Aug 26 05:17:47 vrrp-backup Keepalived_vrrp[2980]: VRRP_Instance(vrrp_group_1) Entering BACKUP STATE
 Aug 26 05:17:47 vrrp-backup Keepalived_healthcheckers[2979]: Using LinkWatch kernel netlink reflector...
 Aug 26 05:22:21 vrrp-backup Keepalived_vrrp[2980]: VRRP_Instance(vrrp_group_1) Transition to MASTER STATE
 Aug 26 05:22:22 vrrp-backup Keepalived_vrrp[2980]: VRRP_Instance(vrrp_group_1) Entering MASTER STATE

You can also watch the VRRP traffic on the wire with this command:

.. code-block:: bash

 $ sudo tcpdump -n -i eth0 proto 112
 05:28:23.651795 IP 10.0.0.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype simple, intvl 1s, length 20
 05:28:24.652909 IP 10.0.0.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype simple, intvl 1s, length 20

You can see the VRRP advertisements every second.

If you bring the master port back up at this point you will be able to see the
master node switch from the backup instance to the master instance:

.. code-block:: bash

 $ neutron port-update $VRRP_MASTER_PORT --admin_state_up=True
 Updated port: 8f1997e4-fd12-41df-9fb9-d4605e5157d8

on ``vrrp-backup``:

.. code-block:: bash

 $ sudo tcpdump -n -i eth0 proto 112
 05:30:11.773655 IP 10.0.0.5 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 50, authtype simple, intvl 1s, length 20
 05:30:11.774311 IP 10.0.0.4 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 100, authtype simple, intvl 1s, length 20
 05:30:12.775156 IP 10.0.0.4 > 224.0.0.18: VRRPv2, Advertisement, vrid 1, prio 100, authtype simple, intvl 1s, length 20

At this point we have successfully setup Keepalived with automatic failover
between instances. If this is all that you require for your setup so you can
stop here.

Resource Cleanup
================

At this point many people will want to cleanup the OpenStack resources we have
been using in this tutorial. Running the following commands should remove all
networks, routers, ports, security groups and instances. Note that the order
you delete resources is important.

.. code-block:: bash

 # delete the instances
 $ nova delete vrrp-master
 $ nova delete vrrp-backup

 # delete instance ports
 $ for port_id in $(neutron port-list | grep 10.0.0 | grep -v 10.0.0.1 | awk '{ print $2 }'); do neutron port-delete $port_id; done

 # delete router interface
 $ neutron router-interface-delete vrrp-router $(neutron subnet-list | grep vrrp-subnet | awk '{ print $2 }')
 Removed interface from router vrrp-router.

 # delete router
 $ neutron router-delete vrrp-router
 Deleted router: vrrp-router

 # delete subnet
 $ neutron subnet-delete vrrp-subnet
 Deleted subnet: vrrp-subnet

 # delete network
 $ neutron net-delete vrrp-net
 Deleted network: vrrp-net

 # delete security group
 $ neutron security-group-delete vrrp-sec-group
 Deleted security_group: vrrp-sec-group


Setup Using HEAT Templates
==========================

Up to this point in this tutorial we have been using the Nova and Neutron
command line clients to setup our system. We have needed to run a large number
of different commands in the right order. It would be nice if we could define
the entire setup in one configuration file and ask OpenStack to create that
setup based on our blueprint.

OpenStack provides just such an orchestration system which is known as heat. In
this section we will run heat in order to recreate the stack we have created
manually using a single command.

It is beyond the scope of this tutorial to explain the syntax of writing heat
templates, thus we will make use of a predefined example from the
cloud-orchestration repository. For more information on writing heat templates
please consult the documentation at :ref:`cloud-orchestration`

That said, there are a number of parts of the heat template we should have a
look at in more detail. The template is located in the
``catalystcloud-orchestration`` repository we cloned earlier.

.. code-block:: bash

 $ cat "$ORCHESTRATION_DIR/hot/ubuntu-14.04/vrrp-basic/vrrp.yaml"

The first thing to note is the Security Group rule for VRRP traffic:

.. code-block:: yaml

 - direction: ingress
   protocol: 112
   remote_group_id:
   remote_mode: remote_group_id

Note that the ``remote_mode`` is set to ``remote_group_id`` and
``remote_group_id`` is not set. If no value is set then the rule uses the
current security group (`heat documentation`_).

.. _heat documentation: http://docs.openstack.org/developer/heat/template_guide/openstack.html#OS::Neutron::SecurityGroup-props

The next code block demonstrates how to configure the port and floating IP that
will be shared between the VRRP instances.

.. code-block:: yaml

 vrrp_shared_port:
   type: OS::Neutron::Port
   properties:
     network_id: { get_resource: private_net }
     fixed_ips:
       - ip_address: { get_param: vrrp_shared_ip }

 vrrp_shared_floating_ip:
   type: OS::Neutron::FloatingIP
   properties:
     floating_network_id: { get_param: public_net_id }
     port_id: { get_resource: vrrp_shared_port }
   depends_on: router_interface

Finally, let's take a look at the Server and Port definition for an instance:

.. code-block:: yaml

 vrrp_master_server:
   type: OS::Nova::Server
   properties:
     name: vrrp-master
     image: { get_param: image }
     flavor: { get_param: servers_flavor }
     key_name: { get_param: key_name }
     user_data_format: RAW
     networks:
       - port: { get_resource: vrrp_master_server_port }
     user_data:
       get_file: vrrp-setup.sh

 vrrp_master_server_port:
   type: OS::Neutron::Port
   properties:
     network_id: { get_resource: private_net }
     allowed_address_pairs:
       - ip_address: { get_param: vrrp_shared_ip }
     fixed_ips:
       - subnet_id: { get_resource: private_subnet }
         ip_address: 10.0.0.4
     security_groups:
        - { get_resource: vrrp_secgroup }

Note the line ``user_data_format: RAW`` in the server properties; this is
required so that cloud init will setup the ``ubuntu`` user correctly (see this
`blog post`__ for details).

__ http://blog.scottlowe.org/2015/04/23/ubuntu-openstack-heat-cloud-init/

The ``allowed_address_pairs`` section associates the shared VRRP address with
the instance port. We are explicitly setting the port IP address to
``10.0.0.4``. This is not required, we are doing it in order to stay consistent
with the manual configuration. If we do not set it we cannot control which IPs
are assigned to instances and which are assigned for DCHP. If we don't set
these the assigned addresses will be inconsistent across heat invocations.

This configuration is mirrored for the backup instance.

Building the VRRP Stack using HEAT Templates
============================================

Before we start, check that the template is valid:

.. code-block:: bash

 $ heat template-validate -f $ORCHESTRATION_DIR/hot/ubuntu-14.04/vrrp-basic/vrrp.yaml

This command will echo the yaml if it succeeds and will return an error if it
does not. Assuming the template validates let's build a stack

.. code-block:: bash

 $ heat stack-create vrrp-stack --template-file $ORCHESTRATION_DIR/hot/ubuntu-14.04/vrrp-basic/vrrp.yaml
 +--------------------------------------+------------+--------------------+----------------------+
 | id                                   | stack_name | stack_status       | creation_time        |
 +--------------------------------------+------------+--------------------+----------------------+
 | e38eab21-fbf5-4e85-bbad-153321bc1f5d | vrrp-stack | CREATE_IN_PROGRESS | 2015-09-01T03:23:38Z |
 +--------------------------------------+------------+--------------------+----------------------+

As you can see the creation is in progress. You can use the ``event-list``
command to check the progress of creation process:

.. code-block:: bash

 $ heat event-list vrrp-stack
 +--------------------------------+--------------------------------------+------------------------+--------------------+----------------------+
 | resource_name                  | id                                   | resource_status_reason | resource_status    | event_time           |
 +--------------------------------+--------------------------------------+------------------------+--------------------+----------------------+
 | vrrp_backup_server             | 40351139-008c-4d42-b4bb-89e761b4caf8 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:24:17Z |
 | vrrp_backup_server             | 4b8b38db-1292-46db-8307-ef5e95c2a51b | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:24:00Z |
 | vrrp_master_server             | 1c48a5a9-bd92-4c05-8513-f02c1b1e4c8b | state changed          | CREATE_COMPLETE    | 2015-09-01T03:24:00Z |
 | vrrp_shared_floating_ip        | e8829f1e-ba73-4fad-b08e-6cc8e4cf9e59 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:50Z |
 | vrrp_backup_server_floating_ip | 8bff5aa5-5b50-4619-86ed-eaa434f2f9f0 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:50Z |
 | vrrp_master_server_floating_ip | 031949ea-45c8-4fc4-859d-9a1b13e37be3 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:50Z |
 | vrrp_master_server_floating_ip | 0975e4f8-922d-41f3-b363-73d0b6d8e407 | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:49Z |
 | vrrp_shared_floating_ip        | 083c7c2b-4c0f-473b-a417-f6a12ea77f9e | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:48Z |
 | vrrp_master_server             | 0a72a874-7346-4df1-adfa-67ee262863c9 | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:47Z |
 | vrrp_backup_server_floating_ip | d157d7b3-c4e1-4e81-a61b-323aa59256bf | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:45Z |
 | router_interface               | 4468ad1c-a850-4145-91c0-ccb55bc51dc1 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:45Z |
 | vrrp_shared_port               | 94d8d1f0-c38e-4831-b4f2-48a2d5172595 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:45Z |
 | vrrp_master_server_port        | 4263d08f-99b4-43bc-b90f-d72fc125a9bf | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:45Z |
 | vrrp_backup_server_port        | 926342ac-e63a-4707-be56-de0a34d6276f | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:44Z |
 | router_interface               | 3a91b996-3eda-4425-a016-5ab93c503a7f | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:43Z |
 | vrrp_shared_port               | ee41a8c2-5451-4f23-861b-6cf74af666df | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:43Z |
 | vrrp_master_server_port        | c9fa1cd9-79fd-478b-9f0f-099cf341ced9 | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:42Z |
 | vrrp_backup_server_port        | 101a9a93-1600-47f7-8194-90b25c0405c7 | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:42Z |
 | private_subnet                 | eeb887aa-828d-4e87-b224-2f873de21061 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:42Z |
 | private_subnet                 | 144d7c8f-9f0d-4a87-9d42-dc068f906caf | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:41Z |
 | private_net                    | c232f2bc-aac0-44aa-b615-9fd464d22d8d | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:41Z |
 | router                         | 2dd769d8-b44b-46c6-866a-5bf3f74de1c2 | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:41Z |
 | vrrp_secgroup                  | 89741526-6a38-4e64-95dd-b826c9921aff | state changed          | CREATE_COMPLETE    | 2015-09-01T03:23:41Z |
 | router                         | 39321e72-dcbf-4e22-805f-ad3e86abd8ef | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:39Z |
 | private_net                    | ac5a2e1b-42c1-4c73-b947-df47c6db23a1 | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:39Z |
 | vrrp_secgroup                  | 6d5229e7-2977-4286-9214-795c1fa2198a | state changed          | CREATE_IN_PROGRESS | 2015-09-01T03:23:38Z |
 +--------------------------------+--------------------------------------+------------------------+--------------------+----------------------+

If you prefer to create this stack in the Wellington region you
can modify the appropriate parameters on the command line:

.. code-block:: bash

 $ OS_REGION_NAME=nz_wlg_2
 $ heat stack-create vrrp-stack --template-file $ORCHESTRATION_DIR/hot/ubuntu-14.04/vrrp-basic/vrrp.yaml /
 --parameters "public_net_id=e0ba6b88-5360-492c-9c3d-119948356fd3;private_net_dns_servers=202.78.240.213,202.78.240.214,202.78.240.215"

The ``stack-show`` and ``resource-list`` commands are useful commands for
viewing the state of your stack. Give them a go:

.. code-block:: bash

 $ heat stack-show vrrp-stack
 $ heat resource-list vrrp-stack

Once all resources in your stack are in the ``CREATE_COMPLETE`` state you are
ready to re-run the tests as described under :ref:`vrrp-testing`. The neturon
``floatingip-list`` command will give you the IP addresses and port IDs you
need:

.. code-block:: bash

 $ neutron floatingip-list

If you wish you can SSH to the master and backup instances as described under
:ref:`instance-access`.

Once satisfied with the configuration we can cleanup and get back to
our original state:

.. code-block:: bash

 $ heat stack-delete vrrp-stack
 +--------------------------------------+------------+--------------------+----------------------+
 | id                                   | stack_name | stack_status       | creation_time        |
 +--------------------------------------+------------+--------------------+----------------------+
 | e38eab21-fbf5-4e85-bbad-153321bc1f5d | vrrp-stack | DELETE_IN_PROGRESS | 2015-09-01T03:23:38Z |
 +--------------------------------------+------------+--------------------+----------------------+

This ends the tutorial on setting up hot swap VRRP instances in the Catalyst
Cloud.

