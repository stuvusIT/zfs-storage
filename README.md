# zfs-vm-storage

This role sets up a storage server to share ZFS filesystems via NFS or ZFS ZVOLs via iSCSI.


## Requirements

an apt-based package manager with source version 16.04 or later, as zfs isn't included in older versions.


## Role Variables

| Name                  | Description                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------|
| `root_prefix`         | prefix filesystem that is applied to all filesystems and zvols (Default: `tank`)            |
| `defaults`            | dict containing zfs attributes that will be applied to all configured zfs filesystems/zvols |
| `groups`              | dict of dicts containing zfs attributes                                                     |
| `filesystems`         | list of zfs filesystem definitions, see [below](#filesystems) for details                   |
| `zvols`               | list of zvol definitions, see [below](#zvols) for details                                   |

### filesystems
Each filesystem can define the following variables:

| Name                  | Description                                                                                                                                |
|-----------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| `name` (mandatory)    | the filesystem name (`root_prefix` will be prepended). Missing parent filesystems will be created without any specific zfs attributes set. |
| `groups`              | list containing the groups defined above. The zfs attributes of all groups will be applied and possibly overridden                         |
| `attributes`          | a dict that contains attributes that should be set, overriding the defaults                                                                |

### zvols
Each entry has the following variables:

| Name                  | Description                                                                            |
|-----------------------|----------------------------------------------------------------------------------------|
| `name` (mandatory)    | the zvol name. `root_prefix` will again be applied and missing parent filesystems created. This name is used as iSCSI target name (with illegal characters replaced by `_`), if `initator_name` is set. |
| `groups`              | list containing the groups defined above. The zfs attributes of all groups will be applied and possibly overridden|
| `attributes`         | a dict that contains attributes that should be set, overriding the defaults |
| `initiator_name`         | name of the iSCSI initiator that is allowed to connect to this targets (needed for iSCSI)|
| `iscsi_ip`         | IP of the local interface where iSCSI should be advertised (Default: `172.0.0.1`)|

Note: The size of every zvol needs to be specified using the zfs attribute `volsize`.

## Dependencies

None

## Example Playbook

### Playbook

```yml
- hosts: zfsstorage
  roles:
    - role: zfs-vm-storage
      root_prefix: "tank/vms"
      defaults:
        compression: "lz4"
        acltype: "posixacl"
      groups:
        vm-fs:
          reservation: "10G"
        vm-image:
          volsize: "20G"
          volblocksize: "4K"
      filesystems:
        - name: "testing"
          attributes:
            - quota: "500G"
        - name: "testing/wiki"
          attributes:
            - sharenfs: "rw=@172.100.100.100"
          groups:
            - vm-fs
      zvols:
        - name: "testing/dns01"
          groups:
            - vm-image
          initiator_name: "iqn.2005-03.org.open-iscsi:9e9c13ff76ac"
          iscsi_ip: "172.100.100.99"
        - name: "testing/ldap01"
          groups:
            - vm-image
          attributes:
            - volsize: "100G"
          initiator_name: "iqn.2005-03.org.open-iscsi:9e9c13ff76ac"
          iscsi_ip: "172.100.100.99"
```

### Result

This example defines three default groups, two filesystems and two zvols. The `zfs` group is used by all filesystems and zvols and specifies the compression and acl type to be used. The other two default groups define quota/reservation defaults for filesystems (`vm-fs`) and volume size and blocksize for zvols (`vm-image`).

Two filesystems are created: `testing/wiki` is exported with read and write access, usable by `172.27.100.100`. The `testing` filesystem would normally be created automatically, if `testing/wiki` is specified, but in this case we want to set further attributes, namely the quota of `500G` (applying to all children of `testing`).
Two zvols are created and exported to the same initiator, and on the same local IP `172.27.100.99`. `testing/dns01` uses the default values for zvols defined in the `vm-image` group. `testing/ldap01` also uses them, but overrides the volsize with the value `100G`.


## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).


## Author Information

 * [Michel Weitbrecht (SlothOfAnarchy)](https://github.com/SlothOfAnarchy) _michel.weitbrecht@stuvus.uni-stuttgart.de_
