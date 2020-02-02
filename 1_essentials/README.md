# Vagrant and Ansible Essentials

Lean how to setup local environnement with Vagrant
Understand ansible commands and configuration management

## What you will learn

* setup vagrant boxes 
* configure ansible with vagrant
* define ansible tasks 
* sync folder

## What you will build

An HTTP server with Nginx to serve local HTML file


## Vagrant 

### 1 - configure boxes

Create a Vagrantfile in your the current directory that conatins the basic configurations
 
```bash
vagrant init bento/ubuntu-18.10
```

Nb: you can use another recommanded box provider "bento"

```bash
vagrant init bento/centos-7.7
vagrant init bento/ubuntu-18.10
```

Run box

```bash
vagrant up
```

#### 1.1 Connect to your vm with SSH

Test that your machine is up using ssh protocol.

[ssh protocol](https://medium.com/@Magical_Mudit/understanding-ssh-workflow-66a0e8d4bf65)

```bash
vagrant ssh
```

### 2 - Set a static  IP for your machine 

```ruby
Vagrant.configure("2") do |config|
    # Configure web server machine
     config.vm.define "web" do |web1|
         # configure a virtual machine that use ubuntu 
         web1.vm.box = "bento/ubuntu-18.10"
        
         # configure a private network ip for the machine
         # Your machine will be accessible through this private static IP address that has been configured
         # Your machine is not accessible from the global internet
         web1.vm.network "private_network", ip: "192.169.10.10"
     end
end
```
As you can see, config.vm.define takes a block with another variable. This variable, such as **web** above, is the exact same as the **config** variable, except any configuration of the inner variable applies only to the machine being defined. Therefore, any configuration on **web** will only affect the **web** machine.



#### 2-1 Restart vm

When you have modified your **Vagrantfile** you need to apply those change to your machine by using this command : 

```bash
vagrant reload
```

This command is usually required for changes made in the Vagrantfile to take effect. After making any modifications to the Vagrantfile, a reload should be called.
or equivalent 

* --provision - Force the provisioners to run.

```bash
vagrant halt
vagrant up
```

Note : 

At this stage you have configured a virtual machine using **Vagrant**. This machine run with Ubuntu OS and we have configured a static private IP to communicate locally with it. 

#### 2-2 Ping machine

[Ping](https://en.wikipedia.org/wiki/Ping_(networking_utility) is a computer network administration software utility used to test the reachability of a host on an Internet Protocol (IP) network.

```bash
ping -c 3 192.169.10.10
```

**Note:** for more information about ping enter 

```
man ping 
```


### 3- Provisoning With shell script


Our goal is to configure an Nginx server allowing us to serve an HTML page. We will modify our
Vagrantfile so that it uses a shell script that does this for us.


#### 3-1 Update your Vagrantfile with provision


**Provisioner**

Provisioners in Vagrant allow you to automatically install software, alter configurations, and more on the machine as part of the vagrant up process.

This is useful since boxes typically are not built perfectly for your use case. Of course, if you want to just use vagrant ssh and install the software by hand, that works. But by using the provisioning systems built-in to Vagrant, it automates the process so that it is repeatable. Most importantly, it requires no human interaction, so you can vagrant destroy and vagrant up and have a fully ready-to-go work environment with a single command. Powerful.

Vagrant gives you multiple options for provisioning the machine, from simple shell scripts to more complex, industry-standard configuration management systems.


[Shell Provision](https://www.vagrantup.com/docs/provisioning/shell.html)


Add this line in the web config 

```ruby
web1.vm.provision :shell do |shell|
    shell.path = "web.sh"
end
```

- **path (string) - Path to a shell script to upload and execute. It can be a script relative to the project Vagrantfile or a remote script (like a gist).**



**For Ubuntu OS**

Your Vagrantfile should look like this now.

```ruby
Vagrant.configure("2") do |config|
    # Configure web server machine
     config.vm.define "web" do |web1|
         web1.vm.box = "bento/ubuntu-18.10"
         web1.vm.network "private_network", ip: "192.169.10.10"
         web1.vm.provision :shell do |shell|
             shell.path = "web.sh"
         end
     end
end
```

#### 3-2 Create script

Create a file name web.sh in your current directory the same as Vagrantfile.

*Note: Your web.sh file can be put somewhere else. Just update the path of your script file in the Vagrantfile*

Shell code  :

**For Debian OS** 

web.sh : 

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y nginx
mkdir -p /var/www/html/
touch /var/www/html/index.html
 echo "Hello !" >> /var/www/html/index.html
```

**For Centos OS** 

web.sh :

```bash
#!/bin/bash
service httpd stop
sudo yum update
sudo yum install -y epel-release nginx 
mkdir -p /var/www/html/
touch /var/www/html/index.html
echo "Hello !" >> /var/www/html/index.html
sudo systemctl start nginx
```

**Dont forget to restart your machine to apply new changes**

```bash
vagrant reload --provision
```


### Sync vm folder with local folder

Add current directory to vm 

```ruby
web1.vm.synced_folder ".", "/home/vagrant/files"s
```

Reload vagrant 

```bash
vagrant reload
```


### Define port forwarding

Forwar host port 8000 to vm guest port 80

```ruby
web.vm.network "forwarded_port", guest: 80, host: 8000
```

Reload vagrant 

```bash
vagrant reload
```

## Ansible commands


Notes about using vagrant with ansible :

* ssh keys are already provisionned by vagrant when creating a box
* the ssh user is vagrant
* ssh private key is located in '.vagrant/machines/<VM_NAME>/virtualbox/private_key' 
* ssh port is 2222
  
Tips to know ports 

```bash
vagrant port
```

### Ansible modules

Use Ping module for all machine with all parameters specified

```bash
ansible all -i '192.168.33.10,' -m ping -u vagrant  --private-key=./.vagrant/machines/default/virtualbox/private_key 
```

> Nb: this method is not the "ansible way", you should use an inventory file

> Nb: note the "," required with inline inventory


Define inventory file 'hosts' and add this line

```bash
192.168.33.10 ansible_ssh_user=vagrant ansible_ssh_private_key_file='.vagrant/machines/default/virtualbox/private_key'
```

Use ping module using inventory 

```bash
ansible all -i hosts  -m ping 
```

> Nb: we did not defined sections in our inventory file so we use 'all'


Use Setup module to get infos about vm 

```bash
ansible all -i hosts  -m setup
```

> Tips: use Sed and Jq to extract infos
> ansible all -i hosts -m setup | sed '1c {'|jq 


Use module apt to install Nginx

```bash
ansible all -i hosts -m apt -b -a "name=nginx state=latest"
```

> Nb : use '-b' argument to make vagrant user use sudo (required)

> Nb : we use apt for debian, use yum package manager for centos

Use module service to restart nginx

```bash
ansible all -i hosts -m service -b -a "name=nginx state=restarted"
```

### Define ansible playbook to install Nginx


Define hosts 

```yaml
---
  - hosts: all
```
> '---' is not mandatory but it designated Yaml file to interpretor

Need Sudo 

```yaml
---
  - hosts: all
    become: true
```

Add task to ensure nginx service is installed

```yaml
---
  - hosts: all
    become: true
    tasks:
    -
      name: "Ensure nginx is at the latest version"
      apt: "name=nginx state=latest"

```


Add task to ensure nginx is started

```yaml
---
  - hosts: all
    become: true
    tasks:
    -
      name: "Ensure nginx is at the latest version"
      apt: "name=nginx state=latest"
    -
      name: "start nginx"
      service:
        name: nginx
        state: started

```


### Use Ansible playbook with Vagrant provider


Use 'ansible_local' provisioning

```ruby
config.vm.provision "ansible_local" do |ans|
    ans.playbook = "playbook.yml"
    ans.install = true
    ans.install_mode = "pip"
end
```

> Nb: use playbook path on remote host ! (ex /home/vagrant/playbook.yml)

Reload VM with new provision

```bash
vagrant provision
```

### Use Ansible playbook in standalone

Replace shell script to use ansible provision :

```bash
ansible-playbook -i hosts playbook.yml
```

> Nb: we do not define host 'all'


## Playbooks useful tips


### Use loop

Example for creating user ssh

```yaml
## users.yml
---
- hosts: "localhost"
  connection: "local"
  vars:
    users:
    - "paul"
    - "tanya"
    - "ruby"
  tasks:
  - name: "Create user accounts"
    user:
      name: "{{ item }}"
      groups: "admin,www-data"
      state: "present"
    with_items: "{{ users }}"
  - name: "Add authorized keys"
    authorized_key:
      user: "{{ item }}"
      key: "{{ lookup('file', 'files/'+ item + '.key.pub') }}"
    with_items: "{{ users }}"
```

Store ssh pub keys in files directory 

```bash
├── files
│   ├── paul.key.pub
│   ├── ruby.key.pub
│   └── tanya.key.pub
├── users.yml
```

Change sudoer list with regex

```yaml
- name: "Allow admin users to sudo without a password"
    lineinfile:
      dest: "/etc/sudoers"
      state: "present"
      regexp: "^%admin"
      line: "%admin ALL=(ALL) NOPASSWD: ALL"
```


### Store task result

Example : Use URI module and debug variable

```yaml
- name: Get page content
  uri: 
    url: "http://169.254.169.254/info"
    return_content: yes
  register: page_result

- name: Show page_result
  debug: var=page_result

```

### Execute shell script

Example to run docker cmd

```yaml
- name: Run Container
  shell: "docker run -p 80:5000 -d playground:pgflask"
  args:
   chdir: "/tmp"
  become: yes
```

> nb: you assure docker cli is installed before

### Task executed only when condition

Download `docker-compose` script only if not exists 

```yaml
- name: Check if docker-compose file exists
  stat: 
    path: /usr/local/bin/docker-compose
  register: docker_compose

- name: Install Docker-compose
  shell: curl -L "https://github.com/docker/compose/releases/download/1.24.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose"
  become: true
  when: docker_compose.stat.exists == False

```

you can cast variable 

```yaml
when: whatever_var|bool
```

## Exercice

### Configure an Nginx HTTP Server 

* Start a VM with static IP 192.169.10.99
* use ansible to configure VM with Nginx server
* Serve local HTML files with VM server

#### How to check ?

You should see your html page content with this command

```bash
curl 192.169.10.99
```

