---
title: Laravel 项目里使用 Swoole
date: 2017-12-31 13:30:21
tags:
  - Laravel
  - Swoole
  - Docker
category: Server
---

一直想用 swoole，却一直没有真正地去了解它。

前些天看到群里有人讨论相关的技术问题，于是下定决心试一试。但是真正搞定它前后折腾了近一个星期。

<!-- more -->

### 起步

试用一下 swoole，很简单，官网的 [swoole server 例子](https://wiki.swoole.com/wiki/page/p-server.html)

本地环境 ab 测一下：

![ab](https://pico.oss-cn-hangzhou.aliyuncs.com/blog/2mq13.jpg)

简直惊叹！当然官方的压测数据更详细和准确： [压力测试](https://wiki.swoole.com/wiki/page/62.html)

### Laravel 结合

Swoole 虽强，但是还是要能和项目结合好才算能用。

调研了几个 laravel 的 swoole 库之后，选择了 [huang-yi/laravel-swoole-http](https://github.com/huang-yi/laravel-swoole-http)，他对 swoole 的封装实现比较好。当然他只实现了 http server，目前也是足够了。

如果项目比较简单，基于 Laravel 并且没有太多的修改，那么可能 1 个小时就能立竿见影了。但是项目复杂了就折腾起来没完了。

研究了下实现的源码，原理还比较简单，和 `artisan serve` 差不多。

1. 管理 swoole `start` `stop` 等几个命令；
2. 启动 Laravel Application；
3. 监听 swoole http server 的 request 等几个事件；
4. 将数据传给 Laravel 的 `request`，并将 Laravel 的 `response` 传出来。

### Docker 结合

自从用了 docker，什么东西都想 docker 化。项目也一直运行在 docker 环境下好几年了。所以必须把 swoole 在 docker 里跑起来。

定制一个 swoole 的 docker 容器倒也简单，[Dockerfile](https://github.com/ruogoo/docker-env/blob/develop/swoole/Dockerfile)
把该装的装上。

docker-compose.yml 里多一条 command:

```dockerfile
  swoole:
    build: ./swoole
    # ...
    command: php /docker/app/someproject/artisan swoole:http start
```

稍麻烦的是 Nginx 的配置，不再是 fastcgi 的方式，而是反向代理到 swoole 的 http server，简单配置示例如下：

```
# laravel swoole config example.
server {
    listen 80;
    server_name swoole.app;

    access_log  /var/log/nginx/host.access.log  main;

    root  /docker/app/someproject/public;
    index index.php;

    location / {
        try_files $uri $uri/ @swoole;
    }

    # 由于 http server 支持不完善，需要传递 header
    proxy_set_header   HOST $host;
    proxy_set_header   SERVER_PORT $server_port;
    proxy_set_header   REMOTE_ADDR $remote_addr;
    proxy_set_header   X-Real-IP $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;

    # 这里注意：如果是无路径的 URL，需要加 / 
    location = /index.php {
        proxy_pass http://swoole:1215/;
    }

    location @swoole {
        proxy_pass http://swoole:1215;
    }

    location ~ /\.ht {
        deny    all;
    }
}
```

### 坑

1. Docker 端口容易混乱

    这个坑算自己的失误，要管理的容器太多了，端口映射就乱了。项目里配置的端口，Dockerfile 里开放的端口，docker-compose 里映射的端口，以及 Nginx 里反向代理的端口，都要对应得上（不需要一样），很容易混乱，导致调了很久还是 502。

2. Nginx 配置

    Nginx 的配置需要揣摩每一项的意思，并思考还缺少什么，分析请求到达 Nginx 之后是如何转给 swoole 的。很多次访问静态文件是可以，但访问 php 页面就 502 了。这里调了很久很久。

3. require_once 大坑货

    Nginx 配置好了，不再 502 了，却一直是 404 了。开始找了很久都没找到原因，一直以为是缺少了什么，导致请求没有找到目标。但是后来一想，Laravel 的 Application 也启动了，那 404 肯定就是 Laravel 内部抛出来的。然后就一点点 log 调试（好吧，当时好像不知道为什么不能看错误栈），终于发现是 route 丢了。

    什么，丢了？

    对，就是本来所有的 route 都加载进去了，但是又丢了。就是在 swoole onRequest 的时候，会 reset 并重新加载一些 ServiceProvider (待考究)。

    PHP 对于引入代码文件一般用 `require_once` 避免重复 require，比如我的 RouteServiceProvider 中需要 require 路由文件：

    ```php
    class RouteServiceProvider extends ServiceProvider
    {
        /**
         * `map()` called by parent's `loadRoutes()`.
         */
        public function map()
        {
            require_once base_path('route/web.php');
            require_once base_path('route/passport.php');
            require_once base_path('route/api_v3.php');
        }
    }
    ```

    就在这里，路由文件是 `require_once` 的，加载一次后就不再被加载了，swoole 启动后实际的路由被清空了，改成 `require` 就好了。。。
