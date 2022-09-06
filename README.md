# Infrastructure as Code

![image](https://user-images.githubusercontent.com/110126036/188429117-139b84ed-5e70-427b-99c4-9170aa266df6.png)

Steps: 

- Launch Vagrant using following vagrant file

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what

# MULTI SERVER/VMs environment 
#
Vagrant.configure("2") do |config|
# creating are Ansible controller
    config.vm.define "controller" do |controller|

     controller.vm.box = "bento/ubuntu-18.04"
    
     controller.vm.hostname = 'controller'
     # synced_folder to run provision.sh to set up ansible controller

     controller.vm.network :private_network, ip: "192.168.33.12"

     #config.hostsupdater.aliases = ["development.controller"] 

    end
# creating first VM called web  
    config.vm.define "web" do |web|

     web.vm.box = "bento/ubuntu-18.04"
     # downloading ubuntu 18.04 image

     web.vm.hostname = 'web'
     # assigning host name to the VM

     web.vm.network :private_network, ip: "192.168.33.10"
     #   assigning private IP

     #config.hostsupdater.aliases = ["development.web"]
     # creating a link called development.web so we can access web page with this link instread of an IP   

    end

# creating second VM called db
    config.vm.define "db" do |db|

     db.vm.box = "bento/ubuntu-18.04"

     db.vm.hostname = 'db'

     db.vm.network :private_network, ip: "192.168.33.11"

     # config.hostsupdater.aliases = ["development.db"]     
    end


end
```

- Run updates and upgrades on all machines (may take a while)
- SSH into the controller and run the following commands

1. `sudo apt-get install software-properties-common`
2. `sudo apt-add-repository ppa:ansible/ansible`
3. `sudo apt-get update -y`
4. `sudo apt-get install ansible -y`
5. `sudo apt install tree`

- Now edit the ansible.cfg file to include host_key_checking = false under the `[defaults]`

```
[defaults]
host_key_checking = false
```

- Now edit the hosts file to include

```
[web]
192.168.33.10 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant

[db]
192.168.33.11 ansible_connection=ssh ansible_ssh_user=vagrant ansible_ssh_pass=vagrant
```

default dir /etc/ansible

### Ansible ad-hoc commands

- Copies a file from controller to host `sudo ansible web -m copy -a "src=/etc/ansible/test.txt dest=/home/vagrant"`
- See response time `sudo ansible web -m shell -a "uptime"`
- See if services are running `sudo ansible all -a "systemctl status nginx"`

You can use all to denote for `all` running machines you can also specify delcaring the host name.

## Provisioning YAML Files

### Install NGINX
```
# create a playbook to install nginx inside web
# --- three dashes at the start of the file    

---

# add hosts or name of the host server
- hosts: web

# indentation is extremely important
# gather live information
  gather_facts: yes

# we need admin permissions
  become: true

# add the instructions
  tasks:
  - name: Install Nginx
    apt: pkg=nginx state=present

# the nginx server status is running
```

### Install NodeJS and NPM
```
---
- hosts: web

  gather_facts: yes
  become: true

  tasks:
  - name: Install NodeJS
    apt: pkg=nodejs

  - name: Install NPM
    apt: pkg=npm
```
### Configure Reverse Proxy
```
---
- name : Configure the Reverse Proxy for NodeAPP
  hosts: web


  tasks:
  - name: Disable NGINX Default Virtual Host
    become: yes
    ansible.legacy.command:
      cmd: unlink /etc/nginx/sites-enabled/default

  - name: Create NGINX conf file
    become: yes
    file:
      path: /etc/nginx/sites-available/node_proxy.conf
      state: touch

  - name: Amend NGINX Conf file
    become: yes
    blockinfile:
      path: /etc/nginx/sites-available/node_proxy.conf
      marker: ""
      block: |
        server {
            listen 80;
            location / {
                proxy_pass http://192.168.33.10:3000;
            }
        }

  - name: Link NGINX Node Reverse Proxy
    become: yes
    command:
      cmd: ln -s /etc/nginx/sites-available/node_proxy.conf /etc/nginx/sites-enabled/node_proxy.conf

  - name: Make sure NGINX service is running
    become: yes
    service:
      name: nginx
      state: restarted
      enabled: yes
```
https://geektechstuff.com/2020/05/05/installing-and-configuring-nginx-as-a-reverse-proxy-via-an-ansible-playbook/

### Transfers app code from controller to web node
```
---
- name: transfer code for app from controller to the web instance 
  hosts: web
  tasks:
  - name: download the files for the nodeapp
    become: true
    copy:
      src: /home/vagrant/eng122_ci_cd/
      dest: /home/vagrant/app
```
### Completes npm install inside the correct folder
```
---
- name: Launch the app
  hosts: web
  become: true
  tasks:
    - name: install npm in the directory
      shell: |
        cd /home/vagrant/app/app
        npm install
```
### Creates a Service which will run the app
```
---
- name: creating service for the nodeapp to run on
  hosts: web
  tasks:
  - name: Create nodeapp service file
    become: yes
    file:
      path: /etc/systemd/system/nodeapp.service
      state: touch

  - name: Amend nodeapp service file
    become: yes
    blockinfile:
      path: /etc/systemd/system/nodeapp.service
      marker: ""
      block: |
        [Unit]
        Description=Launch Node App

        [Service]
        Type=simple
        Restart=always
        user=root
        ExecStart=/usr/bin/nodeapp.sh

        [Install]
        WantedBy=multi-user.target
```
### Copies nodeapp.sh script from Controller to Web Node (not sure how to do write #/bin/bash in the blockinfile module)
```
---
- name: transfer prov script from controller to web
  hosts: web
  tasks:
  - name: download prov script
    become: true
    copy:
      src: /home/vagrant/nodeapp.sh
      dest: /usr/bin/nodeapp.sh
```
### The nodeapp.sh script
```
#!/bin/bash

cd /home/vagrant/app/app

npm start
```

### Launches the app and declares environment variable (through activating the service)
```
---
- name: launch the node app
  hosts: web
  become: true
  tasks:
    - name: make shell executable
      shell: chmod +x /usr/bin/nodeapp.sh

    - name: enable new service
      shell: sudo systemctl daemon-reload

    - name: enable nodeapp service
      shell: sudo systemctl enable nodeapp.service

    - name: start nodeapp service
      shell: sudo systemctl start nodeapp.service

    - name: check status
      shell : sudo systemctl status nodeapp.service
```

### NOTE ALL OF THESE STEPS CAN BE PUT INTO ONE PLAYBOOK

## DATABASE CONFIGURATION (FAIL)
### Using script
```
---
- name: configure the db
  hosts: db
  become: true
  tasks:

  - name: create script file
    become: yes
    file:
      path: /home/vagrant/provisioning.sh
      state: touch

  - name: write the script
    become: yes
    blockinfile:
      dest: /home/vagrant/provisioning.sh
      marker: ""
      block: |
        sudo apt-get update
        sudo apt-get upgrade -y
        sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv D68FA50FEA312927
        echo "deb https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list
        sudo apt-get update
        sudo apt-get upgrade -y
        sudo apt-get install -y mongodb-org=3.2.20 mongodb-org-server=3.2.20 mongodb-org-shell=3.2.20 mongodb-org-mongos=3.2.20 mongodb-org-tools=3.2.20
        sudo systemctl restart mongod
        sudo systemctl enable mongod

  - name: make executable
    become: yes
    shell: chmod +x /home/vagrant/provisioning.sh

  - name: execute shell
    become: yes
    shell: ./provisioning.sh

  - name: sort mongod.conf file
    become: yes
    blockinfile:
      dest: /etc/mongod.conf
      marker: ""
      block: |
        # mongod.conf

        # for documentation of all options, see:
        #   http://docs.mongodb.org/manual/reference/configuration-options/

        # Where and how to store data.
        storage:
          dbPath: /var/lib/mongodb
          journal:
              enabled: true
        #  engine:
        #  mmapv1:
        #  wiredTiger:

        # where to write logging data.
        systemLog:
          destination: file
          logAppend: true
          path: /var/log/mongodb/mongod.log

        # network interfaces
        net:
          port: 27017
          bindIp: 0.0.0.0


        #processManagement:

        #security:

          #operationProfiling:

          #replication:

          #sharding:

          ## Enterprise-Only Options:

          #auditLog:

          #snmp:
```

### Using Shell 
```
---
- name: configure the db
  hosts: db
  become: true
  tasks:
    - name: make shell executable
      blockinfile: |
        dest: /home/vagrant/provisioning.sh 
        marker: ""
        block: |
          sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv D68FA50FEA312927
          echo "deb https://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-3.2.list

        # updates ubuntu
        sudo apt-get update
        sudo apt-get upgrade -y

        sudo apt-get install -y mongodb-org=3.2.20 mongodb-org-server=3.2.20 mongodb-org-shell=3.2.20 mongodb-org-mongos=3.2.20 mongodb-org-tools=3.2.20

        # enables and starts mongodb
        sudo systemctl restart mongod
        sudo systemctl enable mongod

    - name: sort mongod.conf file
      become: yes
      blockinfile:
        dest: /etc/mongod.conf
        marker: ""
        block: |
          # mongod.conf

          # for documentation of all options, see:
          #   http://docs.mongodb.org/manual/reference/configuration-options/      

          # Where and how to store data.
          storage:
            dbPath: /var/lib/mongodb
            journal:
              enabled: true
          #  engine:
          #  mmapv1:
          #  wiredTiger:

          # where to write logging data.
          systemLog:
            destination: file
            logAppend: true
            path: /var/log/mongodb/mongod.log

          # network interfaces
          net:
            port: 27017
            bindIp: 0.0.0.0


          #processManagement:

          #security:

          #operationProfiling:

          #replication:

          #sharding:

          ## Enterprise-Only Options:

          #auditLog:

          #snmp:
```
## Hybrid

Steps:
Install Ansible Vault Dependencies:
- python 3.7 or above
- pip3
- awscli
- ansible vault folder structure /etc/ansible/group_vars/all/file.yml to encrypt AWS keys

### Creating folder for encrypted keys
``` 
cd /etc/ansible/
mkdir group_vars
cd group_vars/
mkdir all/
cd all
sudo ansible-vault create pass.yml
sudo chmod 600 pass.yml
```

When inside the pass.yml file enter
```
access_key_id: xxxx
secret_key_id: xxxx
```
NOTE to exit .vi files enter `esc wq! enter`

### Create a key pair to send to aws

```
cd ~/.ssh/
ssh-keygen -t rsa -b 4096
User
```

### Install the dependencies

```
sudo apt install python3-pip
pip3 install awscli
pip3 install boto boto3
```

### Edit the Hosts File in /etc/ansible

```
[local]
localhost ansible_python_interpreter=/usr/local/bin/python3

[aws]
ec2-instance ansible_host=ec2-ip ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/eng122.pem
```

### Create a Playbook to Create an EC2 Instance
```
# AWS playbook
---

- hosts: localhost
  connection: local
  gather_facts: False

  vars:
    ansible_python_interpreter: /usr/bin/python3
    key_name: eng122
    region: eu-west-1
    image: ami-0a195d26685c18f3c
    id: "db-app-test"
    sec_group: "{{ id }}-sec"


  tasks:

    - name: Facts
      block:

      - name: Get instances facts
        ec2_instance_facts:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          region: "{{ region }}"
        register: result

      - name: Instances ID
        debug:
          msg: "ID: {{ item.instance_id }} - State: {{ item.state.name }} - Public DNS: {{ item.public_dns_name }}"
        loop: "{{ result.instances }}"

      tags: always


    - name: Provisioning EC2 instances
      block:

      - name: Upload public key to AWS
        ec2_key:
          name: "{{ key_name }}"
          key_material: "{{ lookup('file', '/home/vagrant/.ssh/{{ key_name }}.pub') }}"
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"

      - name: Create security group
        ec2_group:
          name: "{{ sec_group }}"
          description: "Sec group for app {{ id }}"
          # vpc_id: 12345
          region: "{{ region }}"
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          rules:
            - proto: tcp
              ports:
                - 22
              cidr_ip: 0.0.0.0/0
              rule_desc: allow all on ssh port
        register: result_sec_group

      - name: Provision instance(s)
        ec2:
          aws_access_key: "{{aws_access_key}}"
          aws_secret_key: "{{aws_secret_key}}"
          key_name: "{{ key_name }}"
          id: "{{ id }}"
          group_id: "{{ result_sec_group.group_id }}"
          image: "{{ image }}"
          instance_type: t2.micro
          region: "{{ region }}"
          wait: true
          count: 2
          # exact_count: 2
          count_tag:
            Name: sam_db1
          instance_tags:
             Name: sam_db

      tags: ['never', 'create_ec2']
```
### Launch the instance
`sudo ansible-playbook playbook.yml --ask-vault-pass --tags ec2_create`
