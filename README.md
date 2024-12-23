# Ansible Playbook: Docker and Docker Swarm Setup

## Overview
This Ansible playbook automates the following tasks:
1. Ensures the user has `sudo` privileges (and adds them if not).
2. Installs Docker and Docker Compose using `apt`.
3. Initializes Docker Swarm on the first manager node and joins the other nodes to the Swarm.

## Prerequisites
Before running the playbook, ensure that:
- Ansible is installed and configured on your local machine.
- You have a list of IP addresses for the manager nodes in the `inventory.ini` file.
- The user specified in `ansible.cfg` (or overridden in the playbook) has SSH access to the target machines.
- Docker and Docker Compose are not already installed on the target nodes.

## Playbook Structure

### 1. Setup

The playbook is designed to run on a group of manager nodes defined in the `inventory.ini` file.

### 2. Tasks
The playbook performs the following steps:

1. **Ensure Sudo Privileges**
   - It checks if the user has `sudo` privileges by running `sudo -v`.
   - If the user doesn't have `sudo` privileges, it adds them to the `sudo` group and configures passwordless sudo.

2. **Install Docker and Docker Compose**
   - The playbook updates the apt cache and installs `docker.io` and `docker-compose`.

3. **Start Docker Service**
   - The playbook ensures that the Docker service is started and enabled to run on system boot.

4. **Initialize Docker Swarm**
   - The playbook initializes Docker Swarm on the first manager node and retrieves the Swarm join token.

5. **Join Other Nodes to the Swarm**
   - Using the join token retrieved from the first manager, the playbook makes the other manager nodes join the Swarm cluster.

## Example Playbook

```yaml
---
- name: Setup Docker and Docker Swarm on Manager Nodes
  hosts: managers
  become: true  # Ensure tasks are run as root
  
  tasks:
    - name: Check if the user has sudo privileges
      command: sudo -v
      register: sudo_check
      ignore_errors: yes

    - name: Add user to sudo group if not already a member
      user:
        name: "{{ ansible_user }}"  # Replace with the username (e.g., ubuntu)
        groups: sudo
        append: yes
      when: sudo_check.failed  # Only add to sudo group if the user doesn't have sudo privileges

    - name: Ensure passwordless sudo for the user
      lineinfile:
        path: /etc/sudoers
        line: "{{ ansible_user }} ALL=(ALL) NOPASSWD:ALL"
        validate: '/usr/sbin/visudo -cf %s'
      when: sudo_check.failed  # Only apply this if sudo check failed

    - name: Update apt and install dependencies
      apt:
        update_cache: yes
        name:
          - docker.io
          - docker-compose
        state: present

    - name: Ensure Docker service is started and enabled
      service:
        name: docker
        state: started
        enabled: true

    - name: Initialize Docker Swarm on the first manager node
      shell: docker swarm init --advertise-addr {{ ansible_host }}
      when: inventory_hostname == groups['managers'][0]
      register: swarm_init

    - name: Get the Docker Swarm manager join token
      shell: docker swarm join-token -q manager
      when: inventory_hostname == groups['managers'][0]
      register: manager_token

    - name: Join the other nodes to the Docker Swarm
      shell: docker swarm join --token {{ hostvars[groups['managers'][0]]['manager_token']['stdout'] }} {{ hostvars[groups['managers'][0]]['ansible_host'] }}:2377
      when: inventory_hostname != groups['managers'][0]
