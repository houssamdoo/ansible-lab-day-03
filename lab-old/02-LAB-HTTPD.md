# Ansible: Ansible playbooks - Install HTTPD

## requirements

```bash
ansible-galaxy collection install ansible.posix
```

## 🔹 Step 01: create an inventory file

`inventory`

```ini
nginx-node ansible_host=15.15.15.12
```

## 🔹 Step 02: Create the Ansible Playbook

`httpd.yaml` file.

```yaml
---
- name: "Install HTTPD"
  hosts: all
  become: yes
  tasks:
    - name: "Install httpd package"
      #package:
      yum:
        name: httpd
        state: present

    - name: Ensure httpd service is running and enabled
      service:
        name: httpd
        state: started
        enabled: yes

    - name: "Open firewall ports for HTTP/HTTPS (optional)"
      firewalld:
        service: "{{ item }}"
        permanent: yes
        immediate: yes
        state: enabled
      loop:
        - http
        - https
      when:
        - ansible_os_family == 'RedHat'
        - ansible_distribution_major_version == '9'

  handlers:
    - name: Reload firewalld
      ansible.builtin.service:
        name: firewalld
        state: reloaded
```

## 🔹 Step 03: Launch Playbook

```bash
ansible-playbook all -i inventory httpd.yaml
```

# Concepts

## Ansible Collection

An Ansible collection is a distribution format for Ansible content that can include playbooks, roles, modules, and plugins. Collections are designed to address a set of related use cases and can be packaged and distributed using Ansible Galaxy or a private Automation Hub instance.

Collections allow Ansible developers to contribute content independently from the core Ansible engine, making it easier to distribute and manage modules, roles, and plugins.
When you install a collection, it is typically placed in the ~/.ansible/collections directory, but you can specify a different path if needed.

To install a collection, you can use the `ansible-galaxy` command, which comes bundled with Ansible. For example, to install the sensu.sensu_go collection, you would run:
