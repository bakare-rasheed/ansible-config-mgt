##  ANSIBLE DYNAMIC ASSIGNMENTS (INCLUDE) AND COMMUNITY ROLES

Ansible has two modes of operation for reusable content: dynamic and static.

In Ansible 2.0, the concept of dynamic includes was introduced. Due to some limitations with making all includes dynamic in this way, the ability to force includes to be static was introduced in Ansible 2.1. 
Because the include task became overloaded to encompass both static and dynamic syntaxes, and because the default behavior of an include could change based on other options set on the Task, Ansible 2.4 introduces the concept of `include vs. import.`

For the business requirements, we need to create, remove and modify the hosts in our infrastructure environment very frequently and to manage those nodes via Ansible, one need to update the inventory file very frequently. So, static inventory file is not suitable.
In such case, we need an inventory file which updates itself.

This type of inventory file is called Dynamic inventory file, which have dynamic contents. 
In our environment, the sources of host nodes can be AWS, OpenStack, Cobbler, LDAP systems. These systems have their list of nodes and Ansible supports to integrates such external dynamic inventory.

The primary advantage of using include_* statements is looping. When a loop is used with an include, the included tasks or role will be executed once for each item in the loop.

In most cases however, it is recommended to use static assignments for playbooks, because it is more reliable. With dynamic ones, it is hard to debug playbook problems due to its dynamic nature.


## STEP 1: Introducing Dynamic Assignment Into Our structure

* In your https://github.com/<your-name>/ansible-config-mgt GitHub repository, start a new branch and call it "dynamic-assignments".

* Therein, create a new folder, name it "dynamic-assignments" and create a file in the new folder "env-vars.yml"

Since we will be using the same Ansible to configure multiple environments, and each of these environments will have certain unique attributes, such as servername, ip-address etc., we will need a way to set values to variables per specific environment. Thus, we shall create a folder to keep each environment’s variables file.

* create a new folder "env-vars", and for each environment (dev,stage,uat and prod), create new YAML files which we will use to set variables.

The tree structure thus looks like the below:

![tree](https://user-images.githubusercontent.com/114327344/206931869-456372c8-c201-46a1-a2a7-b61d2da6d6e3.PNG)

* Update the env-vars.yml file (dynamic-assignments/env-vars.yml) with the code below:

  ```
---
- name: collate variables from env specific file, if it exists
  hosts: all
  tasks:
    - name: looping through list of available files
      include_vars: "{{ item }}"
      with_first_found:
        - files:
            - dev.yml
            - stage.yml
            - prod.yml
            - uat.yml
          paths:
            - "{{ playbook_dir }}/../env-vars"
      tags:
        - always

```
  
  ## STEP 2: UPDATE 'SITE.YML' WITH DYNAMIC ASSIGNMENTS

* Update site.yml file to make use of the dynamic assignment. Our site.yml file should now look like below:

```
---
- hosts: all
- name: Include dynamic variables 
  tasks:
  import_playbook: ../static-assignments/common.yml 
  include: ../dynamic-assignments/env-vars.yml
  tags:
    - always

- hosts: webservers
- name: Webserver assignment
  import_playbook: ../static-assignments/uat-webservers.yml

```


In order to sync our github with the jenkins server directly, we shall configure our VS code to work directly with github;

* On Jenkins-Ansible server make sure that git is installed with git --version, then go to ‘ansible-config-mgt’ directory and run

```
git init
git pull https://github.com/<your-name>/ansible-config-mgt.git
git remote add origin https://github.com/<your-name>/ansible-config-mgt.git
git branch roles-feature
git switch roles-feature
  
    
![p13-gitremote](https://user-images.githubusercontent.com/114327344/206932157-87f8210f-d66c-4cf7-a018-51b5e02357ec.PNG)
  
![p13-gitconfig](https://user-images.githubusercontent.com/114327344/206932166-2065ed45-5ae4-4881-bc87-b1b1f5df85a2.PNG)

  ## STEP 3: Community Roles

Next is to create a role for MySQL database – it should install the MySQL package, create a database and configure users. Instead of re-inventing the wheel however, we shall use roles already developed by open source engineers. With Ansible Galaxy, we can simply download a ready to use ansible role, and keep going.

We will be using a MySQL role developed by 'geerlingguy'.

* Inside roles directory , create your new MySQL role with `ansible-galaxy install geerlingguy.mysql` and rename the folder to mysql;

` mv geerlingguy.mysql/ mysql `
  
  ![p13-mysql](https://user-images.githubusercontent.com/114327344/206932351-24010251-6a68-4bfa-bbd9-0b287855d102.PNG)
  
![p13-mysqlcnf](https://user-images.githubusercontent.com/114327344/206932360-2270479e-9724-4230-a086-25802e90a894.PNG)

  * upload the changes into your GitHub:

```
git add .
git commit -m "Commit new role files into GitHub"
git push --set-upstream origin roles-feature 

```
  
  ## STEP 4: LOAD BALANCER ROLES.

In order to be able to chose which loadbalancer to use (Apache or nginx), we need to have two roles respectively; Apache and Nginx.

* Using the earlier introduced community role; geerlingguy, cd to the role directory, run `ansible-galaxy install geerlingguy.apache` and `ansible-galaxy install geerlingguy.nginx`.

* rename the role to apache and nginx respectively; 
```
mv geerlingguy.apache/ apache
mv geerlingguy.nginx/ nginx

```
  
![p13-nginpache](https://user-images.githubusercontent.com/114327344/206932488-4a32e27e-1346-4b9d-ae12-d5a7bac6af6e.PNG)
  
![p13-nginxConf2](https://user-images.githubusercontent.com/114327344/206932504-2f441412-2947-44e6-bcd4-4c8f1d72c69c.PNG)
  
![p13-nmaes](https://user-images.githubusercontent.com/114327344/206932519-9ae8d101-f236-4e48-9558-4c2e704c726e.PNG)

  The snippet below is placed in the nginx roles' defaults/main.yml file.

```
enable_nginx_lb: false
load_balancer_is_required: false

```

while the below is placed in Apache's defaults/main.yml file.

```
enable_apache_lb: false
load_balancer_is_required: false

```

- We will then use 'env-vars\uat.yml' file to define which loadbalancer to use in UAT environment by setting respective environmental variable to true.

* Activate load balancer, and enable nginx by setting these in the respective environment’s env-vars file.

```
enable_nginx_lb: true
load_balancer_is_required: true

```
The same must work with apache LB, so you can switch it by setting respective environmental variable to true and other to false.

To test this, you can update inventory for each environment and run Ansible against each environment.


## LB CONFIGURATION SETTINGS FOR APACHE ROLE.

From our knowledge of loadbalancing with apache in the previous project, the following points are crucial when configuring apache as a loadbalancer;
We shall highlight the points to guide us to where necessary modifications will be made in the downloaded roles. 

### INSTALLING APACHE AND STARTING THE SERVICE

This will be performed by default as part of the pre-configured settings in the apache role that was downloaded.


  
### ENABLING THE FOLLOWING MODULES

```
sudo a2enmod rewrite
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod headers
sudo a2enmod lbmethod_bytraffic

```
  
  ## LB CONFIGURATION SETTINGS FOR NGINX ROLE.

From our knowledge of loadbalancing with Nginx in the previous project, the following points are crucial when configuring nginx as a loadbalancer;

### Updating the '/etc/hosts' file to include the web servers’ names (e.g. Web1 and Web2) and their private IP addresses.

This was done using Ansible blockinfile module. The module is used to insert, update or remove a block of lines (multi-line text) from files on the remote nodes. These blocks are surrounded by the marker like begin and end which can be a custom marker. The module helps to work with the different types of files.

The code below was thus included in tasks/main.yml file within the vhost configuration section.

```
- name: set webservers host name in /etc/hosts
  become: yes
  blockinfile: 
    path: /etc/hosts
    block: |
      {{ item.ip }} {{ item.name }}
  loop:
    - { name: web1, ip: 172.31.27.70 }
    - { name: web2, ip: 172.31.20.240 }

``` 
  
  
![lb-db-vs](https://user-images.githubusercontent.com/114327344/206933142-986b5c27-7270-4a4d-8e8a-9efccc53527a.PNG)
  
![p13-playbook00](https://user-images.githubusercontent.com/114327344/206933071-a50de089-5e4b-4e13-afbf-97fe19d4c5da.PNG)
  
![p13-playbook](https://user-images.githubusercontent.com/114327344/206933077-acf43845-1b51-4cee-9092-f1b0626b6c6e.PNG)
  
![p13-playbook2](https://user-images.githubusercontent.com/114327344/206933084-41b3975c-cfd9-4623-bd68-ad85a20bc133.PNG)
  
![apache-page](https://user-images.githubusercontent.com/114327344/206933124-20ca6f04-7958-4a23-b3c6-4104ffba51e0.PNG)
  
![p13-apache](https://user-images.githubusercontent.com/114327344/206933181-b7c1fe49-c405-43b4-ba06-8c970604887f.PNG)
