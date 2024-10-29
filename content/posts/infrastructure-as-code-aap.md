---
title: "AAP and Infrastructure as Code"
date: 2024-08-29T15:58:29-05:00
draft: false
toc: false
images:
tags:
  - ansible
  - aap
  - infrastructure as code
---

Picture this. You're tasked with creating a brand new instance of Ansible Automation Platform your team to match the current production instance. Easy right? Red Hat makes this process easy with an intiutive Ansible playbook and structured inventory file to make the installation process relatively painless.

But what happens once the instance is complete? By now, your team most likely has tens, if not hundreds of job templates, projects, inventories, credentials; all of these resources that must now be re-created manually. Unless you really enjoy clicking the same button hundreds of times, this manual process is both inefficient, boring, and prone to user error.

Enter Ansible and "Infrastructure as Code". You may have heard the term thrown around as your boss' new favorite lingo, but what is it actually? Simply put, its a process where we use code to provision infrastructure and its underyling resources, instead of doing everything manually, and Ansible is a great tool to use in this regard.

Red Hat provides a certified Ansible collection, [ansible.controller](https://console.redhat.com/ansible/automation-hub/repo/published/ansible/controller/) that we can use to easily create resources inside of Ansible Automation Platform. In the below tutorial, I'll go over a few examples of how to create job templates, credentials, and inventories using Ansible and this collection.

# Prerequisites

The `ansible.controller` collection requires a few pieces of information to be provided in order for it to successfully run tasks. This information can be provided either through parameters passed to the module, environment variables, or a config file. I've provided an example config file below for reference:

```ini
[general]
host = https://localhost:8043
verify_ssl = true
oauth_token = LEdCpKVKc4znzffcpQL5vLG8oyeku6
```

You can then pass the location of the config file as a parameter to the Ansible module:

```yaml
- name: Add Something
  inventory:
  ...
    controller_config_file: "~/tower_cli.cfg"
```

## Authentication

Authentication can be accomplished using two methods:

- OAuth2 Token
- Username and Password

The OAuth2 token method is preferred by the collection, but either is appropriate. THis tutorial will use a bugus OAuth2 token and host in the below examples. 

## SSL

Unless otherwise specified in the installation, Ansible Automation Controller will install with a self-signed certificate. Your client must be able to trust the SSL certificate of the AAP controller in order to run tasks. 

Another option, although less secure, is to disable certificate varification when using the Ansible module:

```yaml
- name: Add Something
  ansible.controller.inventory:
  ...
    validate_certs: false
```

# Creating Credentials

## Source Control

One of the first tasks when setting up a new instance of Ansible Automation Controller is to sync your team's set of playbooks from Git. Before you can do that, however, you must have appropriate credentials created to access said resource.

Let's assume that I have created an SSH key pair to act as my source control credential. We'll need the private key (the public will be stored in GitHub or GitLab), stored somewhere on our client machine. In this example, my private RSA SSH key is stored in my user's home directory `/home/someuser/.ssh/id_rsa`, and this credential, named `Git Credential`, will be created within the `Default` organization. 

```yaml
- name: Create a Source Credential
  ansible.controller.credential:
    name: Git Credential
    organization: Default
    state: present
    credential_type: Source Control
    inputs:
      username: "{{ git_username }}"
      ssh_key_data: "{{ lookup('file', '/home/someuser/.ssh/id_rsa') }}"
```

With this source control credential, we can now access and sync our playbooks stored in our Source Control instance. However, this is not the only credential we'll need to get started.

## Machine Credential

In order to access our Linux hosts and run the tasks defined in our playbooks, we'll need to define a credential we can use to access said hosts.

We will assume that on our target servers, there is a user that has the ability to gain elevated privileges using sudo. Like before, the credential will be created within the `Default` organization.

```yaml
- name: Create a Machine Credential
  ansible.controller.credential:
    name: "Machine Credential {{ credential_username }}"
    description: "Credential for the {{ credential_username }} account"
    organization: Default
    credential_type: Machine
    state: present
    inputs:
      username: "{{ credential_username }}"
      password: "{{ credential_password }}"
      become_method: sudo
      become_username: root
      become_password: "{{ credential_password }}"
```

There are instances where you wish to prompt for the password of a machine credential, instead of storing it inside of AAP. You can configure this behavior like so:

```yaml
- name: Create a Machine Credential
  ansible.controller.credential:
    name: "Service Credential"
    description: "Credential for the {{ credential_username }} account"
    organization: Default
    credential_type: Machine
    state: present
    inputs:
      username: "{{ credential_username }}"
      password: "ASK"
      become_method: sudo
      become_username: root
      become_password: "ASK"
```

With this, we've now successfully created the credentials needed to sync our playbooks and connect to our target machines.

# Projects

With our source control credential in hand, we can now begin to sync our playbooks and other required resources from our Source Control instance.

Creating a project via Ansible is easy; using the SCM URL of our playbooks git project, plus our earlier created Source Control credential (named `Git Credential`), we can create our project like so:

```yaml
- name: Create a Playbooks Project
  ansible.controller.project:
    name: "Playbooks"
    description: "Playbooks Project"
    organization: Default
    credential: "Git Credential"
    scm_type: git
    scm_url: "{{ git_url }}"
    scm_update_on_launch: true
```

You'll notice that `scm_update_on_launch` is set to `true` in the above example. This allows AAP to automatically update its version of the Playbooks project every time a job is launched using an asset from that project. If you do not wish this behavior, simply set it to `false`.

# Inventories

Playbooks are a critical part of Ansible, but are useless without a set of hosts to run against. Inventories provide this list.

There are two steps to creating an AAP Inventory; creating an inventory, and creating an inventory source. An inventory source is where host data is actually sourced from, such as an inventory file inside a project, VMWare, Red Hat Insights, and many others. A list of inventory plugins can be found [here](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/controller-inventories#ref-controller-inventory-plugins).

We'll first create an inventory using the following Ansible task:

```yaml
- name: Add Inventory
  ansible.controller.inventory:
    name: "Prod Inventory"
    description: "Production Servers"
    organization: "Default"
    state: present
```

Next, we will create an inventory source for the above inventory. Each source type will have its own set of variables that need to be defined. The link provided above that lists inventory plugins also provides required variables for each plugin. For this example, I will simulate the use of VMWare as an inventory source.

```yaml
- name: Add Inventory Source
  ansible.controller.inventory_source:
    name: "VMWare Inventory"
    description: "Prod VMWare Inventory"
    inventory: "Prod Inventory"
    overwrite: true
    update_on_launch: true
    organization: Default
    source_vars:
      strict: false
      hostname: "{{vmware_hostname}}"
      username: "administrator@vsphere.local"
      password: "{{vmware_password}}"
      validate_certs: false
      with_tags: true
```

# Job Templates

Now that we have created our required credentials, projects, and inventories, it is finally time to create job templates so we can begin to run Ansible playbooks inside of AAP. 

In the below example, I will create a job template that calls a fake playbook called `website.yml` located inside the `Playbooks` project, using the `Service Credential` Machine Credential, and the `Prod Inventory` inventory. 

```yaml
- name: Create Job template
  job_template:
    name: "Ping"
    job_type: "run"
    organization: "Default"
    inventory: "Prod Inventory"
    project: "Playbooks"
    playbook: "website.yml"
    credentials:
      - "Service Credential"
    state: "present"
```

One critical aspect of job templates is the use of surveys; a user friendly question-answer form that sets the values of extra variables that can be used to alter the behavior of playbooks. 

Creating a survey for a job template using Ansible requires the creation of a survey config file, the file name of which is passed to the above task, like so:

```yaml
    survey_enabled: yes
    survey_spec: "{{ lookup('file', 'survey.json') }}"
```

I've provided an example survey config file below that you can use to reference when creating your own:

```json
{
  "name": "Example Survey",
  "description": "An Example Survey",
  "spec": [
    {
  	  "type": "multiplechoice",
  	  "question_name": "Production Status",
  	  "question_description": "Production Status to Deploy To",
  	  "variable": "prod_status",
      "choices": ["nonprod","prod"],
  	  "required": true,
  	  "default": "nonprod"
    }
  ]
}
```

More survey spec examples can be found in the Red Hat Developer Ansible Automation Platform Controller API Guide posted below.

We now have officially created everything we need to start running Automation inside of Ansible Automation platform; all using Ansible playbooks to create the resources we need. Hopefully this serves as an excellent starting point as you start to develop Infrastructure as Code workflows during your Ansible journey. 

# See Also

[Red Hat Developer Ansible Automation Platform Controller API Guide](https://developers.redhat.com/api-catalog/api/ansible-automation-controller)

[Ansible Inventory Plugins](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/using_automation_execution/controller-inventories#ref-controller-inventory-plugins)

[VMWare Inventory Source Variables](https://github.com/ansible-collections/community.vmware/blob/371cea3484f774e4e8bca74d66da776bea64f2c8/plugins/inventory/vmware_vm_inventory.py#L178)
