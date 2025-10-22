hosts.ini
[myserver]
13.235.50.20 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/dev-classic-ap-south-1.pem

create_user_with_ssh_keys.yml
---
- name: Create user and SSH key setup
  hosts: all
  become: yes
  tasks:
    - name: Ensure required packages are installed
      apt:
        name:
          - openssh-server
          - sudo
        state: present
        update_cache: yes

    - name: Create user 'thanigai'
      user:
        name: thanigai
        comment: "Thanigai User"
        shell: /bin/bash
        create_home: yes
        state: present
        groups: sudo
        append: yes

    - name: Create .ssh directory for user 'thanigai'
      file:
        path: /home/thanigai/.ssh
        state: directory
        owner: thanigai
        group: thanigai
        mode: '0700'

    - name: Generate SSH key pair for user 'thanigai' (written on remote)
      community.crypto.openssh_keypair:
        path: /home/thanigai/.ssh/id_rsa
        type: rsa
        size: 2048
        owner: thanigai
        group: thanigai
        mode: '0600'
      register: ssh_keypair

    - name: Add public key to authorized_keys
      copy:
        content: "{{ ssh_keypair.public_key }}"
        dest: /home/thanigai/.ssh/authorized_keys
        owner: thanigai
        group: thanigai
        mode: '0600'
		
ansible-playbook -i hosts.ini create_user_with_ssh_keys.yml

dpkg -l | grep ssh
cat /etc/os-release
sudo su - thanigai
ls -la ~/.ssh
cat ~/.ssh/id_rsa.pub
cat ~/.ssh/authorized_keys
ls -l ~/.ssh/*
