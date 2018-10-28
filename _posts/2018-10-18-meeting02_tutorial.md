---
layout: post
title: Meeting02 Variable Tutorial
author: Scott Tully
---

>Meeting02 will cover some basics for installing and running Ansible playbooks. We will breifly look at simple inventories, variables, templates, and playbooks. I have tried to introduce simple tasks with some semi-advanced options to keep everyone interested. 


### Preparation

Since you are here, you have already figured out most of the prep. The only thing left is to create your environment. Running the command below will create a docker container with the name you define. You can use this envrinment to run the rest of the labs.

```
lab make_up_name
```
You should see a mesage like the one below. If it doesn't say "new container" then you might want to pick another name.
```
[root@fwac ansible]# lab make_up_name

Created new container named make_up_name

[root@make_up_name make_up_name]#

```

You are now ready to run your labs. You are already in the docker container and can do whatever you want without causing any damage. You will be in a working directory that is shared to the fwac server so all your work will be saved.

---
## Lab 1 - Install Ansible

Installing Ansible is easy with the proper setup. We will review the configuration and you will install ansible using one of the two following methods
```
yum -y install ansible
```
>or

```
pip install ansible
```

---
## Lab 2 - Inventory

We are going to start with super simple inventory and then add a little as we go. `vi inventory` and add the content below to the file, then `:wq` to save. 

```
[fwac]
10.0.0.1

[fwac:vars]

```
That's about as simple as it gets. `fwac` is a server group, with a single server `10.0.0.1` as a member. We can now create a playbook to reference this group.

---
## Lab 3 - Playbook

We are going to break the playbook down into sections to understand everything that is happening. We are covering a lot of functionality in the few tasks we setup. In this lab we are adding the docker host to the fwac server /etc/hosts. This adds your host into DNS.

#### Getting started
Playbooks are defined using YAML. A YAML file should begin with the first line containing `---` and nothing else. Once you define the file as YAML, you can start defining the key value pairs that build the playbook for Ansible to interpret.

This task will prompt you for your email address and add it to /root/logins

`vi log.yml` and add the following lines
```
---
- hosts: fwac
  vars_prompt:
  - name: "email"
    prompt: "Enter your email address"
    private: no
  tasks:
  - lineinfile:
      path: /root/logins
      line: "{{ email }}"
```
You can run this playbook like this:
```
ansible-playbook -i inventory log.yml
```

Next we will demo how to perform remote actions vs local actions. We will also see how variables can be defined.

Create a new playbook `vi playbook.yml` and add this stuff:

```
---
- hosts: fwac
  tasks:
  - shell: hostname
    register: host
    when: myhost is not defined
  - debug:
      var: host
      verbosity: 2
  - local_action: shell hostname
    register: host
    when: myhost is not defined
  - debug:
      var: host
      verbosity: 2
```


