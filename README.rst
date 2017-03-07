OpenStack Virtual Baremetal
===========================

Introduction
------------

OpenStack Virtual Baremetal is a way to use OpenStack instances to do
simulated baremetal deployments.  This project is a collection of tools
and documentation that make it much easier to do so.  It primarily consists
of the following pieces:

- Patches and documentation for setting up a host cloud.
- A deployment CLI that leverages the OpenStack Heat project to deploy the
  VMs, networks, and other resources needed.
- An OpenStack BMC that can be used to control OpenStack instances via IPMI
  commands.
- A tool to collect details from the "baremetal" VMs so they can be added as
  nodes in the OpenStack Ironic baremetal deployment project.

A basic OVB environment is just a BMC VM configured to control a number
of "baremetal" VMs.  This allows them to be treated largely the same
way a real baremetal system with a BMC would.  A number of additional
features can also be enabled to add more to the environment.

Benefits and Drawbacks
----------------------

OVB started as part of the OpenStack TripleO project.  It was intended to
provide more flexible and scalable development and CI environments.  Previous
methods for doing this focused on setting up all the vms for a given
environment on a single box.  This had a number of drawbacks:

- Each developer needed to have their own system.  Sharing was possible, but
  more complex and generally not done.  Multi-tenancy is a basic design
  tenet of OpenStack so this is not a problem when using it to provision the
  VMs.  A large number of developers can make use of a much smaller number of
  physical systems.
- If a deployment called for more VMs than could fit on a single system, it
  was a complex manual process to scale out to multiple systems.  An OVB
  environment is only limited by the number of instances the host cloud can
  support.
- Pre-OVB test environments were generally static because there was not an API
  for dynamic provisioning.  By using the OpenStack API to create all of the
  resources, test environments can be easily tailored to their intended use
  case.

One drawback to OVB at this time is that it is generally not compatible with
current public clouds.  While it is possible to do an OVB deployment on a
completely stock OpenStack cloud, most public clouds have restrictions (older
OpenStack releases, inability to upload new images, no Heat, etc.) that make
it problematic.  At this time, OVB is primarily used with semi-private clouds
configured for ideal compatibility.  This situation should improve as more
public clouds move to newer OpenStack releases, however.

How-To
------

Instructions for patching the host cloud[1], setting up the base environment,
and deploying a virtual baremetal Heat stack.

1: The host cloud is any OpenStack cloud providing the necessary functionality
to run OVB.  In a TripleO deployment, this would be the overcloud.

.. warning:: This process requires patches and configuration settings that
             may not be appropriate for production clouds.

Patching the Host Cloud
^^^^^^^^^^^^^^^^^^^^^^^

The changes described in this section apply to compute nodes in the
host cloud.

Apply the Nova pxe boot patch file in the ``patches`` directory to the host
cloud Nova.  Examples:

TripleO/RDO::

    sudo patch -p1 -d /usr/lib/python2.7/site-packages < patches/nova/nova-pxe-boot.patch

Devstack:

   .. note:: You probably don't want to try to run this with devstack anymore.
             Devstack no longer supports rejoining an existing stack, so if you
             have to reboot your host cloud you will have to rebuild from
             scratch.

   .. note:: The patch may not apply cleanly against master Nova
             code.  If/when that happens, the patch will need to
             be applied manually.

   ::

      cp patches/nova/nova-pxe-boot.patch /opt/stack/nova
      cd /opt/stack/nova
      patch -p1 < nova-pxe-boot.patch

Configuring the Host Cloud
^^^^^^^^^^^^^^^^^^^^^^^^^^

The changes described in this section apply to compute nodes in the
host cloud.

#. Neutron must be configured to use the NoopFirewallDriver.  Edit
   ``/etc/neutron/plugins/ml2/ml2_conf.ini`` and set the option
   ``firewall_driver`` in the ``[securitygroup]`` section as follows::

       firewall_driver = neutron.agent.firewall.NoopFirewallDriver

#. In Liberty and later versions, arp spoofing must be disabled.  Edit
   ``/etc/neutron/plugins/ml2/ml2_conf.ini`` and set the option
   ``prevent_arp_spoofing`` in the ``[agent]`` section as follows::

        prevent_arp_spoofing = False

#. The Nova option ``force_config_drive`` must _not_ be set.

#. Ideally, jumbo frames should be enabled on the host cloud.  This
   avoids MTU problems when deploying to instances over tunneled
   Neutron networks with VXLAN or GRE.

   For TripleO-based host clouds, this can be done by setting ``mtu``
   on all interfaces and vlans in the network isolation nic-configs.
   A value of at least 1550 should be sufficient to avoid problems.

   If this cannot be done (perhaps because you don't have access to make
   such a change on the host cloud), it will likely be necessary to
   configure a smaller MTU on the deployed virtual instances.  For a
   TripleO undercloud, Neutron should be configured to advertise a
   smaller MTU to instances.  Run the following as root::

       # Replace 'eth1' with the actual device to be used for the
       # provisioning network
       ip link set eth1 mtu 1350
       echo -e "\ndhcp-option-force=26,1350" >> /etc/dnsmasq-ironic.conf
       systemctl restart 'neutron-*'

   If network isolation is in use, the templates must also configure
   mtu as discussed above, except the mtu should be set to 1350 instead
   of 1550.

#. Restart ``nova-compute`` and ``neutron-openvswitch-agent`` to apply the
   changes above.

Preparing the Host Cloud Environment
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Source an rc file that will provide admin credentials for the host cloud.

#. Upload an ipxe-boot image for the baremetal instances::

    glance image-create --name ipxe-boot --disk-format qcow2 --property os_shutdown_timeout=5 --container-format bare < ipxe/ipxe-boot.qcow2

   .. note:: os_shutdown_timeout=5 is to avoid server shutdown delays since
             since these servers won't respond to graceful shutdown requests.

   .. note:: On a UEFI enabled openstack cloud, to boot the baremetal instances
             with uefi (instead of the default bios firmware) the image should
             be created with the parameters --property="hw_firmware_type=uefi".

#. Upload a CentOS 7 image for use as the base image::

    wget http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud.qcow2

    glance image-create --name CentOS-7-x86_64-GenericCloud --disk-format qcow2 --container-format bare < CentOS-7-x86_64-GenericCloud.qcow2

#. (Optional) Create a pre-populated base BMC image.  This is a CentOS 7 image
   with the required packages for the BMC pre-installed.  This eliminates one
   potential point of failure during the deployment of an OVB environment
   because the BMC will not require any external network resources::

    wget https://repos.fedorapeople.org/repos/openstack-m/ovb/bmc-base.qcow2

    glance image-create --name bmc-base --disk-format qcow2 --container-format bare < bmc-base.qcow2

   To use this image, configure ``bmc_image`` in env.yaml to be ``bmc-base`` instead
   of the generic CentOS 7 image.

#. Create recommended flavors::

    nova flavor-create baremetal auto 6144 50 2
    nova flavor-create bmc auto 512 20 1

   These flavors can be customized if desired.  For large environments
   with many baremetal instances it may be wise to give the bmc flavor
   more memory.  A 512 MB BMC will run out of memory around 20 baremetal
   instances.

#. Source an rc file that will provide user credentials for the host cloud.

#. Add a Nova keypair to be injected into instances::

    nova keypair-add --pub-key ~/.ssh/id_rsa.pub default

#. (Optional) Configure quotas.  When running in a dedicated OVB cloud, it may
   be helpful to set some quotas to very large/unlimited values to avoid
   running out of quota when deploying multiple or large environments::

    neutron quota-update --security_group 1000
    neutron quota-update --port -1
    neutron quota-update --network -1
    neutron quota-update --subnet -1
    nova quota-update --instances -1 --cores -1 --ram -1 [tenant uuid]


#. Create provisioning network.

   .. note:: The CIDR used for the subnet does not matter.
             Standard tenant and external networks are also needed to
             provide floating ip access to the undercloud and bmc instances

   .. warning:: Do not enable DHCP on this network.  Addresses will be
                assigned by the undercloud Neutron.

   ::

      neutron net-create provision
      neutron subnet-create --name provision --no-gateway --disable-dhcp provision 192.0.2.0/24

#. Create "public" network.

   .. note:: The CIDR used for the subnet does not matter.
             This can be used as the network for the public API endpoints
             on the overcloud, but it does not have to be accessible
             externally.  Only the undercloud VM will need to have access
             to this network.

   .. warning:: Do not enable DHCP on this network.  Doing so may cause
                conflicts between the host cloud metadata service and the
                undercloud metadata service.  Overcloud nodes will be
                assigned addresses on this network by the undercloud Neutron.

   ::

       neutron net-create public
       neutron subnet-create --name public --no-gateway --disable-dhcp public 10.0.0.0/24

Create the baremetal Heat stack
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

#. Copy the example env file and edit it to reflect the host environment::

    cp templates/env.yaml.example env.yaml
    vi env.yaml

#. Deploy the stack::

    bin/deploy.py

#. Wait for Heat stack to complete:

   .. note:: The BMC instance does post-deployment configuration that can
             take a while to complete, so the Heat stack completing does
             not necessarily mean the environment is entirely ready for
             use.  To determine whether the BMC is finished starting up,
             run ``nova console-log bmc``.  The BMC service outputs a
             message like "Managing instance [uuid]" when it is fully
             configured.  There should be one of these messages for each
             baremetal instance.

   ::

      heat stack-show baremetal

#. Boot a VM to serve as the undercloud::

    nova boot undercloud --flavor m1.large --image centos7 --nic net-id=[tenant net uuid] --nic net-id=[provisioning net uuid]
    neutron floatingip-create [external net uuid]
    neutron port-list
    neutron floatingip-associate [floatingip uuid] [undercloud instance port id]

#. Build a nodes.json file that can be imported into Ironic::

    bin/build-nodes-json
    scp nodes.json centos@[undercloud floating ip]:~/instackenv.json

   .. note:: ``build-nodes-json`` also outputs a file named ``bmc_bm_pairs``
             that lists which BMC address corresponds to a given baremetal
             instance.

#. The undercloud vm can now be used with something like TripleO
   to do a baremetal-style deployment to the virtual baremetal instances
   deployed previously.

#. If using the full network isolation provided by OS::OVB::BaremetalNetworks
   then the overcloud can be created with the network templates in
   the ``network-templates`` directory.
