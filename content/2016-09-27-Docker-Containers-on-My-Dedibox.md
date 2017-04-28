Title: 在新 Dedibox 上部署 Docker
Date: 2016-09-27
Modified: 2017-04-28
Category: Post
Tags: Docker, Dedibox, Online.net, Transmission, Seafile, Lets-Encrypt
Slug: Docker Containers on My Dedibox
Summary: 在 Dedibox 上部署 Docker 全记录

刚刚 Online.net 又开卖 Limited Dediboxes，虽然这次很没诚意，但我还是买了个 C2750/1TB 的。
买后犹豫再三，还是安装了 Fedora 24 而非 Centos 7 或 Debian 8，原因大约是 Fedora 更新快。

后来，Online.net 又开卖 Limited Dediboxes，没忍住换了一个E3/2TB的，系统更新为 Fedora 25。

言归正传，下面开始部署 Docker。

Docker 安装
====================
Check https://fedoraproject.org/wiki/Docker for details!

首先更新个系统:
``` bash
sudo dnf -y update
```

然后安装 Docker，并设置好开机启动顺便启动 Docker:
``` bash
sudo dnf -y install docker
sudo systemctl enable --now docker
```

测试下 Docker 是否正常运行:
``` bash
sudo docker info
```

然后建立用于自动启动 Docker Container 的 Systemd Service 文件：
```bash
sudo tee /etc/systemd/system/docker@.service <<-'EOF'
[Unit]
Description=General systemd service profile for one docker container
Requires=docker.service
After=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a %i
ExecStop=/usr/bin/docker stop -t 2 %i

[Install]
WantedBy=multi-user.target
EOF
```

安装各个应用
---------------------
显示目前运行的容器:
``` bash
sudo docker info
```

显示现有的全部容器:
``` bash
sudo docker info -a
```

安装 Nginx Proxy 和 Lets-Encrypt
---------------------
Check https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion for details！

*下列操作均由 root 用户完成。*

1) 首先弄个 Docker 设置目录，建立 docker 用戶組，並添加你的管理員帳號到 docker 組

``` bash
mkdir -p /etc/docker/certs
chmod 700 /etc/docker/certs

groupadd docker
usermod -a -G docker YOUR_ADMIN_ACCOUNT
```

2) 然后下载 Nginx 的模板文件
``` bash
wget https://github.com/jwilder/nginx-proxy/raw/master/nginx.tmpl -O /etc/docker/nginx.tmpl
```

3) 新建一個 bridge 網絡，並新建一个官方的 Nginx Container，用于实际运行 Nginx
``` bash

docker network create --subnet=172.23.39.0/24 nginx-proxy

docker run -d -p 80:80 -p 443:443 \
    --name nginx --network nginx-proxy \
    -v /etc/nginx/conf.d  \
    -v /etc/nginx/vhost.d \
    -v /usr/share/nginx/html \
    -v /etc/docker/certs:/etc/nginx/certs:ro \
    nginx
```

4) 运行一个 docker-gen Container，用于设置 Nginx 的配置文件
``` bash
docker run -d \
    --name nginx-gen --network nginx-proxy \
    --volumes-from nginx \
    -v /etc/docker/nginx.tmpl:/etc/docker-gen/templates/nginx.tmpl:ro \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    jwilder/docker-gen \
    -notify-sighup nginx -watch -wait 5s:30s /etc/docker-gen/templates/nginx.tmpl /etc/nginx/conf.d/default.conf
```

5) 运行用于生成 Lets-Encrypt 证书的 letsencrypt-nginx-proxy-companion Container
``` bash
docker run -d \
    --name nginx-letsencrypt --network nginx-proxy \
    -e "NGINX_DOCKER_GEN_CONTAINER=nginx-gen" \
    --volumes-from nginx \
    -v /etc/docker/certs:/etc/nginx/certs:rw \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    jrcs/letsencrypt-nginx-proxy-companion
```

6) 开启 firewalld 端口：
``` bash
firewall-cmd --add-service=https --add-service=http --permanent
firewall-cmd --reload
```

7) 开启开机启动：
```bash
tee /etc/systemd/system/docker@nginx-gen.service <<-'EOF'
[Unit]
Description=General systemd service profile for one docker container
Requires=docker.service docker@nginx.service
After=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a nginx-gen
ExecStop=/usr/bin/docker stop -t 2 nginx-gen

[Install]
WantedBy=multi-user.target
EOF

tee /etc/systemd/system/docker@nginx-letsencrypt.service <<-'EOF'
[Unit]
Description=General systemd service profile for one docker container
Requires=docker.service docker@nginx.service docker@nginx-gen.service
After=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a nginx-letsencrypt
ExecStop=/usr/bin/docker stop -t 2 nginx-letsencrypt

[Install]
WantedBy=multi-user.target
EOF

systemctl enable docker@nginx
systemctl enable docker@nginx-gen
systemctl enable docker@nginx-letsencrypt
```

安装 Transmission (作为 Seedbox)
---------------------
建立 Transmission 目录
``` bash
mkdir -p /opt/transmission
```

运行 Docker Container (Transmission 2.9.2 based on Debian:Stretch)
```bash
docker run -it --name transmission -p 127.0.0.1:9091:9091 \
            --network nginx-proxy \
            -p 51413:51413/tcp -p 51413:51413/udp \
            -v /opt/transmission:/var/lib/transmission-daemon \
            -e "TR_PEX_ENABLED=false" \
            -e "TR_LPD_ENABLED=false" \
            -e "TR_DHT_ENABLED=false" \
            -e "TR_DOWNLOAD_DIR=/var/lib/transmission-daemon/downloads" \
            -e "TR_INCOMPLETE_DIR=/var/lib/transmission-daemon/incomplete" \
            -e "TR_INCOMPLETE_DIR_ENABLED=true" \
            -e "TRUSER=<Your Username for Auth>" \
            -e "TRPASSWD=<Your Passcode for Auth>" \
            -e "VIRTUAL_HOST=<your.host.name>" \
            -e "VIRTUAL_PORT=9091" \
            -e "LETSENCRYPT_HOST=<your.host.name>" \
            -e "LETSENCRYPT_EMAIL=your@email" \
            -d coeusite/docker-transmission
```
*注意自己设置域名(VIRTUAL_HOST & LETSENCRYPT_HOST)、邮箱(LETSENCRYPT_EMAIL)、用户名(TRUSER)和密码(TRPASSWD)*

安装 https://github.com/ronggang/transmission-web-control[Transmission Web Control]
```bash
docker exec transmission /root/install-tr-web-control.sh
```

开启 firewalld 端口：
``` bash
firewall-cmd --add-port=51413/tcp --add-port=51413/udp --permanent
firewall-cmd --reload
```

开启开机启动：
``` bash
systemctl enable docker@transmission
```


安装 Seafile (作为私有云)
---------------------
详见 https://github.com/coeusite/docker-seafile

建立后台 MariaDB 数据库
``` bash
docker volume create --name seafile-dbstore

docker run -d -p 127.0.0.1:3306:3306 \
  --network nginx-proxy --ip 172.23.39.98 \
  -v seafile-dbstore:/var/lib/mysql:rw \
  -e MYSQL_ROOT_PASSWORD=<password> \
  -e MYSQL_DATABASE=seafile \
  -e MYSQL_USER=seafile \
  -e MYSQL_PASSWORD=<password> \
  --name seafile-db mariadb:latest
```

建立 Seafile 数据 Container
``` bash
docker run -it --dns=127.0.0.1 \
  --network nginx-proxy --ip 172.23.39.99 \
  -e SEAFILE_DOMAIN_NAME=<YOUR.HOST.NAME> \
  --name seafile-data coeusite/docker-seafile:latest  bootstrap
```

建立 Seafile 运行 Container
``` bash
docker run -d -t --dns=127.0.0.1 -p 127.0.0.1:8080:8080 \
            --network nginx-proxy --ip 172.23.39.99 \
            --volumes-from seafile-data \
            -e SEAFILE_DOMAIN_NAME=<YOUR.HOST.NAME> \
            -e "VIRTUAL_HOST=<YOUR.HOST.NAME>" \
            -e "VIRTUAL_PORT=8080" \
            -e "VIRTUAL_PROTO=https" \
            -e "LETSENCRYPT_HOST=<YOUR.HOST.NAME>" \
            -e "LETSENCRYPT_EMAIL=<YOUR@EMAIL>" \
            --name seafile coeusite/docker-seafile
```

然后就可以通过 https://<YOUR.HOST.NAME> 访问 Seafile 了！
不过现在还需要去后台修正一下端口，从8080修改为443。

开启开机启动：
``` bash
tee /etc/systemd/system/docker@seafile.service <<-'EOF'
[Unit]
Description=General systemd service profile for one docker container
Requires=docker.service docker@seafile-db.service
After=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a seafile
ExecStop=/usr/bin/docker stop -t 2 seafile

[Install]
WantedBy=multi-user.target
EOF

systemctl enable docker@seafile-db
systemctl enable docker@seafile
```
