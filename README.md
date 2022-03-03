# Ansible Role: Exoscale Compute

[![CI](https://github.com/ngine-io/ansible-role-exoscale/workflows/CI/badge.svg?event=push)](https://github.com/ngine-io/ansible-role-exoscale/actions?query=workflow%3ACI)

Manages compute resources on [Exoscale Cloud](https://www.exoscale.com/).

## Role Variables

```yaml
# API to be used
exoscale__api_key:
exoscale__api_secret:
exoscale__api_endpoint: https://api.exoscale.ch/v1

# Zone to be used
exoscale__zone: ch-dk-2

# Security groups to be created
exoscale__security_groups:
  - name: default
    rules:
      - protocol: tcp
        type: ingress
        cidr: 0.0.0.0/0
        start_port: 22
        end_port: 22

# Affinity groups to be created (default [])
exoscale__affinity_groups:
  - name: my cluster group

# Networks to be created (default [])
exoscale__networks:
  - name: private-network
    start_ip: 10.23.12.100
    end_ip: 10.23.12.150
    netmask: 255.255.255.0

# SSH Keys to be uploaded
exoscale__ssh_key_default_name: "{{ lookup('env', 'USER') }}@{{ lookup('pipe', 'hostname') }}"
exoscale__ssh_key_default_pubkey: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
exoscale__ssh_keys:
  - name: "{{ exoscale__ssh_key_default_name }}"
    pubkey: "{{ exoscale__ssh_key_default_pubkey }}"

# Server instance
exoscale__server_name: "{{ inventory_hostname_short }}"
exoscale__server_template: "Rocky Linux 8 (Green Obsidian) 64-bit 2021-08-25-13bb54"
exoscale__server_service_offering: Small
exoscale__server_root_disk_size: 50
exoscale__server_networks: "{{ omit }}"
exoscale__server_ip_to_networks: "{{ omit }}"
exoscale__server_security_groups:
  - default
exoscale__server_affinity_groups: []
exoscale__server_ssh_key: "{{ exoscale__ssh_key_default_name }}"
exoscale__server_user_data: |
  #cloud-config
  manage_etc_hosts: false
  fqdn: "{{ inventory_hostname }}"

exoscale__server_state: present
exoscale__server_allow_reboot: false
```

## Dependencies

See `requirements.txt` for python library dependencies.

## Examples

### Playbook

A typical playbook would look like this.

```yaml
---
- name: Provision Cloud servers
  hosts: all
  serial: 5
  gather_facts: false
  roles:
    - role: ngine_io.exoscale_compute
      delegate_to: localhost

  post_tasks:
    - name: Wait for SSH access
      delegate_to: localhost
      wait_for:
        host: "{{ ansible_host }}"
        port: 22
        timeout: 60

    - name: Wait for cloud-init finished
      wait_for:
        host: "{{ ansible_host }}"
        path: /var/lib/cloud/instance/boot-finished
        timeout: 600
```

### Common Configurations

Typical common configs in a top level group (e.g. `group_vars/all.yml`) for additional security groups and rules, affinity groups and networks:

```yaml
# file: group_vars/all.yml

# API: creds
exoscale__api_key: EXO...
exoscale__api_secret: ...$

# Zone: default ch-dk-2
exoscale__zone: ch-gva-2

# Template: depending on the templates used, there is a default user created which gets an SSH pub key
ansible_user: rockylinux
exoscale__server_template: "Rocky Linux 8 (Green Obsidian) 64-bit 2021-08-25-13bb54"

# Enable IPv6
exoscale__server_ipv6_enabled: true

# Security groups: create these secrurity groups.
# NOTE: Existing security groups with different names won't be touched.
# NOTE: Additional existing rules in the groups below won't be touched.
exoscale__security_groups:
  - name: proxy
    rules:
      - protocol: tcp
        type: ingress
        cidr: 0.0.0.0/0
        port: 443

      - protocol: tcp
        type: ingress
        cidr: 0.0.0.0/0
        port: 80

      - protocol: tcp
        type: ingress
        cidr: 1.2.3.4/32
        start_port: 8000
        end_port:  8999

  - name: mqtt
    rules:
      - port: 1883
      - port: 8883

# Affinity groups: create these groups.
exoscale__affinity_groups:
  - name: cassandra
  - name: proxy

# Create a private networks for backend access.
exoscale__networks:
  - name: my private net
    start_ip: 10.23.12.100
    end_ip: 10.23.12.150
    netmask: 255.255.255.0
```

### Instances config via Group / Host Vars

Assuming an inventory similar to the following:

```ini
[proxy]
proxy1.example.com
proxy2.example.com

[cassandra]
cassandra1.example.com
cassandra2.example.com
cassandra3.example.com
```

Ensure all proxy instances use a different offering and the additional security group proxy:

```yaml
#file: group_vars/proxy.yml

# Large offering (default is Small)
exoscale__server_service_offering: Medium

# Assign security groups to instances (default: [default])
exoscale__server_security_groups:
  - default
  - proxy

# Attach a private network
exoscale__server_networks:
  - my private net
```

Ensure all cassandra instances are on different hardware hosts:

```yaml
#file: group_vars/cassandra.yml

# Assign affinity group to all instances
exoscale__server_affinity_groups: cassandra

# Use CPU related hardware
exoscale__server_service_offering: CPU-extra-large

# Attach a private network
exoscale__server_networks:
  - my private net

```

----
> :information_source: **HINT**: get a list of all offerings `cs listServiceOfferings | jq --raw-output '.serviceoffering[].name'`.
> See https://github.com/exoscale/cs/ how to make use the library used underneath as CLI.
----

## License

MIT

## Author Information

René Moser (@resmo)
