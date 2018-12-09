Ansible role: bootstrap_core
=========

[![MIT licensed][mit-badge]][mit-link]
[![Galaxy Role][role-badge]][galaxy-link]

Ansible role for RaspberryPi vanilla bootstrap.

Role accomplishes the following:

 - expands filesystem
 - configures hostname
 - configures timezone
 - configures network (for now only for Debian)
 - updates firmware
 - creates all specified users and adds them to all specified groups
 - modifies sudoers so group %sudo(Debian) or %wheel(Centos) not to be asked for passwords
 - changes /etc/skel
 - upgrades distro
 - changes root password (if playbook is run by non-root user)

Requirements
------------

NOTE: Role requires Fact Gathering by ansible!

One of the following distros (or derivatives) required:
 - [Minibian][minibian-link] | Raspbian | Debian
 - [CentOS][centos-link]

Flush fresh distro on raspberrypi SD Card.

Do it with *dd* or using for example this script (for Mac):

```
#!/usr/bin/env bash
set -o nounset

diskutil list

echo "Please specify disk number:"
read N

diskutil eraseDisk ExFAT boot "/dev/disk${N}"
diskutil unmountDisk "/dev/disk${N}"
pigz -kdc 2016-03-12-jessie-minibian.tar.gz | tar -xOf - | pv -s 794M | dd of="/dev/rdisk${N}" bs=4m; sync
sleep 5
diskutil eject "/dev/disk${N}"
```

**Minibian:**

Minibian usually comes with ssh enabled for root user.
After flushing the minibian image to SD Card, no need to create a user, adding to sudo group and modify sudoers with
NOPASSWD statement.
All this will be done automatically, just modify the variables in accordance with your needs.

ssh into the host using default credentials (root:raspberry) and install ansible dependencies (python).

Alternatively, do the updates and dependencies installation remotely with ansible `raw` module:

    ansible -m raw -a "echo 'nameserver 9.9.9.9' > /etc/resolv.conf && apt-get update -y && apt-get install -y python" --user root -k rpi

**rpi** - is set to your raspberry pi ip address in `hosts` file like:

    [raspberrypi_3]
    rpi ansible_ssh_host=172.16.42.8

Now the host is ready for provisioning!

**CentOS 7 (minimal):**

Just flush the SD card as shown above. Python is already there.


Role Variables
--------------

| Variable | Description | Default |
|----------|-------------|---------|
| **bootstrap_core_new_root_passwd** | New root password (SHA512 salted hash) |`vault_bootstrap_core_new_root_passwd` |
| **bootstrap_core_user0_passwd** | New user password (SHA512 salted hash) | `vault_bootstrap_core_user0_passwd` |
| **bootstrap_core_sudonopasswd** | Make users from 'sudo' group stop being asked for sudo password | `yes` |
| **bootstrap_core_users** | List of users with passwords and groups to be added to system | see [`defaults/main.yml`](defaults/main.yml) |
| **bootstrap_core_firmware_update** | Wether to update firmware with rpi-update | `no` |
| **bootstrap_core_hostname** | Hostname to be configured for the system | `raspberry.localdomain` |
| **bootstrap_core_locale** | Locale to be configured | `en_US.UTF-8` |
| **bootstrap_core__rpi3_network_wifi_APs** | this overrides the rpi3_network_wifi_APs var of rpi3_network dependency role | see [`defaults/main.yml`](defaults/main.yml) |


**ATTENTION!**


make sure you override the **bootstrap_core__rpi3_network_wifi_APs** var as it contains a sensitive data for your wireless networks,
such as WPA passphrase and network ESSID...

It is highly recommended to encrypt with [ansible-vault][ansible-vault-link].

Before running any playbook which uses this role, decrypt the file *vars/main.yml* with:

    ansible-vault decrypt vars/main.yml --vault-password-file=.vault.key

OR set environment variable:

    export ANSIBLE_VAULT_PASSWORD_FILE=.vault.key

OR (PREFERRED):
add the following to **ansible.cfg**:

    [defaults]
    vault_password_file = .vault.key


**CREATE YOUR OWN PASSWORD HASHES**

If the .vault.pass key file is lost we ca always recreate **vault_bootstrap_core_new_root_passwd**
and **vault_bootstrap_core_user0_passwd** variables and *vars/main.yml* file

NOTE: Following instructions for some reason does not work on Mac

Calculate SHA512 hash with salt using either Python3:

    echo 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))' | python3 -

Or Python2.7:

    echo 'import crypt,getpass; print crypt.crypt(getpass.getpass(), "$6$16CHARACTER_SALT")' | python -

Dependencies
------------

 - [drew_kun.rpi_expandfs][rpi_expandfs-galaxy-link]
 - [drew_kun.rpi3_network][rpi3_network-galaxy-link]

Install them from galaxy:

    ansible-galaxy install drew_kun.rpi_expandfs drew_kun.rpi3_network

Example Playbook
----------------

```yaml
- hosts: raspberrypi
  gather_facts: yes
  roles:
  - drew_kun.bootstrap_core
    bootstrap_core__rpi3_network_wifi_APs:
    - id_str: home
      hidden: no
      essid: "{{ vault_bootstrap_core__rpi3_network_wifi_APs[0].essid }}"
      passphrase: "{{ vault_bootstrap_core__rpi3_network_wifi_APs[0].passphrase }}"
      priority: 10
```

*vars/vault.yml*:

```yaml
vault_bootstrap_core__rpi3_network_wifi_APs:
# only sensitive stuff goes here:
- essid: YourSensitiveESSID
  passphrase: YourSecureWPA_Passphrase
```

License
-------

[MIT][mit-link]

Author Information
------------------

Andrew Shagayev | [e-mail](mailto:drewshg@gmail.com)

[role-badge]: https://img.shields.io/badge/role-drew--kun.bootstrap__core-green.svg
[galaxy-link]: https://galaxy.ansible.com/drew_kun/bootstrap_core/
[mit-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[mit-link]: https://raw.githubusercontent.com/drew_kun/ansible-bootstrap_core/master/LICENSE
[minibian-link]: https://minibianpi.wordpress.com/
[centos-link]: https://wiki.centos.org/Download
[rpi_expandfs-galaxy-link]: https://galaxy.ansible.com/drew_kun/rpi_expandfs/
[rpi3_network-galaxy-link]: https://galaxy.ansible.com/drew_kun/rpi3_network/
[ansible-vault-link]: https://docs.ansible.com/ansible/latest/user_guide/vault.html
