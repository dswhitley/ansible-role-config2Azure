Ansible Role: config4Azure
==========================

This is a fairly rudimentary role meant to keep me from having to repeat the
same steps over and over again.  It accomplishes the steps for
[Configure the RHEL VM image](https://access.redhat.com/articles/uploading-rhel-image-to-azure#configure-the-rhel-vm-image-14)
in Red Hat's 
[How to provision a Red Hat Enterprise Linux virtual machine for Microsoft Azure](https://access.redhat.com/articles/uploading-rhel-image-to-azure)
article.  

There is plenty of room for improvement.

Important Notes
---------------

* Currently, the role is hardcoded to set up your Azure VM with 2Gb of swap

Requirements
------------

Please be aware of these requirements:

* You will need to have a running VM ready to be configured
* You will need to prompt for, or otherwise provide, a user and password that
  can subscribe a system to Red Hat's network for package management (see below)

Role Variables
--------------

Because we need to install a package from a repository we need to temporarily
register and attach a subscription.  So the following variables are 
***required*** (there is minimal sanity checking):

* `sm_user`
  * the user whose login credentials will be used
* `sm_pass`
  * the password for `sm_user`

Example Playbook
----------------

This role can be called on its own:

```yaml
- hosts: allVMs
  remote_user: root
  vars_prompt:
    - name: "sm_user"
      prompt: "Enter subscription-manager username"
      private: no
      when: sm_user is not defined
    - name: "sm_pass"
      prompt: "Enter subscription-manager password"
      private: yes
      when: sm_pass is not defined
  roles:
    - role: config4Azure
```

**But** it will most likely be called in combination with another role, so it is
important to set up your playbook correctly.  I prefer to prompt at the
beginning of a playbook and pass the variables to any subsequent plays, just so
that a user can kick off a playbook, provide the necessary information, and then
walk away.  Here is an example of how I do that when using this play with a role
that 
[creates a VM from a qcow2 image](https://github.com/dswhitley/ansible-role-QCOW2builder):

```yaml
---
- hosts: localhost
  connection: local
  vars_prompt:
    - name: "sm_user"
      prompt: "Enter subscription-manager username"
      private: no
      when: sm_user is not defined
    - name: "sm_pass"
      prompt: "Enter subscription-manager password"
      private: yes
      when: sm_pass is not defined
  roles:
    - role: QCOW2builder
      vm:
        name: rhel74
        memory: 2048
        os_type: Linux
        os_variant: rhel7.4
        network:
          name: default # this is the libvirt network the vm will be on
      qcow2:
        name: rhel-server-7.4-update-4-x86_64-kvm.qcow2
        location: /depot/images/libvirt/ # this must be local
  post_tasks:
    - name: set facts to use in future plays
      set_fact:
        sm_user: "{{ sm_user }}"
        sm_pass: "{{ sm_pass }}"
    - name: wait 30 seconds for VMs to come all the way up
      pause:
        seconds: 30

- hosts: allVMs #the VM created above is added to this group in the QCOW2builder role
  pre_tasks:
    - name: get the subscription-manager info from previous play
      set_fact:
        sm_user: "{{ hostvars['localhost']['sm_user'] }}"
        sm_pass: "{{ hostvars['localhost']['sm_pass'] }}"
  roles:
    - role: config4Azure
```

Inclusion
---------

* I'm imagining this role as a part of other roles, such as to create a 
  Satellite server.  So I have included a fairly generic `meta/main.yml` file
  which allows for something similar to:

        ansible-galaxy install -p ./roles -r requirements.yml

    with `requirements.yml` containing:

        ---
        # get the config4Azure role from github
        - src: https://github.com/dswhitley/ansible-role-config4Azure.git
          scm: git
          name: config4Azure

References
----------

* [How to provision a Red Hat Enterprise Linux virtual machine for Microsoft Azure](https://access.redhat.com/articles/uploading-rhel-image-to-azure)

License
-------

Red Hat, the Shadowman logo, Ansible, and Ansible Tower are trademarks or
registered trademarks of Red Hat, Inc. or its subsidiaries in the United
States and other countries.

All other parts of this project are made available under the terms of the [MIT
License](LICENSE).