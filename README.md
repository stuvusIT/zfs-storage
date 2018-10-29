# zfs-storage

This role installs ZFS on Linux and configures filesystems and ZVOLs. 
`nfs-kernel-server` will be installed if a configured filesystem makes use of the `sharenfs` attribute. NFS exports will be permanently configured in `/etc/exports.d/zfs-storage-ansible.exports`.

## Requirements

Ubuntu version 16.04 or later or Debian stretch.
`systemd` is required in order to scrub pools using timers instead of distribution-specific cron jobs.
`python-jmespath` is needed on the machine executing this role.

## Role Variables

| Name                           | Default / Mandatory | Description                                                                                                                                                                                       |
|:-------------------------------|:-------------------:|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `zfs_parent_fs`                |                     | Existing parent ZFS filesystem for all filesystems and zvols that is optionally prepended to all configurations                                                                                   |
| `zfs_storage_defaults`         |        `{}`         | Dict containing ZFS attributes that will be applied to all configured ZFS filesystems/zvols                                                                                                       |
| `zfs_filesystems`              |        `[]`         | List of zfs_filesystems defined by a `name` and a dict of `attributes` (mandatory for each entry)                                                                                                 |
| `zfs_zpools`                   |        `[]`         | List of dicts having a `name` (mandatory) and `scrub` (optional) key. `scrub` is a boolean and can disable scrubbing for this pool.                                                               |
| `zvols`                        |        `[]`         | List of zvols defined by a `name` and a dict of `attributes` (mandatory for each entry)                                                                                                           |
| `zfs_default_enable_scrub`     |       `True`        | Whether to enable zpool scrubbing by default                                                                                                                                                      |
| `zfs_scrub_frequency`          |      `weekly`       | Frequency of zfs scrubbing - already running scrubs will be aborted. See [systemd.time](https://www.freedesktop.org/software/systemd/man/systemd.time.html#Calendar%20Events) for allowed values. |
| `zfs_kernel_module_parameters` |        `{}`         | Dict containing ZFS kernel module options to be set in `/etc/modprobe.d/zfs.conf`.                                                                                                                |

Note: There are some ZFS attributes that can only be set at creation (see [man zfs](https://linux.die.net/man/8/zfs)). 
These are `utf8only`, `normalization` and `casesensitivity` for filesystems and `volsize` and `volblocksize` for ZVOLs. 
The respective task will fail if you try to change those.
Also, `volsize` is a mandatory attribute for ZVOLs.

If an attribute is not defined, the ZFS default will be configured (even if the attribute is already set to something else), except for these four attributes:
- `acltype`=`posixacl`
- `compression`=`on`
- `relatime`=`on`
- `xattr`=`sa`

All default values for ZFS attributes can be seen in [the defaults](defaults/main.yml).

In addition to the actual ZFS attributes described in the [man page](https://linux.die.net/man/8/zfs), this role sets attributes to control automatic snapshotting using tools such as  [zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot).
This is achieved using the following variables, which are all set to `False` by default:

| ZFS attribute                    | Variable name in this role       | Description                                                                                                                                                                                       |
|:---------------------------------|:---------------------------------|:--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `com.sun:auto-snapshot`          | `com_sun_auto_snapshot`          | Enable automatic snapshotting and cleaning (destroying) of snapshots using all available intervals for the current dataset (basically acts as if all the following attributes are set to `True`). |
| `com.sun:auto-snapshot:frequent` | `com_sun_auto_snapshot_frequent` | Enable snapshotting and cleaning in 15min intervals.                                                                                                                                              |
| `com.sun:auto-snapshot:hourly`   | `com_sun_auto_snapshot_hourly`   | Enable snapshotting and cleaning in hourly intervals.                                                                                                                                             |
| `com.sun:auto-snapshot:daily`    | `com_sun_auto_snapshot_daily`    | Enable snapshotting and cleaning in daily intervals.                                                                                                                                              |
| `com.sun:auto-snapshot:weekly`   | `com_sun_auto_snapshot_weekly`   | Enable snapshotting and cleaning in weekly intervals.                                                                                                                                             |
| `com.sun:auto-snapshot:monthly`  | `com_sun_auto_snapshot_monthly`  | Enable snapshotting and cleaning in monthly intervals.                                                                                                                                            |



## Example Playbook

```yml
- hosts: zfsstorage
  roles:
    - role: zfs-storage
      zfs_parent_fs: tank
      zfs_zpools:
        - name: rpool
          scrub: False
        - name: tank
          scrub: False
      zfs_scrub_frequency: monthly
      zfs_kernel_module_parameters:
        zfs_arc_max: 30064771072 # Allow ARC to grow to 30GB
      zfs_storage_defaults:
        acltype: posixacl
        volsize: 50G
        quota: 50G
      zfs_filesystems:
        - name: testing
          attributes:
            quota: 200G
        - name: testing/wiki
          attributes:
            sharenfs: rw=@172.27.10.13
            compression: off
      zvols:
        - name: testing/dns01
          attributes:
            volsize: 100G
        - name: testing/ldap01
          attributes:
```

### Result

| Name                  | Type       | Attributes                                                                              |
|:----------------------|:-----------|:----------------------------------------------------------------------------------------|
| `tank/testing`        | filesystem | `acltype`=`posixacl`, `quota`=`200G`                                                    |
| `tank/testing/wiki`   | filesystem | `acltype`=`posixacl`, `sharenfs`=`rw=@172.27.10.13`, `compression`=`off`, `quota`=`50G` |
| `tank/testing/dns01`  | zvol       | `volsize`=`100G`                                                                        |
| `tank/testing/ldap01` | zvol       | `volsize`=`50G`                                                                         |

`rpool` won't be scrubbed, `tank` will be scrubbed `monthly`.

## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).

## Author Information

- [Michel Weitbrecht (SlothOfAnarchy)](https://github.com/SlothOfAnarchy) _michel.weitbrecht@stuvus.uni-stuttgart.de_
