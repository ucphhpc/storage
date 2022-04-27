ZFS setup for Lustre
====================

For Lustre a supported version of ZFS is required. This is not always available from the main ZFS repos, because they don't build older versions.

Script for building ZFS 0.7.13
------------------------------

The *build-zfs-0.7* script builds ZFS 0.7.X RPM packages that can be installed manually or put into a yum repo (see *man createrepo* for more). The script is run with two parameters; Where to build and what version to build. I.e.::

 # ./build-zfs-0.7 /dev/shm 0.7.13

After this completes, the RPMs can be found in */dev/shm/spl/\*.rpm* and */dev/shm/zfs/\*.rpm*. The required packages need to be installed. If put into a yum repo a simple *yum install zfs* will install the zfs packages an all the dependencies.


Create hostid
-------------

ZFS has what's called `multihost`_. This ensures that hosts that have access to the same disks cannot import pools simultaniouly. For this to work they need a unique hostid. ZFS has a tool to generate that::

 # zgenhostid

It need to be run on each OSS and MDS.

.. _multihost: https://wiki.lustre.org/Protecting_File_System_Volumes_from_Concurrent_Access

Create ZFS pools
----------------

Creating ZFS pools for Lustre is just like creating normal ZFS pools. In these examples we will create a pool using the multipath block devices defined when setting up multipath for the JBODs.

OST pools
~~~~~~~~~

Here we create a raidz2 pool based on the disk mappings from multipath::

 # zpool create oss00p0 raidz2 /dev/mapper/d?r0s0-*-data /dev/mapper/d1?r0s0-*-data
 # zpool create oss00p1 raidz2 /dev/mapper/d?r1s0-*-data /dev/mapper/d1?r1s0-*-data

 # zpool create oss01p0 raidz2 /dev/mapper/d?r0s1-*-data /dev/mapper/d1?r0s1-*-data
 # zpool create oss01p1 raidz2 /dev/mapper/d?r1s1-*-data /dev/mapper/d1?r1s1-*-data

Note that we here expect more than 10 drive and we want the listed in order, hence the two */dev/mapper/...*.

Now we want to configure the default properties on each pool. Information about these can be found in *man zfs* in the *Native Properties* section. This will setup the *oss00p0* pool::

 # for i in compression=lz4 redundant_metadata=most xattr=sa recordsize=1m dnodesize=auto secondarycache=metadata; do
     zfs set $i oss00p0
   done

Metadata cache devices
^^^^^^^^^^^^^^^^^^^^^^

The SSDs /dev/mapper/\*-meta will for now be used for caching metadata (secondarycache=metadata). Each ZFS pool needs at leach on block device for this. If there is not enough SSDs they need to be partitioned to make enougth. Example adding a cache device to a pool::

 zpool add oss00p0 cache /dev/mapper/d0r0s0-*-meta1

Make sure the *cache* keyword is included, otherwise you could add it as a regular data device, which cannot be removed. A cache device can be removed.

MDT pool
~~~~~~~~

Here we create a pool consisting of 2 times 3 drive mirrors::

 # zpool create -o ashift=9 mds00p0 mirror /dev/mapper/d[0-2]r0s0-*
 # zpool add -o ashift=9 mds00p0 mirror /dev/mapper/d[0-2]r1s0-*

So the mapping should have set of 3 SSDs for each mirror. The *ashift=9* option forces ZFS to use 512 bytes blocks.
Otherwise, it would use the physical block size of the disks, which for SSDs often is 4k. However, with 4k blocks the space overhead is huge.

Now we want to configure the default properties::

 # for i in xattr=sa dnodesize=auto; do
     zfs set $i oss00p0
   done

ZFS tuning
----------

ZFS has a lot knobs you can tune. For Lustre there is a wiki with `ZFS Tuning`_ sugestions. These are not maintained, so we have added our ZFS 0.7.13 */etc/modprobe.d* files for OSS and MDS in the *etc* dir. These denpends on the specific system and might need retuning for newer ZFS versions.

.. _ZFS Tuning: https://wiki.lustre.org/Category:ZFS_OSD_Tuning