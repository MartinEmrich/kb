# Manual setup of a Ceph Bluestore OSD

As `ceph-deploy` or `ceph-disk` had some restrictions, and I just want to know as much of the under-the-hood-stuff as possible, I documented here how to create a Ceph Bluestore OSD without these convenience tools.

I am running
* CentOS 7
* Ceph "luminous" 12.1 RC1

I assume that your cluster (including monitors, manager and stuff) is already up and running.

## Bluestore components

A Bluestore OSD uses several storage components:

* A small data partition, where configuration files, keys and UUIDs reside. `ceph-deploy` allocates 100MB, so that's how we are rolling...
* `block.wal`: The Write-Ahead-Log (WAL). No idea on the sizing, but I'll go with 8GB here. *
* `block.db`: The RocksDB. Again no idea for a reasonable size, so 4GB it is. *
* `block`: The actual block storage for your objects and data. Have a 8TB disk here.

\*) A nice person from the _ceph-users_ mailing list had 1000MB for DB and 600MB for WAL, so I split my 12GB volume into 1/3 and 2/3.

## Prepare the block devices

Partition your big storage disk. Create a small 100MB partition and create an XFS on it, e.g.:

    mkfs -t xfs -f -i size=2048 -- /dev/sdc1

Allocate all the rest for the second partition. There, no filesystem is necessary (that's the point in Bluestore).
The WAL and DB volumes (running here on a smaller but fast SSD) also need no prior treatment. With this tutorial, they can be whole disks, partitions or LVM logical volumes.

## Create OSD

with

    ceph osd create

You create a new OSD. The return value is the new OSD number, starting from 0 for your first OSD. You'll need it later.

## Prepare the data directory

The data directory is usually mounted to /var/lib/ceph/osd/_clustername_-_number_
So if your cluster is named "ceph" (the default), and you got the OSD number 0, permanently mount the data directory to `/var/lib/ceph/osd/ceph-0`.

In there, create:
* A file named `type`, containing only the word `bluestore` with trailing newline. This informs `ceph-osd` that this is in fact a bluestore, not a filestore based OSD.
* A symlink named `block` to your block storage
* A symlink named `block.wal` to your WAL block device
* A symlink named `block.db` to your RocksDB device

Now make sure that **everything** is owned by the *ceph* user, including the block devices themselves (not just the symlinks). 
Add an udev rule (in `/etc/udev/rules.d`) like:

```
KERNEL=="sda*", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", OWNER="ceph", GROUP="ceph", MODE="0660"
KERNEL=="sdb*", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", OWNER="ceph", GROUP="ceph", MODE="0660"
KERNEL=="sdc*", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", OWNER="ceph", GROUP="ceph", MODE="0660"
KERNEL=="sdd*", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", OWNER="ceph", GROUP="ceph", MODE="0660"
KERNEL=="sde*", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", OWNER="ceph", GROUP="ceph", MODE="0660"
KERNEL=="sdf*", SUBSYSTEM=="block", ENV{DEVTYPE}=="partition", OWNER="ceph", GROUP="ceph", MODE="0660"
ENV{DM_LV_NAME}=="osd-*", OWNER="ceph", GROUP="ceph", MODE="0660"
```

This changes the ownership of the relevant block devices to ceph:ceph after reboot.

This should look similar to this:

    # ls -l /var/lib/ceph/osd/ceph-0

    lrwxrwxrwx 1 ceph ceph  9  7. Jul 11:20 block -> /dev/sdc2
    lrwxrwxrwx 1 ceph ceph 20  7. Jul 11:20 block.db -> /dev/ceph/osd-sdc-db
    lrwxrwxrwx 1 ceph ceph 21  7. Jul 11:20 block.wal -> /dev/ceph/osd-sdc-wal
    -rw-r--r-- 1 ceph ceph 10  7. Jul 11:20 type

## Create the Bluestore

Now the tense moment, create the bluestore (insert your OSD number for 0):

    ceph-osd --setuser ceph -i 0 --mkkey --mkfs

This creates all the metadata files, "formats" the block devices, and also generates a Ceph auth key for your new OSD.

## Configure OSD

Now import the new key into your ceph keystore:

    ceph auth add osd.0 osd 'allow *' mon 'allow rwx' mgr 'allow profile osd' -i /var/lib/ceph/osd/ceph-0/keyring

Also insert the new OSD into your CRUSH map, so that it is actually being used.
Although you surely know your CRUSH map if you are reading this, here is an example:

    ceph --setuser ceph osd crush add 0 7.3000 host=`hostname`

(Marketing says it's a 8TB disk, but I got only 7.3TB ;)

## Start OSD

    systemctl enable ceph-osd@0
    systemctl start ceph-osd@0

Now the OSD is running.
