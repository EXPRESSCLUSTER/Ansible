# Ansible

Open Knowlodge of Ansible and EXPRESSCLUSTER

## Table of Contents

* [Evaluation Environment](#evaluation-environment)
* [Installing Ansible](#installing-ansible)
* [Creating Inventory File](#creating-inventory-file)
* [Placing Necessary Files](#placing-necessary-files)
* [Creating Playbook](#creating-playbook)

## Evaluation Environment

* OS: AlmaLinux 8.8
  * Virtual machine on libvirt/KVM
  * VM host: Ubuntu 22.04

### Network Configuration

```
                                +--- cluster ---+
                                |               |
                                |   [primary]   |
[client] ---------------------> |               |
          creates environment   |  [secondary]  |
              with Ansible      |               |
                                +---------------+
```

* client
  * IP address: 192.168.122.112
* primary
  * IP address: 192.168.122.110
* secondary
  * IP address: 192.168.122.111

## Installing Ansible

```
$ sudo dnf update
$ sudo dnf install epel-release
$ sudo dnf install ansible

$ ansible --version
ansible [core 2.14.2]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/vagrant/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3.11/site-packages/ansible
  ansible collection location = /home/vagrant/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.11.2 (main, Apr  5 2023, 11:57:00) [GCC 8.5.0 20210514 (Red Hat 8.5.0-18)] (/usr/bin/python3.11)
  jinja version = 3.1.2
  libyaml = True
```

## Creating Inventory File

1. Create a directory for trying Ansible.

```
$ mkdir ansible
$ cd ansible
```

2. Place ssh private key files for the primary server and the secondary server.

```
$ cp /from/dir1/private_key ./private_key_1
$ cp /from/dir2/private_key ./private_key_2
```

3. Create an inventory file.

```
$ vi inventory.yml
```

Write the inventory file as below.

```
all:
  children:
    cluster:
      hosts:
        primary:
          ansible_host: 192.168.122.110
          ansible_ssh_user: vagrant
          ansible_ssh_private_key_file: ./private_key_1
        secondary:
          ansible_host: 192.168.122.111
          ansible_ssh_user: vagrant
          ansible_ssh_private_key_file: ./private_key_2
```

4. Check the connectivity to the both servers via the ping module of ansible.

```
$ ansible -i inventory cluster -m ping
primary | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
secondary | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
```

## Placing Necessary Files

Place the following files under the current directory.

* EXPRESSCLUSTER's rpm
* EXPRESSCLUSTER's license files
* EXPRESSCLUSTER's configuration file
  * Use Cluster WebUI offline or clpcfadm.py to create a configuration file

```
$ find . -type f
./ecx-files/base-license.key
./ecx-files/conf/clp.conf
./ecx-files/conf/clp.conf.bak
./ecx-files/conf/scripts/failover1/exec1/start.sh
./ecx-files/conf/scripts/failover1/exec1/stop.sh
./ecx-files/conf/scripts/monitor.s/genw1/genw.sh
./ecx-files/expresscls-5.1.0-1.x86_64.rpm
./ecx-files/repl-license-1.key
./ecx-files/repl-license-2.key
./expresscluster.yml
./inventory.yml
./private_key_1
./private_key_2
```

## Creating Playbook

In the automated setting up of EXPRESSCLUSTER, the following processes are required.

1. Transfer necessary files
2. Install EXPRESSCLUSTER
3. Register the licenses
4. Restart the operating system
5. Apply configuration information
6. Restart the services
7. Start cluster


### 0. Create a playbook file.

```
$ vi expresscluster.yml
```


### 1. Transfer files

```
- hosts: cluster
  become: yes
  tasks:
    - name: Copy files (rpm, license keys, clp.conf, etc.)
      copy:
        src: ./ecx-files
        dest: /tmp
```

### 2. Install EXPRESSCLUSTER

```
    - name: Install EXPRESSCLUSTER
      yum:
        name: /tmp/ecx-files/expresscls-5.1.0-1.x86_64.rpm
        state: present
        disable_gpg_check: true
```

If the server cannot access to the yum/dnf repository,
there's a possibility that the above yum task fails.

### 3. Register the licenses

```
    - name: Register EXPRESSCLUSTER's licenses
      shell: clplcnsc -i '/tmp/ecx-files/{{ item }}'
      with_items: '{{ ecx_license_file }}'
```

Also, add ecx_license_file variable for each hosts in inventory.yml.

```
all:
  children:
    cluster:
      hosts:
        primary:
          # (omitted)
          ecx_license_file:
            - base-license.key
            - repl-license-1.key
        secondary:
          # (omitted)
          ecx_license_file:
            - repl-license-2.key
```

The whole example of the inventory file is available from [here](./inventory.yml).

### 4. Restart the operating system

```
    - name: Reboot the cluster servers
      shell: sleep 1 && reboot
      async: 1
      poll: 0
    - name: Wait for OS reboot completion
      pause:
        seconds: 60
```

### 5. Apply configuration information

```
    - name: Apply the configuration (executed only on the primary server)
      shell: clpcfctrl --push -x /tmp/ecx-files/conf --force
      when: inventory_hostname == 'primary'
```

### 6. Restart the services

```
    - name: Restart EXPRESSCLUSTER's services
      systemd:
        name: '{{ item }}'
        state: restarted
      with_items:
        - clusterpro_alertsync
        - clusterpro_webmgr
        - clusterpro_ib
        - clusterpro_api
        - clusterpro_nm
```

### 7. Start cluster

```
    - name: Start cluster
      shell: clpcl -s
```

The whole example of the playbook is available from [here](./expresscluster.yml).

## Testing Playbook

1. Run the playbook.

```
$ ansible-playbook -i inventory.yml expresscluster.yml

PLAY [cluster] *************************************************************************************
TASK [Gathering Facts] *****************************************************************************
ok: [primary]
ok: [secondary]

...

PLAY RECAP *****************************************************************************************
primary                    : ok=9    changed=6    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
secondary                  : ok=7    changed=5    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

2. Login to the primary (or secondary) server and try clpstat

```
$ hostname
primary

$ sudo clpstat
 ========================  CLUSTER STATUS  ===========================
  Cluster : cluster
  <server>
   *server1 .........: Online
      lanhb1         : Normal           LAN Heartbeat
    server2 .........: Online
      lanhb1         : Normal           LAN Heartbeat
  <group>
    failover1 .......: Online
      current        : server1
      exec1          : Online
  <monitor>
    genw1            : Normal
 =====================================================================
```
