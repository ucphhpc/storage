Lustre server and client setup
==============================


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

