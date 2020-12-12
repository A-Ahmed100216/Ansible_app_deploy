# Task - Create a playbook for the Sparta Sample Node App

1. Create host instances for the app and database.
2. Configure the security groups of these hosts to accept traffic from the controller. Additionally, the database should be open to the app server.
3. Connect to the controller and instance and navigate to hosts. Add the app and db as hosts in the inventory.
```
cd /etc/ansible
sudo nano hosts

[host_app]
app_private_ip ansible_connection=ssh ansible_ssh_private_key_file=/home/ubuntu/.ssh/key_name.pem

[host_db]
db_private_ip ansible_connection=ssh ansible_ssh_private_key_file=/home/ubuntu/.ssh/key_name.pem
```
4. Create a directory for playbooks. Within here, create an app.yaml and db.yaml file.

### Installations and Tasks for App
* [**App Playbook**](https://github.com/A-Ahmed100216/Ansible_app_deploy/blob/main/app.yaml)
* Install git
```YAML
- name: install git
  # apt - used to install files, specify the name i.e. pkg
  apt: pkg=git state=present
```
* Install nodejs
```YAML

- name: add apt key for nodesource
  # Apt-key is used to acquire the key to install certain packages
  apt_key: url=https://deb.nodesource.com/gpgkey/nodesource.gpg.key
  # Install nodejs
- name: install nodejs
  apt: name=nodejs
```
* Install pm2
```YAML
# To install pm2, we use npm i.e. node package manager.
- name: Install pm2
  npm:
    name: pm2
    global: yes
```
* Install npm
```YAML
- name: install npm
  apt: name=npm state=present
```
* Install nginx
```YAML
- name: install nginx
  apt: name=nginx state=present
  # Notify used in conjuction with handlers
  notify:
    - Start nginx
```
* Copy app files into controller and then into the hosts
```YAML
- name: Copy app files
  copy:
    src: /home/ubuntu/app
    dest: /home/ubuntu/app
    # Force ensures that if the file has no changes, it is not rewritten
    force: no
```
* Configure nginx reverse proxy:
  * Remove the default file
  ```YAML
  - name: remove nginx default file
    file:
      path: /etc/nginx/sites-enabled/default
      state: absent
  ```
  * Create a new configuration file in /etc/nginx/sites-available
  ```YAML
  - name: create config file
    file:
      path: /etc/nginx/sites-available/custom_server.conf
      state: touch
      # Mode used to define permissions
      mode: 666
  ```
  * Write the reverse proxy commands to the new file
  ```YAML
  - name: Creating a file with content
  # Copy can be used to write contents to a file
    copy:
      dest: "/etc/nginx/sites-available/custom_server.conf"
      content: |
        server {
        listen 80;
        location / {
        proxy_pass http://127.0.0.1:3000;
        }
        }
  ```
  * Create a symbolic link to copy data to the sites-enabled directory
  * Restart nginx
  ```YAML
  - name: Create a symbolic link between sites enabled and sites available
    file:
      src: /etc/nginx/sites-available/custom_server.conf
      dest: /etc/nginx/sites-enabled/custom_server.conf
      state: link
    # Use notify to trigger a restart nginx handle   
    notify:
    -  Restart nginx
    ```

* Run `npm install`
```YAML
- name: run npm install
  become: yes
  shell:
    chdir: /home/ubuntu/app/app
    cmd: npm install
```

* Configure DB_HOST. This is achived using shell commands.  
```YAML
- name: set db host as global variable
    become: true
    shell: |
      export DB_HOST={{ DB_HOST }}
      sed -i '/export DB_HOST=/d' .bashrc
      printf '\nexport DB_HOST={{ DB_HOST }}' >> .bashrc
    args:
      chdir: /home/ubuntu
```


* Start the app
```YAML
- name: Start APP
  become_user: ubuntu
  command: pm2 start app.js --name app chdir=/home/ubuntu/app/app
  ignore_errors: yes
```

## Installations and Tasks for DB
* [**DB Playbook**](https://github.com/A-Ahmed100216/Ansible_app_deploy/blob/main/db.yaml)
* Run updates of source lists
```YAML
- name: apt update and upgrade
  apt:
    upgrade: "yes"
    update_cache: "yes"
    cache_valid_time: 86400
```

* Install Mongodb. This requires the public key and repository.
* Once installed, `notify` is used to trigger the `start mongod` handle.
```YAML
- name: MongoDB - Import public key used by package management system
  apt_key:
    url: https://www.mongodb.org/static/pgp/server-3.2.asc
    state: present

- name: Add repository
  become: True
  apt_repository:
    filename: '/etc/apt/sources.list.d/mongodb-org-3.2.list'
    repo: 'deb http://repo.mongodb.org/apt/ubuntu xenial/mongodb-org/3.2 multiverse'
    state: present
    update_cache: "yes"

- name: Install MongoDB
  apt:
    name: mongodb-org
    state: present
    update_cache: "yes"
  notify:
  - start mongod
```
* Configure mongod and set bindip as 0.0.0.0. This is achievd via runnng a shell commands within the playbook. A mongod restart is subsequently triggered.  
```YAML
- name: set db host as global variable
become: true
shell: |
 sudo sed -i 's/127.0.0.1/0.0.0.0/g' /etc/mongod.conf
args:
 chdir: /etc
notify:
- restart mongod
```
5. Run the plabooks using the commands below:
```
ansible-playbook app.yaml
ansible-playbook db.yaml
```
