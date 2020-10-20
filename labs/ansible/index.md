# Introduction


In this lab, you will get hands-on experience using Ansible to automate common manual tasks.

## Katacoda
Log into [Katacoda Ansible playground](https://www.katacoda.com/openshift/courses/ansibleop/ansible-refresher-311)

Slide terminal to the left so the steps disappear. Follow the below steps: 

## Tasks and hosts
Ansible runs tasks on hosts, and all you need is SSH. In other words, Ansible runs in one place (your laptop, or a deploy box like Rundeck or Tower, or somewhere else), opens up SSH connections to your remote hosts and then runs commands directly on them.

### What is a task?
A task can be anything from creating a bucket in AWS S3, to launching an instance in Azure, installing pip on a server, updating a config file, or simply checking the time on a remote host. In this tutorial we focus on tasks to configure a remote CentOS host.

For example, this task installs pip on CentOS:

```
- name: ensure pip is installed
    yum:
      name: python-pip
      state: installed
```

Every task uses a module. Here we use the yum module, a wrapper for the yum package manager, letting you specify what to do in YAML syntax.

There are hundeds of different [modules](https://docs.ansible.com/ansible/modules_by_category.html) included in Ansible.

### What is a host?
The host is where the tasks get run. It can be any number of remote hosts that you have SSH access to, or localhost.

Your hosts respective IP addresses or hostnames need to be stored in an inventory file for Ansible to be aware of them. Later when you run Ansible, you will specify one or more hosts, or groups of hosts, defined in the inventory file.

## Creating an inventory
In this tutorial you have access to one host named `host01`. To let Ansible know about it, put its hostname in an inventory file. Let's call our inventory `myhosts`, with one group called `group1`.
```
echo "[group1]" > myhosts
```

Now add the host to the group:
```
echo "host01 ansible_ssh_user=cent" >> myhosts
```

Here we're also passing the username to use for the SSH access, as an [inventory parameter](https://docs.ansible.com/ansible/intro_inventory.html#list-of-behavioral-inventory-parameters)

## Running Ansible   
### Ad-hoc Tasks 

To run an ad-hoc task, using a single module, use `ansible`.

For example, to check the current date and time of the remote host:

```
ansible group1 -i myhosts -m command -a date
```

(if the command fails the first time, try again in few seconds, as the test host may not yet have been provisioned)

`group1` = group to run against. You can also use the name of an individual host, so `host01` would work just as well for this example.

`myhosts` = name of the inventory file listing all available hosts

`command` = module to use. The [command module](https://docs.ansible.com/ansible/command_module.html)returns the result of a bash command)

`date` = argument for the module, in this case a standard shell command `date`.

When you run the command above, Ansible establishes an SSH connection with the remote host, executes the command, captures the output and returns it.

### Playbook   
A playbook is an entirely different way of running Ansible, that is far more powerful.   

#### What is a play? 

A play is a set of tasks mapped to a set of hosts. Plays are organised inside a text file called a playbook.


#### Running a playbook

This playbook runs one task, on our one host, host01. Note the indentation - it's important for how the file gets parsed. 

Create a file with the following named `site.yml`

```yaml
---
- hosts: host01
  become: true
  tasks:
    - name: ensure latest sysstat is installed
      yum:
        name: sysstat
        state: latest
```

To run the playbook, use the `ansible-playbook` command with the inventory file `myhosts`:

```
ansible-playbook -i myhosts site.yml
```

#### Playbook breakdown

What happened here?

`---` denotes the beginning of a YAML file   
`hosts`: host tells Ansible to run the tasks on the host host   
`become: true` makes all your tasks run as sudo   
`- name:` is basically a comment, describing what the task does   
`yum` :` specifies the module we want to use   
`name:` is an argument to the yum module, that specifies the name of the package to install.   
To see all arguments for a specific module, allowed values, and other details, you can use the CLI documentation that is included with Ansible:   

```
ansible-doc yum 
```

To close the documentation, enter q in the terminal.   

#### Ansible is declarative   
A fundamental principle of Ansible is that it ensures a desired state.   

From the previous example:   

```
- name: ensure latest sysstat is installed
    yum:
      name: sysstat
      state: latest
```

This tells Ansible to check if the latest version of sysstat is installed. If it is, Ansible will do nothing more, since the "latest" state is already achieved. If sysstat is not already there, or of an older version than what is available, Ansible installs the latest version.

If you run the playbook again, Ansible does the former, and instead of "Changed: 1", you will get "OK: 2, Changed 0". Try it out:   

```
ansible-playbook -i myhosts site.yml
```

#### Ensure a package is not present 

The `yum` module allows you to specify the state you wish the package to be in. If you want a specific version, you append it to the package name, for example:   

```
- name: ensure sysstat is installed at version 10.2.0-1
    yum:
      name: sysstat=10.2.0-1
      state: installed
```

If you want to ensure that the package is not installed, you can declare that with `state: absent`, and Ansible will ensure it.

Update the playbook to remove sysstat.

```
sed -i -e 's/state: latest/state: absent/' -e 's/ensure.*/ensure sysstat is removed/' site.yml
```

Then re-run the playbook:

```
ansible-playbook -i myhosts site.yml
```

In summary, the same playbook should always be able to run on a host and - assuming there are no errors - produce exactly the desired state, regardless of what state the host started in. For example, should your playbook halt halfway through due to a network error, it would still finish successfully when you re-run it.

## BONUS:
See if you can install the cowsay package.

## Lab Complete
