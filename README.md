Sometimes you may find difficulties while configuring wordpress on your server, but when you need to do the same on multiple servers with multiple operating systems the process may take more time than expected.

by using ansible, you don’t need to worry about this anymore. By running a single playbook you can set up wordpress without any worries.

So let’s start.

In this example, we are setting up wordpress on an amazon Linux machine & ubuntu machine (LAMP & LEMP)

![lamp,lemp](https://user-images.githubusercontent.com/61390678/219973785-5992b6f2-e62d-4e3d-b51c-3fcfe355d485.png)



So we need 3 servers in our lab environment,
1x master server with ansible installed
2x client servers with ubuntu 20.04 & amazon linux installed.

You can install ansible using the following command, assuming your master machine is running on amazon Linux.
```
 $ sudo amazon-linux-extras install ansible2 -y
```

Once it is installed, you can check the ansible version by the following command,

```
$ansible - version
```

Before proceeding to the playbook we need to configure the inventory file. This file keeps the details of the client machine, which includes the IP address, ansible username, ssh port key file, etc.

The following file is an example of what an inventory fike looks like.

```
$ cat inventory.txt
[amazon]
172.31.44.89 ansible_user="ec2-user" ansible_port="22" ansible_ssh_private_key_file="key.pem"
[ubuntu]
172.31.34.215 ansible_user="ubuntu" ansible_port="22" ansible_ssh_private_key_file="key.pem"
```

let’s break it out.

[amazon] defines the name of the group, if you add any hosts to the group, they all will be part of the group, this is useful when you need to run the playbook on a specific group from the inventory file.
172.31.44.89 is the IP address of your client machine.
since these clients are on the same machine, you can configure them with the private IP address.
ansible_user defines the user name for ansible to connect to the server via ssh.
ansible_port defines the ssh port of the client machine.
ansible_ssh_private_key_file defines the key used to connect to the server.

You can add multiple servers to a group and you can configure multipl groups in the inventory file.

Our playbook uses some variables for its execution. Instead of adding the variables in the code, you can define them in a different directory called group_vars,

If you define the variables in this directory, those values will be assigned to the hosts inside the group. The advantage is that you can assign different values for the same variables for different groups,

To set up this you need to create a directory called group_vars in the master machine. inside the directory, you can create files for specifying the variables and values, in our playbook we are setting up wordpress on LAMp & LEMP stack, so the values to the variables are different. If you specify them in group_vars, they will fetch the values from this file.

Please note that group_vars follows YAML format, so you need to add — — — at the top of your file.

The files will be like the following.

```
$ cat group_vars/amazon
---
port: "8080"
owner: "apache"
group: "apache"
domain: "example.com"
mysql_root_password: "mysqlroot@123456"
mysql_db: "wordpress"
mysql_user: "wordpress"
mysql_password: "wordpress@123"
wp_url: "https://wordpress.org/wordpress-6.1.1.tar.gz"
```

```
$ cat group_vars/ubuntu
---
port: "8080"
owner: "www-data"
group: "www-data"
domain: "example.com"
mysql_root_password: "mysqlroot@123456"
mysql_db: "wordpress"
mysql_user: "wordpress"
mysql_password: "wordpress@123"
wp_url: "https://wordpress.org/wordpress-6.1.1.tar.gz"
```

while comparing the files, you can see that the same variables are assigned different values.
the port defines the port number.
owner defines the owner of the website files, In amazon Linux the owner is apache and in ubuntu, it is www-data, the same values are applicable for group too,
domain defines the domain name, in our example, it is example.com
mysql_root_password used to specify the root password for our MySQL server.
mysql_db defines the database base for our wordpress website and mysql_user defines the wordpress user.
wp_url defines the wordpress tar file, which will be used in the wordpress installation.

As you know, we need to make a few changes to files such as wp-config.php, virtual host file to setting up wordpress, so we are copying modified versions of these files to the servers using the template feature,

Since we mentioned a few variables in the code, they must be replaced with the corresponding values while copying to the client servers.

Using the template, we can perform the above-mentioned operation, it will replace the variables with their corresponding values.

For setting up wordpress on our LAMP stack, we need to copy the following files.

httpd.conf
wp-config.php
virtual host

the extension for these files must be either .tmpl or .j2, in our example, we are using .tmpl format.

Let’s check on these files.

httpd.conf — it is the default configuration file of the apache webserver, we are only modifying a small portion, the listen port will be replaced by our “port” variable.

```
$ cat httpd.conf.tmpl | grep port

Listen "{{port}}"
```

This is the change that we mentioned earlier, the rest of the file will be the same while executing the playbook, the port will be replaced with the corresponding value, in our example, it will be 8080.

In the wp-config.php file, we are providing the database details as variables, so the database section of the wp-config file will be the following.

```
$ cat wp-config.php.tmpl |DB

define( 'DB_NAME', '{{mysql_db}}' );
define( 'DB_USER', '{{mysql_user}}' );
define( 'DB_PASSWORD', '{{mysql_password}}' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8' );
define( 'DB_COLLATE', '' );
```

virtual.conf.tmpl defines the virtual host entry for our website.

```
$ cat virtual.conf.tmpl

<virtualhost *:{{ port }}>
  servername {{ domain }}
  documentroot /var/www/html/{{ domain }}
  directoryindex index.html index.php
  <directory "/var/www/html/{{ domain }} ">
    allowoverride all
  </directory>

</virtualhost>
```

Now regarding the LEMP stack, there are some changes, the wp-config.php.tmpl will remain the same.

the virtual host entry will be like the following.

```
$ cat nginx-virtualhost.tmpl

server {
listen "{{port}}";
root "/var/www/{{domain}}";
index index.php index.html index.htm;
server_name "{{domain}}";
location / {
try_files $uri $uri/ =404;
}
location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
     }
}
```

Now let’s proceed with our playbook, in our playbook we are performing the actions are tasks, since the tasks/actions for different operations systems are different, we are adding the tasks using the include_tasks option.

it is a very useful feature while writing a playbook with more tasks.

Let’s check on the tasks for Amazon-Linux for setting up the LAMP stack.

![lamp](https://user-images.githubusercontent.com/61390678/219973848-7710bd42-7a94-4dca-8a71-e2266b8ee3d8.png)


```
$cat amazon.yml

---
    - name: "installing apache"
      yum:
        name: httpd
        state: present
      tags:
        - lamp

    - name: "installing php"
      shell: amazon-linux-extras install php7.4 -y
      tags:
        - lamp

    - name: "copying httpd.config"
      template:
        src: ./httpd.conf.tmpl
        dest: /etc/httpd/conf/httpd.conf
      tags:
        - lamp
        - wordpress

    - name: "creating virtual host"
      template:
        src: "./virtual.conf.tmpl"
        dest: "/etc/httpd/conf.d/{{domain}}.conf"
        owner: "{{owner}}"
        group: "{{group}}"
      tags:
        - lamp
        - wordpress
    - name: "creating document root"
      file:
        path: "/var/www/html/{{domain}}"
        state: directory
        owner: "{{owner}}"
        group: "{{group}}"
      tags:
        - lamp

    - name: "creating sample files"
       copy:
        src: "{{item}}"
        dest: "/var/www/html/{{domain}}/"
        owner: "{{owner}}"
        group: "{{group}}"
      with_items:
        - sample.php
        - sample.html
      tags:
        - lamp
        - wordpress

    - name: "installing mariadb package"
      yum:
        name:
          - mariadb-server
          - MySQL-python
        state: present
      tags:
        - lamp
        - wordpress
        - mariadb
    - name: "restarting service"
      service:
        name: mariadb
        state: restarted
        enabled: true
      tags:
        - mariadb
        - wordpress
        - lamp
    - name: "updating root password"
      ignore_errors: true
      mysql_user:
        login_user: "root"
        login_password: ""
        name: "root"
        password: "{{mysql_root_password}}"
        host_all : true
      tags:
        - lamp
        - wordpress
        - mariadb

    - name: "removing anonymous users"
      mysql_user:
        login_user: "root"
        login_password: "{{mysql_root_password}}"
        name: ""
        password: ""
        host_all: true
        state: absent
      tags:
       - wordpress
       - lamp
       - mariadb

    - name: "creating database"
      mysql_db:
        login_user: "root"
        login_password: "{{mysql_root_password}}"
        name: "{{mysql_db}}"
        state: present
      tags:
        - lamp
        - wordpress
        - mariadb
    - name: "creating user"
      mysql_user:
        login_user: "root"
        login_password: "{{mysql_root_password}}"
        name: "{{mysql_user}}"
        password: "{{mysql_password}}"
        state: present
        host: "%"
        priv: "{{mysql_db}}.*:ALL"
      tags:
       - wordpress
       - mariadb
       - lamp

    - name: "downloading wordpress files"
      get_url:
        url: "{{wp_url}}"
        dest: "/tmp/wordpress.tar.gz"
      tags:
        - wordpress
        - lamp
        - mariadb
    - name: "extracting files"
      unarchive:
        src: "/tmp/wordpress.tar.gz"
        dest: "/tmp/"
        remote_src: true
      tags:
        - lamp
        - mariadb
        - wordpress
    - name: "copying website files"
      copy:
        src: "/tmp/wordpress/"
        dest: "/var/www/html/{{domain}}"
        owner: "{{owner}}"
        group: "{{group}}"
        remote_src: yes
    - name: "setting up wp-config"
      template:
        src: ./wp-config.php.tmpl
        dest: "/var/www/html/{{domain}}/wp-config.php"
        owner: "{{owner}}"
        group: "{{group}}"
    - name: "restarting services"
      service:
        name: "{{item}}"
        state: restarted
        enabled: true
      with_items:
        - httpd
        - mariadb
      tags:
        - httpd
        - mariadb
    - name: "cleanup"
      file:
        name: "{{item}}"
        state: absent
      with_items:
        - "/tmp/wordpress/"
        - "/tmp/wordpress.tar.gz"
      tags:
        - Wordpress
 ```
 
Let’s have a closer look at the tasks.

task1 — installation of httpd using yum module
task2 — installation of php7.4 using shell module
task3 — copying of httpd.conf the updated httpd.conf will be replaced with the default httpd conf.
task4 — setting up the virtual host
task6 — creating document root for the website. (/var/www/html/example.com)
task 7 — copying 2 sample files called sample.html and sample.php for checking the php and apache installation status
task 8 — MySQL installation
task 9 — restarting MySQL service
task 9 — updating the root password of the MySQL server (using the mysql_user module)
task 10 — removing anonymous users from MySQL
task 11 — creating a database for setting up wordpress (using the mysql_db module)
task 12 — creating a wordpress user and granting required privileges.
task 13- downloading wordpress files using wget (get_url is used for wget)
task 14- extracting wordpress files to /tmp
task 15- copying websites files to document root and updating ownership
task 16 — setting up wp-config in the document root.
task 17- restarting services (apache, MySQL)
task 18 — removing unwanted files.

Now let’s check the tasks for the LEMP stack on the ubuntu server.

![lemp](https://user-images.githubusercontent.com/61390678/219973956-42a02321-58ff-4936-bba7-6482723aa8c0.png)


```
$ cat ubuntu.yml

---
    - name: "installing nginx"
      apt:
        update_cache: true
        name: nginx
        state: present
      tags:
        - lamp

    - name: "installing php"
      apt:
        name:
          - php-mysql
          - php-fpm
        update_cache: true
        state: present
      tags:
        - lamp

    - name: "creating virtual host"
      template:
        src: "./nginx-virtualhost.tmpl"
        dest: "/etc/nginx/sites-available/{{domain}}"
      tags:
        - lamp
        - wordpress
    - name: "creating document root"
      file:
        path: "/var/www/{{domain}}"
        state: directory
        owner: "{{owner}}"
        group: "{{group}}"
      tags:
        - lamp

    - name: "creating sample files"
      copy:
        src: "{{item}}"
        dest: "/var/www/{{domain}}/"
        owner: "{{owner}}"
        group: "{{group}}"
        with_items:
        - sample.php
        - sample.html
      tags:
        - lamp
        - wordpress

    - name: "link"
      file:
        src: "/etc/nginx/sites-available/{{domain}}"
        dest: "/etc/nginx/sites-enabled/{{domain}}"
        state: link

    - name: "installing mysql package"
      apt:
        update_cache: true
        name:
          - mariadb-server
          - python3-pymysql
        state: present

    - name: "updating root password"
      ignore_errors: true
      mysql_user:
        login_unix_socket: /var/run/mysqld/mysqld.sock
        login_user: "root"
        login_password: ""
        name: "root"
        password: "{{mysql_root_password}}"
        host_all : true
      tags:
        - lamp
        - wordpress
        - mariadb

    - name: "removing anonymous users"
      mysql_user:
        login_user: "root"
        login_password: "{{mysql_root_password}}"
        name: ""
        password: ""
        host_all: true
        state: absent
         tags:
       - wordpress
       - lamp
       - mariadb

    - name: "creating database"
      mysql_db:
        login_user: "root"
        login_password: "{{mysql_root_password}}"
        name: "{{mysql_db}}"
        state: present
      tags:
        - lamp
        - wordpress
        - mariadb
    - name: "creating user"
      mysql_user:
        login_user: "root"
        login_password: "{{mysql_root_password}}"
        name: "{{mysql_user}}"
        password: "{{mysql_password}}"
        state: present
        host: "%"
        priv: "{{mysql_db}}.*:ALL"
      tags:
        - wordpress
        - mariadb
        - lamp

    - name: "downloading wordpress files"
      get_url:
        url: "{{wp_url}}"
        dest: "/tmp/wordpress.tar.gz"
      tags:
        - wordpress
        - lamp
        - mariadb
    - name: "extracting files"
      unarchive:
        src: "/tmp/wordpress.tar.gz"
        dest: "/tmp/"
        remote_src: true
      tags:
        - lamp
        - mariadb
        - wordpress
    - name: "copying website files"
      copy:
        src: "/tmp/wordpress/"
        dest: "/var/www/{{domain}}"
        owner: "{{owner}}"
        group: "{{group}}"
        remote_src: yes
    - name: "setting up wp-config"
      template:
        src: ./wp-config.php.tmpl
        dest: "/var/www/{{domain}}/wp-config.php"
        owner: "{{owner}}"
        group: "{{group}}"
    - name: "restarting services"
      service:
        name: "{{item}}"
        state: restarted
        enabled: true
      with_items:
        - nginx
        - mariadb
      tags:
        - nginx
        - mysql
    - name: "cleanup"
      file:
        name: "{{item}}"
        state: absent
      with_items:
        - "/tmp/wordpress/"
        - "/tmp/wordpress.tar.gz"
      tags:
         - Wordpress
```

The tasks are almost similar here, but there are a few changes.

Task1 — installing nginx using apt module
task3 — creating virtual host in the folder /etc/nginx/sites-available
task6 — creating symlink (to sites-enabled)

All other tasks are similar to LAMP stack installation.

By using the feature of include_tasks, you are able to run tasks that are written in different files., Which makes the codes easy to manage.

Also, we are setting up a few conditions for executing the right file, In our example, the playbook fetches the os family and package manager of the client machines from facts gathering and it will execute the playbook based on the result.

Now let’s have a look at the playbook.

Please note that the playbooks are using the YAML format.

```
$ cat main.yml

---

- name: "wordpress installation"
  become: yes
  gather_facts: true
  hosts: amazon, ubuntu

  tasks:

    - include_tasks: "amazon.yml"
      when: ansible_distribution == "Amazon" and ansible_pkg_mgr == "yum"

    - include_tasks: "ubuntu.yml"
      when: ansible_distribution == "Ubuntu" and ansible_pkg_mgr == "apt"
 
 ```
 
In this code, you can see that our subcodes amazon.yml and ubuntu.yml as include_tasks. These tasks will be executed when the mentioned condition is satisfied.

As mentioned before we are fetching the values of ansible_distribution & ansible_pkg_mgr from the setup module, so we are keeping the gather_facts as enabled.

In order to install packages on the server, you need to have sudo privileges. so we are setting up privilege escalation to true, so the commands will be executed with sudo privileges.

hosts define the groups to which the code needs to be applied.

Now let’s proceed with executing the playbook, before that please make sure that there are no syntax errors in the code.

You can verify the same using the following command.

```
$ ansible-playbook -i inventory.txt main.yml - syntax-check
playbook:main.yml
```

If you are getting an output like the above, the code is error-free.

Now, let’s execute the codes.

```
$ ansible-playbook -i inventory.txt main.yml 
```

Once it is executed successfully, you can visit the browser and set up wordpress.

Conclusion
— — — — — -
As already mentioned earlier, you can easily perform wordpress installation on servers regardless of the number of servers using ansible playbook.

I hope this document was able to provide a clearcut idea about deploying wordpress on servers using the ansible playbook.

If you are facing any difficulties or have any doubts, feel free to contact me via LinkedIn.

Thanks for reading :)
