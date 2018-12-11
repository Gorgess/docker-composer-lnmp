# docker-composer-lnmp

1.安装docker及docker-compose
centos7安装docker及docker-compose

2.docker-compose常用命令
#  在含有 docker-compose.yml 的文件夹下 构建容器
#  如有使用 Dockerfile 在修改 Dockerfile 文件之后再次执行如下即可应用修改
docker-compose up -d
 
#  停止 docker-compose.yml 里面的所有容器
docker-compose stop
 
#  删除 docker-compose.yml 里面的所有容器
docker-compose rm
 
#  查看 docker-compose.yml 里面 nginx 的日志
docker-compose logs -f nginx
 
#  重启 docker-compose.yml 里面的某一个容器
docker-compose restart nginx
####  start 启动 stop 停止 #####
3.docker-compose.yml文件内容
因为我宿主机上面已经安装 mysql redis php nginx 所以下面的端口都不是默认的端口

version : '2'
services :
    mysql :
        build : ./mysql # 使用Dockerfile文件
        ports :
            - "3366:3306" # 宿主机端口:容器端口
        environment :
            - MYSQL_ROOT_PASSWORD=123456 # 设置mysql的root密码
        volumes:
            - /home/docker/mysql/data:/var/lib/mysql:rw # mysql数据文件
        networks:
            - mysql-net # 加入网络
        container_name : mysql57 # 设置容器名字
    redis :
        build : ./redis
        ports :
            - "127.0.0.1:6398:6379" # 如不需外网访问容器里面的服务 设置ip地址为127.0.0.1即可
        environment :
            - appendonly=yes # 打开redis密码设置
            - requirepass=123456 # 设置redis密码
        networks:
            - redis-net
        container_name : redis40
    php :
        build : ./php
        ports :
            - "127.0.0.1:9800:9000"
        volumes :
            - /home/docker/web:/var/www/html:rw # web站点目录
            - /home/docker/php/php.ini:/usr/local/etc/php/php.ini:ro
            - /home/docker/php/www.conf:/usr/local/etc/php-fpm.d/www.conf:ro
        networks:
            - php-net
            - mysql-net
            - redis-net
        container_name : php72
    nginx :
        build : ./nginx
        ports :
            - "8080:80" # 如果宿主机有安装nginx或者apache并且在运行则需要映射到其他端口
            - "8081:81" # 设置多个站点
            - "8082:82"
            - "8083:83"
        depends_on :
            - "php"
        volumes :
            - /home/docker/web:/var/www/html:rw
            - /home/docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
            - /home/docker/nginx/server.conf:/etc/nginx/conf.d/default.conf:ro
        networks:
            - php-net
        container_name : nginx114
networks: # 创建网络
    php-net:
    mysql-net:
    redis-net:
查看其他文件请点击

4.如果宿主机和docker同时使用nginx的访问办法
第一种:和上面配置一样,访问容器的nginx地址为==>宿主机IP地址:端口==>xxx.xxx.x.x:8080

但是这样会出现一个问题=>>宿主机是开启了firewall防火墙且查看防火墙通过端口并没有8080端口,而外网还能访问8080端口进行网页浏览

第二种:就是第一种的基础上不让外网访问8080端口

第一步=>在docker-compose.yml文件中nginx配置的端口修改如下

        ports :
            - "127.0.0.1:8080:80"
            - "127.0.0.1:8081:81"
            - "127.0.0.1:8082:82"
            - "127.0.0.1:8083:83"
这样做的意义是这个8080端口只有宿主机可以访问,并且外网扫描不到8080端口

 如配置端口时只填入端口即让所有外网IP都可以访问,并且自动透过防火墙,但是firewall防火墙里面并没有8080端口或者服务

第二部:配置宿主机的nginx站点和docker里面的nginx站点

宿主机nginx站点

server {
        listen       80;
        server_name  xx1.xx.com;
 
        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8080;
        }
 
    }
 
server {
        listen       80;
        server_name  xx2.xx.com;
        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8081;
        }
 
    }
 
server {
        listen       80;
        server_name  xx3.xx.com;
 
        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8082;
         }
    }
 
#################################### https 配置样例  docker里面的nginx配置不用修改
######## 浏览器出现https不是完全安全的原因就是页面中含有http地址
######## 当页面没有http地址的时候就会有锁的图标 提示是安全连接
server {
        listen 80;
        server_name  xxx.com;
        return 301 https://xxx.com$request_uri;
}
server {
        listen 443 ssl;
        server_name  xxx.com;
        root         /home/www;
        ssl on;
        ssl_certificate /etc/letsencrypt/live/xxx.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/xxx.com/privkey.pem;
        ssl_session_cache shared:SSL:20m;
        ssl_session_timeout  10m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
        ssl_prefer_server_ciphers on;
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /etc/letsencrypt/live/xxx.com/chain.pem;
        #启用 HSTS 用于通知浏览器强制使用 https 通信
        add_header Strict-Transport-Security "max-age=31536000";
        #减少点击劫持
        add_header X-Frame-Options SAMEORIGIN;
        #禁止服务器自动解析资源类型
        add_header X-Content-Type-Options nosniff;
        #防XSS攻擊
        add_header X-Xss-Protection 1;
        resolver 8.8.8.8 8.8.4.4;
 
        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://127.0.0.1:8083;
        }
 
    }
容器里面的nginx站点  只展示了2个

server {
        listen       80;
        server_name  127.0.0.1;
        root         /var/www/html/web1;
 
        location / {
			index  index.html index.htm index.php l.php;
			if (!-e $request_filename) {
				rewrite  ^(.*)$  /index.php?s=/$1  last;
				break;
    		}
			autoindex  off;
        }
 
        location ~ \.php(.*)$  {
            fastcgi_pass   php72:9000;
            fastcgi_index  index.php;
            fastcgi_split_path_info  ^((?U).+\.php)(/?.+)$;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO  $fastcgi_path_info;
            fastcgi_param  PATH_TRANSLATED  $document_root$fastcgi_path_info;
            include        fastcgi_params;
        }
 
        error_page 404 /404.html;
            location = /40x.html {
        }
 
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
 
server {
        listen       81;
        server_name  127.0.0.1;
        root         /var/www/html/web2;
 
        location / {
			index  index.html index.htm index.php l.php;
			if (!-e $request_filename) {
                rewrite  ^(.*)$  /index.php?s=/$1  last;
                break;
            }
			autoindex  off;
        }
 
        location ~ \.php(.*)$  {
            fastcgi_pass   php72:9000;
            fastcgi_index  index.php;
            fastcgi_split_path_info  ^((?U).+\.php)(/?.+)$;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO  $fastcgi_path_info;
            fastcgi_param  PATH_TRANSLATED  $document_root$fastcgi_path_info;
            include        fastcgi_params;
        }
 
        error_page 404 /404.html;
            location = /40x.html {
        }
 
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }
