---
title: "Using Ansible to Interact with the Satellite API"
date: 2024-08-21T15:54:51-05:00
draft: false
toc: false
images:
tags:
  - ansible
  - satellite
  - red hat
  - api
---

Red Hat Satellite is an essential tool in large scale enterprise environments that can be used to manage repository content, as well as provision, manage, and scan deployed systems. 

Managing connected hosts is an essential task when maintaining any deployed instance of Red Hat Satellite, and one of the easiest tools to accomplish this on a mass scale is using Ansible. 

Red Hat provides the certified [redhat.satellite](https://github.com/RedHatSatellite/satellite-ansible-collection) Ansible collection that can accomodate most workflows, of which its upstream is the [theforeman.foreman](https://github.com/theforeman/foreman-ansible-modules) Ansible collection. In this tutorial however, we'll make the API calls ourselves using Ansible's builtin [uri](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/uri_module.html) module, as it will help demonstrate the API output and how to use it. 

An important tool in organizing hosts within Satellite is the use of host collections, which can be used by later tools to perform an action on multiple hosts at once. We can use Ansible and the Satellite API to automate the mass addition of multiple content hosts to a given host collection. 

# Prerequisites

An account with sufficient read/write permissions to interact with host collections, organizations, and hosts is necessary to perform the changes detailed in this guide. 

For the sake of this tutorial, let's assume the following exists in our demo environment:

- Satellite Instance: `satellite-01.demo.lab`
- Content Host: `host-01.demo.lab`
- Host Collection: `demo-collection`
- Organization: `demo-org`

## A Note on Authentication

### SSL

Upon creation of the instance, Red Hat Satellite uses a self-signed certificate for SSL. In order for API requests to function, your client machine must trust Satellite's certificate. You can use the following example commands to download Satellite's certificate and add it to your trusted certificate store on your client:

```bash
scp user@satellite-01.demo.lab:/var/www/html/pub/katello-server-ca.crt /etc/pki/ca-trust/source/anchors/satellite-01.demo.lab-katello-server-ca.crt
chmod 644 /etc/pki/ca-trust/source/anchors/satellite-01.demo.lab-katello-server-ca.crt
update-ca-trust --extract
```

Another workaround, although not secure, is to disable certificate vaidation in each Ansible task:

```yaml
- name: Example without Validation
  ansible.builtin.uri:
    url: "https://satellite-01.demo.lab/katello/api/v2/organizations"
    method: GET
    user: "{{ satellite_username }}"
    password: "{{ satellite_password }}"
    validate_certs: false
```

### Authentication Methods

The tutorial below uses basic authentication when querying the Satellite API for simplicity, but there are a variety of other options available to you for authentication:

- Personal Access Tokens
- Limited OAuth 1.0

Using these authentication methods is beyond the scope of this tutorial, but Red Hat provides some [examples](https://docs.redhat.com/en/documentation/red_hat_satellite/6.15/html/api_guide/chap-red_hat_satellite-api_guide-authenticating_api_calls#Authenticating_API_Calls-HTTP_Authentication_Overview) using `curl` to get you started. 

# Finding your Organization ID

Before we can grab the ID of the host collection we wish to add our content hosts to, we'll first need to get the ID of the organization our content hosts and host collections belong to. 

```yaml
- name: Find all Organizations
  ansible.builtin.uri:
    url: "https://satellite-01.demo.lab/katello/api/v2/organizations"
    method: GET
    user: "{{ satellite_username }}"
    password: "{{ satellite_password }}"
    force_basic_auth: true
  register: org_output
```

This task will send a GET request to the organizations endpoint, and save all outputted data to the `org_output` variable. Further inspection of `org_output` reveals the following (shortened for readability):

```json
"org_output": {
  "json": {
    "page": 1,
    "per_page": 20,
    "results": [
      {
        "created_at": "",
        "description": "",
        "id": 1,
        "label": "demo-org",
        "name": "demo-org",
        "title": "demo-org",
        "updated_at": ""
      },
      {
        "created_at": "",
        "description": "",
        "id": 2,
        "label": "demo-org2",
        "name": "demo-org2",
        "title": "demo-org2",
        "updated_at": ""
      }
```

The endpoint outputs a list of all organizations that currently exist on the Satellite instance. We can use the `select` filter to find an item from the organization list where one of its attribute values contains the name of the organization we are looking for, and grab its ID. `organization` is a variable containing the name of our organization.

```yaml
- name: Set Organization ID Fact
  ansible.builtin.set_fact:
    org_id: "{{ (org_output.json.results | select('search', organization))[0].id }}"
```

# Finding the Host Collection ID

We'll need to do the same when finding the ID of our desired host collection and the content host. We'll use the above organization ID to search through only content hosts and host collections that belong to our desired organization. In the below example, `collection_name` is a variable containing the name of our desired host collection.

```yaml
- name: Find All Host Collections
  ansible.builtin.uri:
    url: "https://{{ hostvars[inventory_hostname]['servers']['satellite'].keys() | first }}/katello/api/v2/organizations/{{ org_id }}/host_collections"
    method: GET
    user: "{{ satellite_username }}"
    password: "{{ satellite_password }}"
    force_basic_auth: true
  register: collection_output
 
- name: Set Host Collection ID
  ansible.builtin.set_fact:
    collection_id: "{{ (collection_output.json.results | select('search', collection_name))[0].id }}"
```

# Finding the Content Host ID

When finding the ID of our host, we'll structure our API call slightly differently, adding `?search=` to limit the output to only hosts that match our desired query. Otherwise, depending on the size of the environment, we would be flooded with thousands of unneeded results. 

```yaml
- name: Get Host
  ansible.builtin.uri:
    url: "https://satellite-01.demo.lab/api/hosts?search={{ ansible_nodename }}"
    method: GET
    user: "{{ satellite_username }}"
    password: "{{ satellite_password }}"
    force_basic_auth: true
  register: host_output
 
- name: Set Host ID
  ansible.builtin.set_fact:
    host_id: "{{ (host_output.json.results | select('search', ansible_nodename))[0].id }}"
```

# Adding the Content Host to the Host Collection

Now that we have the IDs of our content host and host collection, we can use the Satellite API to add our content host to said collection.

```yaml
- name: Add Host to Host Collection
  ansible.builtin.uri:
    url: "https://satellite-01.demo.lab/katello/api/v2/host_collections/{{ collection_id }}/add_hosts"
    method: PUT
    user: "{{ satellite_username }}"
    password: "{{ satellite_password }}"
    force_basic_auth: true
    body_format: json
    headers:
      Content-Type: 'application/json'
      Accept: 'application/json'
    body: {
      "host_ids": [ "{{ host_id }}" ]
    }
```

Ta-da! Your selected content host is now a part of your desired host collection. Hopefully this gives you enough of an overview on interacting with Satellite's API to apply this to other IT workflows.

For reference, I've provided links to the Satellite API Guide, and the Foreman API Guide (Foreman is the upstream project for RH Satellite).

# See Also

[Foreman API Guide](https://apidocs.theforeman.org/foreman/latest/apidoc/v2.html)

[Red Hat Satellite 6.15 API Guide](https://docs.redhat.com/en/documentation/red_hat_satellite/6.15/html/api_guide/index)

Red Hat documentation requires a Red Hat account with an active subscription to view. A (free) Red Hat Developer subscription is sufficient to meet this requirement and can be obtained by visiting the [Red Hat Developer Program](https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux).