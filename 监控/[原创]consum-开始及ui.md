+++
date = "2015-10-20T16:30:09+08:00"
draft = true
title = "【原创】consul开始及ui"

+++
## consul
执行命令`./consul agent -server -bootstrap-expect 1 -data-dir /home/consul -node Litao-MacBook-Pro -dc sz-1 -bind 45.32.21.60 -ui-dir /media/monitor/ui -atlas=ATLAS_USERNAME/demo -atlas-token=xxx`
 
#### 注意 

- `-ui-dir`后面标注的是ui包的地址，也就是index.html所在地址，并且最后的`ui/`是访问ui的uri。
- `-atlas=ATLAS_USERNAME/demo -atlas-token=xxx`是否必须，还未验证

#### 启动的时候，有些端口是回环地址，我用nginx做代理进行访问
 
        server {
            listen       80;
            server_name 45.32.21.60;
            charset utf-8;
            access_log  /home/xiaoju/nginx/logs/access.log;
            root /media/monitor;
                       index index.html;
            location / {
                       proxy_pass http://127.0.0.1:8500;
              }
            location /ui {
                       root /media/monitor;
                       index index.html;
              }
        }

我的vps: http://45.32.21.60/ui/(还没上域名)

参考:

https://www.consul.io/docs/upgrading.html
https://www.consul.io/intro/getting-started/ui.html
http://txt.fliglio.com/2014/05/encapsulated-services-with-consul-and-confd/
https://blog.coding.net/blog/intro-consul
