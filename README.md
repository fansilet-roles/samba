samba
======

This role deploy a [Standalone Samba Server](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Standalone_Server).  
It can be deployed either to a regular GNU/Linux box or by creating a 
[Podman Quadlet](https://github.com/containers/quadlet) container.  

The role uses ansible tags that allows use `host` mode or `quadlet` mode.
Use `ansible-playbook playbook.yml --tags host` to install the Samba Server in a
host machine. 

Use `ansible-playbook playbook.yml --tags quadlet` to build a Samba Server
Container which will run as a Quadlet. _When using quadlet the service
`samba-server` will manage the SMB and NMB services_  

Requirements
------------

* `podman` - version `4.7.2+`  

Role Variables
--------------

* `samba_mode`: (`str`) - Define the samba server installation method. Support
  `host` or `quadlet`. (default `host`)  
* `samba_workgroup`: (`str`) - Defines the samba server workgroup. (default
  `WORKGROUP`).  
* `samba_home`: (`bool`) - Toggle the Home User shares. (default: `false`)  
* `samba_protocol`: (`str`) - Define the minimal client and [server protocol](https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html#SERVERMAXPROTOCOL). It can be one of `CORE`, `COREPLUS`, `LANMAN1`, `LANMAN2`, `NT1`, `SMB2_02`, `SMB2_10`, `SMB3_00`, `SMB3_02`, `SMB3_11` or `SMB2_FF`  
* `samba_server_string`: (`str`) - Defines the samba server string  
* `samba_netbios`: (`str`) - Define the netbios name  
* `samba_all_printers`: (`bool`) - Toggle to enable all system printers shares
  (default `false`)  
* `samba_listen`: (`str`) - A list of interfaces for listening e.g `lo eth0`  
* `samba_hosts_allow`: (`str`) - A space separated list of allowed addresses  
* `samba_shares`: (`list`) - A list of shares to be created.  
* `samba_shares.name`: The name of share to be created.  
* `samba_shares.comment`: Define a comment description to the share  
* `samba_shares.path`: Creates the path in the OS and the share in smb.conf
  file.  
* `samba_shares.mode`: The OS directory permission mode  
* `samba_shares.owner`: The OS directory owner. (if undeclared default `root`)  
* `samba_shares.group`: The OS directory group owner. (if undeclared default
  `root`)  
* `samba_shares.browseable`: Toggle the share `browseable` flag  
* `samba_shares.guest`: Toggle samba share `guest ok` flag  
* `samba_shares.force_user`: Define the share user owner  
* `samba_shares.read_only`: Toggle the samba share `read_only` flag  
* `samba_shares.writable`: Toggle the samba share `writable` flag
* `samba_shares.dir_mask`: Define the directory creation mask to the share  
* `samba_shares.create_mask`: Define the creation mask to the share  
* `samba_packages`: The list of packages to be installed  
* `samba_services`: The list of `smb` services, default `smb, nmb`  
* `samba_quadlet_rebuild`: (`bool`) - Remove the previous container image to
  trigger a clean rebuild (default: `false`) _when set to `false` speeds up the process
  however if new changes were made in the templates, it's recommended set it to
  `true`_  
* `samba_selinux`: (`bool`) - Toggle SELinux configurations on the target
  server, this will manipulate `sebooleans` and set the proper context type for
  the shares  
* `samba_booleans`: (`dict`) - Define a list of SELinux booleans to be enable or
  disable  
  
For more information of `SMB` share flags [refere the samba official
docs](https://wiki.samba.org/index.php/Setting_up_Samba_as_a_Standalone_Server#Creating_the_Shared_Directories) 

The [`smb.conf`](templates/smb.conf) template also support custom user share
inclusion. The drop-in file `/etc/samba/usershares.conf` can be created with
additional custom shares. When using quadlet this file will be mounted.  

Dependencies
------------

None

Example Playbook
----------------

* Creating a public share:  

```yaml
---
- name: "Creating a Standalone Samba Server with a Public Share"
  hosts: homelab
  gather_facts: false
  vars:
    samba_workgroup: "HomeLab"
    samba_server_string: "Samba Box"
    samba_shares:
      - name: "Public"
        path: "/mnt/public"
        mode: "1777"
        owner: root
        group: root
        browseable: true
        guest: true
        force_user: nobody
        read_only: false
        writable: true

  roles:
    - role: samba
```
_when using the role from the
[`server`](https://galaxy.ansible.com/ui/repo/published/mrbrandao/server/)
collection call the role as `mrbrandao.server.samba` e.g:_  

```yaml

---
- name: "Creating a Standalone Samba Server Using the Collection"
  hosts: homelab
  gather_facts: false
  vars:
    samba_workgroup: "HomeLab"
    samba_server_string: "Samba Box"
    samba_shares:
      - name: "Public"
        path: "/mnt/public"
        mode: "1777"
        owner: root
        group: root
        browseable: true
        guest: true
        force_user: nobody
        read_only: false
        writable: true

  roles:
    - role: mrbrandao.server.samba
```

* Example playbook using the `quadlet` mode:  

```yaml
---
- name: "Creating a Quadlet Samba Server Using the Collection"
  hosts: homelab
  gather_facts: false
  vars:
    samba_mode: "quadlet"
    samba_workgroup: "HomeLab"
    samba_server_string: "Samba Box"
    samba_shares:
      - name: "Public"
        path: "/mnt/public"
        mode: "1777"
        owner: root
        group: root
        browseable: true
        guest: true
        force_user: nobody
        read_only: false
        writable: true

  roles:
    - role: mrbrandao.server.samba

```

* Example playbook using SELinux and Quadlet:  

```yaml
---
- name: "Deploy Samba Server"
  hosts:
    - mysamba
  become: true
  tasks:
    - name: "Install missing packages"
      ansible.builtin.dnf:
        name: rsync
        state: present
      tags:
        - quadlet

    - name: "Importing Samba Role"
      ansible.builtin.import_role:
        name: mrbrandao.server.samba
      vars:
        samba_mode: "quadlet"
        samba_protocol: "NT1"
        samba_listen: "lo eth0 192.168.1.252/24"
        samba_hosts_allow: 192.168.1.0/24
        samba_selinux: true
```

_this role uses the `tags` `host` or `quadlet`, when using the above playbooks
in to install a samba server in a host machine use the tag `host` e.g:_  

```bash
ansible-playbook playbook.yml --tags host
```

To install the samba server in a container use the quadlet tag, e.g:  

```bash
ansible-playbook playbook.yml --tags quadlet
```

Developing and Testing
----------------------

This role was developed using [ansible
molecule](https://ansible.readthedocs.io/projects/molecule/).
The use of molecule is optional but recommended.  
  
* Testing:  
Unit tests for checking code regression are available in the [`tests` directory](tests/).
use the `verify` or `test` commands, e.g:  

```bash
molecule test
```

while developing use `verify` instead:  

```bash
molecule create
molecule verify
```

License
-------

[GPL-2.0-or-later](https://spdx.org/licenses/GPL-2.0-or-later.html)

Author Information
------------------

@mrbrandao - Igor Brandão
