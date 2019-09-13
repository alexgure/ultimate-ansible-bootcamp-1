# Getting Started with Ansible (Ad Hoc Server Management)

## Creating Project Specific Ansible Configuration

The default configurations for ansible resides at /etc/ansible/ansible.cfg. Instead of relying on defaults, we are going to creates  a custom configuration file for our project. The advantage with that is we could take this configurations on any host and execute it the same way, without touching the default system configurations.  This custom configurations will essentially  override the values in /etc/ansible/ansible/cfg.

###  Ansible configuration file

Change into /vagrant/code/chap3 directory on your ansible host. Create a file called ansible.cfg  Add  the following contents to the file.

On Ansible Control node,

```
cd chap4
ansible --version
```

Create **ansible.cfg** in chap4

```
[defaults]
remote_user = devops
inventory   = environments/prod
retry_files_save_path = /tmp
host_key_checking = False
log_path=~/ansible.log

```

Validate that your new configs are picked up,

```
ansible --version
ansible-config dump
```

## Creating Host Inventory

Since you are going to create a environment specific inventory,
create a **environments** directory and a  file  inside it called *prod*

```
mkdir environments

```
Create inventory file

file: environments/prod

Let's create three groups as follows,


```
[local]
localhost ansible_connection=local

[lb]
lb

[app]
app1
app2


[db]
db

[prod:children]
lb
app
db

```

* First group contains the localhost, the control host. Since it does not need to be connected over ssh, it mandates we add ansible_connection=local option
* Second group contains  Application Servers. We will add  two app servers to this group.
* Third group holds the information about the database servers.

The inventory file should look like below.

### Working with Inventory Command

```
ansible-inventory --help

ansible-inventory --list

ansible-inventory --graph



```

### Ansible ping

We will use Ansible to make sure all the hosts are reachable

```
ansible all -m ping

```

[Output]

```
lb | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
app1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
app2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
db | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Ad Hoc commands

Try running following *fire-and-forget* Ad-Hoc commands...

### Run *hostname* command on all hosts

Let us print the hostname of all the hosts

```
ansible all -a hostname
```

[output]

```
localhost | SUCCESS | rc=0 >>
ansible

192.168.61.11 | SUCCESS | rc=0 >>
db

192.168.61.12 | SUCCESS | rc=0 >>
app

192.168.61.13 | SUCCESS | rc=0 >>
app
```

### Check the *uptime*

How long the hosts are *up*?

```
ansible all -a uptime
```

[Output]

```
localhost | SUCCESS | rc=0 >>
 13:17:13 up  2:21,  1 user,  load average: 0.16, 0.03, 0.01

192.168.61.12 | SUCCESS | rc=0 >>
 13:17:14 up  1:50,  2 users,  load average: 0.00, 0.00, 0.00

192.168.61.13 | SUCCESS | rc=0 >>
 13:17:14 up  1:47,  2 users,  load average: 0.00, 0.00, 0.00

192.168.61.11 | SUCCESS | rc=0 >>
 13:17:14 up  1:36,  2 users,  load average: 0.00, 0.00, 0.00
```

### Check memory info on app servers

Does my app servers have any disk space *free*?

```
ansible app -a free
```

[Output]

```
192.168.61.13 | SUCCESS | rc=0 >>
             total       used       free     shared    buffers     cached
Mem:        372916     121480     251436        776      11160      46304
-/+ buffers/cache:      64016     308900
Swap:      4128764          0    4128764

192.168.61.12 | SUCCESS | rc=0 >>
             total       used       free     shared    buffers     cached
Mem:        372916     121984     250932        776      11228      46336
-/+ buffers/cache:      64420     308496
Swap:      4128764          0    4128764
```

### Installing packages

Let us *install* nano editor on app servers

```
ansible app -a "yum install -y nano"
```

This command will fail.

[Output]

```
192.168.61.13 | FAILED | rc=1 >>
Loaded plugins: fastestmirror, prioritiesYou need to be root to perform this command.

192.168.61.12 | FAILED | rc=1 >>
Loaded plugins: fastestmirror, prioritiesYou need to be root to perform this command.
```

Run the fillowing command with sudo permissions.

```
ansible app -s -a "yum install -y nano"
```

This will install docker in our app servers

[Output]

```
================================================================================
 Package         Arch              Version                Repository       Size
================================================================================
Installing:
 nano            x86_64            2.0.9-7.el6            base            436 k

Transaction Summary
================================================================================
Install       1 Package(s)

Total download size: 436 k
Installed size: 1.5 M
Downloading Packages:
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : nano-2.0.9-7.el6.x86_64                                      1/1
  Verifying  : nano-2.0.9-7.el6.x86_64                                      1/1

Installed:
  nano.x86_64 0:2.0.9-7.el6

Complete!

```

### Running commands one machine at a time

Do you want a command to run on *one machine at a time* ?

```
ansible all -f 1 -a "free"
```

## Using *modules* to manage the state of infrastructure

### Creating users and groups using *user* and *group*

To create a group

```
ansible app -s -m group -a "name=admin state=present"
```

The output will be,

```
192.168.61.13 | SUCCESS => {
    "changed": true,
    "gid": 501,
    "name": "admin",
    "state": "present",
    "system": false
}
192.168.61.12 | SUCCESS => {
    "changed": true,
    "gid": 501,
    "name": "admin",
    "state": "present",
    "system": false
}
```

To create a user

```
ansible app -s -m user -a "name=devops group=admin createhome=yes"
```

This will create user *devops*,

```
192.168.61.13 | SUCCESS => {
    "changed": true,
    "comment": "",
    "createhome": true,
    "group": 501,
    "home": "/home/devops",
    "name": "devops",
    "shell": "/bin/bash",
    "state": "present",
    "system": false,
    "uid": 501
}
192.168.61.12 | SUCCESS => {
    "changed": true,
    "comment": "",
    "createhome": true,
    "group": 501,
    "home": "/home/devops",
    "name": "devops",
    "shell": "/bin/bash",
    "state": "present",
    "system": false,
    "uid": 501
}
```

### Copy a file using *copy* modules

We will copy file from control node to app servers.

```
touch /tmp/test.txt
ansible app -m copy -a "src=/tmp/test.txt dest=/tmp/test.txt"
```

File will be copied over to our app server machines...

```
192.168.61.13 | SUCCESS => {
    "changed": true,
    "checksum": "3160f8f941c330444aac253a9e6420cd1a65bfe2",
    "dest": "/tmp/test.txt",
    "gid": 500,
    "group": "vagrant",
    "md5sum": "9052de4cff7e8a18de586f785e711b97",
    "mode": "0664",
    "owner": "vagrant",
    "size": 11,
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1472991990.29-63683023616899/source",
    "state": "file",
    "uid": 500
}
192.168.61.12 | SUCCESS => {
    "changed": true,
    "checksum": "3160f8f941c330444aac253a9e6420cd1a65bfe2",
    "dest": "/tmp/test.txt",
    "gid": 500,
    "group": "vagrant",
    "md5sum": "9052de4cff7e8a18de586f785e711b97",
    "mode": "0664",
    "owner": "vagrant",
    "size": 11,
    "src": "/home/vagrant/.ansible/tmp/ansible-tmp-1472991990.26-218089785548663/source",
    "state": "file",
    "uid": 500
}

```

## Exercises:
1. Add another system group (not inventory group) called *lb* in inventory with respective host ip
2. Add a system user called *joe* on all  app servers. Make sure that the user has a home directory
3. Install  package *vim* using the correct *Ad-Hoc* command
4. Examine all the available module
http://docs.ansible.com/ansible/modules_by_category.html
5. Find out the difference between the *command* module and *shell* module. Try running the following command with both these modules,

```
"free | grep -i swap"
```

6. Use command module to show *uptime* on the host  
7. Install docker-engine using the yum/apt module
8. Using docker-image module, pull *hello-world* image on web server
