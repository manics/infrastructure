Useful GPFS notes
=================

This is a summary of some useful GPFS concepts and commands.
Before doing any administrative work on a GPFS system you should consult the official docs.
- http://www.ibm.com/support/knowledgecenter/STXKQY/ibmspectrumscale_welcome.html

Some commands can be run as root from any node in the GPFS cluster including clients.
Depending on how the cluster is configured some commands can only be run from a limited number of administrative nodes.

Note: be careful about copying commands from the IBM documentation, as some characters may have been replaced by other unicode characters (such as dashes `-`).


GPFS Filesets
-------------

A GPFS fileset is a subtree of a GPFS file-system that can be managed separately.
For example, policies can be set so that data is only stored on a subset of disks for performance, and (read-only) snapshots can be created of the subtree.
If you want to use snaphots you must create the fileset with the `--inode-space new` parameter, and you may want to specify the maximum number of inodes using `‐‐inode‐limit MaxNumInodes`.
A larger number of inodes requires more metadata space, so it should not be too large.
- http://www.ibm.com/support/knowledgecenter/STXKQY_4.1.1/com.ibm.spectrum.scale.v4r11.adv.doc/bl1adv_filesets.htm

For example, if you have a GPFS filesystem called `gpfs-1` mounted on `/data` and you want to create a new fileset called `downloads` with an initial maximum of 100k inodes:

    mmcrfileset gpfs-1 downloads --inode-space new --inode-limit 100k
    mmlinkfileset gpfs-1 downloads -J /data/downloads

You can increase the maximum number of inodes after creating the fileset:

    mmchfileset gpfs-1 downloads ‐‐inode‐limit 1m

Note it is not possible to convert an existing directory into a fileset.

To list existing filesets:

    mmlsfileset gpfs-1

Add `-L` to see additional information include any inode limits.

To delete a fileset:

    mmunlinkfileset gpfs-1 downloads
    mmdelfileset gpfs-1 downloads

If the fileset was not empty pass `-f` to `mmdelfileset`.


GPFS Snapshots
--------------

GPFS snapshots are **read-only** snapshots of either the entire GPFS filesystem, or a single fileset.
They are useful for taking consistent backups of an subtree, or for tagging versions of a directory for future reference.
GPFS uses a form of copy-on-write so is not the same as a backup.
- http://www.ibm.com/support/knowledgecenter/STXKQY_4.1.1/com.ibm.spectrum.scale.v4r11.adv.doc/bl1adv_logcopy.htm
- http://www.ibm.com/support/knowledgecenter/STXKQY_4.1.1/com.ibm.spectrum.scale.v4r11.adv.doc/bl1adv_fslevelsnaps.htm%23bl1adv_fslevelsnaps

To create a snapshot of the `downloads` fileset called `downloads-0.1`:

    mmcrsnapshot gpfs-1 downloads-0.1 -j downloads

To list all snapshots:

    mmlssnapshot gpfs-1

To delete a snapshot:

    mmdelsnapshot gpfs-1 downloads-0.1 -j downloads


Setting up cluster export services (CES) such as NFS
----------------------------------------------------

Cluster export services (CES) uses a virtual IP address to provide automatic failover between multiple file servers (NFS, Samba, etc) in a GPFS cluster.
This virtual IP should be dedicated to the use of CES, must be distinct from existing IP addresses, and must also have a reverse DNS record.

See http://www.ibm.com/support/knowledgecenter/STXKQY_4.1.1/com.ibm.spectrum.scale.v4r11.adv.doc/bl1adv_ces.htm.
In particular
- [CES cluster setup](http://www.ibm.com/support/knowledgecenter/STXKQY_4.1.1/com.ibm.spectrum.scale.v4r11.adv.doc/bl1adv_cesclustersetup.htm)
- [CES network configuration](http://www.ibm.com/support/knowledgecenter/STXKQY_4.1.1/com.ibm.spectrum.scale.v4r11.adv.doc/bl1adv_cesnetworkconfig.htm)

See also `man mmces`, it includes some useful examples.

Some of these commands must be run from an admin node, so it's best to run all these commands from an admin node.
To setup a CES cluster:

1. Create a shared GPFS directory for CES data, optionally in a new filesystem or fileset `mkdir /data/ces-data` or `mmcrfileset ...; mmlinkfileset ...`
2. Configure this directory `mmchconfig cesSharedRoot=/data/ces-data`
3. Enable CES on one or more nodes `mmchnode --ces-enable -N gpfs-node1,...`
4. Optionally place all CES nodes into a new class (e.g. called `protocol`) for convenience `mmcrnodeclass protocol -N gpfs-node1,...`
5. Add the CES IPs to the CES address pool `mmces address add --ces-ip A.B.C.D` (if you get an error `Incorrect value for --ces-ip option` check that there is a corresponding reverse DNS record)
6. Enable the service `mmces service enable NFS` (other protocols are available including `SMB` and `OBJ`)
6. Start the service `mmces service start NFS -a` (similarly for other protocols)
7. Check services are started `mmces service list -a`

Once the CES cluster is created you can create some shares.
Note that NFS shares **must** be configured using GPFS commands such as `mmnfs`, and not the standard Linux NFS commands.

If the service did not start you may need to configure authentication `mmuserauth service create --data-access-method file --type ...`

To setup an NFS export: `mmnfs export add /data/shared --client '10.0.0.0/24(Access_Type=RO,Squash=all_squash)'`

Note if you are modifying an existing CES configuration you may need to shutdown GPFS on the CES nodes `mmshutdown -N protocol`
