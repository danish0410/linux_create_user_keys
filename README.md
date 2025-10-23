hosts.ini
[localhost]
localhost ansible_connection=local

[targets][dev]
13.232.244.20 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/dev-classic-ap-south-1.pem

[myserver]
13.235.50.20 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/dev-classic-ap-south-1.pem

create_user_with_ssh_keys.yml
---
# ────────────────────────────────────────────────
# 1️⃣ Install AWS CLI and Secure Security Group
# ────────────────────────────────────────────────
- name: Install AWS CLI and update security group with current public IP
  hosts: localhost
  become: yes
  become_flags: "-E"
  gather_facts: no
  vars:
    security_group_id: sg-0a7742fe627ae00f5    # ✅ Replace with your SG ID
    region: ap-south-1                         # ✅ Replace with your region

  tasks:
    - name: Update apt package index
      apt:
        update_cache: yes

    - name: Install AWS CLI
      apt:
        name: awscli
        state: present

    - name: Verify AWS CLI installation
      command: aws --version
      register: aws_version
      changed_when: false

    - name: Show AWS CLI version
      debug:
        msg: "AWS CLI installed version: {{ aws_version.stdout }}"

    - name: Get current public IP
      uri:
        url: https://checkip.amazonaws.com
        return_content: yes
      register: my_ip

    - name: Show detected public IP
      debug:
        msg: "Current public IP: {{ my_ip.content | trim }}/32"

    - name: Remove insecure 0.0.0.0/0 SSH rule (if exists)
      command: >
        aws ec2 revoke-security-group-ingress
        --group-id {{ security_group_id }}
        --protocol tcp
        --port 22
        --cidr 0.0.0.0/0
        --region {{ region }}
      ignore_errors: yes
      delegate_to: localhost

    - name: Allow SSH from current public IP
      command: >
        aws ec2 authorize-security-group-ingress
        --group-id {{ security_group_id }}
        --protocol tcp
        --port 22
        --cidr {{ my_ip.content | trim }}/32
        --region {{ region }}
      delegate_to: localhost

    - name: Wait for security group rule to propagate
      pause:
        seconds: 10


# ────────────────────────────────────────────────
# 2️⃣ SSH into EC2 and Configure Users & SSH Keys
# ────────────────────────────────────────────────
- name: Create Linux users, group, and SSH key setup (ED25519)
  hosts: all
  become: yes
  vars:
    dev_group: dev
    dev_users:
      - thanigai
      - rajasekar
      - merryla

  tasks:
    - name: Ensure required packages are installed
      apt:
        name:
          - openssh-server
          - sudo
        state: present
        update_cache: yes

    - name: Create group "{{ dev_group }}"
      group:
        name: "{{ dev_group }}"
        state: present

    - name: Create users in the "{{ dev_group }}" group
      user:
        name: "{{ item }}"
        comment: "{{ item | capitalize }} User"
        shell: /bin/bash
        create_home: yes
        state: present
        groups: "{{ dev_group }},sudo"
        append: yes
      loop: "{{ dev_users }}"

    - name: Create .ssh directory for each user
      file:
        path: "/home/{{ item }}/.ssh"
        state: directory
        owner: "{{ item }}"
        group: "{{ dev_group }}"
        mode: '0700'
      loop: "{{ dev_users }}"

    - name: Generate ED25519 SSH key pairs for each user
      community.crypto.openssh_keypair:
        path: "/home/{{ item }}/.ssh/id_ed25519"
        type: ed25519
        owner: "{{ item }}"
        group: "{{ dev_group }}"
        mode: '0600'
      register: ssh_keys
      loop: "{{ dev_users }}"
      loop_control:
        label: "{{ item }}"

    - name: Add public key to authorized_keys for each user
      copy:
        content: "{{ item.public_key }}"
        dest: "/home/{{ item.item }}/.ssh/authorized_keys"
        owner: "{{ item.item }}"
        group: "{{ dev_group }}"
        mode: '0600'
      loop: "{{ ssh_keys.results }}"

    - name: Display generated public keys
      debug:
        msg: "User '{{ item.item }}' ED25519 public key:\n{{ item.public_key }}"
      loop: "{{ ssh_keys.results }}"
	  

export AWS_ACCESS_KEY_ID="YOUR_AWS_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_AWS_SECRET_KEY"
export AWS_DEFAULT_REGION="ap-south-1"

aws configure

ansible-playbook -i hosts.ini create_user_with_ssh_keys.yml


dpkg -l | grep ssh
cat /etc/os-release
sudo su - thanigai
sudo su - rajasekar
sudo su - merryla
ls -la ~/.ssh
cat ~/.ssh/authorized_keys
ls -l ~/.ssh/*
