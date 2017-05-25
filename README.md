# 打造基于Docker的Ubuntu 14.04 + PHP-FPM + Nginx运行环境

### 准备工作
- 安装[docker][1]、docker-compose
- 更改docker[镜像源地址][2]，加速访问

### 创建Dockerfile

1. 在用户目录下创建docker项目文件夹 
```
$ mkdir -p ~/docker && cd ~/docker
```
2. 创建Dockerfile
```
$ vim Dockerfile
```
3. 由于我们是基于Ubuntu14.04做的镜像，所以定义好官方Ubuntu地址即可，在Dockerfile插入:`FROM ubuntu:trusty`，保存Dockerfile，然后运行`docker build -t ubuntu:14.04-php-nginx .`，如果你是第一次运行，需要从上面修改的docker源拉取ubuntu的镜像，会比较久。

![](/images/14956931611685.jpg)

4. 我们用`docker images`来看结果，会有两个镜像，一个是拉下来的`ubuntu:trusty`，一个是我们定制的镜像: `ubuntu:14.04-php-nginx`

![](/images/14956932635561.jpg)

5. 现在我们已经拥有一个本地image，使用**交互模式**启动容器准备进行定制 `docker run -t -i ubuntu:14.04-php-nginx /bin/bash`

![](/images/14956933269280.jpg)

6. 把它当成一台装了Ubuntu的虚拟机，该怎么样怎么样，我们准备安装PHP-FPM和Nginx及一些PHP扩展，那么正常命令是这样的：(容器以root用户登录，且默认没有sudo命令，则不需要sudo)
```
apt-get update 
apt-get install -y nginx nginx php5 php5-fpm php5-mongo php5-redis php5-gd php5-curl php5-mcrypt imagemagick php5-imagick supervisor
```
漫长的等待时间之后，安装完了
7. 由于默认的系统是UTC时间，我们将它改为CST
```
rm /etc/localtime
echo "Asia/Shanghai" > /etc/timezone
dpkg-reconfigure -f noninteractive tzdata
```
成功之后会输出当前的`local time`

![](/images/14956933774657.jpg)

8. 接下来我们不需要再安装其他软件了，那么可以删掉没有必要的apt缓存：`rm -rf /var/lib/apt/list/*`
9. 由于docker是无状态的，且启动时只能保留一条`RUN`来执行我们要启动的命令，但是我们需要FPM和Nginx都一起启动，怎么办？于是我们通过`supervisor`来控制，那么我们需要配置`supervisor`：
supervisord配置：/etc/supervisor/conf.d/supervisord.conf 
```
[supervisord]
nodaemon=true 
```
nginx配置：/etc/supervisor/conf.d/nginx.conf 
```
[program:nginx] 
command=/usr/sbin/nginx 
autostart=true 
autorestart=true 
priority=10'
```
fpm配置：/etc/supervisor/conf.d/php-fpm.conf
```
[program:php-fpm] 
command=/usr/sbin/php5-fpm 
autostart=true 
autorestart=true 
priority=5'
```
10. 好了，需要的东西都安装配置了，接下来就剩下启动`supervisor`了，我们把这个命令放在Dockerfile来启动`CMD ["/usr/bin/supervisord", "-n", "-c",  "/etc/supervisor/supervisord.conf"]`

### 完整的文件是这样子的
```
#FROM ubuntu:14.04.05
FROM ubuntu:trusty

# 更换apt源
RUN sed -i 's/http:\/\/archive\.ubuntu\.com\/ubuntu\//http:\/\/cn\.archive\.ubuntu\.com\/ubuntu\//g' /etc/apt/sources.list \
    && apt-get update \
    && apt-get install -y nginx php5 php5-fpm php5-mongo php5-redis php5-gd php5-curl php5-mcrypt imagemagick php5-imagick supervisor \
    && rm /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && dpkg-reconfigure -f noninteractive tzdata \
    # remove caches
    && rm -rf /var/lib/apt/lists/* \
    # add supervisord conf before CMD
    && mkdir -p /etc/supervisor/conf.d \
    && echo '[supervisord] \nnodaemon=true' > /etc/supervisor/conf.d/supervisord.conf \
    && echo '[program:nginx] \ncommand=/usr/sbin/nginx \nautostart=true \nautorestart=true \npriority=10' > /etc/supervisor/conf.d/nginx.conf \
    && echo '[program:php-fpm] \ncommand=/usr/sbin/php5-fpm \nautostart=true \nautorestart=true \npriority=5' > /etc/supervisor/conf.d/php-fpm.conf

CMD ["/usr/bin/supervisord", "-n", "-c",  "/etc/supervisor/supervisord.conf"]

```

### 使用方法
- git pull 本工程
- cd 进入`docker-compose.yml`文件所在目录（如ubuntu14.04）
- `docker-compose up -d`，具体用法命令行输入`docker-compose --help`查看

### docker-compose修改
- 修改ports端口映射
- 修改目录映射
    - 修改www目录，将自己本机的目录共享给docker镜像
    - 修改log目录，以便不需要ssh进docker镜像也可以查看log
    > nginx/php-fpm/supervisor master process user为root，因此可以直接写入/var/log目录
    > php-fpm的worker process user为www-data(默认)，因此将worker的error output设置成(php_errors = /tmp/php_error.log)以写入/tmp
    > 不建议修改系统默认文件夹权限设置，特别是将一些关键目录设置成777或者使用root权限运行程序
- 修改`docker-compose.yml`文件完毕之后请运行`docker-compose down && docker-compose up -d`重新启动建立映射关系

### docker交互模式
- 通过`docker-compose up -d`进入后台进程，可以使用`docker exec -i -t $containerId bash`进入交互模式
- 通过`docker ps`查看所有运行的docker情况
- 通过`docker-compose ps -q`查看当前镜像运行的containerId
- 直接使用`docker exec -it $(docker-compose ps -q) bash`进入交互模式
- 建议添加function到shell以便快速进入`sshdocker() { docker exec -it $(docker-compose ps -q) bash}`(alias在shell载入时会计算出结果，所以不能用alias)

### 其他参考
- 为什么只用一条RUN命令?

> 每一次RUN都会导致镜像被包装一层，从而有不必要的额外信息被记录，更多详细信息参考：https://yeasy.gitbooks.io/docker_practice/content/image/build.html

- 有什么要注意的？

> supervisor启动的进程不要以daemon形式启动，所以在`php-fpm.conf`和`nginx.conf`都将daemon关闭了，具体详细参数请参考示例工程

- 示例工程：

> 请参考[docker-ubuntu-php-nginx][3]，包含ubuntu14.04和ubuntu16.04的示例

- 更多docker的使用技巧请参考[官网][1]和[git book][4]

[1]: https://www.docker.com/
[2]: https://c.163.com/wiki/index.php?title=DockerHub%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F
[3]: https://yeasy.gitbooks.io/docker_practice/content/introduction/
[4]: https://yeasy.gitbooks.io/docker_practice/content/introduction/

