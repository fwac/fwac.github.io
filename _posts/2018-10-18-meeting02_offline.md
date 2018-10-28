---
layout: post
title: Setting up an offline repository
author: Scott Tully
---

This document will provide guidance on setting up the necessary repositories to support most environments that host Ansible, Docker, and Python installs.  I will try to provide step-by-step instructions for the important parts of the procedure, but I’m sure you will need to fill gaps. There are many ways to do this… This is one.

Prerequisites: >=128GB thumb drive or disk, and a Linux machine that matches the flavor of the offline install media you want. You will need Internet connectivity to download the packages.  

## Yum Repo
I am using CentOS here so yum will be the package manager of choice.  From your internet box, mount your drive to a directory.  I mounted /opt, but you can use /mnt or whatever. I will call it $MNT

`# mount /dev/sdX $MNT`

Let’s install some packages to get ready for some reposyncing. On RHEL docker is in the extras repo. 

`# yum -y install yum-utils docker-ce epel-release`

You can also use the following if epel-release is not available
`wget https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm`


You can now check what repos you have by running 

`# yum repolist`

You will need to note the “repo id” in the left most column. This will be the values provided in the next steps. 
This is it. It’s going to take a while for each reposync to complete.  But when you are done, you will have everything to install rpms offline. You should make a directory tree like this ( match your architecture) 

```
# mkdir –vp $MNT/centos/7/os/x86_64/CentOS/RPMS
# mkdir –vp $MNT/centos/7/epel/x86_64/CentOS/RPMS
# mkdir –vp $MNT/centos/7/update/x86_64/CentOS/RPMS
# mkdir –vp $MNT/centos/7/extras/x86_64/CentOS/RPMS
# mkdir –vp $MNT/centos/7/docker-ce-stable/x86_64/CentOS/RPMS
```

```
# cd $MNT/centos
# reposync --repoid=base --repoid=etxra --repoid=updates --repoid=epel --repoid=docker-ce-stable
# mv base/Packages/* $MNT/centos/7/os/x86_64/CentOS/RPMS
# mv epel/Packages/* $MNT/centos/7/epel/x86_64/CentOS/RPMs
# mv updates/Packages/* $MNT/centos/7/update/x86_64/CentOS/RPMS
# mv extras/Packages/* $MNT/centos/7/extras/x86_64/CentOS/RPMS
# mv docker-ce-stable/Packages/* $MNT/centos/7/docker/x86_64/CentOS/RPMS
```

```
# createrepo $MNT/centos/7/os/x86_64
# createrepo $MNT/centos/7/epel/x86_64
# createrepo $MNT/centos/7/update/x86_64
# createrepo $MNT/centos/7/extras/x86_64
# createrepo $MNT/centos/7/docker-ce-stable/x86_64
```
If you are running RHEL you can do something like this instead. You will need to check the subscriptions you have available. The example below includes OpenShift.
```
for repo in rhel-7-server-rpms rhel-7-server-extras-rpms rhel-7-fast-datapath-rpms rhel-7-server-ansible-2.4-rpms rhel-7-server-ose-3.9-rpms
do   
  reposync --gpgcheck -lm --repoid=${repo} --download_path=$MNT;   
  createrepo -v  $MNT/${repo} -o $MNT/${repo}; 
done

```

Okay, now that we have a copy, let’s disable those Internet repos.  
```
# yum-config-manager --disable \*
# yum repolist
```
The repolist should confirm you have no repos enabled.
Now use the sample yum configuration below to add your local repos.

```
[fwac-base]
name=FWAC-$releasever - Base
baseurl=http://fwac/centos/$releasever/os/$basearch/
gpgcheck=0
enabled=1
#released updates

[fwac-update]
name=FWAC-$releasever - Updates
baseurl=http://fwac/centos/$releasever/update/$basearch/
gpgcheck=0
enabled=1

[fwac-extras]
name=FWAC-$releasever - Extras
baseurl=http://fwac/centos/$releasever/extras/$basearch/
gpgcheck=0
enabled=1

[fwac-epel]
name=FWAC-$releasever - EPEL
baseurl=http://fwac/centos/$releasever/epel/$basearch/
gpgcheck=0
enabled=1

[fwac-docker]
name=FWAC-$releasever - Docker
baseurl=http://fwac/centos/$releasever/docker-ce-stable/$basearch/
gpgcheck=0
enabled=1
```

Before you can use this configuration, you must setup apache, or nginx to serve the rpms.  I used Nginx.  You can either point the document root to your $MNT directory, or like I did, create a symlink inside /usr/share/nginx/html to /opt/centos/. I did this because I thought I might use the webserver for other stuff too.

`# ln –s /opt/centos /usr/share/nginx/html/centos`

With this configuration in place you should be able to see your repos
`# yum repolist`

## PyPi Repo

The second part of this is duplicating the PyPi servers so the pip python package manager can be used offline. Believe me this will make your life better if you do not have internet access from your hosts.
We need to install a couple things to set this up.  If you don’t have pip already you can install it using the yum package manager.
```
# yum install python-pip
# pip install pypiserver[cache]
# pip install minirepo
```

Assuming that all goes smoothly, the next step is running minirepo. You can research it to find out more, but I was fine with the defaults.  The first time you run it you will get a prompt for a directory to use to store the thousands of files you are about to download.  Enter something like $MNT/python/packages.  This will probably take a couple hours. 

We will use the pypiserver to actually serve the index for pip to search for packages.  It gets a little weird here. You have options. Maybe my solution is not the one for you.  I ended up doing a nginx proxy and alias for https.  Since we are doing the entire pypi repository,  performance with pypiserver can be an issue.  Every request causes the server to rescan the directory and is a huge performance hit.  You should also use TLS/https for connections. I’m going to leave the TLS discussion out of this for now because I don’t feal like writing about it. 

To run the pypi server, I use this command:
```
# pypi-server --disable-fallback -i 127.0.0.1 -p 5050 -v --log-file /var/log/pypi.log $MNT/python/packages
```
The `--disable-fallback` will block requests that don't find an answer in the offline index from attmepting to go out to the internet.

To put a front end that support TLS connections, I use nginx with this config. The server on port 80 is actually used for my rpm repo.

```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

# Settings for a TLS enabled server.

    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  _;
        root         /opt/python;

        ssl_certificate "/opt/ca/pypi.local.crt";
        ssl_certificate_key "/opt/ca/pypi.local.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers on;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
          proxy_set_header Host $http_host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto https;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          add_header Pragma "no-cache";
          proxy_pass http://127.0.0.1:5050;
        }
        location /packages/ {
          root /opt/python/;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

}
```

This is the important part for the server on 443:

```
        location / {
          proxy_set_header Host $http_host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-Proto https;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          add_header Pragma "no-cache";
          proxy_pass http://127.0.0.1:5050;
        }

        location /packages/ {
          root /opt/python/;
        }
```
The `location /` directive takes all the requests for index search and proxies them to pypiserver.  The `location /packages/` directive is the performance boost.  When the client receives the url for the download, nginx will serve the file, not pypiserver.  This happens because the url has a match for the ‘/packages/’ directory. 

Now that you have a shiny new PyPi server setup you will need to make pip use it.  You will need to create a file named `.pip/pip.conf` in your home directory with this in it. Replace values as needed.
```
[global]
index-url = https://pypi.local
cert = /opt/fwac/meeting02/myCA.pem
```

Maybe I will come back and give more explanation on setting up a little CA and issueing certificates. But for now that's all I have. You can reference the [offline playook](https://github.com/fwac/fwac/blob/master/meeting02/ansible/offline.yml) and the [shell scripts](https://github.com/fwac/fwac/blob/master/meeting02/bin/) for info.
