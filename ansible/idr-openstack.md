# OpenStack Installation

These instructions were last tested with OpenStack Liberty, February 2016:

    $ packstack --version
    packstack Liberty 7.0.0.dev1682.g42b3426

Run `idr-initial.yml` and any other playbooks required for setting up the system such as `idr-gpfs-client.yml`. Run `idr-openstack.yml`. Reboot the system to ensure everything is up-to-date.

Log in as a normal user (you do not need to be `root`).

Setup ssh keys so `ssh root@localhost` works without a password (you may wish to delete these keys later):

    ssh-keygen
    sudo sh -c "cat $HOME/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys"

The general plan:
1. Create a near-default allinone setup
2. Reconfigure Cinder to use gpfs instead of a loopback device
3. Configure external network bridge

Create a single-node allinone setup, excluding external network configuration and (optionally) without swift object storage. Enable self-signed SSL for the Horizon web interface so that it's easy to modify with real SSL certificates.

    packstack --allinone --os-horizon-ssl=y --os-swift-install=n --provision-demo=n

This will take a while. Packstack uses Puppet to deploy OpenStack, so can be run multiple times during installation using the generated answers file, however it may revert any manual configuration changes so take care before running this on an operational OpenStack instance.

    packstack --answer-file packstack-answers-YYYYMMDD-hhmmss.txt


# Optional Configuration

OpenStack installs the `crudini` utility which can be used to edit ini files.

## Optional: GPFS

See http://docs.openstack.org/liberty/config-reference/content/GPFS-driver-options.html for the full list of options.

Create a directory for the Cinder volumes and images (as root):

    OS_GPFS=/gpfs/openstack
    mkdir -p $OS_GPFS/{block,images}

Reconfigure Cinder to use GPFS (as root):

    CINDER_CONF=/etc/cinder/cinder.conf
    cp $CINDER_CONF $CINDER_CONF.bak

    crudini --set $CINDER_CONF DEFAULT enabled_backends gpfs

    crudini --set $CINDER_CONF gpfs gpfs_images_dir $OS_GPFS/images
    crudini --set $CINDER_CONF gpfs gpfs_images_share_mode copy_on_write
    crudini --set $CINDER_CONF gpfs gpfs_mount_point_base $OS_GPFS/block
    crudini --set $CINDER_CONF gpfs gpfs_sparse_volumes True
    crudini --set $CINDER_CONF gpfs volume_driver cinder.volume.drivers.ibm.gpfs.GPFSDriver

Optionally set an alternative storage pool

    crudini --set $CINDER_CONF gpfs gpfs_storage_pool system

The current version of packstack has a bug https://bugzilla.redhat.com/show_bug.cgi?id=1272572

    crudini --set $CINDER_CONF keystone_authtoken auth_uri http://localhost:5000/

Restart Cinder:

    systemctl restart openstack-cinder-api
    systemctl restart openstack-cinder-scheduler
    systemctl restart openstack-cinder-volume


#openstack-config --set /etc/cinder/cinder.conf keystone_authtoken auth_uri http://localhost:5000/
#systemctl restart openstack-cinder-api

# https://www.rdoproject.org/networking/neutron-with-existing-external-network/

#openstack-config --set /etc/neutron/plugins/ml2/openvswitch_agent.ini ovs bridge_mappings extnet:br-ex
#openstack-config --set /etc/neutron/plugin.ini ml2 type_drivers vxlan,flat,vlan
#for s in network neutron-openvswitch-agent neutron-server ; do systemctl restart $s; done

Setup login credentials for the OpenStack API

    source keystonerc_admin


#. keystonerc_admin
#neutron net-create external_network --provider:network_type flat --provider:physical_network extnet  --router:external --shared

#neutron subnet-create --name public_subnet --enable_dhcp=False --allocation-pool=start=10.0.0.64,end=10.0.0.127 --gateway=10.0.0.254 external_network 10.0.0.0/24
#neutron router-create router1
#neutron router-gateway-set router1 external_network

#neutron net-create private_network
#neutron subnet-create --name private_subnet private_network 192.168.100.0/24

#neutron router-interface-add router1 private_subnet

#curl http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img | glance image-create --name='cirros image' --visibility public --container-format=bare --disk-format=qcow2

#keystone tenant-create --name internal --description "internal tenant" --enabled true
#keystone user-create --name internal --tenant internal --pass "ome" --email test@example.org --enabled true

#curl http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1601.qcow2 | glance image-create --name='CentOS 7 x86_64 1601' --visibility public --container-format=bare --disk-format=qcow2
# Note for GPFS use raw images so COW works

#

# Adding more nodes
# https://www.rdoproject.org/install/adding-a-compute-node/
