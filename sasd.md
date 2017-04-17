# TRIM(Discard) Support

## Introduction
A Trim command (known as TRIM in the ATA command set, and UNMAP in the SCSI
command set) allows an operating system to inform underlying storage driver
which blocks of data are no longer considered in use and can be wiped
internally.

Solid-state, flash-based storage devices are getting larger and cheaper, to
the point that they are starting to displace rotating disks in an increasing
number of systems. While flash requires less power, makes less noise, and is
faster, it has some peculiar quirks of its own. Due to the architecture of
SSDs, continuous use results in degraded performance if not accounted for
and mitigated. The TRIM command is an operation that allows the operating
system to propagate information down to the SSD about which blocks of data
are no longer in use. This allows the SSD's internal systems to better manage
wear leveling and prepare the device for future writes. Wear leveling is a
process that is designed to extend the life of solid state storage devices.
Solid state storage is made up of microchips that store data in blocks.
Each block can tolerate a finite number of program/erase cycles before becoming
unreliable. Wear leveling arranges data so that write/erase cycles are
distributed evenly among all of the blocks in the device. Trim can have a major
impact on the device's performance over time and its overall longevity.

## Enable Trim requests on block device
For enabling trim request on the block device, QUEUE_FLAG_DISCARD has to be set
while initializing device request queue in the driver. Once this is done device
is ready to handle trim requuse from the file system.

Now we need some way to communicate to the file system that underlying device is
ready to handle trim requests. There are two ways by which we can achieve this-

* Mount a file system on a device with -o discard option. By doing this we would
get trim requests on device in real-time, whenever a file, directory, symlink
etc is been removed from that device.

```
mount –o discard <block device path> <mountpoint>
```

* On the other hand trim request can be issued in batch using 'fstrim'. It is
used on a mounted filesystem to trim (or discard) blocks which are not in use
by the filesystem. By default, it will discard all unused blocks in the
filesystem. Options may be used to modify this behavior based on range or size.

```
fstrim [-o offset] [-l length] [-m minimum-free-extent] [-v] <mountpoint>
```

Running fstrim frequently, or even using mount -o discard, might negatively
affect the lifetime of poor-quality SSD devices. For most desktop and server
systems a sufficient trimming frequency is once a week.

## Trim support with Starling
In starling, the actual user data is stored in the key-value store. Ctree is
in memory and on disk starling's structure which maps the logical offset of
the starling device to the key. Using this key, the data is read and written
in the KV store. Key is represented by CtLeaf structure. Each leaf node can
represent user data between chunk size(64KB default) and the max chunk
size(64KB * #num entries in leaf node). The leaf node entry will be for the
start chunk index in case entry spans more than a chunk.

File system can issue minimum trim request of 1 page(4KB). Since a leaf node is
having 16 pages(64KB/4KB) and there may be case where trim request came for
partial chunk(i.e. for < 16 pages). For handling all such scenarios we need
to have some mechanism for tracking information, for the set of pages of leaf
got the trim request. Once we got the trim request for the entire chunk we can
delete data represented by that leaf from the KV store. For maintaining crash
consistency the SSDLog entry is also added for each broken trim request.

For tracking such information we would be using a bitmap of size 16 bits(since a
leaf node can have maximum of 16 pages). Each bit would be representing 1 page
inside a chunk. A set bit of bitmap indicates that trim request came on that
particular page. If a leaf entry spans more than a chunk then trim information
would be kept on each leaf node.

e.g.
* If a leaf node covers 64KB data and trim request came for page 0, 2 and 7 then,
we would be deleting data once we would be getting trim request for all
remaining pages of this leaf node.

* If a leaf node covers 128KB data then this entry would span over two chunks. Now
consider trim request came for entire chunk-2, we can’t delete this data until
we get trim request for chunk-1 also.


## Snapshot/clone handling:
TBD

## References
[1] Block layer discard requests
[link](https://lwn.net/Articles/293658/)

[2] How To Configure Periodic TRIM for SSD Storage on Linux Servers
[link](https://www.digitalocean.com/community/tutorials/how-to-configure-periodic-trim-for-ssd-storage-on-linux-servers)
