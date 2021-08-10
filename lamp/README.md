Building a simple LAMP stack and deploying Application using Ansible Playbooks.
-------------------------------------------

These playbooks require Ansible 1.2.

These playbooks are meant to be a reference and starter's guide to building
Ansible Playbooks. These playbooks were tested on CentOS 7.x so we recommend
that you use CentOS or RHEL to test these modules.

RHEL7 version reflects changes in Red Hat Enterprise Linux and CentOS 7:
1. Network device naming scheme has changed
2. iptables is replaced with firewalld
3. MySQL is replaced with MariaDB

This LAMP stack can be on a single node or multiple nodes. The inventory file
'hosts' defines the nodes in which the stacks should be configured.

        [webservers]
        localhost

        [dbservers]
        bensible

Here the webserver would be configured on the local host and the dbserver on a
server called `bensible`. The stack can be deployed using the following
command:

        ansible-playbook -i hosts site.yml

Once done, you can check the results by browsing to http://localhost/index.php.
You should see a simple test page and a list of databases retrieved from the
database server.

# Obligatorio 2021 - Taller de Linux

Primeramente se modifica el archivo de configuracion para que el playbook busque el inventario dentro de la ruta ./Inventory dentro del repositorio.
[centos]
centos1 ansible_host=192.168.56.110

[ubuntu]
ubuntu1 ansible_host=192.168.56.112

[linux:children]
ubuntu
centos

[webservers:children]
centos
ubuntu

[dbservers:children]
centos
ubuntu

Luego se configura el archivo site.yml para que el playbook se ejecute con el usuario creado especialmente para ansible llamado "ansible".
- name: apply common configuration to all nodes                 
  hosts: all                                                    
  remote_user: ansible                                          
  become: yes                                                   
                                                                
  roles:                                                        
  - common                                                      
                                                                
- name: configure and deploy the webservers and application code
  hosts: webservers                                             
  remote_user: ansible                                          
  become: yes                                                   
                                                                
  roles:                                                        
    - web                                                       
                                                                
- name: deploy MySQL and configure the databases                
  hosts: dbservers                                              
  remote_user: ansible                                          
  become: yes                                                   
                                                                
  roles:                                                        
    - db                                                        

Luego bajo la siguiente ruta "/lamp/roles/common/tasks" se modifica el archivo main.yml, para que este ejecute correctamente los modulos y parametros en ambos sistemas operativos (Centos y Ubuntu)

- name: Install chrony
  yum:
    name: chrony
    state: present
  tags: ntp
  when: ansible_os_family == "RedHat"

- name: Install ntp
  apt:
    package: ntp,ntpdate
    state: present
    update_cache: yes
  tags: ntp
  when: ansible_os_family == "Debian"

- name: Install common dependencies
  yum:
    name: "{{ item }}"
    state: present
  loop:
  - python3-libselinux.x86_64
  - policycoreutils-python-utils
  - firewalld
  when: ansible_os_family == "RedHat"

- name: Install common dependencies
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - python3
    - policycoreutils
    - ufw
  when: ansible_os_family == "Debian"

- name: Configure ntp file
  template:
    src: chrony.conf.j2
    dest: /etc/chrony.conf
  tags: ntp
  notify:
    - restart chrony
  when: ansible_os_family == "RedHat"

- name: Configure ntp file
  template:
    src: ntp.conf.j2
    dest: /etc/ntp.conf
  tags: ntp
  notify:
    - restart ntp ubuntu
  when: ansible_os_family == "Debian"
  
  - name: Start the ntp service
  service:
    name: chronyd
    state: started
    enabled: yes
  tags: ntp
  when: ansible_os_family == "RedHat"

- name: Start the ntp service
  service:
    name: ntp
    state: started
    enabled: yes
  tags: ntp
  when: ansible_os_family == "Debian"
  
  Se procede a configurar la seccion de DB dentro de la ruta "/lamp/roles/db/tasks" el archivo main.yml con la sentencia "when" indicando segun que version de sistema operativo contenga el equipo remoto (Centos o Ubuntu) a que archivo debera tomar para enviarle las configuraciones correspondientes (centos.yml o ubuntu.yml)
  
main.yml

- include: centos.yml
  when: ansible_os_family == "RedHat"

- include: ubuntu.yml
  when: ansible_os_family == "Debian"
  
  centos.yml
  
  - name: Install MariaDB package
  yum:
    name: "{{ item }}"
    state: installed
  loop:
  - mariadb-server
  - python3-PyMySQL

- name: Configure SELinux to start mysql on any port
  seboolean:
    name: mysql_connect_any
    state: true
    persistent: yes

- name: Create Mysql configuration file
  template:
    src: my.cnf.j2
    dest: /etc/my.cnf
  notify:
  - restart mariadb

- name: Create MariaDB log file
  file:
    path: /var/log/mysqld.log
    state: touch
    owner: mysql
    group: mysql
    mode: 0775

- name: Create MariaDB PID directory
  file:
    path: /var/run/mysqld
    state: directory
    owner: mysql
    group: mysql
    mode: 0775

- name: Start MariaDB Service
  service:
    name: mariadb
    state: started
    enabled: yes

- name: Start firewalld
  service:
    name: firewalld
    state: started
    enabled: yes
    
 - name: insert firewalld rule      
  firewalld:                       
    port: "{{ mysql_port }}/tcp"   
    permanent: true                
    state: enabled                 
    immediate: yes                 
                                   
- name: Create Application Database
  mysql_db:                        
    name: "{{ dbname }}"           
    state: present                 
                                   
- name: Create Application DB User 
  mysql_user:                      
    name: "{{ dbuser }}"           
    password: "{{ upassword }}"    
    priv: '*.*:ALL'                
    host: '%'                      
    state: present                 
                                   
ubuntu.yml

- name: Install MariaDB package
  apt:
    name: "{{ item }}"
    state: present
  loop:
  - mariadb-server
  - python3-pymysql

- name: Create Mysql configuration file
  template:
    src: my.cnf.j2
    dest: /etc/my.cnf
  notify:
  - restart mariadb

- name: Create MariaDB log file
  file:
    path: /var/log/mysqld.log
    state: touch
    owner: mysql
    group: mysql
    mode: 0775

- name: Create MariaDB PID directory
  file:
    path: /var/run/mysqld
    state: directory
    owner: mysql
    group: mysql
    mode: 0775

- name: Start MariaDB Service
  service:
    name: mariadb
    state: started
    enabled: yes

- name: Start firewalld
  service:
    name: ufw
    state: started
    enabled: yes

- name: insert firewalld rule
  ufw:
  #    port: "{{ mysql_port }}"
    port: "3306"
    proto: tcp
    state: enabled

- name: Create Application Database
  mysql_db:
    name: "{{ dbname }}"
    state: present
    
- name: Create Application DB User
  mysql_user:
    name: "{{ dbuser }}"
    password: "{{ upassword }}"
    priv: '*.*:ALL'
    host: '%'
    state: present
    
Por ultimo se configura la seccion WEB dentro de la siguiente ruta "/lamp/roles/web/tasks" editando el archivo main.yml para que dependiendo de que sistema operativo contenga el equipo remoto que archivo de configuracion debe dirigirse (install_httpd.yml e install_httpdU.yml)

main.yml

- include: install_httpd.yml
  when: ansible_os_family == "RedHat"

- include: install_httpdU.yml
  when: ansible_os_family == "Debian"

- include: copy_code.yml

install_httpd.yml

- name: Install httpd and php                                         
  yum:                                                                
    name: "{{ item }}"                                                
    state: present                                                    
  loop:                                                               
  - httpd                                                             
  - php                                                               
  - php-mysqlnd                                                       
                                                                      
- name: Install web role specific dependencies                        
  yum:                                                                
    name: "{{ item }}"                                                
    state: installed                                                  
  loop:                                                               
  - git                                                               
                                                                      
- name: Start firewalld                                               
  service:                                                            
    name: firewalld                                                   
    state: started                                                    
    enabled: yes                                                      
                                                                      
- name: insert firewalld rule for httpd                               
  firewalld:                                                          
    port: "{{ httpd_port }}/tcp"                                      
    permanent: true                                                   
    state: enabled                                                    
    immediate: yes                                                    
                                                                      
- name: http service state                                            
  service:                                                            
    name: httpd                                                       
    state: started                                                    
    enabled: yes                                                      
                                                                      
- name: Configure SELinux to allow httpd to connect to remote database
  seboolean:                                                          
    name: httpd_can_network_connect_db                                
    state: true                                                       
    persistent: yes                                                   
   
install_httpdU.yml

- name: Install httpd and php                 
  apt:                                        
    name: "{{ item }}"                        
    state: present                            
  loop:                                       
  - apache2                                   
  - php                                       
  - php-mysqlnd                               
                                              
- name: Install web role specific dependencies
  apt:                                        
    name: "{{ item }}"                        
    state: present                            
  loop:                                       
  - git                                       
                                              
- name: Start UFW                             
  service:                                    
    name: ufw                                 
    state: started                            
    enabled: yes                              
                                              
- name: insert firewalld rule for httpd       
  ufw:                                        
    port: "{{ httpd_port }}/tcp"              
    state: enabled                            
                                              
- name: apache service state                  
  service:                                    
    name: apache2                             
    state: started                            
    enabled: yes                              
    
copy_code.yml

- name: Copy the code from repository
  git:
    repo: "{{ repository }}"
    dest: /var/www/html/
  environment:
    GIT_SSL_NO_VERIFY: True

- name: Creates the index.php file
  template:
    src: index.php.j2
    dest: /var/www/html/index.php
