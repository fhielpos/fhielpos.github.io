---
layout: post
title: "Setting up a Wazuh lab - Part 1 [Installation]"
---

Ever wanted to mount your own Wazuh lab? I wanted to make my labs more "realistic" so, I decided to throw a Droplet on the internet and install Wazuh on it to see what would happen. This, is my journey building this lab.

<img src="assets/img/wazuh.png" alt="Wazuh Open Source Security Platform" width=500px height=auto>

**Wazuh** is an Open Source Host-based Intrusion Detection System. *Disclaimer: I work on Wazuh, but I **really** love this tool.* It is a great tool to provide observability on security events.

In this first part, I will focus on how I deployed this box.

## What I installed:

* Wazuh Manager 4.2.5 (Latest release)
* EFK OpenDistro (Elasticsearch, Filebeat, Kibana)

## Machine used

This lab runs on a very lightweight (and cheap) machine on DigitalOcean. Here are the specs:

* 2 GB RAM
* 1 vCPU
* 25GB Disk
* Ubuntu LTS 18.04 x64

## Installation

First things first, I added my `SSH public key` to have quick access to the VM. Easy and quick access is mandatory for all my labs.

With the access resolved, installing the `Wazuh stack` (Wazuh and the EFK stack) is as easy as running the following command:

```bash
curl -so ~/unattended-installation.sh https://packages.wazuh.com/resources/4.2/open-distro/unattended-installation/unattended-installation.sh && \
    bash ~/unattended-installation.sh
```

However, if you are like me, and want to dig around the installation, you can follow these steps:
* [Wazuh All-in-One Step by Step](https://documentation.wazuh.com/current/installation-guide/open-distro/all-in-one-deployment/all-in-one.html)


## Users customization

This is a lab, but I don't want a `by default` lab, so I made some customizations, starting with my users. The `users`, `roles`, and `roles_mapping` are managed by the `OpenDistro Security` plugin and defined by `YAML` files.

### 1. Gathering the actual users

To obtain the current users, we can create a backup of the `.opendistro_security` index inside Elasticsearch, to do so, we need to call the `securityadmin.sh` script:

```bash
/usr/share/elasticsearch/plugins/opendistro_security/tools/securityadmin.sh \
-backup ~/security_backup/ \
-cacert /etc/elasticsearch/certs/root-ca.pem \
-cert /etc/elasticsearch/certs/admin.pem \
-key /etc/elasticsearch/certs/admin-key.pem \
-nhnv
```

If you see any `WARNING: JAVA_HOME not set` errors, export the following variable:

```bash
export JAVA_HOME=/usr/share/elasticsearch/jdk/
```

This will download all the `YAML` files inside our **home** into the **security_backup** directory. There, I want to modify the following:
1. My `wazuh` user password.
2. The `wazuh_ui_user` role.
3. Create a new `wazuh_ro` user and assign it to the `wazuh_ui_user` role.

### 2. Modifying passwords and creating new users

So, lets get started. First, I will need to create a `hash` for my new password. This can be done with the `hash.sh` tool as follows:

```bash
/usr/share/elasticsearch/plugins/opendistro_security/tools/hash.sh -p "MyVerySecurePassword"
# OUTPUT:
$2y$12$elpaNQWK0MJvJod1nUwiK./AkE0j.uS2/Uv8LxR8QUvFCPNH09KwS
```

I will repeat this step to create a hash for my `wazuh_ro` user. Once I have both, I can modify the `~/security_backup/internal_users.yml` file:

```bash
vi ~/security_backup/internal_users.yml
```

Here, I will modify the `wazuh` hash and add the `wazuh_ro` user at the end of the file. I also deleted all the "unnecessary" users.

**internal_users.yml**
```yaml
---
_meta:
  type: "internalusers"
  config_version: 2
wazuh:
  hash: "$2y$12$elpaNQWK0MJvJod1nUwiK./AkE0j.uS2/Uv8LxR8QUvFCPNH09KwS"
  reserved: true
  backend_roles:
  - "admin"
  description: "Wazuh admin user"
kibanaserver:
  hash: "$2y$12$elpaNQWK0MJvJod1nUwiK./AkE0j.uS2/Uv8LximyUvFCPNH09KwS"
  reserved: true
  description: "Demo kibanaserver user"
wazuh_ro:
  hash: "$2y$12$39oVvvBV6n2s3jP3TeoOeOgN./AkE0j.uS2/W1z.8jimyqZ/RIpRS"
  reserved: true
  description: "Wazuh Read-Only user"
```

*The `reserved: true` definition prevents the user from being modified from the Kibana UI*

### 3. Modifying the roles

To modify the `roles` we need to follow the same steps as before, simply calling a `vi` or `nano` (or any text editor of your choice) to modify the `roles.yml` file:

```bash
vi ~/security_backup/roles.yml
```

Here I will only focus on the `wazuh_ui_user` role. I want it to be the only role needed to access the whole interface with `RO` permissions. I will show the before and after:

**roles.yml (before)**
```yaml
wazuh_ui_user:
  reserved: true
  hidden: false
  cluster_permissions: []
  index_permissions:
  - index_patterns:
    - "wazuh-*"
    dls: ""
    fls: []
    masked_fields: []
    allowed_actions:
    - "read"
  tenant_permissions: []
  static: false    
```

This role is **enough** for access to the `Wazuh` indices. However, we also need access to the `.kibana` indices and the `Global tenant`. I also decided to mask some fields like `srcip`, `full_log`, and `dstip` so any sensitive data is not shown.

**roles.yml (after)**
```yaml
wazuh_ui_user:
  reserved: true
  hidden: false
  cluster_permissions: []
  index_permissions:
  - index_patterns:
    - "wazuh-*"
    - ".kibana*"
    dls: ""
    fls: []
    masked_fields:
    - "data.srcip"
    - "full_log"
    - "previous_output"
    - "data.dstip"
    allowed_actions:
    - "read"
  tenant_permissions:
  - tenant_patterns:
    - "global_tenant"
    allowed_actions:
    - "kibana_all_read"
  static: false
```

It might look complicated, but we only did a couple of things:

* Adding `.kibana*` index to the allowed indices. This will give access to **Saved Objects** like `Dashboards`, `index-patterns`, and other required things to access apps on Kibana.
* Masking the `data.srcip`, `full_log`, `previous_output` and `data.dstip` fields. This will prevent this user from seeing the content of the fields. Instead, they will see a hashed string. This is completely not needed. I just wanted to play around.
* Adding `read_only` permissions to the `global_tenant`.

### 4. Assigning the `wazuh_ro` user to the `wazuh_ui_user` role

Finally, we will modify the `roles_mapping.yml` file to assign users to a specific role:

```bash
vi ~/security_backup/roles_mapping.yml
```

**roles_mapping.yml**
```yaml
wazuh_ui_user:
  reserved: true
  hidden: false
  backend_roles: []
  hosts: []
  users:
  - "wazuh_user"
  - "wazuh_ro"
  and_backend_roles: []
```

Here, we only added `wazuh_ro` to the `users` array.


### 5. Applying the new config

Now that we have all of our config files ready, we can push this new config to the `.opendistro_security` index with the `securityadmin.sh` script with the following command:

```bash
/usr/share/elasticsearch/plugins/opendistro_security/tools/securityadmin.sh \
-cd ~/security_backup/ \
-cacert /etc/elasticsearch/certs/root-ca.pem \
-cert /etc/elasticsearch/certs/admin.pem \
-key /etc/elasticsearch/certs/admin-key.pem \
-nhnv
```

As you can see, the command is similar to the `backup` one, but in this case, we swapped `backup` with `cd` which, specifies the directory where the `yaml` files are.

Here is a successful output:

```log
Open Distro Security Admin v7
Will connect to localhost:9300 ... done
Connected as CN=admin,OU=Docu,O=Wazuh,L=California,C=US
Elasticsearch Version: 7.10.2
Open Distro Security Version: 1.13.1.0
Contacting elasticsearch cluster 'elasticsearch' and wait for YELLOW clusterstate ...
Clustername: elasticsearch
Clusterstate: YELLOW
Number of nodes: 1
Number of data nodes: 1
.opendistro_security index already exists, so we do not need to create one.
Populate config from /root/security_backup/
Will update '_doc/config' with /root/security_backup/config.yml
   SUCC: Configuration for 'config' created or updated
Will update '_doc/roles' with /root/security_backup/roles.yml
   SUCC: Configuration for 'roles' created or updated
Will update '_doc/rolesmapping' with /root/security_backup/roles_mapping.yml
   SUCC: Configuration for 'rolesmapping' created or updated
Will update '_doc/internalusers' with /root/security_backup/internal_users.yml
   SUCC: Configuration for 'internalusers' created or updated
Will update '_doc/actiongroups' with /root/security_backup/action_groups.yml
   SUCC: Configuration for 'actiongroups' created or updated
Will update '_doc/tenants' with /root/security_backup/tenants.yml
   SUCC: Configuration for 'tenants' created or updated
Will update '_doc/nodesdn' with /root/security_backup/nodes_dn.yml
   SUCC: Configuration for 'nodesdn' created or updated
Will update '_doc/whitelist' with /root/security_backup/whitelist.yml
   SUCC: Configuration for 'whitelist' created or updated
Will update '_doc/audit' with /root/security_backup/audit.yml
   SUCC: Configuration for 'audit' created or updated
Done with success
```


That's it! Now I can go to my Droplet's IP and access the Kibana interface with my `wazuh` user.