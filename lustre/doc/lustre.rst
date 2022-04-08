Lustre server and client setup
==============================

This is describe a very specific Lustre setup. Detailed information can be found in `Lustre manual`_ and `Lustre wiki`_

.. _Lustre manual: https://doc.lustre.org/lustre_manual.xhtml
.. _Lustre wiki: https://wiki.lustre.org

Bulding Lustre
--------------


Setting up build environment
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Install the following packages::

 yum -y groupinstall "Development Tools"
 yum -y install libselinux-devel kernel-devel
 yum -y install net-snmp-devel
 yum -y install python-docutils

 # RHEL 7
 yum -y install libyaml-devel
 # RHEL 8
 dnf --enablerepo=powertools -y install libyaml-devel
 dnf -y install kernel-rpm-macros kernel-abi-whitelists

For server build the ZFS develeopment packages are also required. These can be installed with yum if you created a ZFS repo::

 yum -y install libzfs2-devel libzfs2-devel kmod-zfs-devel

Build
~~~~~

Create a build user, clone and configure the Lustre git repo::

 useradd -m build
 su - build

 git clone git://git.whamcloud.com/fs/lustre-release.git
 cd lustre-release

 release=2.12.8
 git checkout $release
 sh ./autogen.sh

The *release* variable sets the version you want to build.

Configure server::

 ./configure --disable-ldiskfs --with-zfs --enable-quota --enable-utils --enable-gss --enable-snmp

Configure client for OS provided RDMA drivers::

  ./configure --disable-server --enable-quota --enable-utils --enable-gss 

For Mellanox OFED (RDMA)::

 ./configure --disable-server --enable-quota --enable-utils --enable-gss --with-o2ib=/usr/src/ofa_kernel/default/

Start the build::

 make rpms

Now, the current directory will contain all the RPMs (*\*.rpm*). Not all are needed in the normal case, but it is recommended to put them in a yum repo.

Install Lustre
--------------

If you created yum repos for ZFS and Lustre this is simply a yum install. For server::

 # yum install lustre

For client::

 yum install lustre-client

Manually installing the RPMs is also posible, but then you need to install all package dependencies as well.

Setup Lustre networking
-----------------------

The `Lustre manual`_ has details about networking, but for this setup based on TCP this simple config file in */etc/lnet.conf* will do::

 net:
     - net type: tcp0
       local NI(s):
         - nid: <IP>@tcp0
           status: up
           interfaces:
               0: <Interface>
           tunables:
               peer_credits: 63
               credits: 2560

<IP> needs to be replaced with the IP of the host and <Inteface> with the network interface (i.e. eth0).

Then we just need to enable LNet::

 systemctl enable lnet
 

Creating Lustre targets
-----------------------

We will use two OSS server pairs and an MDS server pair in this example. We plan to make 2 separate filesystems. The pairs mds[00-01], oss[00-01] and oss[02-03] will have thfollowing IPs and ZFS pools::

 mds00:
   ip: 10.10.10.10
   zfs pools: mds00p0
 mds01:
   ip: 10.10.10.11
   zfs pools: mds01p0

 oss00:
   ip: 10.10.10.100
   zfs pools: oss00p[0-4]

 oss01:
   ip:10.10.10.101
   zfs pools: oss01p[0-3]

 oss02:
   ip: 10.10.10.102
   zfs pools: oss02p[0-4]

 oss03:
   ip:10.10.10.103
   zfs pools: oss03p[0-3]

Let us call the two filesystems erda and sif. The ZFS pools to filesystems mapping will be::

 erda: mds00p0, oss00p[0-4], oss01p[0-3], oss02p[0-4]
 sif: mds01p0, oss03p[0-3]

The OST pools can be mapped however you like, but keeping things separate can be an advantage if posible.

First we create the MGSes, which store information about the cluster. An MGS can service more filesystems, but to avoid this dependence we will create two. There can only be one active MGS per server, so they should not be on the same pair. Therefor we choose the OSS, but one could be on an MDS::

 oss00# mkfs.lustre --backfstype=zfs --mgs --servicenode=10.10.10.100@tcp0:10.10.10.101@tcp0 oss00p0/mgs-erda
 oss03# mkfs.lustre --backfstype=zfs --mgs --servicenode=10.10.10.103@tcp0:10.10.10.102@tcp0 oss02p0/mgs-sif

The option *servicenode* defines where the MGS can be hosted. First it the normal location and second is the failover location.

Next we create an MDT for each filesystem::

 mds00# mkfs.lustre --backfstype=zfs --mdt --fsname erda --index=0 \
   --mgsnode=10.10.10.100@tcp0:10.10.10.101@tcp0 \
   --servicenode=10.10.10.10@tcp0:10.10.10.11@tcp0 \
   mds00p0/erda0

 mds01# mkfs.lustre --backfstype=zfs --mdt --fsname erda --index=0 \
   --mgsnode=10.10.10.103@tcp0:10.10.10.102@tcp0 \
   --servicenode=10.10.10.11@tcp0:10.10.10.10@tcp0 \
   mds01p0/erda0

The option *mgsnode* defines which nodes the MGS can be located at.

The OSTs follow a simular format::

 oss00# mkfs.lustre --backfstype=zfs --ost --fsname erda --index=0 \
   --mgsnode=10.10.10.100@tcp0:10.10.10.101@tcp0 \
   --servicenode=10.10.10.100@tcp0:10.10.10.101@tcp0 \
   oss00p0/erda0

 oss01# mkfs.lustre --backfstype=zfs --ost --fsname erda --index=5 \
   --mgsnode=10.10.10.100@tcp0:10.10.10.101@tcp0 \
   --servicenode=10.10.10.101@tcp0:10.10.10.100@tcp0 \
   oss01p0/erda5

 oss02# mkfs.lustre --backfstype=zfs --ost --fsname erda --index=9 \
   --mgsnode=10.10.10.100@tcp0:10.10.10.101@tcp0 \
   --servicenode=10.10.10.102@tcp0:10.10.10.103@tcp0 \
   oss02p0/erda9

 oss03# mkfs.lustre --backfstype=zfs --ost --fsname sif --index=0 \
   --mgsnode=10.10.10.103@tcp0:10.10.10.102@tcp0 \
   --servicenode=10.10.10.103@tcp0:10.10.10.102@tcp0 \
   oss03p0/sif0

This is only for the first pool on each server. The rest should be done in the same way, with unique indexes for each target on each filesystem.

Configuratation
---------------

A configuration mapping hosts to targets need to be created. Due to the MGS separation this cannot be the same file on all servers. For the example this is the basis to put in */etc/ldev.conf*::

 mds00 mds01 erda-MDT0000 zfs:mds00p0/erda0
 mds01 mds00 sif-MDT0000 zfs:mds01p0/sif0

 oss00 oss01 erda-OST0000 zfs:oss00p0/erda0
 oss00 oss01 erda-OST0001 zfs:oss00p1/erda1
 oss00 oss01 erda-OST0002 zfs:oss00p2/erda2
 oss00 oss01 erda-OST0003 zfs:oss00p3/erda3
 oss00 oss01 erda-OST0004 zfs:oss00p4/erda4

 oss01 oss00 erda-OST0005 zfs:oss01p0/erda5
 oss01 oss00 erda-OST0006 zfs:oss01p1/erda6
 oss01 oss00 erda-OST0007 zfs:oss01p2/erda7
 oss01 oss00 erda-OST0008 zfs:oss01p3/erda8

 oss02 oss03 erda-OST0009 zfs:oss02p0/erda9
 oss02 oss03 erda-OST000A zfs:oss02p1/erdaA
 oss02 oss03 erda-OST000B zfs:oss02p2/erdaB
 oss02 oss03 erda-OST000C zfs:oss02p3/erdaC

 oss03 oss02 sif-OST0000 zfs:oss03p0/sif0
 oss03 oss02 sif-OST0001 zfs:oss03p0/sif1
 oss03 oss02 sif-OST0002 zfs:oss03p0/sif2
 oss03 oss02 sif-OST0003 zfs:oss03p0/sif3

On oss[00-01] add this::

 oss00 oss01 MGS zfs:oss00p0/mgs-erda

On oss[02-03] add this::

 oss02 oss03 MGS zfs:oss02p0/mgs-sif

TODO:

start Lustre (server, client)

modprobe.d

tuned-adm

