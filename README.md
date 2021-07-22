# Ansible-playbook-to-install-Docker-and-Webserver-Container-Launch with in an AWS infra

##### CODE FLOW >>>>>>


######  - name: "Creating Aws Infrastructure"  
######  - name: "Aws - Creating KeyPair"   
######  - name: "Aws - Creating Secutiry Group"     
######  - name: "Aws - Creating Ec2 Instance"   
######  - name: "Aws - Creating Dynamic Inventory"
######  - name: Docker Configuration and Webserver
######  - name: Create Docker Yum Repo
######  - name: Install Docker (compatible version)
######  - name: Start and Enable Docker Service
######  - name: Install Python3
######  - name: Install Docker-Python module
######  - name: Pull httpd Image from Docker Hub
######  - name: Create Directory
######  - name: Copy webpage content
######  - name: Launch Webserver Container

# ###############################################################################################################################################
````sh
---
- name: "Creating Aws Infrastructure"
  hosts: localhost
  connection: local
  vars_files:
     - new.vars
     - credential.vars


  tasks:

    - name: "Aws - Creating KeyPair"
      ec2_key:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        name: "{{ key_name }}"
        state: present
      register: key_data

    - name: "Aws - Saving KeyPair"
      when: key_data.changed == true
      copy:
        content: "{{ key_data.key.private_key }}"
        dest: "{{ key_name }}.pem"
        mode: 0400


    - name: "Aws - Creating Secutiry Group"
      ec2_group:
        name: "{{ sg_name }}"
        description: "Created By Ansible"
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        rules:
          - proto: tcp
            from_port: 80
            to_port: 80
            cidr_ip: 0.0.0.0/0

          - proto: tcp
            from_port: 8080
            to_port: 8080
            cidr_ip: 0.0.0.0/0

          - proto: tcp
            from_port: 22
            to_port: 22
            cidr_ip: 0.0.0.0/0

      register: sg

    - name: "Aws - Creating Ec2 Instance"
      ec2:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        key_name: "{{ key_name }}"
        group_id:
          - "{{ sg.group_id }}"
        instance_type: t2.micro
        image: "ami-0ba62214afa52bec7"
        wait: yes
        instance_tags:
          Name: Ansible
        count_tag:
          Name: Ansible
        exact_count: 1
      register: ec2


    - name: "Fetching Ec2 Instance Details"
      ec2_instance_info:
        aws_access_key: "{{ access_key }}"
        aws_secret_key: "{{ secret_key }}"
        region: "{{ region }}"
        filters:
          "tag:Name": "Ansible"
          instance-state-name: [ "running"]
      register: ec2

    - name: "debug instance details"
      debug:
        var: ec2


    - name: "Aws - Creating Dynamic Inventory"
      add_host:
        name: "{{ item.public_ip_address }}"
        groups: "instances"
        ansible_host: "{{ item.public_ip_address }}"
        ansible_port: 22
        ansible_user: "ec2-user"
        ansible_ssh_private_key_file: "{{ key_name }}.pem"
        ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
      with_items:
        - "{{ ec2.instances }}"


- name: Docker Configuration and Webserver
  hosts: instances
  become: true
  vars_files:
     - new.vars
  tasks:

  - name: Create Docker Yum Repo
    yum_repository:
       name: Docker
       description: docker yum repo for rhel8
       baseurl: https://download.docker.com/linux/centos/7/x86_64/stable/
       gpgcheck: no

  - name: Install Docker
    command: "yum install docker-ce --nobest -y"

  - name: Start and Enable Docker Service
    service:
       name: "docker"
       state: started
       enabled: yes

  - name: Install Python3
    package:
       name: "python3-pip"
       state: present

  - name: Install Docker-Python module
    pip:
       name: "docker"
       state: present

  - name: Pull httpd Image from Docker Hub
    docker_image:
       name: "httpd"
       source: pull

  - name: Create Directory
    file:
       path: /webpages
       state: directory

  - name: Copy webpage content
    copy:
       content: "<center><h1>TESTING WEBSERVER CONTAINER</h1></center>"
       dest: "/webpages/index.html"

  - name: Launch Webserver Container
    docker_container:
       name: "webserver"
       image: "httpd"
       state: started
       exposed_ports: "80"
       ports: "8080:80"
       volumes: /webpages:/usr/local/apache2/htdocs
````
