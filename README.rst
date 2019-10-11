Overview
========

Welcome to TrinityX!

TrinityX is as open-source HPC platform. It is designed from the ground up to provide all services required in a modern HPC system, and to allow full customization of the installation. Also it includes optional modules for specific needs, such as  Docker on the compute nodes.

The full documentation is available in the ``doc`` subdirectory. See the instructions how to build it below.


Quick start
===========

In standard configuration TrinityX provides the following services to the cluster:

* Luna, our default super-efficient node provisioner https://github.com/clustervision/luna
* OpenLDAP
* SLURM
* Zabbix
* NTP
* and more

It will also set up:

* NFS-shared home and application directories
* OpenHPC modules
* rsyslog
* and more

It can also setup the Open OnDemand web portal and configure remote desktop access to the compute nodes using NoVNC


Steps to install TrinityX without Luna e.g. on a cloud platform
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Provision all cloud instances with a CentOS minimal image and configure an internal netowrk that all nodes are inside

2. Configure passwordless authentication to the controller and all of the computes

3. Enable the EPEL repository::

   # yum install epel-release

   if you are using a local mirror of this repo please use that repo file instead

4. Setup luna repository:: (still needed for luna_ansible module)

    # curl https://updates.clustervision.com/luna/1.2/centos/luna-1.2.repo > /etc/yum.repos.d/luna-1.2.repo

    if you are using a local mirror of this repo please use that repo file instead

5. Install ``git``, ``ansible`` and ``luna-ansible``::

    # yum install git ansible luna-ansible

6. Clone TrinityX repository into your working directory and go to the site directory::

    # git clone https://github.com/antonycleave/trinityX.git
    # cd trinityX/site
    # git checkout openstack-dev

7. Based on whether you are installing a single-controller or a high-availability (HA) setup, you might want to update the configuration files:

   * ``group_vars/all``

     controller IPs should be set to the IPs of your instances and hostnames will be applied by the playbooks:

       trix_ctrl1_ip: 10.15.255.254
     
       trix_ctrl1_bmcip: 10.148.255.254
     
       trix_ctrl1_heartbeat_ip: 192.168.0.1
     
       trix_ctrl1_hostname: controller1

     The hostname will overwrite anything that your cloud provider has set.
     
     It is ok to leave in the bmcip and heartbeat IP even if you are not using them, they will be ignored if you are not using luna and if you are not enabling HA

     Do the same for controller 2 if you are using it, having them in does no harm if you are not

     the following need to match your subnet as you defined when setting up your cloud network

       trix_cluster_net:  10.15.0.0
     
       trix_cluster_netprefix:  16
     

     You might also want to check if the default firewall parameters in the same file apply to your situation. If you are in a cloud env your controller node probably has 1 virtual NIC and one floating IP:
   
       firewalld_public_interfaces:
     
       firewalld_trusted_interfaces: ['{{ ansible_default_ipv4.interface }}']
     
       firewalld_public_tcp_ports:

     No firewall public ports. This is a cloud node so you should be using your cloud providers security groups to restrict access to allow ping, ssh (22) and https (443) to the controller node.


     most times you will  not want selinux enabled

       enable_selinux: false

     most times you will  not want password login enabled

       allow_password_login: 'no'


     if you have specific DNS server(s) you want to use then configure them here leaving this blank will configure the cluster DNS server to use the google DNS servers:

       ``bind_dns_forwarders:``
       
       ``  - 'ip1'``
       
       ``  - 'ip2'``

     the following 2 settings are essential if not using luna, they tell ansible to take over configuring resolv.conf on all nodes and dns zones on on the controller and rename compute instances

       **configure_resolveconf: true**
       
       **use_inventory_hostname: true**


     if you are installing Open OnDemand you probably want to enable the vnc desktop option. If so then currently you probably want to set:

       enable_vnc_to_nodes: true

     This is used by the ood- modules to determine whether to build the modules (on the controller node),  configure the bc_desktop module (on the portal node) and install the graphical desktop packages (on the staticcompute nodes) so currently is makes sense to set it here in one place. Going forward as more node groups are supported out of the box you may want to define it for each separate group. e.g. in ``group_vars/controllers``, ``group_vars/staticcomputes`` and ``group_vars/portal`` files

     Final settings are for the compute node count so that ansible can build the slurm config file

       heat:
       
         ctrl_ip: '{{ trix_ctrl_ip }}'
         
         ctrl_hostname: '{{ trix_ctrl_hostname }}'

         static_compute_partition_name: defq
         
         static_compute_host_name_base: node
         
         static_compute_start_number: 20
         
         static_compute_initial_number: 20
         
         static_compute_max_number: 20
         
         static_compute_min_number: 1
         
    Hopefully these are fairly self explanitory. The ctrl_ip and hostname should NOT be changed from these values unless you really know what you are doing. The rest are used to build the slurm config file on the controller. 

      start_number - this is the number we start COUNTing from normally 1 but could be 10 if you want to have the nodes all called node0001-node0020 and have a different partition for nodes0010 to 0020

      initial_number - this is how many nodes you have created initially, the plan is to allow the scheduler to power up and down the nodes or maybe even delete and recreate instances

      max_number - this is the maximum nuber of nodes that can be up in this partition (not used yet should be same as inital number for now)

      min_number - this is the minimum number of nodes that should stay powered up (not used yet until power saving is implemented leave as 1)

This can be auto-populated by openstack heat as the hash name suggests. It assumes that the nodes in the same parition will be sequential counting from the max to min. The above example is for a 20 node system creating node0001 to node0020. **These need to match the hostnames you use in the hostfile** an example for cpu0005 to cpu0019 is shown below

       heat:
       
         ctrl_ip: '{{ trix_ctrl_ip }}'
         ctrl_hostname: '{{ trix_ctrl_hostname }}'
         static_compute_partition_name: defq
         static_compute_host_name_base: cpu
         static_compute_start_number: 5
         static_compute_initial_number: 14
         static_compute_max_number: 14
         static_compute_min_number: 1
         

   * ``group_vars/staticcomputes``

     these are settings for just the compute nodes If these are baremetal provisioned nodes using something like ironic then you might have mellanox cards in. If so you will want to use the mellanox OFED instead of the default CentOS 7 rdma packages. This can be enabled by seting the following varable in this file (or in ``group_vars/all`` if you have baremental nodes with melllanox everywhere)

       use_mellanox_ofed: true

     if you are using the open on demand portal  with remote desktop enabled then you will need to pick from mate or xfce desktops. I have experienced issues with mate to I recommend that xfce be tried first this  should only be set on the nodes you want a graphical desktop login to.

       vnc_desktop: 'xfce'

   * ``group_vars/controllers``

     The default option is to disallow LDAP logins to anyone except users in the default Admins group (the default rsupport user is a member automatically). This is equivalent to setting the following variable in this config file:

       sss_allowed_groups:
         - Admins

     To allow all groups override this by setting it to null like so:

       sss_allowed_groups:

     To add more groups add them to the list but **remember to include the Admins group or you will break the default rsupport user**

       sss_allowed_groups:
         - Admins
         - slurm

   Remember that it is perfectly ok to shake things up and move some settings from ``group_vars/all`` into the individual groups. On example that you might want to do this with is the ``allow_password_login: 'no'`` setting. This makes perfect sense on the controller but if your compute nodes are protected behind a security group then you might want to set ``allow_password_login: 'no'`` in  ``group_vars/controllers`` and ``group_vars/portal`` and then set it to 'yes' for the compute nodes in ``group_vars/staticcomputes``

8. You will need a security group to allow access from the compute node subnet to all ports on each node in the cluster **and** for you to reach ports 22 and 443 on the controller node. You will probabaly want at least 2 different security groups to configure this.


9. Install ``OndrejHome.pcs-modules-2`` and ``ome.network`` from the ansible galaxy::

    # ansible-galaxy install OndrejHome.pcs-modules-2 ome.network

10. Configure ``hosts`` file to allow ansible to address nodes. In ALL cases it is very important that the IPs match the IP's assgned by your cloud provider. The hostnames will be set by ansible and DNS will be configured to point these ip's at these hostnames

   Example for non-HA setup with no web portal::

       [controllers]
       vcontroller ansible_host=10.15.255.254
       [staticcomputes]
       node0001 ansible_host=10.15.0.1
       node0002 ansible_host=10.15.0.2

   Example for non-HA setup with a web portal and allowing vnc remote desktop connections to the compute nodes (just ommit the vncnodes setion if you don't want this and it will not be configured)::

       [controllers]
       vcontroller ansible_host=10.15.255.254
       [staticcomputes]
       node0001 ansible_host=10.15.0.1
       node0002 ansible_host=10.15.0.2
       [ood]
       portal ansible_host=10.15.255.241
       [vncnodes]
       node0001 ansible_host=10.15.0.1
       node0002 ansible_host=10.15.0.2

    If you want a separate queue for the remote esktop nodes, this is currently left as an exercise for the reader. This will require creating a new group of nodes and modifying the slurm role to configure these new nodes

   Example for HA setup::

       [controllers]
       vcontroller1 ansible_host=10.15.255.254
       vcontroller2 ansible_host=10.15.255.253
       [staticcomputes]
       node0001 ansible_host=10.15.0.1
       node0002 ansible_host=10.15.0.2

11. Start TrinityX installation::

     # ansible-playbook controller.yml --skip-tags=luna

     the skip-tags entry is essential or bad things will happen with the suggested network config. Doing it this way allows us to use the same controller file for both types of install.

    **Note**: If errors are encoutered during the installation process, analyze the error(s) in the output and try to fix it then re-run the installer.

    **Note**: By default, the installation logs will be available at ``/var/log/trinity.log``

12. Deploy the compute nodes

    # ansible-playbook static-compute.yml

13. Deploy the portal nodes (if needed and assuming that your portal node is called portal in the inventory file above)

    # ansible-playbook ood-portal.yml -l portal 

    the -l portal is not strictly required but if you are doing this at the beginning of the cluster install and not adding a portal node to an existing VM this stops it repeating actions which ahve already been allplied in the controller and compute node setup.

Now you have your controller(s), portal and computes!


Customizing your installation
-----------------------------

Now, if you want to tailor TrinityX to your needs, you can modify the ansible playbooks and variable files.

Descriptions to configuration options are given inside ``controller.yml`` and ``group_vars/*``. Options that might be changed include:

* Controller's hostnames and IP addresses
* Shared storage backing device
* DHCP dynamic range
* Firewall settings
* mellanox ofed
  * if you want this installed on only the 

You can also choose which components to exclude from the installation by modifying the ``controller.yml`` ``.yml`` playbook.

OpenHPC Support
===============

The OpenHPC project provides a framework for building, managing and maintain HPC clusters. This project provides packages for most popular scientific and HPC applications. TrinityX can integrate this effort into it's ecosystem. In order to enable this integration set the flag ``enable_openhpc`` in ``group_vars/all`` to ``true``. 
Currently when OpenHPC is enabled standart environment modules, slurm and pdsh from TrinityX gets disabled and OpenHPC versions are used instead. 

Steps to install TrinityX with Luna
~~~~~~~~~~~~~~~~~~~~~~~~~

1. Install CentOS Minimal on your controller(s)

2. Configure network interfaces that will be used in the cluster, e.g public, provisioning and MPI networks

3. Configure passwordless authentication to the controller itself or/and for both controllers in the HA case

4. Setup luna repository::

    # curl https://updates.clustervision.com/luna/1.2/centos/luna-1.2.repo > /etc/yum.repos.d/luna-1.2.repo

5. Enable the EPEL repository::

   # yum install epel-release

6. Install ``git``, ``ansible`` and ``luna-ansible``::

    # yum install git ansible luna-ansible

7. Clone TrinityX repository into your working directory and go to the site directory::

    # git clone http://github.com/clustervision/trinityX
    # cd trinityX/site

8. Based on whether you're installing a single-controller or a high-availability (HA) setup, you might want to update the configuration files:

   * ``group_vars/all``

   You might also want to check if the default firewall parameters in the same file apply to your situation::

      firewalld_public_interfaces: [eth0]
      firewalld_trusted_interfaces: [eth1]
      firewalld_public_tcp_ports: [22, 443]

   **Note**: In the case of an HA setup you will most probably need to change the default name of the shared block device set by ``shared_fs_device``.

9. Install ``OndrejHome.pcs-modules-2`` from the ansible galaxy::

    # ansible-galaxy install OndrejHome.pcs-modules-2

10. Configure ``hosts`` file to allow ansible to address controllers.

   Example for non-HA setup::

       [controllers]
       controller ansible_host=10.141.255.254

   Example for HA setup::

       [controllers]
       controller1 ansible_host=10.141.255.254
       controller2 ansible_host=10.141.255.253

11. Start TrinityX installation::

     # ansible-playbook controller.yml

    **Note**: If errors are encoutered during the installation process, analyze the error(s) in the output and try to fix it then re-run the installer.

    **Note**: By default, the installation logs will be available at ``/var/log/trinity.log``

11. Create a default OS image::

    # ansible-playbook compute.yml

Now you have your controller(s) installed and the default OS image created!


Customizing your installation
-----------------------------

Now, if you want to tailor TrinityX to your needs, you can modify the ansible playbooks and variable files.

Descriptions to configuration options are given inside ``controller.yml`` and ``group_vars/*``. Options that might be changed include:

* Controller's hostnames and IP addresses
* Shared storage backing device
* DHCP dynamic range
* Firewall settings

You can also choose which components to exclude from the installation by modifying the ``controller.yml`` playbook.

OpenHPC Support
===============

The OpenHPC project provides a framework for building, managing and maintain HPC clusters. This project provides packages for most popular scientific and HPC applications. TrinityX can integrate this effort into it's ecosystem. In order to enable this integration set the flag ``enable_openhpc`` in ``group_vars/all`` to ``true``. 
Currently when OpenHPC is enabled standart environment modules, slurm and pdsh from TrinityX gets disabled and OpenHPC versions are used instead. 

Documentation
=============

To build the full set of the documentation included with TrinityX:

1. Install ``git``::

    # yum install git

2. Clone TrinityX repository into your working directory and go to the directory containing the documentation::

    # git clone http://github.com/clustervision/trinityx
    # cd trinityX/doc

3. Install ``pip``, e.g. from EPEL repository::

    # yum install python34-pip.noarch

4. Install ``sphinx`` and ``Rinohtype``::

    # pip3.4 install sphinx Rinohtype

6. Build the PDF version of the TrinityX guides::

   # sphinx-build -b rinoh . _build/

If everything goes well, the documentation will be saved as ``_build/TrinityX.pdf``


Contributing
============

To contribute to TrinityX:

1. Get familiar with our `code guidelines <Guidelines.rst>`_
2. Clone TrinityX repository
3. Commit your changes in your repository and create a pull request to the ``dev`` branch in ours.
