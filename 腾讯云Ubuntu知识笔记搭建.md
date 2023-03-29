首先先设置root用户的登录

本地ssh到云服后

```shell
# 设置root密码
sudo passwd root
# 设置root远程登录
sudo vim /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
PermitRootLogin yes

# 重启ssh服务
sudo service ssh restart
#查看防火墙状态
sudo ufw status
sudo ufw enable
```

安装docker

```shell
#  docker需要ubuntu的内核高于3.10
uname -r

mkdir /etc/docker
vim daemon.json
# 写入
{
   "registry-mirrors": [
       "https://mirror.ccs.tencentyun.com"
  ]
}

# 新增更新源
sudo echo "deb https://download.docker.com/linux/ubuntu zesty edge" > /etc/apt/sources.list
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
newgrp docker
docker version
# 设置docker远程访问
vim /usr/lib/systemd/system/docker.service
#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock -H tcp://0.0.0.0:2375
systemctl daemon-reload 
systemctl restart docker 
```

安装docker-compose

```shell
#源码安装
sudo curl -L "https://kgithub.com/docker/compose/releases/download/v2.17.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version

# pip安装 注意以“root”用户运行pip可能导致权限中断，并与系统包管理器的行为冲突。建议使用虚拟环境。
pip install setuptools
pip install --upgrade pip
pip install docker-compose
```

创建docker网络

```shell
# 24这个位置看网段，网段占用了就换一个
docker network create -d bridge --subnet 172.24.0.0/16 leanote_nw
```

安装mongodb

```shell
mkdir /root/mongodb
cd /root/mongodb
mkdir data log
# 使用leanote放数据到MongoDB的前置步骤
mkdir /root/leanote
#上传文件或wget
wget https://udomain.dl.sourceforge.net/projects/leanote-bin/files/2.6.1/leanote-linux-386-v2.6.1.bin.tar.gz/download
tar -xvf leanote-linux-amd64-v2.6.1.bin.tar.gz
cp -r /root/leanote/leanote/mongodb_backup/ /root/mongodb/data/db/
vim docker-compose.yml
docker-compose up -d
```

```yml
version: "2"

services:
  mongo:
    image:  mongo:4.2.2
    container_name: mongodb
    # 占用主端口
    network_mode: "leanote_nw"
    command: mongod --auth
    mem_limit: 1g
    restart: always
    privileged: true
    ports:
     - "27017:27017/tcp"
    environment:
     - MONGO_DATA_DIR=/data/db
     - MONGO_LOG_DIR=/data/logs
     - MONGO_INITDB_ROOT_USERNAME=admin
     - MONGO_INITDB_ROOT_PASSWORD=Hzj2023
    volumes:
     - /etc/localtime:/etc/localtime
     - $PWD/data/db:/data/db
     - $PWD/log:/data/logs
```

设置MongoDB密码

```shell
#进入mongoDB容器
docker exec -it mongodb bash
#数据还原
mongorestore -h localhost -u admin -p Hzj2023 --authenticationDatabase=admin -d leanote --dir /data/db/mongodb_backup/leanote_install_data/
#退出mongo服务
mongo
use admin;
db.auth("admin", "xxx");
use leanote;
db.createUser({user: 'xxx',pwd: 'xxx',roles: [{role: 'dbOwner', db: 'leanote'}]});
db.auth("xxx", "xxx");
db.users.update({"Username" : "admin"},{$set: { "Email" : "xxx@163.com"}});
exit
```

数据备份命令！

mongodump -h localhost -u root -p xxx--authenticationDatabase=admin -d leanote --dir

！！！记得端口打开

安装git

```shell
apt install git
git config --global user.name xxx
git config --global user.email "xxx@163.com"
git config --list
```

安装leanote 

```shell
# 如果在MongoDB先做了这一步就可以省略 直到cp这一步 如果还没则需要先把MongoDB容器删除重建
mkdir /root/leanote
#上传文件或wget

tar -xvf leanote-linux-amd64-v2.6.1.bin.tar.gz
cp -r /root/leanote/leanote/mongodb_backup/ /root/mongodb/data/db/

cd /root/leanote
mkdir data  conf
cp -r /root/leanote/leanote/public /root/leanote/data/
cp /root/leanote/leanote/conf/app.conf /root/leanote/conf
vim /root/leanote/conf/app.conf 
#修改如下信息
site.url=http://xxx:9000 # or http://x.com:8080, http://www.xx.com:9000
db.host=xxx
db.dbname=leanote # required
db.username=xxx # if not exists, please leave it blank
db.password=xxx # if not exists, please leave it blank

cd /root/leanote
vim docker-compose.yml
docker-compose up -d
# 防火墙开放端口
#ufw allow 9000/tcp
#ufw reload
```

```yml
version: '2'
services:
  leanote:
    image: axboy/leanote:2.6.1
    restart: always
    user: root
    mem_limit: 1g
    network_mode: "leanote_nw"
    container_name: leanote
    ports:
      - "9000:9000"
    volumes:
      - $PWD/data:/leanote/data
      - $PWD/db:/data/db
      - $PWD/conf/app.conf:/leanote/conf/app.conf
```

安装nginx

```shell
mkdir /root/nginx
docker run --name nginx -p 80:80 -d nginx:1.22.0

#从容器nginx中复制nginx.conf文件到宿主机
docker cp nginx:/etc/nginx/nginx.conf /root/nginx
docker cp nginx:/etc/nginx/conf.d/ /root/nginx
docker cp nginx:/usr/share/nginx/html/ /root/nginx
docker cp nginx:/var/log/nginx/ /root/nginx
#删除原来的容器
docker stop nginx
docker rm nginx
cd /root/nginx/conf.d
rm default.conf
cd ../
vim /root/nginx/nginx.conf
```

```json
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  10240;
}
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    				  '$status $body_bytes_sent "$http_referer" '
    				  '"$http_user_agent" "$http_x_forwarded_for"';

    log_format json '{"time_iso8601":"$time_iso8601",'
    '"host":"$host",'
    '"uri":"$uri",'
    '"connection":$connection,'
    '"connection_requests":$connection_requests,'
    '"server_addr":"$server_addr",'
    '"server_port":$server_port,'
    '"remote_addr":"$remote_addr",'
    '"remote_user":"$remote_user",'
    '"http_x_user":"$http_x_user",'
    '"http_x_forwarded_for":"$http_x_forwarded_for",'
    '"http_user_agent":"$http_user_agent",'
    '"http_referer":"$http_referer",'
    '"body_bytes_sent":$body_bytes_sent,'
    '"request":"$request",'
    '"request_uri":"$request_uri",'
    '"request_length":$request_length,'
    '"request_time":$request_time,'
    '"request_method":"$request_method",'
    '"upstream_connect_time":"$upstream_connect_time",'
    '"upstream_header_time":"$upstream_header_time",'
    '"upstream_response_time":"$upstream_response_time",'
    '"upstream_addr":"$upstream_addr",'
    '"status":$status}';
	
    access_log  logs/access.log  main;
    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
	upstream leanote{
		server x:9000;
	}

	  
	server {
        listen       80;
        server_name  localhost;
        client_max_body_size 10M;
        root  /root/app/compose/leanote/data/public;

        location / {
          proxy_read_timeout  600;
          proxy_pass http://leanote;
          #root   html;
          #index  index.html index.htm;
          proxy_set_header Host                $host:$server_port;
          proxy_set_header X-Real-IP           $remote_addr;
          proxy_set_header X-Forwarded-For     $proxy_add_x_forwarded_for;
          proxy_http_version 1.1;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "";
        }
		location /demo{
            deny all;
        }
        location ~ (/index.html|/html/loginPage.html|/css/wms/|/js/wms/|/images/packing/|/asset/media/audio/|/images/prod-logo.png|/images/prod-logo.svg|/images/wms/widgets/|/assets/media/audio/|/images/wms/returnReceipt/|/css/font-awesome-5.15.4/) {
          root /root/app/compose/leanote/data/public/;
        }
	}
}
```

```shell
#重新跑一个
docker run -p 80:80 \
-v /root/nginx/nginx.conf:/etc/nginx/nginx.conf \
-v /root/nginx/logs:/var/log/nginx \
-v /root/nginx/html:/usr/share/nginx/html \
-v /root/nginx/conf.d:/etc/nginx/conf.d \
-v /etc/localtime:/etc/localtime \
--network=leanote_nw \
--name nginx \
--restart=always \
-d nginx:1.22.0

sudo ufw allow 80/tcp
sudo ufw reload
```
