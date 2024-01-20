# Notes for Ansible Devops Beginners

# Configuration Files
A default configuration of ansible file is `/etc/ansible/ansible.cfg`
```sh
    [defaults]
    gathering   = implicit # search config file in dir
    [inventory]
```
You must to set *environment variable* for custom configuration file location
```sh
$ANSIBLE_CONFIG=/opt/ansible-web.cfg ansible-playbook playbook.yml
```
**Custom** configuration file will overwrite the **default** configuration file

There are four level of configuration file 
1. custom config
2. dir config
3. user home dir config
4. etc config
It works like *union* operation, same instruction will overwrite with the highest priority.

### Change config with Ansible env variable
```sh
ANSIBLE_GATHERING=explicit ansible-playbook playbook.yml # only for playbook.yml
```

### List all configurations
```sh
ansible-config list # lists all configurations
ansible-config view # shows the current config file
ansible-config dump # shows the comprehensive current settings
```

# What is YAML?
> **dictionary** is *unordered* collection and **list** is *ordered* collection

```yaml
# Key-Value Pair
Fruit: Apple    # must have a space between : and apple
Vegetable: Carrot
Liquid: Water
Meat: Chicken

# Array/Lists
Fruits:
-   Orange
-   Apple
-   Banana
Vegetables:
-   Carrot
-   Cauliflower
-   Tomato

# Dictionary/Map
Banana:
    Calories: 105
    Fat: 0.4g
    Carbs: 27g
Grapes:
    Calories: 62
    Fat: 0.3g
    Carbs: 16g
```

# Ansible Inventory
- Ansible establishes the connections to servers with `SSH` in Linux and Powershell Remoting in windows.
- You don't need any additonal software or agent to work with *Ansible*
- The default *Ansible inventory* location is `/etc/Ansible/hosts`
- It uses `.ini` file format
```sh
# /etc/Ansible/hosts
server1.company.com
server2.company.com

[mail]
server3.company.com
server4.company.com

[db]
server4.company.com
server5.company.com

[web]
server6.company.com
server7.company.com

# using alias
web     ansible_host=server1.company.com    ansible_connection=ssh
db      ansible_host=server2.company.com    ansible_connection=winrm
mail    ansible_host=server3.company.com    ansible_connection=ssh
web2    ansible_host=server4.company.com    ansible_connection=winrm

localhost   ansible_connection=localhost

[web_servers]   # group
web
[db_servers]    # group
db

[all_servers:children]  # group child-group
web_servers
db_servers

# Other Inventory Parameters
    #   ansible_connection  - ssh/winrm/localhost
    #   ansible_port        - 22/5986
    #   ansible_user        - root/administrator
    #   ansible_ssh_pass    - password
    #   ansible_password    - password # for window
```

# Inventory Formats
There are *two* main types of *Inventory Format* types
1. *ini* file
2. *yaml* file

```sh
# ini file format
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
```
```yaml
# yaml file format
all:
    children:
        webservers:
            hosts:
                web1.example.com
                web2.example.com
        dbservers:
            hosts:
                db1.example.com
                db2.example.com
```

# Grouping and Parent-Child Relationships
- In *ini* format, *groups* are defined in square brackets
```ini
# In ini file
[webservers:children]
webservers_us
Webservers_eu

[webservers_us]
server1_us.com ansible_host=192.168.8.101
server2_us.com ansible_host=192.168.8.102

[webservers_eu]
server1_eu.com ansible_host=10.12.0.101
server2_eu.com ansible_host=10.12.0.102
```
```yaml
# In yaml file
all:
    children:
        webservers:
            children:
                webservers_us:
                    hosts:
                        server1_us.com:
                            ansible_host: 192.168.8.101
                        server2_us.com:
                            ansible_host: 192.168.8.102
                webservers_eu:
                    hosts:
                        server1_eu.com:
                            ansible_host: 10.12.0.101
                        server2_eu.com:
                            ansible_host: 10.12.0.102

```

# Ansible Variables
```yaml
- 
    name: Add DNS Server to resolv.conf
    hosts: localhost
    vars:
        dns_server: 10.1.250.10
    tasks:
        - lineinfile:
            path: /etc/resolv.conf
            line: 'nameserver {{ dns_server }}'

## Firewall Configuration
# {{ }} is called Jinja2 Templating
# source: {{ inter_ip_range }}  # not correct
# source: '{{ inter_ip_range}}' # correct
# source: Something{{ inter_ip_rnage }}Something # correct

http_port: 8081
snmp_port: 161-162
inter_ip_range: 192.0.2.0
-
    name: Set Firewall Configurations
    hosts: web
    tasks:
        - firewalld:
            service: https
            permanent: true
            state: enabled
        - firewalld:
            port: '{{http_port}}'/tcp
            permanent: true
            state: disabled
        - firewalld:
            port: '{{snmp_port}}'/udp
            permanent: true
            state: disabled
        - firewalld:
            source: '{{inter_ip_range}}'/24
            Zone: internal
            state: enabled
```
```ini
# with ini file
Web http_port=8081 snmp_port=161-162 inter_ip_range=192.0.2.0
```

# Variable Types
### Number Variables
It can hold *integer, float* and can do *math* operations
### Boolean Variables
It is for holding truthy or falsy values
```yaml
True, 'true', 't', 'yes', 'y', 'on', '1', 1, 1.0    # truthy values
False, 'false', 'f', 'no', 'n', 'off', '0', 0, 0.0  # falsy values
```
### List variables
To hold ordered collection of values and the values can be of any type
```yaml
packages:
    - nginx
    - postgresql
    - git

# access with {{ packages[0] }}
```
### Dictionary Variables
To hold a collection of key-value pairs and the keys and values can be of any type.
```yaml
user: 
    name: "admin"
    password: "secret"

# access with {{ user.name }}
```

# Registering Variables and Variable Precendence
```ini
web1 ansible_host=172.20.1.100
web2 ansible_host=172.20.1.101  dns_server=10.5.5.4 # this is call `host` variable
web3 ansible_host=172.20.1.102

[web_servers]
web1
web2
web3

[web_servers:vars]
dns_server=10.5.5.3 # this is call `group` variable

# overwrite 10.5.5.3 with 10.5.5.4 (inline call host) for web2
# host variables (inline) have over precedence to group variables
```

```yaml
---
-   name: Configure DNS Server
    hosts: all
    vars:
        dns_server: 10.5.5.5
    tasks:
        - nsupdate: 
            server: '{{ dns_server }}'

# play variables have over precendence to host variables
# **play > host > group**
```
```sh
ansible-playbook playbook.yml --extra-vars "dns-server=10.5.5.6"

# extra-vars is the highest precedence over all variables
# extra-vars > play > host > group
```
```yaml
---
-   name: Check /etc/hosts file
    hosts: all  # hosts directive to check all hosts
    tasks:      # shell script commands
        -   shell: cat /etc/hosts
            register: result    # register directive to hold the command result
        -   debug:              # output
            var: result.stdout  # stdout contained in result
# any variable specified with `register` directive falls under the scope of that host
-   name: Play2
    hosts: all
    tasks:
        - debug: 
            var: result.rc  # still can access from the above
```
```bash
# if you don't want to use the `debug` module, pass options to command
ansible-playbook -i inventory playbook.yaml -v # -v is equal to debug module
```

# Variable Scopes
```sh
# /etc/ansible/hosts
web1 ansible_host=172.20.1.100
web2 ansible_host=172.20.1.101  dns_server=10.5.5.4
web3 ansible_host=172.20.1.102 
```
```yaml
---
-   name: Print dns server
    hosts: all
    tasks:
    -   debug: 
        msg: '{{ dns_server }}'
```
### Variable Scope - Host
> *Host* variable is accessible within the *play* variable.
```ini
web1 ansible_host=172.20.1.100
web2 ansible_host=172.20.1.101  dns_server=10.5.5.4
web3 ansible_host=172.20.1.102
```
```yaml
---
-   name: Configure DNS Server
    hosts: all
    vars:
        dns_server: 10.5.5.5
    tasks:
        - nsupdate: 
            server: '{{ dns_server }}'
```

### Variable Scope - Play
> Variable defined in *one play* can not reference from *another play*
```yaml
---
-   name: Play1
    hosts: web1
    vars:
        ntp_server: 10.1.1.1
    tasks:
        - debug:
            var: ntp_server # 10.1.1.1

-   name: Play2
    hosts: web1
    tasks:
        - debug:
            var: ntp_server # VARIABLE IS NOT DEFINED!
```

### Variable Scope - Global
```bash
ansible-playbook playbook.yml --extra-vars "ntp_server=10.1.1.1"

# global variables are the highest precedence variables
```

# Magic Variables
```ini
web1 ansible_host=172.20.1.100
web2 ansible_host=172.20.1.101 dns_server=10.5.5.4
web3 ansible_host=172.20.1.102

# grouping
[web_servers]
web1
web2
web3

[americas]
web1
web2

[asia]
web3

# this will interpolate to
# web1
inventory_hostname=web1
ansible_host=172.20.1.100

# web2
inventory_hostname=web2
ansible_host=172.20.1.101
dns_server=10.5.5.4

#web3
inventory_hostname=web3
ansible_host=172.20.1.103
```
### Magic Variable - hostvars
```yaml
-   name: Print dns server
    hosts: all
    tasks:
        - debug:
            msg: '{{ dns_server}}'
            msg: '{{ hostvars['web2'].dns_server }}'
            msg: '{{ hostvars['web2'].ansible_host }}'
            msg: '{{ hostvars['web2'].ansible_facts.architecture }}'
            msg: '{{ hostvars['web2'].ansible_facts.devices }}'
            msg: '{{ hostvars['web2'].ansible_facts.mounts }}'
            msg: '{{ hostvars['web2'].ansible_facts.processor }}'
            msg: '{{ hostvars['web2']['ansible_facts']['processor'] }}'
# here web1 & web3 dns_server is not defined and cannot access
# but we can use magic variable to get the web2 dns_server and insert to web1 & web3
# **hostvars['web2'] is magic variable
```

### Magic Variable - group
```yml
-   name: Print dns server
    hosts: all
    tasks:
        - debug:
            msg: '{{ groups['americas'] }}' # web1 & web2
```

### Magic Variable - group_names
```yml
-   name: Print dns server
    hosts: web1
    tasks:
        - debug:
            msg: '{{ group_names }}' # web_severs & americas
```

### Magic Variable - inventory_hostname
```yml
-   name: Print dns server
    hosts: web1
    tasks:
        - debug:
            msg: '{{ inventory_hostname }}' # web1
```

# Ansible Facts
When Ansible connects to a target machine, it first collects the system information such as architecture, version of OS, processor details, memory details, serial numbers, different interfaces, IP addresses, FQDN, mac address, diff disks, mounts, amount of space available on them, date and time and etc. These are facts of Ansible and gather by `setup` module
```yaml
---
-   name: Print hello message
    hosts: all
    tasks:
    -   debug: 
            msg: Hello from Ansible!
```
All facts gathered by Ansible are stored in variable called `ansible_facts`. Use the `var` in `debug` module to print the `ansible_facts`
```yaml
---
-   name: Print hello message
    hosts: all
    tasks:
    -   debug: 
            var: ansible_facts
``` 
*You can disable gather facts with `gather_facts: no` if you don't want to use `ansible_facts`*
```yaml
---
-   name: Print hello message
    hosts: all
    gather_facts: no
    tasks:
    -   debug: 
            var: ansible_facts  # {}
``` 
Disalbe implicit gathering, you can specify in `/etc/ansible/ansible.cfg`
```ini
# /etc/ansible/ansible.cfg
gathering   = implicit
```

# Ansible Playbooks
A playbook is a single `YAML` file
- **play**: defines a set of activities
- **task**: an action to be performed on the host
    - execute a command
    - run a script
    - install a package
    - shutdown/restart
```yaml
-
    name: Play 1
    hosts: localhost
    tasks:
        -   name: Execute command 'date'
            command: date
        -   name: Execute script on server
            command: test_script.sh
        -   name: Install httpd service
            yum:
                name: httpd
                state: present
        -   name: Start Web server
            service:
                name: httpd
                state: started
-
    name: Play 2
    hosts: localhost
    tasks:
        -   name: Install Web service
            yum:
                name: httpd
                state: present
        -   name: Start web server
            service:
                name: httpd
                state: started
```
### Module
The different actions run by tasks are called *modules*.

### Run
```bash
ansible-playbook playbook.yml
ansible-playbook --help
```

# Verifying Playbooks
Ansible provides serveral modes for verifying playbooks which are *check* and *diff* modes. The *check* mode 
- **check** - Ansible execute playbook without making any actual changing on the hosts. It  all you to see what changes the playbook will make without applying them. Use the `--check` option as `ansible-playbook playbook.yaml --check`
- **diff** - shows the current state and the state after the playbook run. `ansible-playbook playbook.yaml --diff`

### syntax check mode
check the syntax for any error and ensures playbook syntax is error-free. Use the `--syntax-check` option
```yaml
# configure_nginx.yml
---
-
    hosts: webservers
    tasks:
        -   name: Ensure the configuration line is present
            lineinfile:
                path: /etc/nginx.conf
                line: 'client_max_body_size 100M;'
            become: yes
```

# Ansible Lint
- It is a command-line tool that performs linting on Ansible playbooks, roles and collections.
- It checks your code for potential errors, bugs, stylistic errors, supicious constructs.
- It is an seasoned Ansible guiding you, providing valuable insights and catching issues that might have slipped past your notice.
```yaml
# style_example.yaml

```

# Ansible Variables
```yaml
---
- name: Install NGINX
  hosts: debian_hosts
  tasks: 
  - name: Install NGINX on Debian
    apt: 
      name: nginx
      state: present
- name: Install NGINX (on Redhat)
  hosts: redhat_hosts
  tasks:
  - name: Install NGINX on Redhat
    yum: 
      name: nginx
      state: present
# can i combine this to one?
```
### Conditional when, or, and
```yaml
# combining aboves into one
- name: Install NGINX
  hosts: all
  tasks: 
  - name: Install NGINX on Debian
    apt: 
      name: nginx
      state: present
    when: ansible_os_family == "Debian" and ansible_distribution_version == "16.04"
  - name: Install NGINX on Redhat
    yum: 
      name: nginx
      state: present
    when: ansible_os_family == "RedHat" or ansible_os_family == "SUSE"
```

### Conditional Loops
```yaml
- name: Install NGINX
  hosts: all
  vars:
    packages:
        - name: nginx
          required: True
        - name: mysql
          required: True
        - name: apache
          required: False
  tasks: 
  - name: Install "{{ item.name }}" on Debian
    apt: 
      name: "{{ item.name }}"
      state: present
    loop: "{{ packages }}"
    when: item.required == True
  - name: Install NGINX on Redhat
    yum: 
      name: nginx
      state: present
    when: ansible_os_family == "RedHat" or ansible_os_family == "SUSE"
```

### Conditional and Register
```yaml
- name: Check status of a service and email if its down
  hosts: localhost
  tasks:
    - command: service httpd status
      register: result
    - mail:
        to: admin@company.com
        subject: Service Alert
        body: Httpd Service is down
        when: result.stdout.find('down') != -1
```

# Ansible Conditionals based on facts, variables, re-use
```yaml
# Scenario 1
- name: Install nginx on Ubuntu 18.04
  apt: 
    name: nginx=1.18.0
    state: present
  when: ansible_facts['os_family'] == 'Debian' and ansible_facts['distribution_major_version'] == '18'
- name: Deploy configuration files
  template: 
    src: "{{ app_env }}_config.j2"
    dest: "/etc/myapp/config.conf"
  vars:
    app_env: production
- name: Install required packages
  apt:
  name: 
    - package1
    - package2
  state: present

- name: Create necessary directories and set permissions
- name: Start web application service
  service: 
    name: myapp
    state: started
  when: environment == 'production'
```

# Ansible Loops
```yaml
- name: Create users
  hosts: localhost
  tasks:
    - user: name='{{ item }}' state=present
      loop:
        - joe
        - george
        - ravi
        - mani
        - kiran
    - user: name='{{ item.name }}' state=present uid='{{ item.uid }}'
      loop:
        - name: joe
          uid: 1010
        - name: george
          uid: 1011
        - name: ravi
          uid: 1012
        - name: mani
          uid: 1013
        - name: kiran
          uid: 1014
```
### With_*
> After the *with_* string is a *lookup plugin*.
```yaml
-
  name: Create users
  hosts: all
  tasks:
    - user: name='{{ item }}' state=present
      with_items:
        - joe
        - george
        - ravi
        - mani
-
  name: View Config Files
  hosts: localhost
  tasks:
    - debug: var=item
      with_file:
        - "/etc/hosts"
        - "/etc/resolv.conf"
        - "/etc/ntp.conf"
-
  name: Get from multiple URLs
  hosts: localhost
  tasks:
    - debug: var=item
      with_url: 
        - "https://site1.com/get-servers"
        - "https://site2.com/get-servers"
        - "https://site3.com/get-servers"
-
  name: Check multiple mongodbs
  hosts: localhost
  tasks:
    - debug: msg="DB={{ item.database }} PID={{ item.pid }}"
      with_mongodb:
        - database: dev
          connection_string: "mongodb://dev.mongo/"
        - database: prod
          connection_string: "mongodb://prod.mongo/"
```

# Modules
- system
- commands
- files
- databases
- cloud
- windows ...

### System modules
> System modules are actions to be performed at a system level such as modifying the users, group, iptables, firewall, logical volume groups, mouting, and working with services and etc.

### Command modules
> Command modules are used to execute command or script on a host. 
- command - simple command
- expect - interactive command
- raw 
- script
- shell

```yaml
-   name: Play 1
    hosts: localhost
    tasks:
        -   name: Execute command 'date'
            command: date   # free-form
        -   name: Display resolv.conf contents
            command: cat /etc/resolv.conf   # free-form
        -   name: Display resolv.conf contents
            command: cat resolv.conf chdir=/etc # change to path before running the command (not free-form)
        -   name: Create folder if doesn't exist
            command: mkdir /folder creates=/folder # (not free-form)
# free-form indicates this module takes a free form command
#   - not able to input parameters
```

#### Script Module
> Runs a local script on a remote node after transfering it
```yaml
-
    name: Play 1
    localhost: localhost
    tasks:
        -   name: Run a script on remote server
            script: /some/local/script.sh -arg1 -arg2
```

#### Service module 
> Manage services - Start, Stop, Restart
```yaml
-
    name: Start Services in order
    hosts: localhost
    tasks:
        -   name: Start the database server
            service: name=postgresql state=started
# same as below
        -   name: Start the database server
            service:
                name: postgresql
                state: stared
        -   name: Start the httpdq server
            service:
                name: httpd
                state: started
        -   name: Start the nginx server
            service: name=nginx state=started
```
[Note]
*To ensure the service __started__, we set state as __started__*

#### lineinfile
> Search for a line in a file and replace it or add it if it doesn't exist.
```ini
# /etc/resolv.conf
nameserver  10.1.250.1
nameserver  10.1.250.2
```
```yaml
-
    name: Add DNS server to resolv.conf
    hosts: localhost
    tasks:
        - lineinfile:
            path: /etc/resolv.conf
            line: 'nameserver 10.1.250.10'
```

### File modules
> File module help works with files.
- acl - set and retrieve ACL information on file
- archive - compressing
- copy
- file
- find
- lineinfile
- replace
- stat
- template
- unarchive

### Database modules
> Database modules help work with databases.
- mongodb
- mysql
- mssql
- postgresql
- proxysql
- vertica

### Cloud
> Cloud modules help work with clouds 
- amazon
- azure
- docker
- google
- linode
- cloudstack
- digital ocean
- vmware
### Windows
> Window modules help work with window environment
- win_copy
- win_command
- win_domain
- win_file
- win_iis_website
- win_msg
- win_msi
- win_package
- win_ping
- win_path
- win_robocopy
- win_regedit
- win_shell
- win_service
- win_user
- ...

# Intro to Ansible Plugins
In Ansible, a plugin is a piece of code that extends or modifies the functionality of Ansible. Plugins can be used in various aspects of Ansible such as *inventory, modules, callbacks, filters and more*. An Ansible plugin can be in the form of *inventory plugin, module plugin, action plugin and callback plugin*
- **Dynamic  inventory plugin** - enables you to have an up-to-date view of your infrastructure ensuring *accurate and reliable automation*.
- **Module plugin** - when you develop a module plugin, you gain the power to provision cloud resources with custom configurations.
- **Action Plugin** - This plugin simplifies *load balancer* management.

### Other plugins
- **Lookup plugins** - fetch data from external sources like databases or APIs and let you use that data within your playbooks
- **Filter plugins** - offer additional data manipulation and transformation capabilities within your playbooks, allowing you to modify variables or format output.
- **Connection plugins** - enable Ansible to connect and communicate with various target systems such as `ssh`, `winrm`, `docker`
- **Dynamic Inventory Plugins** - allow Ansible to retrieve inventory information from various sources lie cloud providers or configuration management databases
- **Callback plugins** - allow you to capture events and perform custom actions during playbook execution. 

# Modules & Plugins Index

# Introduction to Handlers
> To effect the changes in the configuration file immediately, we must the restart the servers. Manual changes and restart the servers time consume and error prones.
> With handler, we can restart the server service and associte with task that modifies the configuration file.
> This creates the dependency between the task and handler

### Ansible Handlers
- Tasks triggered by events/notifications
- Defined in playbook, executed when notified by a task
- Manage actions based on system/configuration changes
```yaml
- name: Deploy application
  hosts: application_servers
  tasks:
    - name: Copy Application Code
      copy:
        src: app_code/
        dest: /opt/application
      notify: Restart Application Service
  handlers: 
    - name: Restart Application Service
      service:
        name: application_service
        state: started
```

# Ansible Roles
- reusable Ansible configs are packaged into a role to use later
- To use roles, move *roles* into own *playbook's roles* directory or move into default directory `/etc/ansible/roles` or change the `roles_path` in `/etc/ansible/ansible.cfg`

### Find Roles
You can find roles in Ansible [Ansible Galaxy](https://galaxy.ansible.com/) or with command `ansible-galaxy search mysql` and then use with command `ansible-galaxy install geerlingguy.mysql`
```yaml
-
  name: Install and Configure mysql
  hosts: db-server
  roles:
    - geerlingguy.mysql
#or
    - role: geerlingguy.mysql
      # give escalating privileges
      become: yes   # additonal options to role
      vars:
        mysql_user_name: db-user
```
- use `ansible-galaxy list` to view the location where roles would be installed
- use `ansible-config dump | grep ROLE` to check the configuration
- use `ansible-galaxy install geerlingguy.mysql -p ./roles` to install into roles dir of current dir

# Ansible Collections
```bash
ansible-galaxy collection install network.cisco # install cisco collections
```
### What are Ansible collections?
> Package and distribute modules, roles, plugins, etc.
> self-contained unit
> community and vendor-created

### Benefits
```yaml
# expanded functionality

# first install 
$ ansible-galaxy collection install amazon.aws
---
- hosts: localhost
  collections: 
    - amazon.aws

  tasks: 
    - name: Create an S3 bucket
      aws_s3_bucket:    # extended in amazon.aws collection
        name: my-bucket
        region: us-west-1
```
```yaml
# modularity and re-usability
- hosts: localhost
  collections: 
    - my_namespace.my_collection

  roles: 
    - my_custom_role

  tasks:
    - name: Use custom module
      my_custom_module:
        param: value
```

```yaml
# simplified distribution and managment
# requirements.yaml
---
collections:
    - name: amazon.aws
      version: "1.5.0"
    - name: community.mysql
      src: https://github.com/ansible-collections/community.mysql
      version: "1.2.1"

$ ansible-galaxy collection install -r requirements.yaml
```

# Introduction to Templating

### String mainpulation - Filters
```jinja
# my_name = Bond
The name is {{ my_name }} # The name is Bond
The name is {{ my_name | uppercase}}  # The name is BOND
The name is {{ my_name | lower }}   # The name is bond
The name is {{ my_name | title }}   # The name is Bond
The name is {{ my_name | replace("Bond", "Bounce")}}  # The name is Bounce
The name is {{ first_name | default("James")}} {{ my_name }} # The name is Jame Bond
```

### Filters - List and Set
```jinja
{{ [1,2,3] | min }} # 1
{{ [1,2,3] | max }} # 3
{{ [1,2,3, 2] | unique }} # 1,2,3
{{ [1,2,3,4] | union([4,5]) }}  # 1,2,3,4,5
{{ [1,2,3,4] | intersect([4,5]) }} # 4
{{ 100 | random }}  # random number
{{ ["The", "name", "is", "Bond"] | join(" ") }} # The name is Bond
```

### Loops
```jinja
{% for number in [0,1,2,3,4] %}
{{ number }}
{% endfor %}  # 0 1 2 3 4
```

### Conditions
```jinja
{% for number in [0,1,2,3,4] %}
  {% if number == 2 %}
    {{ number }}
  {% endif %}
{% endfor %}        # 2
```

# Jinja2 Templates for Dynamic Configs 

### Filters - file
```jinja
{{ "etc/hosts" | basename }}  # hosts
{{ "c:\windows\hosts" | win_basename }} # hosts
{{ "c:\windows\hosts" | win_splitdrive }} # ["c:", "\windows\hosts" ]
{{ "c:\windows\hosts" | win_splitdrive | first }} # "c:"
{{ "c:\windows\hosts" | win_splitdrive | last }} # "\windows\hosts"
```

### Jinja2 in Playbooks
```ini
web1  ansible_host=172.20.1.100 dns_server=10.5.5.4
web2  ansible_host=172.20.1.101 dns_server=10.5.5.4
web3  ansible_host=172.20.1.102 dns_server=10.5.5.4
```
```yaml
---
- name: Update dns server
  hosts: all
  tasks:
    - nsupdate:
        server: '{{ dns_server }}'
```
```yaml
# Output
---
- name: Update dns server
  hosts: all
  tasks:
    - nsupdate:
        server: 10.5.5.4
```

```ini
# /etc/ansible/hosts
[web_servers]
web1  ansible_host=172.20.1.100
web2  ansible_host=172.20.1.101
web3  ansible_host=172.20.1.102
```
```jinja
<!-- index.html.j2 -->
<!doctype>
<html>
  <body>
    This is {{ inventory_hostname }} server
  </body>
</html>
```
```yaml
# playbook.yaml
-
  hosts: web_servers
  tasks: 
    - name: Copy index.html to remote servers
      template: 
        src: index.html.j2
        dest: /var/www/nginx-default/index.html
```
```html
<!-- index.html -->
<!doctype>
<html>
  <body>
    <!-- for web1 -->
    This is web1 server
  </body>
</html>
```

```nginx
# nginx.conf.j2
server {
  location / {
    fastcgi_pass  {{host}}:{{port}};
    fastcgi_param QUERY_STRING  $query_string;
  }

  location ~ \ gif|jpg|png $ {
    root {{ image_path }};
  }
}
```
```nginx
# nginx.conf
server {
  location / {
    fastcgi_pass  localhost:9000;
    fastcgi_param QUERY_STRING  $query_string;
  }

  location ~ \ gif|jpg|png $ {
    root /data/images;
  }
}
```

### Redis conf
```ini
#redis.conf.j2 
bind {{ ip_address }}
protected-mode yes
port {{ redis_port | default('6379') }}
tcp-backlog 511

# Unix Socket.
timeout 0

# Tcp keepalive
tcp-keepalive {{ tcp_keepalive | default('300') }}

daemonize no
supervised no
```
```ini
# redis.conf
bind 192.168.1.100
protected-mode yes
port 6379
tcp-backlog 511

# Unix Socket.
timeout 0

# Tcp keepalive
tcp-keepalive 300
daemonize no
supervised no
```

### resolv.conf
```yaml
---
vars:
  name_server:
    - 10.1.1.2
    - 10.1.1.3
    - 8.8.8.8
```
```jinja
{% for name_server in name_servers %}
nameserver {{ name_server }}
{% endfor %}
```
```ini
# /etc/resolv.conf
nameserver 10.1.1.2
nameserver 10.1.1.3
nameserver 8.8.8.8
```

# Revisiting
### Creating file
```yaml
---
- hosts: node01
  become: true
  tasks:
    - name: Creating blog.txt file
      file:
        path: /opt/news/blog.txt
        state: touch
        group: sam

- hosts: node02
  become: true
  tasks:
    - name: Creating story.txt file
      file:
        path: /opt/news/story.txt
        state: touch
        owner: sam
```

```yaml
---
- hosts: node01
  become: true
  tasks:
    - name: create a file
      copy:
        dest: /opt/file.txt
        content: "This file is created by Ansible!"
```
```yaml
---
- hosts: all
  become: true
  tasks:
    - copy:
        src:  /usr/src/blog/index.html
        dest: /opt/blog
        remote_src: yes
```
```yaml
---
- hosts: node01
  become: true
  tasks:
    - replace:
        path: /opt/music/blog.txt
        regexp: 'Kodekloud'
        replace: 'Ansible'

- hosts: node02
  become: true
  tasks:
    - replace:
        path: /opt/music/story.txt
        regexp: 'Ansible'
        replace: 'Kodekloud'
```

### Setting up roles
```bash
cd roles
ansible-galaxy init package
cd package/tasks/main.yml
```
```yaml
# package/tasks/main.yml
- name: Install Nginx
  ansible.builtin.package:
    name: nginx
    state: latest
- name: Start Nginx Service
  ansible.builtin.service:
    name: nginx
    state: started
```
```yaml
# playbook.yml
- hosts: all
  become: yes
  roles:
    - roles/package
```