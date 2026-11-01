# MDADM Role

See defaults for all documented options and configuration.

Example playbooks at the root of the collection.

The role does nothing when `mdadm_arrays` is undefined or empty.

## Array options

```yaml
mdadm_arrays:
  md0:
    level: raid5          # raid0, raid1, raid5, raid6, raid10
    drives: [/dev/sdb, /dev/sdc, /dev/sdd]
    chunk: 512            # KiB, ignored for raid1 (default 256)
    bitmap: internal      # default internal for raid1/5/6, avoids a full resync after an unclean shutdown
    assume_clean: false   # default false
    force: false          # default false
    name: data            # optional
    metadata: "1.2"       # optional, default 1.2
```

Input is validated before anything is changed: known level, minimum drive count (raid1 >= 2, raid5 >= 3,
raid6 >= 4), `drives` is a list of unique devices.

### assume_clean

`--assume-clean` skips the initial sync. On RAID5/6 that leaves parity inconsistent and a later disk failure can
silently rebuild garbage, so it is only passed when you set `assume_clean: true` (acceptable for raid0).

### Refusing to overwrite

Before `mdadm --create` every member is checked with `mdadm --examine` and `wipefs`. If any carries an md
superblock, a filesystem, an LVM signature or a partition table the role fails and names the devices. Set
`force: true` on that array to create over them, which destroys whatever was there. The check only runs
when the array is about to be created; arrays already present in `/proc/mdstat` or `mdadm.conf` are left alone.

### Persistence

`mdadm --detail --scan` is written to `mdadm.conf` (one `ARRAY` line per array, updated in place) and
`update-initramfs -u` runs whenever an array is created or `mdadm.conf` changes.
