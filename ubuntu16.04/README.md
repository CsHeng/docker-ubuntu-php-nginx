# ubuntu 16.04/xenial 基础镜像定制

- 修改apt源
- 修改时区为CST
- 安装nginx php-fpm supervisor
- 安装PHP mongo/redis等库，具体请看`Dockerfile`

### build本地镜像
- `docker build -t ubuntu:16.04-php-nginx .`

### push到仓库
- login `docker login --username=[用户名] [仓库地址]`(本地登录Docker registry，后续的push、pull操作需要)
- tag `docker tag ubuntu:16.04-php-nginx [仓库地址]/ubuntu:16.04-php-nginx`
- push `docker push [仓库地址]/ubuntu:16.04-php-nginx`

### 使用本镜像的Dockerfile
- `FROM [仓库地址]/ubuntu:16.04-php-nginx`
- 镜像默认已启动命令：`CMD ["/usr/bin/supervisord", "-n", "-c",  "/etc/supervisor/supervisord.conf"]`