# Docker最佳实践之Dockerfile

上一篇讲了镜像构建，这一篇讲Dockerfile，这是构建的核心。最近写了好几个镜像的Dockerfile，主要都托管在github上（这没什么可以值得隐瞒的），重要的如下：

- 前端开发镜像：https://github.com/IBBD/dockerfile-node-dev 
- php-fpm镜像：https://github.com/IBBD/dockerfile-php-fpm 
- nginx镜像：https://github.com/IBBD/dockerfile-nginx 
- mariadb镜像：https://github.com/IBBD/dockerfile-mariadb 
- redis镜像：https://github.com/IBBD/dockerfile-redis
- mongoDB镜像：https://github.com/IBBD/dockerfile-mongo

下面已php-fpm的dockerfile为例，先看dockerfile文件：

```sh 
FROM php:5.6-fpm

MAINTAINER Alex Cai "cyy0523xc@gmail.com"

# sources.list
# git clone git@github.com:IBBD/docker-compose.git
ADD ext/sources.list   /etc/apt/sources.list

# Install modules
RUN apt-get update \
    && apt-get install -y \
        libmcrypt-dev \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libpng12-dev \
        libssl-dev \
    && rm -r /var/lib/apt/lists/*

# install php modules
# composer需要先安装zip
RUN  docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install gd \
    && docker-php-ext-install iconv mcrypt pdo pdo_mysql tokenizer mbstring zip

# PHP config
#ADD conf/php.ini        /usr/local/etc/php/php.ini
#ADD conf/php-fpm.conf   /usr/local/etc/php-fpm.conf

# pecl install php modules
RUN  mkdir /home/php
#COPY ext/redis.tgz    /home/php/redis.tgz 
#COPY ext/mongo.tgz    /home/php/mongo.tgz 
COPY ext/msgpack.tgz  /home/php/msgpack.tgz 
#COPY ext/memcache.tgz /home/php/memcache.tgz 

# 安装php扩展
RUN cd /home/php \
    && pecl install redis \
    && echo "extension=redis.so" > /usr/local/etc/php/conf.d/redis.ini \
    && pecl install memcache \
    && echo "extension=memcache.so" > /usr/local/etc/php/conf.d/memcache.ini \
    && pecl install msgpack.tgz \
    && echo "extension=msgpack.so" > /usr/local/etc/php/conf.d/msgpack.ini \
    && pecl install mongo \
    && echo "extension=mongo.so" > /usr/local/etc/php/conf.d/mongo.ini \
    && pecl install swoole \
    && echo "extension=swoole.so" > /usr/local/etc/php/conf.d/swoole.ini \
    && pecl install xdebug \
    && echo "zend_extension=xdebug.so" > /usr/local/etc/php/conf.d/xdebug.ini 

# composer 
# composer中国镜像
# phpunit
COPY ext/composer.php /home/php/composer.php
COPY ext/phpunit.phar /home/php/phpunit.phar
RUN php /home/php/composer.php \
    && mv composer.phar /usr/local/bin/composer \
    && chmod 755 /usr/local/bin/composer \
    && composer config -g repositories.packagist composer http://packagist.phpcomposer.com \
    && chmod +x /home/php/phpunit.phar \
    && mv /home/php/phpunit.phar /usr/local/bin/phpunit \
    && phpunit --version \
    && rm -rf /home/php \


WORKDIR /var/www 

# 解决时区问题
env TZ "Asia/Shanghai"

# Define mountable directories.
VOLUME /var/www

EXPOSE 9000
```

这个镜像主要是为了满足laravel5而开发的，安装了必须的扩展如mcrypt,pdo等，也安装了我们平时常用的扩展，如redis，mongoDB，swoole等，还安装了一些必要的工具，如composer，phpunit等。

- 基础版本选择 

我们选择的5.6，而非7，毕竟php7还在测试中。

- 启动服务的配置文件不要在docker中指定，应该在运行的时候指定

运行的时候可能会使用不同的配置，运行的时候指定会更灵活。

- 一个dockerfile不要执行太多的操作

执行太多的操作，如果中间出现错误，就得重新build, 这样可能会导致花费很多时间在构建上。可以合理的拆分子镜像。

- docker镜像是采用分层的结构，不要太多的层次

每个run语句都会层，后面的层构建在前面的层的基础上，构建的时候，系统会智能的使用缓存，如果对应的构建语句没有变化的话。改变一个run语句，会使后面的缓存都失效。层应该合理的内聚。

上面这个dockerfile有优化的空间




---------

Date: 2015-10-11  Author: alex cai <cyy0523xc@gmail.com>
