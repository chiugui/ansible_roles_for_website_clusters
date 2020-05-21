# ansible_roles搭建web集群架构



## 1.环境准备

### 1.1安装ansible以及依赖

```
yum install -y ansible
yum install -y python-pip
pip install redis
```

### 1.2添加被控主机及主机组

```
[root@m01 ~]# cd /ansible
[root@m01 ~/ansible]# vim hosts
[lbs]
172.16.1.5 state=MASTER priority=150
172.16.1.6 state=BACKUP priority=120
[webservers]
172.16.1.7 
172.16.1.8 
[redis]
172.16.1.71
#基于账户和密码的方式控制主机
#172.16.1.31 ansible_ssh_port=22 ansible_ssh_user=root ansible_ssh_pass='123456'
```

### 1.3创建ansible配置文件

```
[root@m01 ~/ansible]# vim ansible.cfg
[defaults]
inventory = ./hosts

#启用facts变量缓存到redis中
gathering = smart
fact_caching_timeout = 86400
fact_caching = redis
fact_caching_connection = 172.16.1.71:6379

#关闭首次连接被控主机需要验证yes/no的问题
# uncomment this to disable SSH key host checking
host_key_checking = False
```

### 1.4分发公钥到被控主机





## 2.roles编写

编写完成后的目录结构：

```
[root@m01 ~/ansible]# tree
.
├── 1st_run.yml
├── ansible.cfg
├── group_vars
│   ├── all
│   ├── lbs
│   └── webservers
├── hosts
├── lbs
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   └── main.yml
│   └── templates
│       ├── keepalived_tmp.conf.j2
│       ├── nginx.conf.j2
│       └── proxy_osker.com.conf.j2
├── main.yml
├── nginx
│   ├── handlers
│   │   └── main.yml
│   ├── tasks
│   │   └── main.yml
│   └── templates
│       ├── nginx.conf.j2
│       ├── osker.com.conf.j2
│       ├── php.ini.j2
│       └── php_www.conf.j2
├── Process_env.yml
├── README.md
└── redis
    ├── handlers
    │   └── main.yml
    ├── tasks
    │   └── main.yml
    └── templates
        └── redis.conf.j2
```



### 2.1 main roles编写

```
vim main.yml
- hosts: redis
  roles:
    - role: redis
- hosts: webservers
  roles:
    - role: nginx
- hosts: lbs
  roles:
    - role: lbs
```



### 2.2 redis roles编写

```
redis-role

vim redis/tasks/main.yml
- name: install redis service
  yum:
    name: redis
    state: present

- name: config redis.conf file
  template:
    src: redis.conf.j2
    dest: /etc/redis.conf
  notify: Restart Redis Service

- name: Started Redis Service
  systemd:
    name: redis
    state: started
    enabled: yes

vim redis/handlers/main.yml
- name: Restart Redis Service
  systemd:
    name: redis
    state: restarted
    
vim redis/templates/redis.conf.j2
bind 127.0.0.1 {{ansible_eth1.ipv4.address}}
```



### 2.3 nginx_php-roles编写

```
vim nginx/tasks/main.yml
- name: Install Nginx And PHP Service
  yum:
    name: "{{packages}}"
    state: present
    
- name: groupadd and useradd
  include: ../../Process_env.yml

  
- name: config nginx.conf and osker.com.conf files,php.ini and php-fpm.d/www.conf 
  template:
    src: "{{item.src}}"
    dest: "{{item.dest}}"
    backup: yes
  loop:
    - {src: nginx.conf.j2, dest: /etc/nginx/nginx.conf}
    - {src: osker.com.conf.j2, dest: /etc/nginx/conf.d/osker.com.conf}
    - {src: php.ini.j2, dest: /etc/php.ini}
    - {src: php_www.conf.j2, dest: /etc/php-fpm.d/www.conf}
  notify: Restart Nginx And PHP Service

- name: check nginx config files
  shell: nginx -t
  register: ck_ngx
  changed_when:
    - false                              ###nginx语法检查，没有对被控端进行文件改变，将返回状态设置为ok。
    - ck_ngx.stdout.find('successful')   ###检查ck_ngx.stdout返回的字符中有没有successful字段，有状态为ok，没有状态为failed，停止剧本。（此段功能主要为检查配置文件是否有语法错误，若有不会继续执行重启nginx服务，以免在用的nginx重启后服务不能正常启动）
- name: check php-fpm config files
  shell: php-fpm -t
  register: ck_php
  changed_when:
    - false
    - ck_php.stdout.find('successful')
    

- name: Started Nginx and PHP Service
  systemd:
    name: "{{item}}"
    state: started
    enabled: yes
  loop:
    - nginx
    - php-fpm

vim nginx/handlers/main.yml
- name: Restart Nginx And PHP Service
  systemd:
    name: "{{item}}"
    state: restarted
  loop:
    - nginx
    - php-fpm

mkdir -p host group_vars/webservers  
vim group_vars/webservers
packages:
  - nginx
  - php71w
  - php71w-cli
  - php71w-common    
  - php71w-devel
  - php71w-embedded
  - php71w-gd
  - php71w-mcrypt
  - php71w-mbstring
  - php71w-pdo
  - php71w-xml
  - php71w-fpm
  - php71w-mysqlnd
  - php71w-opcache
  - php71w-pecl-memcached
  - php71w-pecl-redis
  - php71w-pecl-mongodb
Web_Site: osker.com
Web_Root_Dir: /code

vim group_vars/all
Web_Port: 80


Process_Group: www
Process_User: www
Gid: 666
Uid: 666


vim Process_env.yml
- name: groupadd for nginx and php service
  group:
    name: "{{Process_Group}}"
    gid: "{{Gid}}"
- name: useradd for nginx and php service
  user:
    name: "{{Process_User}}"
    uid: "{{Uid}}"
    group: "{{Process_Group}}"
    shell: /sbin/nologin
    create_home: no

    
    
vim nginx/templates/nginx.conf.j2
user  {{Process_User}};
worker_processes  {{ansible_processor_vcpus * 2}};


vim nginx/templates/osker.com.conf.j2
server  {
	listen {{Web_Port}};
	server_name {{Web_Site}};
	root {{Web_Root_Dir}};
	client_max_body_size 100m;

	location / {
		index index.php;
	}

	location ~ \.php$ {
		fastcgi_pass 127.0.0.1:9000;
		fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		fastcgi_param HTTPS on;
		include fastcgi_params;
	}
  }

  
vim nginx/templates/php.ini.j2
session.save_handler = redis
 {% for host in groups['redis'] %}
session.save_path = "tcp://{{host}}:6379?weight=1"
{% endfor %}


vim nginx/templates/php_www.conf.j2
user = {{Process_User}}
group = {{Process_Group}}
###注意需要搜索出{%的关键字，并删除，否则报错playbook。

;php_value[session.save_handler] = files  注释本行
;php_value[session.save_path]    = /var/lib/php/session 注释本行
```



### 2.4 lbs roles编写

```
lbs

vim lbs/tasks/main.yml

- name: install nginx service
  yum:
    name: "{{item}}"
    state: present
  loop:
    - nginx
    - keepalived

- name: config nginx.conf and proxy_osker.com.conf file
  template:
    src: "{{item.src}}"
    dest: "{{item.dest}}"
  loop:
    - {src: proxy_osker.com.conf.j2, dest: /etc/nginx/conf.d/proxy_osker.com.conf}
  notify: Restart Nginx Service

  
- name: template keepalived.conf file
  template:
    src: keeplived_tmp.conf.j2
    dest: /etc/keepalived/keepalived.conf
  notify: Restart Keepalived Service

  
- name: Started Nginx Service
  systemd:
    name: nginx
    state: started
    enabled: yes
- name: Started Keepalived Service
  systemd:
    name: keepalived
    state: started
    enabled: yes
    
    
    

    
vim lbs/handlers/main.yml
- name: Restart Nginx Service
  systemd:
    name: nginx
    state: restarted
- name: Restart Keepalived Service
  systemd:
    name: keepalived
    state: restarted
    

vim group_vars/lbs
Proxy_Web_Port: 80
Web_Site: osker.com



vim lbs/templates/proxy_osker.com.conf.j2
upstream {{Web_Site}} {
 {% for host in groups['webservers'] %}
	server {{host}}:{{Web_Port}};
 {% endfor %}
 }
server {
	listen {{Proxy_Web_Port}};
	server_name {{Web_Site}};

	location / {
        proxy_pass http://{{Web_Site}};
        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
 }

 
 
vim lbs/templates/keepalived_tmp.conf.j2
global_defs {
    router_id {{ansible_hostname}}
}
vrrp_instance VI_1 {
    state {{state}}
    priority {{priority}}

    interface eth0
    virtual_router_id 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
 }
    virtual_ipaddress {
        10.0.0.4
    }
 } 
```



### 2.5 生成测试页面

```
vim 1st_run.yml
 - hosts: webservers
   tasks:
     - name: create group and user www
       include: ./Process_env.yml
     - name: create web_dir
	   file:
         path: /code
         state: directory
         owner: www
         group: www
         mode: '0755'
         recurse: yes
     - name: touch index.php
	   shell: echo 'test' >/code/index.php
```



## 3.运行测试

a.先运行ansible-playbook 1st_run.yml 创建一个测试页面
b.再运行ansible-playbook main.yml 