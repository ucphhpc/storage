JBOD Setup
==========

Considering the overview in the diagram below::
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
