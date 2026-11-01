# LVG Role

See defaults for all documented options and configuration.

Example playbooks at the root of the collection.

The role installs `lvm2`, `thin_check` (`thin-provisioning-tools` on Debian, `device-mapper-persistent-data`
on RedHat) and `xfsprogs`. Override the list with `lvg_packages`.

## Linear volumes

`logical_volumes` creates plain (linear) LVs and is unchanged. It can be used next to `thin_pools`.

## Thin pools and thin volumes

```yaml
volume_groups:
  vg_hdd1:
    drives: [/dev/md0]
    thin_pools:
      pool:                      # dict key = pool LV name
        size: 90%FREE            # never 100%
        chunksize: 1m            # optional
        metadata_size: 4g        # optional (--poolmetadatasize)
        zero: false              # optional, default false (--zero n)
        thin_volumes:
          data: {size: 8T}       # virtual size, may exceed the pool
```

The volume is then available as `/dev/vg_hdd1/data` (or `/dev/mapper/vg_hdd1-data`), ready for the `format` role.

* The pool is created with `lvcreate` only when `<vg>/<pool>` does not exist yet. An existing pool is never
  resized, recreated or modified by this role, that is what keeps repeated runs free of changes.
* Thin volumes are created with `community.general.lvol` (`thinpool`). `size` is the virtual size, use an
  absolute size (`8T`), not a percentage. Volumes are never shrunk. `opts` and `resizefs` are passed through.
* Snapshots of thin volumes are not CoW-per-write like classic LVM snapshots, which is the point of using them.

### Sizing the pool

LVM reserves a pool metadata spare (`[lvol0_pmspare]`) equal to the pool metadata size, in addition to the
metadata itself. A pool with `100%FREE` therefore fails or leaves no room for it. The role refuses
`100%...` pool sizes; use something like `90%FREE`. Leaving headroom also lets you grow the pool or its
metadata later (`lvextend`).

### Monitoring

A full thin pool is the dangerous failure mode: writes start to fail or block, and a full metadata device
can corrupt the pool. Watch `data_percent` and `metadata_percent` in `lvs -o+data_percent,metadata_percent`.
Auto-extension (`thin_pool_autoextend_threshold` / `thin_pool_autoextend_percent` in `lvm.conf`/`lvmlocal.conf`)
is not managed by this role.
