# zfs-vm-storage

This role sets up ZFS filesystem that can be shared via NFS.


## Requirements

an apt-based package manager with source version 16.04 or later, as zfs isn't included in older versions.


## Role Variables

| Name                  | Description                                                                                 |
|-----------------------|---------------------------------------------------------------------------------------------|
| `parent_fs`           | existing parent zfs filesystem for all filesystems and zvols (Default: `tank`)              |
| `defaults`            | dict containing zfs attributes that will be applied to all configured zfs filesystems/zvols |
| `filesystems`         | list of filesystems defined by a `name` and a dict of `attributes` (both mandatory)         |
| `zvols`               | list of zvols defined by a `name` and a dict of `attributes` (both mandatory)               |

Note: There are some zfs attributes that can only be set at creation. Also, `volsize` is a mandatory attribute for zvols.

## Dependencies

zfs, nfs

## Example Playbook

### Playbook

```yml
- hosts: zfsstorage
  roles:
    - role: zfs-vm-storage
      parent_fs: tank
      defaults:
        acltype: posixacl
        volsize: 50G
        quota: 50G
      filesystems:
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

This example creates two filesystems and two zvols.
`tank/testing` has `acltype`=`posixacl`, `quota`=`200G`
`tank/testing/wiki` has `acltype`=`posixacl`, `sharenfs`=`rw=@172.27.10.13`, `compression`=`off`, `quota`=`50G`
`tank/testing/dns01` has `volsize`=`100G`
`tank/testing/ldap01` has `volsize`=`50G`


## License

This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/).


## Author Information

 * [Michel Weitbrecht (SlothOfAnarchy)](https://github.com/SlothOfAnarchy) _michel.weitbrecht@stuvus.uni-stuttgart.de_
