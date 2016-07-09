---
layout: post
title: "OpenStack Mitaka Multi Node on CentOS 7"
description: "Setup OpenStack Mitaka Multi-Node on CentOS 7"
category: "OpenStack"
tags: []
---
{% include JB/setup %}

<img src="https://cloud.githubusercontent.com/assets/10396579/16709212/d1daa63e-4627-11e6-8a91-6144c614aeeb.jpg" width="930">

OpenStack setup without the aid of a deployment tool can be a damper for people taking their baby steps in to the world of OpenStack. There are multiple options available in OpenStack today for this kind of setup like Devstack, Packstack which make it easier to setup OpenStack.

# RDO and Packstack

In this blog I would like share my experience deploying a multi-node OpenStack PoC (Proof of Concept) setup using RDO and Packstack.

- **RDO** is simply a distribution of OpenStack for Red Hat Enterprise Linux (RHEL), Fedora and derivatives (e.g.: CentOS).

- **PackStack** is a Puppet based tool that simplifies the deployment of RDO.

Most of the documentation about RDO and PackStack was related to all-in-one setup which lets you setup OpenStack on a single machine Physical or Virtual.

All right, lets get started with a description of OpenStack Mitaka components we are going to setup.

## Controller

The Controller node is where most of the shared OpenStack services and other tools run. The Controller node supplies API, scheduling, and other shared services for the cloud. The Controller node has the dashboard, the image store, and the identity service. Additionally, Nova compute management service as well as the Neutron server are also configured in this node.

	System Requirements
	1  GB Ram
	30 GB HDD

	Networking - Single NIC card
	Management (eth0)


## Networking

The Network node provides virtual networking and networking services to Nova instances using the Neutron Layer 3 and DHCP network services.

	System Requirements
	1  GB Ram
	30 GB HDD

	Networking - Three NIC card
	Management (eth0)
	Guest Data (eth1)
	External Network (eth2)

## Compute

The Compute node is where the VM instances (Nova compute instances) are installed. The VM instances use iSCSI targets provisioned by the Cinder volume service. There are multiple options for compute node like KVM, Hyper-v and even docker. KVM was used in this particular case.

	System Requirements
	4  GB Ram
	60 GB HDD (excluding OS)

	Networking - Two NIC card
	Management (eth0)
	Guest Data (eth1)

## Networking
Few words on networking. Although there are many other options available for networking, OVS was considered for this setup. Ideally **Management network should be isolated from the rest of the Network for security purposes**.
	This being a PoC setup all the networking was done on a single VLAN although they are physically separated.

#### Management (eth0 on all hosts)
This network is used for management only.

#### Guest data (eth1 on all hosts)
This is used by guests to communicate with each other. We will be using a single VLAN for this PoC.

#### Public (eth2 on all hosts)
This is used by hosts to communicate with external network routed through the Network node. External hosts will be able to reach the guests(instances) on the basis of external ip and security groups.

## Host configuration

#### Server Hostname
- stack-pocbox-controller

- stack-pocbox-network

- stack-pocbox-compute

#### Network
	This being a PoC setup all the networking was done on a
	single VLAN although they are physically separated.
	VLAN 150
	172.21.64.1 - 172.21.67.254
	172.21.64.0/22

**Note**
		You can use any Network Subnet or VLAN ID as per your requirement.

#### Operating System

	CentOS 7

#### Server configuration (repeat this on all nodes)

*Update OS to latest*  

	$ yum update

*Disable* *FirewallD*, *NetworkManager*:

	$ systemctl disable Firewalld
	$ systemctl disable NetworkManager
	$ systemctl mask Firewalld

*Set* *Hostname*

	systemctl set-hostname		

*Disable* *selinux*

	/etc/sysconfig/selinux
	set to “permissive”

*Enable iptables*

	$ yum install iptables-services
	$ systemctl enable iptables

*Create* *hosts* *file* *and* *copy* *this* *to* *all* *hosts*
**Note**

- Management (eth0) ip is used for hostname on the nodes
- Static IP
- I prefer updating /etc/hosts even though i have a working DNS

 		 cat /etc/hosts
		127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
		::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
		172.21.64.1 stack-pocbox-controller.local stack-pocbox-controller
		172.21.64.2 stack-pocbox-network.local stack-pocbox-network
		172.21.64.5 stack-pocbox-compute.local stack-pocbox-compute

*If you are using non-English locale make sure your /etc/environment is populated:*

	LANG=en_US.utf-8
	LC_ALL=en_US.utf-8

*Network Configuration*

***Repeat the same for eth0, eth1 and eth2 on all the nodes***

	cat /etc/sysconfig/network-scripts/ifcfg-eth0
	TYPE="Ethernet"
	BOOTPROTO="static"
	DEFROUTE="yes"
	PEERDNS="yes"
	PEERROUTES="yes"
	IPV4_FAILURE_FATAL="no"
	IPV6INIT="no"
	NAME="eth0"
	UUID="9c31d94b-d0d1-45bc-90c3-3c2bfb6c166b"
	DEVICE="eth0"
	ONBOOT="yes"
	IPADDR=172.21.64.1
	NETMASK=255.255.252.0
	GATEWAY=172.21.67.254
	DOMAIN="bedford.progress.com"
	DNS1=172.21.33.253
	DNS2=172.21.33.253

*Configure Chrony NTP for server and clients*

- Server (	Controller Node)

		$ chkconfig ntp off
		$ yum remove ntp
		$ yum install chrony
		Open chrony.conf and replace ‘allow <IP/CIDR>’ with your Management network

*Enable chrony*

		$ systemctl enable chronyd.service
		$ systemctl start chronyd.service
		$ systemctl restart chronyd.service

*Reboot*

	init 6

## Install OpenStack

#### Install PackStack (Controller Node)
	$ sudo yum install -y centos-release-openstack-mitaka
	$ sudo yum update -y
	$ sudo yum install -y openstack-packstack

*Generate answer file (Controller Node)*

	$ packstack --gen-answer-file=packstack_answers.conf

*Modify answer file to change ip address of Network and Compute node*

	CONFIG_COMPUTE_HOSTS=172.21.64.5
	CONFIG_NETWORK_HOSTS=172.21.64.2

**Answer is already populated with random passwords for all your services, change them as required**

*(Optional) I choose to install **Heat** as well*

	CONFIG_HEAT_INSTALL=y


*Install openstack*

	$ packstack --answer-file=packstack_answers.conf

*Once install is successful you will see a screen similar to this*

	*** Installation completed successfully ***

	Additional information:
	 * Time synchronization installation was skipped. Please note that unsynchronized time on server instances might be problem for some OpenStack components.
	 * File /root/keystonerc_admin has been created on OpenStack client host 172.21.64.1. To use the command line tools you need to source the file.
	 * To access the OpenStack Dashboard browse to http://172.21.64.1/dashboard .
	Please, find your login credentials stored in the keystonerc_admin in your home directory.
	 * To use Nagios, browse to http://172.21.64.1/nagios username: nagiosadmin, password: 8908d16396354aa4
	 * The installation log file is available at: /var/tmp/packstack/20160621-160219-uCI8ju/openstack-setup.log
	 * The generated manifests are available at: /var/tmp/packstack/20160621-160219-uCI8ju/manifests

*Open Horizon Dashboard*

	http://172.21.64.1/dashboard
	The root password setup by PackStack script is available in /root/keystonerc_admin

*Source /root/keystonerc_admin file*

	$ . /root/keystonerc_admin

## Configure OpenStack

The following tasks were performed as a part of initial configuration.

- Network Configuration
- Clean up default network created by PackStack script
- Create new Public, Private network and its subnet
- Create a new router
- Upload a CentOS qcow2 image
- Create a Key pair
- Configure Security Group
- Deploy an instance from CentOS image
- Attach floating IP
- Login to instance


#### Network Configuration (Network Node)

	$ ovs-vsctl show
    Bridge "br-eth1”
        Port "br-eth1”
            Interface "br-eth1”
                type: internal
        Port “eth1”
            Interface “eth1”
    Bridge br-int
        fail_mode: secure
        Port br-int
            Interface br-int
                type: internal
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port "tap15df1b42-06"
            tag: 1
            Interface "tap15df1b42-06"
                type: internal
        Port "qr-494def92-bc"
            tag: 1
            Interface "qr-494def92-bc"
                type: internal
    Bridge br-ex
        Port “eth2”

*Note that eth1 is a port on Bridge br-eth1 and eth2 is a port on Bridge br-ex. This would be missing in your Network node at this point. Let’s go ahead and add this.*

	Networking
		Management (eth0)
		Guest Data (eth1)
		External Network (eth2)

	$ ovs-vsctl add-br br-eth1
	$ ovs-vsctl add-port br-eth1 eth1
	$ ovs-vsctl add-port br-ex eth2

*Repeat the steps for Compute Node*

	$ ovs-vsctl add-br br-eth1
	$ ovs-vsctl add-port br-eth1 eth1

#### Clean up default network created by PackStack script

*Open Horizon Dashboard*

	http://172.21.64.1/dashboard
	The admin password setup by PackStack script is available in /root/keystonerc_admin

![login_screen](https://cloud.githubusercontent.com/assets/10396579/16363915/e0b6b4c6-3bf6-11e6-9240-6c9643ee2480.png)

*Source /root/keystonerc_admin file*  

	$ . /root/keystonerc_admin

*Delete existing network and routers*

<img src="https://cloud.githubusercontent.com/assets/10396579/16553155/2ceebd0a-41e4-11e6-97e4-0ec151952fb5.png" width="930">

<img src="https://cloud.githubusercontent.com/assets/10396579/16553031/2b4d7884-41e3-11e6-8cc1-bbcf55b0f4b9.png" width="930">

*Create new Public Network*

This network will have access to the outside (**external**) network. I am using the same as VLAN 150 mentioned before. The public-subnet(more about this in a moment) will be created by using a subgroup of ip’s from the 172.21.64.0/22 subnet.

![](https://cloud.githubusercontent.com/assets/10396579/16553291/60c7ab72-41e5-11e6-8d1e-6cc31a553666.png)

*Create Public Subnet*

I have assigned a subgroup of IP from VLAN 150 to the public Sidney. This will be used by OpenStack to assign elastic IP for the VM's


![](https://cloud.githubusercontent.com/assets/10396579/16553495/a306e150-41e6-11e6-8650-9424e2c91939.png)
![](https://cloud.githubusercontent.com/assets/10396579/16553498/a501ac88-41e6-11e6-8699-d6f559cba881.png)

	Note that DNS server although a different subnet is routable on VLAN 150 and this is done on the network side.

	Ideally the external Network should be able to reach Internet. This needs to be configured on the network end

*Create new Private Network*

<img src="https://cloud.githubusercontent.com/assets/10396579/16553686/a35b2afc-41e7-11e6-8908-960cf9de43d9.png" width="930">

*Create Private Subnet*

You can choose a Private IP range appropriate to your organisation.

![](https://cloud.githubusercontent.com/assets/10396579/16555338/c91efa3e-41f1-11e6-82a9-68dfa3d6fd53.png)

![](https://cloud.githubusercontent.com/assets/10396579/16555339/c9e14968-41f1-11e6-9450-191258d3db0d.png)

*Configure Security Group*

	Note: The following security group configuration was done with PoC in mind. DO NOT USE THIS FOR PRODUCTION

We will create a security group that will allow all Incoming and Outgoing traffic.

From **Project** -> **Access and Security** -> **Security** **Groups** Select default and click on Manage rules.

Configure the group to look like this.

![security_group](https://cloud.githubusercontent.com/assets/10396579/16556538/4154f34a-41f8-11e6-8b78-fadde418c911.png)

*Create a Key Pair*

From **Project** -> **Access and Security** -> **Key** **Pairs** select **+Create Key Pair**

Download the .pem file for later use

	You can also create your own RSA key and upload if you prefer doing so

*Upload CentOS 7 image*

The default Cirros image does not do much.Lets upload create a CentOS image for creating our first instance.

Copy the url CentOS latest version ISO image from [here](http://cloud.centos.org/centos/7/images/)

	Copy the ISO url.There is no need to download the ISO Image.

From **Admin** -> **Images** select **Create Image**

Paste the URL in to the field for **Image Location**.

(https://cloud.githubusercontent.com/assets/10396579/16555562/0460c1b2-41f3-11e6-9b0a-e05855e02819.png)

*Launch instance and attach floating ip*

Now that we uploaded a CentOS image. Let's go ahead and Launch Instance and attach a floating IP

From **Project** -> **Instances** -> **Launch** **Instance**

	- Details
			Provide Instance name(Leave rest to defaults)
	- Source
			Select the CentOS image that was created
	- Flavour
			Select Flavour(depending on the compute instance)
	- Networks
			Select "Private Network"
	- Network Ports
			Leave default
	- Security Groups
			Select "default" security group that was configured
	- Key pair
			Select the key pair that created(Might be the only option unless you have created multiple

**Launch** **Instance** to create instance.

*Attach floating IP*

From **Project** -> **Instances** -> **Instance** **Name** -> **Action** drop down select **Associate** **Floating** **IP**

![](https://cloud.githubusercontent.com/assets/10396579/16556720/37fd6e0c-41f9-11e6-9a84-7127b3d423a3.png)

![](https://cloud.githubusercontent.com/assets/10396579/16556726/398e842c-41f9-11e6-88bf-b2949a5bfe9d.png)

*Login to instance*

Once the instance is up and running. You should be able to login with the floating IP assigned.

![](https://cloud.githubusercontent.com/assets/10396579/16556897/3d7a1f28-41fa-11e6-82b0-a3fda88a208e.png)

	$ ssh -l CentOS -i <pem _key> <floating_ip>


## Further Reading

[Openstack essential Guide](https://anturis.com/blog/openstack-the-essential-guide/)

[RDO multinode](https://cloudbase.it/rdo-multi-node/)

[RDO multinode a very useful Youtube video by Seth Jennings](https://www.youtube.com/watch?v=eOlIB323c8s)
