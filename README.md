# Ansible role: rpi_bootstrap

[![MIT licensed][mit-badge]][mit-link]
[![Galaxy Role][role-badge]][galaxy-link]

Ansible role for RaspberryPi vanilla bootstrap.

Role accomplishes the following:

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
----

NOTE: Role requires Fact Gathering by ansible!

One of the following distros (or derivatives) required:

 - Debian | [Raspberry Pi OS][rpi-os-link] | Raspbian | [Minibian][minibian-link]
 - [CentOS][centos-link]

Flush fresh distro on raspberrypi SD Card.

Do it with `dd` or using for example this script (for Mac):

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

Minibian comes with ssh enabled for root user.
After flushing the minibian image to SD Card, no need to create a user, adding to sudo group and modify sudoers with
NOPASSWD statement.
All this will be done automatically, just modify the variables in accordance with your needs.

ssh into the host using default credentials (root:raspberry) and install ansible dependencies (python).

Alternatively, do the updates and dependencies installation remotely with ansible `raw` module:

```
ansible -m raw -a "echo 'nameserver 9.9.9.9' > /etc/resolv.conf && apt-get update -y && apt-get install -y python" --user root -k rpi
```

**rpi** - is set to your raspberry pi ip address in `hosts` file like:

```
[raspberrypi_3]
rpi ansible_ssh_host=172.16.42.8
```

Now the host is ready for provisioning!

**CentOS 7 (minimal):**

Just flush the SD card as shown above. Python is already there.


Role Variables
----

| Variable | Description | Default |
|----------|-------------|---------|
| `rpi_bootstrap_new_root_passwd` | New root password (SHA512 salted hash) |`vault_rpi_bootstrap_new_root_passwd` |
| `rpi_bootstrap_user0_passwd` | New user password (SHA512 salted hash) | `vault_rpi_bootstrap_user0_passwd` |
| `rpi_bootstrap_sudonopasswd` | Make users from 'sudo' group stop being asked for sudo password | `yes` |
| `rpi_bootstrap_users` | List of users with passwords and groups to be added to system | see [`defaults/main.yml`](defaults/main.yml#L33) |
| `rpi_bootstrap_firmware_update` | Wether to update firmware with rpi-update | `no` |
| `rpi_bootstrap_timezone` | Timezone to be configured for the system | `UTC` |
| `rpi_bootstrap_hostname` | Hostname to be configured for the system | `raspberry.domain` |
| `rpi_bootstrap_locale` | Locale to be configured | `en_US.UTF-8` |
| `rpi_bootstrap__rpi_network_wifi_APs` | this overrides the rpi_network_wifi_APs var of rpi_network dependency role | see [`defaults/main.yml`](defaults/main.yml#L53) |


`ATTENTION!`

make sure you override the `rpi_bootstrap__rpi_network_wifi_APs, rpi_bootstrap_new_root_passwd, rpi_bootstrap_user0_passwd` vars as they may contain a sensitive data,
such as user accounts passwords, WPA passphrase and network ESSID...

It is highly recommended to encrypt with [ansible-vault][ansible-vault-link].

Before running any playbook which uses this role, decrypt the file *vars/main.yml* with:

```
ansible-vault decrypt vars/main.yml --vault-password-file=.vault.key
```

OR set environment variable:

```
export ANSIBLE_VAULT_PASSWORD_FILE=.vault.key
```

OR (PREFERRED):
add the following to **ansible.cfg**:

```
[defaults]
vault_password_file = .vault.key
```

**CREATE YOUR OWN PASSWORD HASHES**

NOTE: Following instructions for some reason does not work on Mac

Calculate SHA512 hash with salt using either Python3:

```
echo 'import crypt,getpass; print(crypt.crypt(getpass.getpass(), crypt.mksalt(crypt.METHOD_SHA512)))' | python3 -
```

Or Python2.7:

```
echo 'import crypt,getpass; print crypt.crypt(getpass.getpass(), "$6$16CHARACTER_SALT")' | python -
```

Dependencies
----

 - [drew1kun.rpi_network][rpi_network-galaxy-link]

Install them from galaxy:

```
ansible-galaxy install drew1kun.rpi_network
```

Example Playbook
----

```yaml
 # Run this bootstrap role as root, but do not run other roles within the playbook as root:
 name: 'Running bootstrap-core play as root user'
  hosts: raspberrypi_3
  remote_user: root

  # User ansible-vault encrypted variables file:
  vars_files:
  - vars/vault.yml

  # It's a good idea to prompt for a user's password to be set dynamically:
  vars_prompt:
  - name: pihole_playbook_prompt_user0_passwd
    prompt: "password for user0 ('{{ vault_rpi_bootstrap_users[0].username }}') to be set in 'rpi_bootstrap' play"
    private: yes
    encrypt: "sha512_crypt"
    confirm: yes
    salt_size: 16

  roles:
  - role: drew1kun.rpi_bootstrap
    rpi_bootstrap_new_root_passwd: "{{ vault_rpi_bootstrap_new_root_passwd }}"
    rpi_bootstrap_users:
    - username: "{{ vault_rpi_bootstrap_users[0].username }}"
      passwd: "{{ pihole_playbook_prompt_user0_passwd }}"
      groups:
      - "{{ rpi_bootstrap_su_group }}"
    rpi_bootstrap__rpi_network_wifi_APs: "{{ vault_rpi_bootstrap__rpi_network_wifi_APs }}"
```

*vars/vault.yml*:

```yaml
# Or you can specify the password hash statically:
# P@$$w0rd
vault_rpi_bootstrap_new_root_passwd:
  $6$pXE2fAZd0vgD77JQ$.c6I7FAFd/b1VV3Q4fm6VKnC29eXf3X.5TR5acIJ9xr3y/Bt0umoEH.b8nX3SqcZgZ3h5uhaoqNN6EAsU69Yn.

# Usernames may be secrets too:
vault_rpi_bootstrap_users:
- username: drew

# Wireless networks for rpi_network role (which is dependency for current role):
vault_rpi_bootstrap__rpi_network_wifi_APs:
- id_str: home_wifi
  hidden: no
  essid: YourSensitiveESSID
  passphrase: YourSecureWPA_Passphrase
  priority: 1
```

License
----

[MIT][mit-link]

Author Information
----

Andrew Shagayev | [e-mail](mailto:drewshg@gmail.com)

[role-badge]: https://img.shields.io/badge/role-drew--kun.bootstrap__core-green.svg
[galaxy-link]: https://galaxy.ansible.com/drew1kun/rpi_bootstrap/
[mit-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[mit-link]: https://raw.githubusercontent.com/drew1kun/ansible-rpi_bootstrap/master/LICENSE
[minibian-link]: https://minibianpi.wordpress.com/
[centos-link]: https://wiki.centos.org/Download
[rpi_expandfs-galaxy-link]: https://galaxy.ansible.com/drew1kun/rpi_expandfs/
[rpi_network-galaxy-link]: https://galaxy.ansible.com/drew1kun/rpi_network/
[ansible-vault-link]: https://docs.ansible.com/ansible/latest/user_guide/vault.html
[rpi-os-link]: https://www.raspberrypi.com/software/operating-systems/
