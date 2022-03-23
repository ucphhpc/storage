JBOD Setup
==========

This describes how to setup multipath on JBODs, in a way that utilizes the bandwidth of all
paths. Examples are based on two Dell ME484 JBODs with 162 HDDs and 6 SSDs. The disks are
named into 9 18x HHD raid sets on two servers. 

Multipath
---------

Disks with multiple SAS paths to the host will show as two seperate /dev/sdX
devices. The device used will define which path is used. Selecting which paths
are used can double the bandwidth to the JBODs.

We still want failover in case of a failing path, so Linux multipath is the way to go.
Multipath has no smart way of defining how disks paths are selected.
I.e. defining which path a disk slot should use by default. The only know way is to
create an /etc/multipath.conf config that specifies which sdX should be the prefered path.

Multipath also makes it posible to name disks in a way that makes sense for the use case
(i.e. RAID sets). For this we have a script that creates a multipath.conf based on a config
defining target JBODs and a mapping from disk slots to a disk name and prefered path. This
is an example of the created mapping in /etc/multipath.conf::

        multipath {
                uid 0
                gid 0
                mode 0600
                wwid "35000039af8182611"
                alias d3r4s0-e1s20-data
                prio "weightedpath"
                prio_args "devname sdik 50 sdu 1 "
        }

'alias' defines the name and 'prio_args' which sdX has priority. There are details in the man
page of multipath.conf.

Check the documentation of your Linux distribution for inforamtion about setting up multipath.

Script Config
-------------

The script needs sg3_utils, which is a package of the same name in CentOS 7.

The config for generating multipath.conf is by default located in /etc/jbod/jbod.conf. It contains
a list of WWIDs of the target JBOD enclosures and a function mapping enclosure number and disk slot to a disk name
and a path number. Example config::

.. highlight:: bash

 encs=(500c0ff0f23e583c 500c0ff0f23e5b3c)

 # name_path(enc, slot) -> name, path
 function name_path() {
   local enc="$1"
   local slot="$2"
   local type drawer row col disk raid server path s2

   type="data"

   let drawer=slot/42      # Drawer number in enclosure
   let row=(slot%42)/14    # Row number in drawer. Each have 3 rows of 14 disks.
   let col=slot%14         # Column in drawer. I.e. disk 0,14 and 28 are column 0.

   # Transpose to a column first layout.
   let s2=6*col+3*drawer+row 

   # Set server, disk and raid number for HDDs (type=data).
   if [ "$s2" -lt "81" ]; then
     let server=s2/45
     let disk=s2%9*2+enc
     let raid=(s2%45)/9
     let path=(s2+enc)%2
   # The same for SSDs (type=meta), which should be placed in last column of lower drawers.
   else
     type="meta"
     let s2=s2-81
     let server=(s2+enc)%2
     let disk=s2
     let raid=0
     let path=enc    
   fi

   # Print disk name and path
   echo "d${disk}r${raid}s${server}-e${enc}s${slot}-${type}" "$path" 
 }

The script is called 'jbod' and should be placed somewhere in the PATH (i.e. /usr/local/sbin), together
with a helper script called 'disklist'. The script can be used to find the JBOD WWIDs that need to go
into the 'encs' variable::

# jbod enc list
ID                  VENDOR              MODEL
0x500c0ff0f23e583c  DellEMC             Array584EMM
0x500c0ff0f23e5b3c  DellEMC             Array584EMM

When jbod.conf is ready, the multipath.conf can be created with::

# jbod multipath > /etc/multipath.conf

After this, run the following to apply the new config::

# multipath -r
# systemctl restart multipathd

Unfortunately, The config have to be rebuild after reboot, because sdX names can change. It is on the TODO
to move this into a service, but for now it can be done in /etc/rc.local.
