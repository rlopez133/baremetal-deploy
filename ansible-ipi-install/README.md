Table of contents
=================

<!--ts-->
   * [Introduction](#introduction)
   * [Prerequisites](#prerequisites)
   * [Tour of the Ansible Playbook](#tour-of-the-ansible-playbook)
   * [Running the Ansible Playbook](#running-the-ansible-playbook)
      * [Create the `ansible.cfg` file](#create-the-ansiblecfg-file)
      * [Copy local ssh key to provision node](#copy-local-ssh-key-to-provision-node)
      * [Modifying the `inventory/hosts` file](#modifying-the-inventoryhosts-file)
      * [Reviewing `group_vars/all` file](#reviewing-group_varsall-file)
      * [Adding Variable Values to the `playbook.yml`](#adding-variable-values-to-the-playbookyml)
   * [Verifying Installation](#verifying-installation)
   * [Troubleshooting](#troubleshooting)
      * [Unreachable Host](#unreachable-host)
      * [Permission Denied Trying To Connect To Host](#permission-denied-trying-to-connect-to-host)
<!--te-->

# Introduction

This write-up will guide you through the process of using the Ansible playbooks
to deploy a Baremetal Installer Provisioned Infrastructure (IPI) of Red Hat
OpenShift 4.

For the manual details, visit our [Installation Guide](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md)

# Prerequisites

* Best Practice Minimum Setup: 6 Physical servers (1 provision node, 3 master 
and 2 worker nodes)
* Minimum Setup: 4 Physical servers (1 provision node, 3 master nodes)
* Each server needs 2 NICs pre-configured. NIC1 for the private network and 
NIC2 for the external network. NIC interface names need to be identical. 
See [issue](https://github.com/openshift/installer/issues/2762)
* Each server should have a RAID-1 configured and initialized
* Each server must have IPMI configured
* Each server must have DHCP setup for external NICs
* Each server must have DNS setup for the API, wildcard applications
* A DNS VIP is IP on the `baremetal` network is required for reservation. 
Reservation is done via our DHCP server (though not required).  
* Optional - Include DNS entries for the external hostnames for each of the 
servers
* Download a copy of your [Pull secret](https://cloud.redhat.com/openshift/install/metal/user-provisioned)
* Append to the `pull-secret.txt` the [Pull secret](https://docs.google.com/document/d/1pWRtk7IbnfPo6cSDsopUMrxS22t3VJ2PuN39MJp9tHM/edit) 
with access to `registry.svc.ci.openshift.org` and `registry.redhat.io`


Due to the complexities of properly configuring an environment, it is 
recommended to review the following steps prior to running the Ansible playbook
as without proper setup, the Ansible playbook won't work.

The sections to review and ensure proper configuration are as follows:
* [Networking Requirements](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#networking-requirements)
* [Configuring Servers](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#configuring-servers)
* [Reserve IPs for the VIPs and Nodes](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#reserve-ips-for-the-vips-and-nodes)
* One of the Create DNS records sections
  * [Create DNS records on a DNS server (Option 1)](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#create-dns-records-on-a-dns-server-option-1)
  * [Create DNS records using dnsmasq (Option 2)](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#create-dns-records-using-dnsmasq-option-2)
* One of the Create DHCP reservation sections
  * [Create DHCP reservations (Option 1)](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#create-dhcp-reservations-option-1)
  * [Create DHCP reservations using dnsmasq (Option 2)](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#create-dhcp-reservations-using-dnsmasq-option-2)


Once the above is complete, the final step is to install Red Hat Enterprise Linux
(RHEL) 8.x on your provision node and create a user (i.e. `kni`) to deploy as
non-root and provide that user `sudo` privileges.

For simplicity, the steps to create the user named `kni` is as follows:
1. Login into the provision node via `ssh`
2. Create a user (i.e `kni`) to deploy as non-root and provide that user `sudo` privileges
    ~~~sh
    useradd kni
    passwd kni
    echo "kni ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/kni
    chmod 0440 /etc/sudoers.d/kni
    ~~~

# Tour of the Ansible Playbook

The `ansible-ipi` playbook consists of 3 main directories:
* `group_vars` - contains the file `all` that has all the default variable values
and their defintion
* `inventory` - contains the file `hosts` that requires the setting up of your 
provision node, master nodes, and worker nodes. Each section will require 
additional details (i.e. Management credentials).
* `roles` - contains two roles: `node-prep` and `installer`. `node-prep` handles
all the prerequisites that the provisioner node requires prior to running the 
installer. The `installer` role handles extracting the installer, setting up
the manifests, and running the Red Hat OpenShift installation. 

The tree structure is shown below:

~~~sh
├── group_vars
│   └── all
├── inventory
│   └── hosts
├── playbook.yml
└── roles
    ├── installer
    │   ├── defaults
    │   │   └── main.yml
    │   ├── files
    │   ├── handlers
    │   │   └── main.yml
    │   ├── meta
    │   │   └── main.yml
    │   ├── tasks
    │   │   ├── 01_get_release_img.yml
    │   │   ├── 02_get_oc.yml
    │   │   ├── 03_extract_installer.yml
    │   │   ├── 04_create_metal3.yml
    │   │   ├── 05_create_manifest.yml
    │   │   ├── 06_extramanifests.yml
    │   │   ├── 07_deploy_ocp.yml
    │   │   ├── 08_cleanup_sub_man_registeration.yml
    │   │   └── main.yml
    │   ├── templates
    │   │   └── metal3-config.j2
    │   ├── tests
    │   │   ├── inventory
    │   │   └── test.yml
    │   └── vars
    │       └── main.yml
    └── node-prep
        ├── defaults
        │   └── main.yml
        ├── handlers
        │   └── main.yml
        ├── library
        │   └── nmcli.py
        ├── meta
        │   └── main.yml
        ├── tasks
        │   ├── 01_sub_man_register.yml
        │   ├── 02_bridge.yml
        │   ├── 03_req_packages.yml
        │   ├── 04_modify_sudo_user.yml
        │   ├── 05_enabled_services.yml
        │   ├── 06_enabled_fw_services.yml
        │   ├── 07_libvirt_pool.yml
        │   ├── 08_create_config_install_dirs.yml
        │   ├── 09_create-install-config.yml
        │   ├── 10_power_off_cluster_servers.yml
        │   └── main.yml
        ├── templates
        │   ├── dir.xml.j2
        │   ├── install-config.j2
        │   └── pub_nic.j2
        ├── tests
        │   ├── inventory
        │   └── test.yml
        └── vars
            └── main.yml

~~~


# Running the Ansible Playbook

The following are the steps to successfully run the Ansible playbook. 

## Create the `ansible.cfg` file

While the `ansible.cfg` may vary upon your environment a sample is provided 
below.

~~~sh
$ cat ansible.cfg 
[defaults]
inventory=/path/to/ansible-ipi/inventory
remote_user=kni

[privilege_escalation]
become=true
become_method=sudo
~~~

NOTE: Ensure to change the `remote_user` and `inventory` path as deemed
apprioriate for your environment. The `remote_user` is the user previously
created on the provision node. 

## Copy local ssh key to provision node

With the `ansible.cfg` file in place, the next step is to ensure to copy our
public `ssh` key to our provision node using `ssh-copy-id`.

~~~sh
$ ssh-copy-id <user>@provisioner.example.com
~~~

NOTE: <user> should be the user previously created on the provision node. 

## Modifying the `inventory/hosts` file

Within the hosts file ensure to set all your nodes that will be used to deploy
IPI on baremetal. There are 3 groups: `masters`, `workers`, and `provisioner`. 
The `masters` and `workers` group collects information about the host such as
its name, role, user management (i.e. iDRAC) user, user management (i.e. iDRAC)
password, ipmi_address to access the server and the mac address of NIC1 that
resides on the provisioning network. 

Below is a sample of the inventory/hosts file

~~~sh
[masters]
master-0 name=master-0 role=master user=admin password=password ipmi_address=192.168.1.1 mac=ec:f4:bb:da:0c:58
master-1 name=master-1 role=master user=admin password=password ipmi_address=192.168.1.2 mac=ec:f4:bb:da:32:88
master-2 name=master-2 role=master user=admin password=password ipmi_address=192.168.1.3 mac=ec:f4:bb:da:0d:98

[workers]
worker-0 name=worker-0 role=worker user=admin password=password ipmi_address=192.168.1.4 mac=ec:f4:bb:da:0c:18
worker-1 name=worker-1 role=worker user=admin password=password ipmi_address=192.168.1.5 mac=ec:f4:bb:da:32:28

[provisioner]
provisioner.example.com
~~~

NOTE: The `ipmi_address` can take a fully qualified name assuming it is 
resolvable.

WARNING: If no `workers` are included, do not remove the workers group 
(`[workers]`) as it required to properly build the `install-config.yaml` file.


## Reviewing `group_vars/all` file

Review the variables that have been set within `group_vars/all`. Default values
have been given to all the variables and they may be overwritten in the 
`playbook.yml` file. 

## Adding Variable Values to the `playbook.yml`

In the sample `playbook.yml` you will find all the variables that are required
to be overwritten by the user. However, more can be added. For example, say you
don't like where the `extract_dir` variables extracts the openshift-installer, 
this variable may be added here under the vars section. 

Sample playbook.yml
~~~sh
---
- name: IPI on Baremetal Installation Playbook
  hosts: provisioner
  vars:
    activation_key: "mykey"
    org_id: "111111"
    myuser: kni
    version: "4.3.0-0.nightly-2019-12-09-035405"
    pullsecret: 'ENTER SECRET HERE'
  roles:
    - role: node-prep
      vars:
        domain: example.com
        cluster: cluster-name
        extcidrnet: 10.1.1.0/21
        dnsvip: 10.1.1.10
        numworkers: 2
        nummasters: 3
        apivip: 10.1.1.11
        ingressvip: 10.1.1.12
        ns1vip: 10.1.1.10
    - role: installer
~~~

NOTE: A description of the vars under the `node-prep` role are all reflected of
what is required within your `install-config.yaml`. Ensure to review 
[Configure the install-config and metal3-config](https://github.com/openshift-kni/baremetal-deploy/blob/master/install-steps.md#configure-the-install-config-and-metal3-config)
for more information if unsure how to populate. 

## Adding Extra Configurations to the OpenShift Installer
Prior to the installation of Red Hat OpenShift, you may want to include
additional configuration files to be included during the installation. The
`installer` role handles this. 

In order to include the extraconfigs, ensure to place your `yaml` files within
the `roles/installer/files` directory. All the files provided here will be
included when the OpenShift manifests are created. 

NOTE: By default this directory is empty. 

## Running the `playbook.yml`

With the `playbook.yml` set and in-place, run the `playbook.yml`

~~~sh
$ ansible-playbook -i inventory/hosts playbook.yml
~~~

# Verifying Installation
Once the playbook has successfully completed, verify that your environment is
up and running.

1. Log into the provision node 
~~~sh
ssh kni@provisioner.example.com
~~~
NOTE: `kni` user is my privileged user. 
2. Export the `kubeconfig` file located in the `~/clusterconfigs/auth` directory
~~~sh
export KUBECONFIG=~/clusterconfigs/auth/kubeconfig
~~~
3. Verify the nodes in the OpenShift cluster
~~~sh
[kni@worker-0 ~]$ oc get nodes
NAME                                         STATUS   ROLES           AGE   VERSION
master-0.openshift.example.com               Ready    master          19h   v1.16.2
master-1.openshift.example.com               Ready    master          19h   v1.16.2
master-2.openshift.example.com               Ready    master          19h   v1.16.2
worker-0.openshift.example.com               Ready    worker          19h   v1.16.2
worker-1.openshift.example.com               Ready    worker          19h   v1.16.2
~~~


# Troubleshooting

The following section troubleshoots common errors that may arise when running
the Ansible playbook. 


## Unreachable Host

One of the most common errors is not being able to reach the provisioner host
and seeing an error similar to

~~~sh
$ ansible-playbook -i inventory/hosts playbook.yml 

PLAY [IPI on Baremetal Installation Playbook] **********************************

TASK [Gathering Facts] *********************************************************
fatal: [provisioner.example.com]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: ssh: Could not resolve hostname provisioner.example.com: Name or service not known", "unreachable": true}

PLAY RECAP *********************************************************************
provisioner.example.com    : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0   
~~~

In order to solve this issue, ensure your provisioner hostname is pingable. 

1. The system you are currently on can `ping` the provisioner.example.com
~~~sh
ping provisioner.example.com
~~~
2. Once pingable, ensure that you have copied your public ssh key from
your local system to the privileged user via the `ssh-copy-id` command.
~~~sh
ssh-copy-id kni@provisioner.example.com
~~~
NOTE: When prompted, enter the password of your privileged user (i.e. `kni`).
3. Verify connectivity using the `ping` module in Ansible
~~~sh
ansible -i inventory/hosts provisioner -m ping
provisioner.example.com | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
~~~
4. Re-run the Ansible playbook
~~~sh
$ ansible-playbook -i inventory/hosts playbook.yml 
~~~

## Permission Denied Trying To Connect To Host

Another very common error is getting a permission denied error similar to:

~~~sh
$ ansible-playbook -i inventory/hosts test-playbook.yml 

PLAY [IPI on Baremetal Installation Playbook] *****************************************************************************************************

TASK [Gathering Facts] ****************************************************************************************************************************
fatal: [provisioner.example.com]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect to the host via ssh: rlopez@provisioner.example.com: Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).", "unreachable": true}

PLAY RECAP ****************************************************************************************************************************************
provisioner.example.com : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0   
~~~

The above issue is typically related to a problem with
your `ansible.cfg` file. Either it does not exist, has errors inside it, or you
have not copied your ssh public key onto the provisioner.example.com system. If
you notice closely, the Ansible playbook attempted to use my `rlopez` user instead
of my `kni` user since my local `ansible.cfg` did not exist **AND** I had not yet
set the `remote_user` parameter to `kni` (my privileged user).

1. When working with the Ansible playbook ensure you have an `ansible.cfg` 
located in the same directory as your `playbook.yml` file. The contents of the
`ansible.cfg` should look similar to the below with the exception of changing
your inventory path (location of `inventory` dir) and potentially your
privileged user if not using `kni`. 
~~~sh
$ cat ansible.cfg 
[defaults]
inventory=/home/rlopez/git_projects/baremetal-deploy/ansible-ipi-install/inventory
remote_user=kni

[privilege_escalation]
become=true
become_method=sudo
~~~
2. Next, ensure that you have copied your public ssh key from
your local system to the privileged user via the `ssh-copy-id` command.
~~~sh
ssh-copy-id kni@provisioner.example.com
~~~
NOTE: When prompted, enter the password of your privileged user (i.e. `kni`).
3. Verify connectivity using the `ping` module in Ansible
~~~sh
ansible -i inventory/hosts provisioner -m ping
provisioner.example.com | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false,
    "ping": "pong"
}
~~~
4. Re-run the Ansible playbook
~~~sh
$ ansible-playbook -i inventory/hosts playbook.yml 
~~~


