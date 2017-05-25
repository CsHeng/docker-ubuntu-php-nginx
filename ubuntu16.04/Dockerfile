#FROM ubuntu:16.04
FROM ubuntu:xenial

# 更换apt源
RUN sed -i 's/http:\/\/archive\.ubuntu\.com\/ubuntu\//http:\/\/cn\.archive\.ubuntu\.com\/ubuntu\//g' /etc/apt/sources.list \
    && apt-get update \
    && apt-get install -y nginx php php-fpm php-curl php-redis php-mongodb php-gd php-mcrypt supervisor \
    && rm /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && dpkg-reconfigure -f noninteractive tzdata \
    # remove caches
    && rm -rf /var/lib/apt/lists/* \
    # add supervisord conf before CMD
    && mkdir -p /etc/supervisor/conf.d \
    && echo '[supervisord] \nnodaemon=true' > /etc/supervisor/conf.d/supervisord.conf \
    && echo '[program:nginx] \ncommand=/usr/sbin/nginx \nautostart=true \nautorestart=true \npriority=10' > /etc/supervisor/conf.d/nginx.conf \
    && echo '[program:php-fpm] \ncommand=/usr/sbin/php-fpm7.0 \nautostart=true \nautorestart=true \npriority=5' > /etc/supervisor/conf.d/php-fpm.conf

CMD ["/usr/bin/supervisord", "-n", "-c",  "/etc/supervisor/supervisord.conf"]
