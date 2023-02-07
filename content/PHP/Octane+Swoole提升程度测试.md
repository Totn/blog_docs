```json
{
    "date":"2021.08.03 16:39"，//最少需要
    "tags": ["BLOG"，"PHP", "并发"]，//可以不填，不过最好添加一些tag，后面可以做一些好玩的东西。
    "author": "Letotn"， //文章作者，可以不用填写，现在也没有使用到
}
```


测试`Octane`+`Swoole`提升程度

---

**写在前面**
laravel项目使用`octane`+`swoole`对于性能的提升非常显著（220%+的QPS提升），勉强能与静态语言一较高下。

**测试过程**
硬件：虚拟机VirtualBox, 1核2G， CPU为i5-8400;
系统: Centos7 + 宝塔;
php环境： 启用`opcache`,  `session`启用`memcached`缓存，文件缓存启用`redis`;

部署项目及引入`octane`
```bash
composer create-project laravel/laravel octance.sw
cd ./octance.sw
## 引入octane
composer require laravel/octane
## 安装octane
php artisan octane:install
```
安装这一步使用`RoadRunner`出了很多问题，最后跑起来也没能正常访问，遂放弃使用RR; 

使用`swoole`倒是非常简单，宝塔面板中装好`swoole`扩展，`octane:install`时选`swoole`直接完成。

对`laravel`进行普通的生产环境优化；
```bash
composer install --no-dev

## 设置配置文件，关掉debug
vi .env
APP_ENV=production
APP_DEBUG=false
## :wq保存
## 执行优化
composer dump-autoload -o
php artisan optimize
```

到中件间文件`app/Http/Kernel.php`中注释掉`api`下的`throttle`中间件，以免`ab`压测时报429的错误

测试以下内容：在控制器中返回一个0-100的随机数，控制器代码如下
```php
function random()
{
	$i = mt_rand(0, 100);
	return response()->json([
		'code' => 0,
		'msg' => 'random: ' . str_pad($i, 3, '0', STR_PAD_LEFT)
	]);
}

```
1， 压测`php-fpm`
配置Nginx转发到php-fpm；然后进行`ab`压测，一次压测结果如下：
```bash
 .\ab -n2000 -c8 http://octane.sw/api/random
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking octane.sw (be patient)
Completed 200 requests
Completed 400 requests
Completed 600 requests
Completed 800 requests
Completed 1000 requests
Completed 1200 requests
Completed 1400 requests
Completed 1600 requests
Completed 1800 requests
Completed 2000 requests
Finished 2000 requests


Server Software:        nginx
Server Hostname:        octane.sw
Server Port:            80

Document Path:          /api/random
Document Length:        30 bytes

Concurrency Level:      8
Time taken for tests:   17.194 seconds
Complete requests:      2000
Failed requests:        0
Total transferred:      512000 bytes
HTML transferred:       60000 bytes
Requests per second:    116.32 [#/sec] (mean)
Time per request:       68.777 [ms] (mean)
Time per request:       8.597 [ms] (mean, across all concurrent requests)
Transfer rate:          29.08 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.4      0       5
Processing:     7   68  68.2     53     641
Waiting:        7   68  68.2     53     641
Total:          7   69  68.2     53     641

Percentage of the requests served within a certain time (ms)
  50%     53
  66%     59
  75%     61
  80%     64
  90%     95
  95%    243
  98%    303
  99%    346
 100%    641 (longest request)
```
测试结果可见，普通优化下，php8的`php-fpm`能提供110+的QPS; 

2, 压测`octane`+`swoole`
启动工作进程：
```bash
php /path/octane.sw/artisan octane:start --host="0.0.0.0" --port=8080 --workers=4 --max-requests=10000 --task-workers=10
```
并加入到`Supervisor`进程守护中；
配置nginx，将所有请求都转发到`127.0.0.1:8080`
进行`ab`压测，一次压测结果如下：
```bash
 .\ab -n2000 -c8 http://octane.sw/api/random
This is ApacheBench, Version 2.3 <$Revision: 1879490 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking octane.sw (be patient)
Completed 200 requests
Completed 400 requests
Completed 600 requests
Completed 800 requests
Completed 1000 requests
Completed 1200 requests
Completed 1400 requests
Completed 1600 requests
Completed 1800 requests
Completed 2000 requests
Finished 2000 requests


Server Software:        nginx
Server Hostname:        octane.sw
Server Port:            80

Document Path:          /api/random
Document Length:        30 bytes

Concurrency Level:      8
Time taken for tests:   5.234 seconds
Complete requests:      2000
Failed requests:        0
Total transferred:      592000 bytes
HTML transferred:       60000 bytes
Requests per second:    382.10 [#/sec] (mean)
Time per request:       20.937 [ms] (mean)
Time per request:       2.617 [ms] (mean, across all concurrent requests)
Transfer rate:          110.45 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.7      0      11
Processing:     4   21  12.4     18     118
Waiting:        3   21  12.4     18     118
Total:          4   21  12.5     18     118

Percentage of the requests served within a certain time (ms)
  50%     18
  66%     21
  75%     23
  80%     25
  90%     32
  95%     43
  98%     65
  99%     76
 100%    118 (longest request)
```
 对`octane`+`swoole`的压测的QPS为380+
 
结论：QPS从110+提升到380+，提升幅度达到220%+，提升可以说很大, 一般情况下将代码从`php-fpm`模式重构成`cli`模式所花费的成本可能不会太高, 为了200%+的提升值得也需要好好考虑一下。