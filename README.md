yii2 swoole
===============================

让yii2运行在swoole上。


性能
----------------
运行在swoole上的yii2是运行在php-fpm上yii2的5倍以上，而且一句代码也不用修改。
 
 
在线demo性能体验
----------------
各个演示站点后台   **用户名:feehicms 密码123456**
  * php7.1.8 (php-fpm+nginx+yii2)
    * 前台[http://demo.cms.qq.feehi.com/admin](http://demo.cms.qq.feehi.com/admin)
    * 后台1[http://demo.cms.qq.feehi.com/admin](http://demo.cms.qq.feehi.com/admin)
  * php7.1.8 (swoole+nginx+yii2)
    * 前台[http:/swoole.demo.cms.qq.feehi.com/](http://swoole.demo.cms.qq.feehi.com)
    * 后台[http://swoole-admin.demo.cms.qq.feehi.com/](http://swoole-admin.demo.cms.qq.feehi.com)

以上demo均采取同一[docker镜像](https://www.github.com/liufee/docker)部署，docker容器运行在同一服务器上，分配相同的资源。

这里用作比较的demo是采用yii2框架开发的一款cms系统[FeehiCMS](http://www.github.com/liufee/cms)，因为FeehiCMS只开发基础cms功能，未对yii框架做任何封装、改造，故选择此作为体验demo。
 
安装
---------------
1. 使用composer
     composer的安装以及国内镜像设置请点击[此处](http://www.phpcomposer.com/)
     
     ```bash
     $ cd /path/to/yii2-app
     $ composer require "feehi/yii2-swoole"
     $ composer install -vvv
     ```
 

配置yii2
-------------
打开console/config/main.php，在顶层配置中加入如下配置。（注意：并不是配置在components里面，而应该在最外层，即与components同级）。[完整示例](https://github.com/liufee/cms/blob/master/console/config/main.php)

```bash
 'id' => 'app-console',
 ...//其他配置
'controllerMap'=>[
     ...//其他配置项
     'swoole' => [
            'class' => feehi\console\SwooleController::className(),
            'rootDir' => str_replace('console/config', '', __DIR__ ),//yii2项目根路径
            'app' => 'frontend',//app目录地址
            'host' => '127.0.0.1',//监听地址
            'port' => 9999,//监听端口
            'swooleConfig' => [//标准的swoole配置项都可以再此加入
                'reactor_num' => 2,
                'worker_num' => 4,
                'daemonize' => false,
                'log_file' => __DIR__ . '/../../frontend/runtime/logs/swoole.log',
                'log_level' => 0,
                'pid_file' => __DIR__ . '/../../frontend/runtime/server.pid',
            ],
    ],
    'swoole-backend' => [
            'class' => feehi\console\SwooleController::className(),
            'rootDir' => str_replace('console/config', '', __DIR__ ),//yii2项目根路径
            'app' => 'backend',
            'host' => '127.0.0.1',
            'port' => 9998,
            'swooleConfig' => [
            'reactor_num' => 2,
            'worker_num' => 4,
            'daemonize' => false,
            'log_file' => __DIR__ . '/../../backend/runtime/logs/swoole.log',
            'log_level' => 0,
            'pid_file' => __DIR__ . '/../../backend/runtime/server.pid',
        ],
    ]
    ...//其他配置
 ]
 ...//其他配置
```


启动命令
-------------
前台

    * 启动 /path/to/php /path/to/yii swoole/start
    * 关闭 /path/to/php /path/to/yii swoole/stop
    * 重启 /path/to/php /path/to/yii swoole/restart

后台

    * 启动 /path/to/php /path/to/yii swoole-backend/start
    * 关闭 /path/to/php /path/to/yii swoole-backend/stop
    * 重启 /path/to/php /path/to/yii swoole-backend/restart

Nginx配置
-------------
虽然swoole从1.9.17版本以后底层支持作为静态资源web服务器，但毕竟没有完全实现http协议，强烈推荐配合nginx使用，把swoole仅作为应用服务器。

```bash
 *
 * 前台
 *
 server {
    set $web /www/cms-swoole/frontend/web;
    root $web;
    server_name swoole.cms.test.docker;

    location ~* .(ico|gif|bmp|jpg|jpeg|png|swf|js|css|mp3) {
    root  $web;
    }

    location ~ timthumb\.php$ {//若部分功能仍需要使用php-fpm则做类似配置，否则删除此段
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }

    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "keep-alive";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host http://swoole.cms.test.docker;
        proxy_pass http://127.0.0.1:9999;
    }
 }
 
 *
 * 后台
 *
 server {
    set $web /www/cms-swoole/backend/web;
    root $web;
    server_name swoole-admin.cms.test.docker;

    location ~* .(ico|gif|bmp|jpg|jpeg|png|swf|js|css|mp3) {
        root  $web;
    }

    location / {
        proxy_http_version 1.1;
        proxy_set_header Connection "keep-alive";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host http://swoole-admin.cms.test.docker;
        proxy_pass http://127.0.0.1:9998;
    }
 }
```

其他
-------------
以上是把swoole启动/关闭/重启命令集成到了yii2 console里面，如果你并不想使用集成到yii2 console的swoole，可以复制vendor/feehi/yii2-swoole下的backend.php和frontend.php，修改$rootDir = "/path/to/project"为真正的yii2项目根目录,然后执行php backend.php以及php frontend.php启动swoole