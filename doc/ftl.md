# Flash Translation Layer {#ftl}

The Flash Translation Layer library provides block device access on top of devices
implementing bdev_zone interface.
It handles the logical to physical address mapping, responds to the asynchronous
media management events, and manages the defragmentation process.

## Terminology {#ftl_terminology}

### Logical to physical address map

- Shorthand: L2P

Contains the mapping of the logical addresses (LBA) to their on-disk physical location. The LBAs
are contiguous and in range from 0 to the number of surfaced blocks (the number of spare blocks
are calculated during device formation and are subtracted from the available address space). The
spare blocks account for zones going offline throughout the lifespan of the device as well as
provide necessary buffer for data [defragmentation](#ftl_reloc).

### Band {#ftl_band}

A band describes a collection of zones, each belonging to a different parallel unit. All writes to
a band follow the same pattern - a batch of logical blocks is written to one zone, another batch
to the next one and so on. This ensures the parallelism of the write operations, as they can be
executed independently on different zones. Each band keeps track of the LBAs it consists of, as
well as their validity, as some of the data will be invalidated by subsequent writes to the same
logical address. The L2P mapping can be restored from the SSD by reading this information in order
from the oldest band to the youngest.

             +--------------+        +--------------+                        +--------------+
    band 1   |   zone 1     +--------+    zone 1    +---- --- --- --- --- ---+     zone 1   |
             +--------------+        +--------------+                        +--------------+
    band 2   |   zone 2     +--------+     zone 2   +---- --- --- --- --- ---+     zone 2   |
             +--------------+        +--------------+                        +--------------+
    band 3   |   zone 3     +--------+     zone 3   +---- --- --- --- --- ---+     zone 3   |
             +--------------+        +--------------+                        +--------------+
             |     ...      |        |     ...      |                        |     ...      |
             +--------------+        +--------------+                        +--------------+
    band m   |   zone m     +--------+     zone m   +---- --- --- --- --- ---+     zone m   |
             +--------------+        +--------------+                        +--------------+
             |     ...      |        |     ...      |                        |     ...      |
             +--------------+        +--------------+                        +--------------+

              parallel unit 1              pu 2                                    pu n

The address map and valid map are, along with a several other things (e.g. UUID of the device it's
part of, number of surfaced LBAs, band's sequence number, etc.), parts of the band's metadata. The
metadata is split in two parts:

       head metadata               band's data               tail metadata
    +-------------------+-------------------------------+------------------------+
    |zone 1 |...|zone n |...|...|zone 1 |...|           | ... |zone  m-1 |zone  m|
    |block 1|   |block 1|   |   |block x|   |           |     |block y   |block y|
    +-------------------+-------------+-----------------+------------------------+

- the head part, containing information already known when opening the band (device's UUID, band's
  sequence number, etc.), located at the beginning blocks of the band,
- the tail part, containing the address map and the valid map, located at the end of the band.

Bands are written sequentially (in a way that was described earlier). Before a band can be written
to, all of its zones need to be erased. During that time, the band is considered to be in a `PREP`
state. After that is done, the band transitions to the `OPENING` state, in which head metadata
is being written. Then the band moves to the `OPEN` state and actual user data can be written to the
band. Once the whole available space is filled, tail metadata is written and the band transitions to
`CLOSING` state. When that finishes the band becomes `CLOSED`.

### Ring write buffer {#ftl_rwb}

- Shorthand: RWB

Because the smallest write size the SSD may support can be a multiple of block size, in order to
support writes to a single block, the data needs to be buffered. The write buffer is the solution to
this problem. It consists of a number of pre-allocated buffers called batches, each of size allowing
for a single transfer to the SSD. A single batch is divided into block-sized buffer entries.

                 write buffer
    +-----------------------------------+
    |batch 1                            |
    |   +-----------------------------+ |
    |   |rwb    |rwb    | ... |rwb    | |
    |   |entry 1|entry 2|     |entry n| |
    |   +-----------------------------+ |
    +-----------------------------------+
    | ...                               |
    +-----------------------------------+
    |batch m                            |
    |   +-----------------------------+ |
    |   |rwb    |rwb    | ... |rwb    | |
    |   |entry 1|entry 2|     |entry n| |
    |   +-----------------------------+ |
    +-----------------------------------+

When a write is scheduled, it needs to acquire an entry for each of its blocks and copy the data
onto this buffer. Once all blocks are copied, the write can be signalled as completed to the user.
In the meantime, the `rwb` is polled for filled batches and, if one is found, it's sent to the SSD.
After that operation is completed the whole batch can be freed. For the whole time the data is in
the `rwb`, the L2P points at the buffer entry instead of a location on the SSD. This allows for
servicing read requests from the buffer.

### Defragmentation and relocation {#ftl_reloc}

- Shorthand: defrag, reloc

Since a write to the same LBA invalidates its previous physical location, some of the blocks on a
band might contain old data that basically wastes space. As there is no way to overwrite an already
written block, this data will stay there until the whole zone is reset. This might create a
situation in which all of the bands contain some valid data and no band can be erased, so no writes
can be executed anymore. Therefore a mechanism is needed to move valid data and invalidate whole
bands, so that they can be reused.

                    band                                             band
    +-----------------------------------+            +-----------------------------------+
    | ** *    * ***      *    *** * *   |            |                                   |
    |**  *       *    *    * *     *   *|   +---->   |                                   |
    |*     ***  *      *            *   |            |                                   |
    +-----------------------------------+            +-----------------------------------+

Valid blocks are marked with an asterisk '\*'.

Another reason for data relocation might be an event from the SSD telling us that the data might
become corrupt if it's not relocated. This might happen due to its old age (if it was written a
long time ago) or due to read disturb (media characteristic, that causes corruption of neighbouring
blocks during a read operation).

Module responsible for data relocation is called `reloc`. When a band is chosen for defragmentation
or a media management event is received, the appropriate blocks are marked as
required to be moved. The `reloc` module takes a band that has some of such blocks marked, checks
their validity and, if they're still valid, copies them.

Choosing a band for defragmentation depends on several factors: its valid ratio (1) (proportion of
valid blocks to all user blocks), its age (2) (when was it written) and its write count / wear level
index of its zones (3) (how many times the band was written to). The lower the ratio (1), the
higher its age (2) and the lower its write count (3), the higher the chance the band will be chosen
for defrag.

## Usage {#ftl_usage}

### Prerequisites {#ftl_prereq}

In order to use the FTL module, a device capable of zoned interface is required e.g. `zone_block`
bdev or OCSSD `nvme` bdev.

### FTL bdev creation {#ftl_create}

Similar to other bdevs, the FTL bdevs can be created either based on JSON config files or via RPC.
Both interfaces require the same arguments which are described by the `--help` option of the
`bdev_ftl_create` RPC call, which are:

- bdev's name
- base bdev's name (base bdev must implement bdev_zone API)
- UUID of the FTL device (if the FTL is to be restored from the SSD)

## Configuring SPDK {#ftl_spdk_config}

To verify that the drive is emulated correctly, one can check the output of the NVMe identify app
(assuming that `scripts/setup.sh` was called before and the driver has been changed for that
device):

```
$ build/examples/identify
=====================================================
NVMe Controller at 0000:00:0a.0 [1d1d:1f1f]
=====================================================
Controller Capabilities/Features
================================
Vendor ID:                             1d1d
Subsystem Vendor ID:                   1af4
Serial Number:                         deadbeef
Model Number:                          QEMU NVMe Ctrl

... other info ...

```

## FTL usage with zone block bdev {#ftl_zone_block}

Zone block bdev is a bdev adapter between regular `bdev` and `bdev_zone`. It emulates a zoned
interface on top of a regular block device.

In order to create FTL on top of a regular bdev:
1) Create regular bdev e.g. `bdev_nvme`, `bdev_null`, `bdev_malloc`
2) Create zone block bdev on top of a regular bdev created in step 1 (user could specify zone capacity
and optimal number of open zones)
3) Create FTL bdev on top of bdev created in step 2

Example:
```
$ scripts/rpc.py bdev_nvme_attach_controller -b nvme0 -a 00:05.0 -t pcie
	nvme0n1

$ scripts/rpc.py bdev_zone_block_create -b zone1 -n nvme0n1 -z 4096 -o 32
	zone1

$ scripts/rpc.py bdev_ftl_create -b ftl0 -d zone1
{
	"name": "ftl0",
	"uuid": "3b469565-1fa5-4bfb-8341-747ec9f3a9b9"
}
```
