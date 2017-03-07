Deploying a Standalone Baremetal Stack
======================================

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
