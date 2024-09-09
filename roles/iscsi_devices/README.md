# Ansible Role: iSCSI Devices

An [Ansible Galaxy](https://galaxy.ansible.com/) role for configuring iSCSI devices. Includes scanning for iSCSI targets, configuring multipath and udev rules, creating raw character device nodes, and optionally creating filesystems.

> [!WARNING]
> There may be a **risk of data loss** when using this role against previously provisioned hosts. For additional safety, this role creates a lock file in `/etc` and will refuse to execute again so long as the lock file exists. See [Reprovisioning Hosts](#reprovisioning-hosts) for more information should this role need to be applied to hosts that contain existing data.

## Table of contents

* [iSCSI Devices][1]
  * [Raw Character Device Configuration][2]
  * [Filesystem Configuration][3]
* [Reprovisioning Hosts][4]
* [Example Requirements File][5]
* [Example Playbook][6]
* [License][7]

[1]: #iscsi-devices
[2]: #raw-character-device-configuration
[3]: #filesystem-configuration
[4]: #reprovisioning-hosts
[5]: #example-requirements-file
[6]: #example-playbook
[7]: #license

## iSCSI Devices

iSCSI devices are configured using a combination of the `iscsi_devices_config` group/host variable and external configuration stored in Hashicorp Vault (internally known by the host fact named `iscsi_devices_vault_config`).

---

The `iscsi_devices_config` variable should be defined as a list of dictionaries, each of which supports the following parameters:

| Name                        | Default | Description                                                                           |
|-----------------------------|---------|---------------------------------------------------------------------------------------|
| `alias`                     |         | A unique alias for the storage device.                                                |
| `multipath`                 |         | A boolean value indicating whether this device uses multipath I/O. Configuration for each multipath device will be added to the file `/etc/multipath.conf`. No configuration will be added for non-multipath devices. |
| `raw_character_device`      |         | _Optional_. A dictionary specifying raw character device configuration. See [Raw Character Device Configuration][2] for more information. |
| `filesystem`                |         | _Optional_. A dictionary specifying filesystem configuration. See [Filesystem Configuration][3] for more information. |

> [!NOTE]
> One of `filesystem` or `raw_character_device` must be provided for each item in the `iscsi_devices_config` list, but not both.

---

The internal host fact `iscsi_devices_vault_config` is set during execution of this role, and its value is retrieved from Hashicorp Vault using the path specified by the mandatory role variable `iscsi_devices_vault_path`. This configuration takes the form of a JSON object with the following attributes:

```json
{
  "<ansible-inventory-host>": {
    "iscsi_device_wwids": {
      "<alias>": "<wwid>"
    }
    "iscsi_initiator_name": "<iscsi-initiator-name>",
    "iscsi_portal_ips": [
      "<ip-address>"
    ]
  }
}
```
### Raw Character Device Configuration

If defined, the _optional_ `raw_character_device` parameter requires the following:

| Name         | Default | Description                                                                           |
|--------------|---------|---------------------------------------------------------------------------------------|
| `group`      |         | The group to be used for ownership of the raw character device node.                  |
| `owner`      |         | The user to be used for ownership of the raw character device node.                   |
| `path`       |         | The path to the raw character device node. This should take the form `/dev/raw/raw<N>` where `<N>` is a non-negative integer value (e.g. `/dev/raw/raw1`, `/dev/raw/raw2` and so on). See [raw(8)](https://www.man7.org/linux/man-pages/man8/raw.8.html) for more information. |

> [!NOTE]
> Each raw character device node is created via a `udev` rule and bound to the WWID of the iSCSI device. The `multipath` setting for each device influences how this `udev` rule behaves:
>
> - Multipath devices are assumed to be managed by [Device Mapper](https://www.kernel.org/doc/html/latest/admin-guide/device-mapper/index.html) and have a corresponding `/dev/mapper` device node assigned to them. This device node is used in combination with the `DM_UUID` device property value to determine the device's WWID value.
> - Non-multipath devices are assumed to have kernel names matching `/dev/sd*[!0-9]` (i.e. they are _unpartitioned_) and `scsi_id` is used to determine the device's WWID value.
>
> Furthermore, a symbolic link will be created in `/dev` for each device that includes a `raw_character_device` parameter. The name of the symbolic link will match that of the `alias` in the device configuration and will point to the raw character device node. These symbolic links can be used in configuration files to distinguish different raw character devices (e.g. the symbolic link `/dev/scud` in place of `/dev/raw/raw1`).

### Filesystem Configuration

If defined, the _optional_ `filesystem` parameter requires the following:

| Name         | Default | Description                                                                           |
|--------------|---------|---------------------------------------------------------------------------------------|
| `group`      |         | The group to be used for ownership of the filesystem mount path.                      |
| `mode`       |         | The mode to be used for permissions fo the filesystem mount path.                     |
| `owner`      |         | The user to be used for ownership of the filesystem mount path.                       |
| `path`       |         | The path where the filesystem will be mounted. This path will be created if it does not already exist. |
| `type`       |         | The filesystem type to be created. Refer to the Ansible [filesystem module documentation](https://docs.ansible.com/ansible/latest/collections/community/general/filesystem_module.html) for valid options.                                         |

## Reprovisioning Hosts

> [!WARNING]
> There is a **risk of data loss** if the role is executed against hosts that have previously been provisioned. A lock file is created in `/etc` by this role to provide a basic level of protection against this. The role will refuse to execute again until manual intervention is taken.

To provision a remote host with this role a second time, accepting the potential risk of data loss:

* Stop any active processes on the remote host(s) that may be using the storage device(s)
* Remove the corresponding lock file from remote host(s) (refer to role variable `iscsi_devices_role_lock_file_path` for the absolute path)
* Rerun this role against the same host(s)

## Example Requirements File

```yml
---

collections:
  - name: companieshouse.middleware
    version: "1.0.0"
```

## Example Playbook

```yml
- name: Provision Informix DB management tooling
  hosts: all
  roles:
    - role: companieshouse.middleware.iscsi_devices
    - hosts: servers
      roles:
        - role: iscsi-devices
          vars:
            iscsi_devices_vault_path: /example/path/iscsi
```

## License

This project is subject to the terms of the [MIT License](LICENSE).
