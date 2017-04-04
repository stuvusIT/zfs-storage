# zfs-vm-storage

This role sets up a storage server to share ZFS filesystems via NFS or ZFS ZVOLs via iSCSI.


## Requirements

an apt-based package manager with source version 16.04 or later, as zfs isn't included in older versions.


## Role Variables

This role has four main variables: `root_prefix` sets a prefix filesystem that is applied to all filesystems and zvols (Default: `tank`). `defaults`, `filesystems`, `zvols` are described in the following sections.

###defaults
`defaults` expects a dict defining groups containing zfs attributes. This way, different default zfs attributes can be used for different filesystems/zvols. Each dict entry has a `name` variable and a `attributes` dict defining zfs attributes (using the same keys and values as ZFS does).

###filesystems
`filesystems` is a dict that contains zfs filesystems and their attributes. Each filesystem can have the following variables:
`name` (mandatory) - the filesystem name (`root_prefix` will be prepended). Missing parent filesystems will be created without any specific zfs attributes set.
`default_groups` is a list of default groups defined above in `default`. Attributes defined in multiple groups will be overridden using the list ordering.
`attributes` is a dict that contains attributes that should be set, overriding the defaults. Again, keys and values are adopted directly.

###zvols
`zvols` is a dict that contains ZVOL configurations. Each entry has the following variables:
	`name` (mandatory) - the zvol name. `root_prefix` will again be applied and missing parent filesystems created. This name is used as iSCSI target name, if `initator_name` is set.
	`default_groups` - usage see above
	`attributes` - usage see above
	`initiator_name` (needed for iSCSI) - name of the iSCSI initiator that is allowed to connect to this target
	`iscsi_ip` (Default: `172.0.0.1`) - IP of the local interface where iSCSI should be advertised

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
        zfs:
          compression: "lz4"
          acltype: "posixacl"
        vm-fs:
          reservation: "10G"
        vm-image:
          volsize: "20G"
          volblocksize: "4K"
      filesystems:
        - name: "testing"
          attributes:
            - quota: "500G"
          default_groups:
            - zfs
        - name: "testing/wiki"
          attributes:
            - sharenfs: "rw=@172.100.100.100"
          default_groups:
            - zfs
            - vm-fs
      zvols:
        - name: "testing/dns01"
          default_groups:
            - zfs
            - vm-image
          initiator_name: "iqn.2005-03.org.open-iscsi:9e9c13ff76ac"
          iscsi_ip: "172.100.100.99"
        - name: "testing/ldap01"
          default_groups:
            - zfs
            - vm-image
          attributes:
            - volsize: "100G"
          initiator_name: "iqn.2005-03.org.open-iscsi:9e9c13ff76ac"
          iscsi_ip: "172.100.100.99"
```

### Result

This example defines three default groups, two filesystems and two zvols. The `zfs` group is used by all filesystems and zvols and specifies the compression and acl type to be used. The other two default groups define quota/reservation defaults for filesystems (`vm-fs`) and volume size and blocksize for zvols (`vm-image`).

Two filesystems are created: `testing/wiki` is exported with read and write access, usable by `172.27.100.100`. The `testing` filesystem would normally be created automatically, if `testing/wiki` is specified, but in this case we want to set further attributes, namely the quota of 500G (applying to all children of `testing`).
Two zvols are created and exported to the same initiator, and on the same local IP `172.27.100.99`. `testing/dns01` uses the default values for zvols defined in the `vm-image` group. `testing/ldap01` also uses them, but overrides the volsize with the value `100G`.


## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).


## Author Information

 * [Michel Weitbrecht (SlothOfAnarchy)](https://github.com/SlothOfAnarchy) _michel.weitbrecht@stuvus.uni-stuttgart.de_
