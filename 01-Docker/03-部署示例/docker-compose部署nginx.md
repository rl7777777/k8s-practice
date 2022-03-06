# 安装Nginx

docker挂载文件时会覆盖掉容器里面的目录，因此需准备一份默认的配置文件

## docker直接启动

```shell
# 启动nginx
docker run -d --name nginx -p 80:80 nginx:latest

# 拷出默认配置文件
mkdir -p /root/nginx/{config,data,logs}
docker cp nginx:/etc/nginx /root/nginx/config/ 
docker cp nginx:/usr/share/nginx/html /root/nginx/data/
docker cp nginx:/var/log/nginx /root/nginx/logs/ 

# 持久化
docker run --name nginx -p 80:80 \
-v /root/nginx/config/nginx/:/etc/nginx \
-v /root/nginx/data/html:/usr/share/nginx/html \
-v /root/nginx/logs/:/var/log/nginx \
-d nginx:latest
```

## docker-compose

```
vim docker-compose.yml
```

```yaml
version: '3.1'
services:
  api-server-nginx:
    image: nginx         # 镜像
    container_name: api-server-nginx # 容器名
    restart: always      # 开机自动重启
    ports:               # 端口号绑定（宿主机:容器内）
      - 16443:16443
    privileged: true
    volumes:             # 目录映射（宿主机:容器内）
      - ./volumes/config/nginx/:/etc/nginx
      - ./volumes/data/html:/usr/share/nginx/html
      - ./volumes/logs/:/var/log/nginx
```

```
mkdir -p ./volumes/{config,data,logs}

docker cp api-server-nginx:/etc/nginx ./volumes/config/ 
docker cp api-server-nginx:/usr/share/nginx/html ./volumes/data/
docker cp api-server-nginx:/var/log/nginx ./volumes/logs/ 
```

```
user  nginx;
worker_processes  4;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}

stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
                server 172.16.2.11:6443;
                server 172.16.2.12:6443;
                server 172.16.2.13:6443;
            }

    server {
       listen 16443;
       proxy_pass k8s-apiserver;
    }
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

- 健康检查脚本

```
cd /etc/keepalived
vim check_nginx.sh
chmod+x
```

```
#!/bin/bash

count=$(docker ps -a|grep api-server-nginx | grep Up|wc -l)  # 查docker中api-server-nginx容器是否正常运行

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi
```

- nginx

```
docker run --name nginx -d -p 80:80 nginx:latest
```

```
scp -r ha 172.16.2.12:/root/
scp /usr/local/bin/docker-compose 172.16.2.12:/usr/local/bin/
```

### 