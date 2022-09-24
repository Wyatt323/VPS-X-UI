# VPS搭建X-UI+伪装网站 教程

#第一步：安装
 *更新软件源 并 开启BBR
  更新软件源
  ```shell
  apt update
  ```
  开启BBR
  ```shell
  echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
  echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
  sysctl -p
  ```
  
  *安装X-UI
  ```shell
  bash <(curl -Ls https://raw.githubusercontent.com/vaxilu/x-ui/master/install.sh)
  ```
  安装过程中选择Y，输入自定义用户名、密码、端口
  
  *安装Nginx
  ```shell
  apt install nginx
  ```
  
  *安装acme：
  ```shell
  curl https://get.acme.sh | sh
  ```
  
  *添加软链接：
  ```shell
  ln -s  /root/.acme.sh/acme.sh /usr/local/bin/acme.sh
  ```
  
  *切换CA机构： 
  ```shell
  acme.sh --set-default-ca --server letsencrypt
  ```
  
  *申请证书： 
  ```shell
  acme.sh  --issue -d 你的域名 -k ec-256 --webroot  /var/www/html
  ```
  
  *安装证书：
  ```shell
  acme.sh --install-cert -d 你的域名 --ecc --key-file       /etc/x-ui/server.key  --fullchain-file /etc/x-ui/server.crt --reloadcmd     "systemctl force-reload nginx"
  ```
  
 #第二步：寻找适合的伪装站（http站点优先，个人网盘符合单节点大流量特征）
  谷歌搜索关键字：
  ```shell
  intext:登录 Cloudreve
  ```
  
 #第三步：配置nginx
  配置文件路径：/etc/nginx/nginx.conf
  ```shell
  /etc/nginx/nginx.conf
  ```
  修改成下面代码：
  ```shell
  user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    gzip on;

    server {
        listen 443 ssl;
        
        server_name nicename.co;  #你的域名
        ssl_certificate       /etc/x-ui/server.crt;  #证书位置
        ssl_certificate_key   /etc/x-ui/server.key; #私钥位置
        
        ssl_session_timeout 1d;
        ssl_session_cache shared:MozSSL:10m;
        ssl_session_tickets off;
        ssl_protocols    TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers off;

        location / {
            proxy_pass https://bing.com; #伪装网址
            proxy_redirect off;
            proxy_ssl_server_name on;
            sub_filter_once off;
            sub_filter "bing.com" $server_name;
            proxy_set_header Host "bing.com";
            proxy_set_header Referer $http_referer;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header User-Agent $http_user_agent;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header Accept-Encoding "";
            proxy_set_header Accept-Language "zh-CN";
        }


        location /ray {   #分流路径
            proxy_redirect off;
            proxy_pass http://127.0.0.1:10000; #Xray端口
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        
        location /xui {   #xui路径
            proxy_redirect off;
            proxy_pass http://127.0.0.1:9999;  #xui监听端口
            proxy_http_version 1.1;
            proxy_set_header Host $host;
        }
    }

    server {
        listen 80;
        location /.well-known/ {
               root /var/www/html;
            }
        location / {
                rewrite ^(.*)$ https://$host$1 permanent;
            }
    }
}
  ```
  每次修改nginx配置文件后必须使用 systemctl reload nginx 命令重新加载配置文件
 
 添加节点到v2中，修改节点：
 端口——>443
 加密方式——>zero
 打开TLS

 如需导入到手机或者openwrt中使用，请分享V2中修改过的节点连接，这样就不需要在别的客户端修改了
 
