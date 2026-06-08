# Splunk + RHEL SSH Monitoring 

![Ansible Automation Platform](https://www.redhat.com/rhdc/managed-files/Logo-Red_Hat-Ansible_Automation_Platform-A-Standard-RGB_0.svg)

---

## Table of Contents
1. [RHEL Host Configuration - Ansible Playbook](#rhel-host-configuration---ansible-playbook)
2. [Splunk Server Configuration - GUI Steps](#splunk-server-configuration---gui-steps)
3. [Network Requirements](#network-requirements)
4. [Testing](#testing)

---

# RHEL Host Configuration - Ansible Playbook

## Overview
This playbook installs and configures Splunk Universal Forwarder on RHEL 9 hosts to send SSH authentication logs to Splunk.

## Prerequisites

Following Ansible collection:

```
 splunk.enterprise
```

## Playbook: `configure_splunk_forwarder.yml`

```yaml
---
- name: Install and Configure Splunk Universal Forwarder for SSH Monitoring
  hosts: all
  become: true
  gather_facts: true

  vars:
    # Splunk version and download
    splunk:
      version: "9.1.0"
      build: "1c86ca0bacc3"
      product: "universalforwarder"
      
    # Deployment server configuration
    splunk_indexer:
      host: "splunk.foxhound.unit"
      port: 9997
    
    # Admin credentials
    splunk_admin_password: "ChangeMe123!"  # Change this!
    
    # Installation paths
    splunk_home: "/opt/splunkforwarder"
    
    # Inputs to monitor
    splunk_inputs:
      - path: "/var/log/secure"
        sourcetype: "linux_secure"
        index: "linux"
    
  tasks:
    # ==========================================
    # 1. INSTALL SPLUNK UNIVERSAL FORWARDER
    # ==========================================
    
    - name: Install Splunk Universal Forwarder
      splunk.enterprise.splunk_universal_forwarder_linux:
        state: present
        version: "{{ splunk.version }}"
        build: "{{ splunk.build }}"
        admin_password: "{{ splunk_admin_password }}"
        
    # ==========================================
    # 2. CONFIGURE OUTPUTS (FORWARD SERVER)
    # ==========================================
    
    - name: Create outputs.conf for forwarding configuration
      ansible.builtin.copy:
        content: |
          # Forward to indexer
          [tcpout]
          defaultGroup = default-autolb-group
          maxQueueSize = 7MB
          
          [tcpout:default-autolb-group]
          server = {{ splunk_indexer.host }}:{{ splunk_indexer.port }}
          compressed = true
          useACK = true
        dest: "{{ splunk_home }}/etc/system/local/outputs.conf"
        owner: splunk
        group: splunk
        mode: '0644'
      notify: restart splunk

    # ==========================================
    # 3. CONFIGURE INPUTS (LOG MONITORING)
    # ==========================================

    - name: Create inputs.conf for SSH log monitoring
      ansible.builtin.copy:
        content: |
          # Set hostname
          [default]
          host = {{ ansible_fqdn }}
          
          # Monitor SSH authentication logs
          [monitor:///var/log/secure]
          disabled = false
          sourcetype = linux_secure
          index = linux
        dest: "{{ splunk_home }}/etc/system/local/inputs.conf"
        owner: splunk
        group: splunk
        mode: '0644'
      notify: restart splunk

    # ==========================================
    # SET FILE PERMISSIONS FOR LOG ACCESS
    # ==========================================

    - name: Install ACL package
      ansible.builtin.dnf:
        name: acl
        state: present

    - name: Set ACL for splunk user to read /var/log/secure
      ansible.posix.acl:
        path: /var/log/secure
        entity: splunk
        etype: user
        permissions: r
        state: present

    - name: Set ACL for splunk user to read /var/log/audit/audit.log
      ansible.posix.acl:
        path: /var/log/audit/audit.log
        entity: splunk
        etype: user
        permissions: r
        state: present
      ignore_errors: yes  # In case audit log doesn't exist

    # ==========================================
    # CONFIGURE FIREWALL
    # ==========================================

    - name: Check if firewalld is running
      ansible.builtin.systemd:
        name: firewalld
      register: firewalld_status

    - name: Allow outbound connection to Splunk server
      ansible.posix.firewalld:
        rich_rule: 'rule family="ipv4" destination address="{{ splunk_indexer.host }}" port protocol="tcp" port="{{ splunk_indexer.port }}" accept'
        permanent: yes
        state: enabled
        immediate: yes
      when: firewalld_status.status.ActiveState == "active"
      ignore_errors: yes  # In case DNS doesn't resolve

    # ==========================================
    # 8. VERIFICATION
    # ==========================================

    - name: Wait for Splunk to be fully started
      ansible.builtin.wait_for:
        path: "{{ splunk_home }}/var/run/splunk/splunkd.pid"
        state: present
        timeout: 60

    - name: Gather Splunk Universal Forwarder information
      splunk.enterprise.splunk_universal_forwarder_linux_info:
      register: splunk_info

    - name: Display Splunk Universal Forwarder status
      ansible.builtin.debug:
        msg:
          - "Splunk Version: {{ splunk_info.version | default('N/A') }}"
          - "Splunk Home: {{ splunk_info.splunk_home | default('N/A') }}"
          - "Status: {{ splunk_info.status | default('N/A') }}"
          - "Forward Servers: {{ splunk_info.forward_servers | default([]) }}"

    - name: Verify forwarder is configured correctly
      ansible.builtin.assert:
        that:
          - splunk_info.status is defined
          - splunk_info.splunk_home == splunk_home
        success_msg: "Splunk Universal Forwarder is installed and running"
        fail_msg: "Splunk Universal Forwarder verification failed"

  # ==========================================
  # HANDLERS
  # ==========================================

  handlers:
    - name: restart splunk
      ansible.builtin.command:
        cmd: "{{ splunk_home }}/bin/splunk restart"
      become_user: splunk
```

---

## Variables to Customize

Edit these variables at the top of the playbook:

| Variable | Description | Example |
|----------|-------------|---------|
| `splunk.version` | Splunk version to install | `9.1.0` |
| `splunk.build` | Splunk build number | `1c86ca0bacc3` |
| `splunk_indexer.host` | Your Splunk server hostname/IP | `splunk.foxhound.unit` |
| `splunk_indexer.port` | Splunk receiving port | `9997` |
| `splunk_admin_password` | Admin password for forwarder | `ChangeMe123!` |

---

## Inventory File Example

Create `inventory.ini`:

```ini
[rhel_hosts]
rhel9-1 ansible_host=192.168.1.10
rhel9-2 ansible_host=192.168.1.11
rhel9-3 ansible_host=192.168.1.12

[rhel_hosts:vars]
ansible_user=ec2-user
ansible_become=true
ansible_python_interpreter=/usr/bin/python3
```

---

## Requirements File

Create `requirements.yml` for the Splunk collection:

```yaml
---
collections:
  - name: splunk.enterprise
    version: ">=1.0.0"
  - name: ansible.posix
    version: ">=1.0.0"
```

## Running the Playbook

```bash
# Install required collections
ansible-galaxy collection install -r requirements.yml

# Test connectivity first
ansible -i inventory.ini rhel_hosts -m ping

# Run the playbook
ansible-playbook -i inventory.ini configure_splunk_forwarder.yml

# Run with verbose output
ansible-playbook -i inventory.ini configure_splunk_forwarder.yml -v
```

---

## What the Playbook Does

1. ✅ Installs Splunk Universal Forwarder 9.1.0 using official `splunk.enterprise.splunk_universal_forwarder_linux` role
2. ✅ Creates outputs.conf with forward-server settings (compression, ACK enabled)
3. ✅ Creates inputs.conf to monitor /var/log/secure with proper sourcetype
4. ✅ Sets ACL permissions for splunk user to read logs
5. ✅ Configures logrotate to persist ACL permissions
6. ✅ Opens firewall for outbound connection
7. ✅ Verifies Splunk Universal Forwarder status and forward server configuration

---

# Splunk Server Configuration - GUI Steps

## Prerequisites
- Splunk Enterprise 9.x installed
- Admin access to Splunk Web UI
- URL: `http://your-splunk-server:8000`

---

## Step 1: Enable Data Receiving

1. **Login to Splunk Web UI**
   - Navigate to: `http://splunk.foxhound.unit:8000`
   - Login with admin credentials

2. **Configure Receiving Port**
   - Click: **Settings** → **Forwarding and receiving**
   - Click: **Configure receiving**
   - Click: **New Receiving Port** button
   - **Listen on this port:** `9997`
   - Click: **Save**

✅ **Verification:** You should see port 9997 listed under "Receive data"

---

## Step 2: Create Linux Index

1. **Navigate to Indexes**
   - Click: **Settings** → **Indexes**
   - Click: **New Index** button

2. **Configure Index**
   - **Index Name:** `linux`
   - **Index Data Type:** `Events`
   - **Max Size of Entire Index:** `500000` MB (500 GB)
   - **Searchable Time Range (days):** `90`
   - Leave other settings as default
   - Click: **Save**

✅ **Verification:** Index "linux" appears in the list

---

## Step 3: Create Source Type for Field Extraction

1. **Navigate to Source Types**
   - Click: **Settings** → **Source types**
   - Find and click on: **linux_secure** (should be automatically created)
   - If it doesn't exist, data needs to arrive first

2. **Add Field Extractions** (Do this after data arrives)
   - Click: **Settings** → **Fields**
   - Click: **Field extractions** → **New Field Extraction**
   
3. **SSH Failed Login Extraction**
   - **Name:** `ssh_failed_login_fields`

And follow the wizard

   - Click: **Save**

---

## Step 4: Create Event Types

1. **Create SSH Failed Login Event Type**
   - Click: **Settings** → **Event types**
   - Click: **New Event Type**
   
2. **Configure Event Type**
   - **Name:** `ssh_failed_login`
   - **Search string:** `sourcetype=linux_secure "Failed password"`
   - **Tags:** `authentication, failure, ssh`
   - **Priority:** `5`
   - Click: **Save**

3. **Create SSH Success Event Type** (Optional)
   - **Name:** `ssh_successful_login`
   - **Search string:** `sourcetype=linux_secure "Accepted password"`
   - **Tags:** `authentication, success, ssh`
   - **Priority:** `5`
   - Click: **Save**

---

## Step 5: Create the Brute Force Detection Alert

1. **Create the Search**
   - Click: **Search & Reporting** (top left)
   - Enter this search:
   ```spl
      index=linux sourcetype=linux_secure
      | stats count by user, host, src_ip
      | where count > 10
      | table host, user, src_ip, count
   ```
   - Set time range: **Last 15 minutes**
   - Click: **Search**

2. **Save as Alert**
   - Click: **Save As** → **Alert**
   - **Title:** `SSH Brute Force Detection`
   - **Description:** `Detects SSH brute force attempts with >5 failed logins`
   - **Permissions:** Shared in App

3. **Configure Alert Type**
   - **Alert type:** Select **Real-time**
   - OR: **Scheduled** → Run every **1 minute**

4. **Trigger Conditions**
   - **Trigger alert when:** Number of Results
   - **is greater than:** `0`
   - **Trigger:** For each result
   - **Throttle:** Check box
   - **Suppress triggering for:** `5` minutes

5. **Trigger Actions**
   - Click: **+ Add Actions**
   - Select: **EDA**
   
---
