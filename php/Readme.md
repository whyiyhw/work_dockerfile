### 构建镜像

```shell
docker build -t php81-laravel:v1 .
```

### 构建运行时

```shell
docker run -i -t -d \
 --restart=always --privileged=true \
 -p 8100:9000 \
 -v /pathto/code:/pathto/code  --name=laravel8  php81-laravel:v1
Copy
```

### 内外文件路径不一致的问题

- 容器内一般默认路径为 `/var/www/html` 
- 可以在容器内挂载与外部一致的路径

### 容器内 php.ini 中配置的修改

```shell
# 因为php.ini 默认会去加载 /usr/local/etc/php/conf.d/*.ini 文件
# 所以只需要 新增一个新的配置文件，默认就会将配置覆盖
touch /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini
# 内存限制
echo "memory_limit=256M" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
# 上传限制
echo "upload_max_filesize=50M" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini && \
# post 大小限制...
echo "post_max_size=50M" >> /usr/local/etc/php/conf.d/docker-php-ext-OPcache.ini
```

### 执行者权限问题

- 在容器内 `php-fpm` 的运行时分组 为 `www-data`

```shell
# 容器内
id  www-data 
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

- 所以其生成的 log 文件也是 属于这个分组下
- 在宿主机上 如果没有 33 的分组与用户， 展示的则为 `tape tape`
- 如若在宿主机上使用 crontab 势必会导致 log 权限的问题，
- 解决方式 就是保证内外用户uid 的一致 如:

```shell
RUN  useradd -u 1001 www && RUN sed -i "s/www-data/www/g" /usr/local/etc/php-fpm.d/www.conf
Copy
```

### laravel 相关任务

```shell
# 可以直接在宿主机上 执行 容器内的命令 如
# 将容器内的 composer 进行更新
docker exec composer update -vvv 
#（此时以容器内的 root 用户去执行的命令）

# 所以在容器内不安装 crontab 的情况下，可以在宿主机进行配置
# 以 root 用户去启动 crontab  在容器内转换成 www 用户去执行 php 完美
crontab -e

* * * * *  docker exec laravel8 sudo -u www php artisan schedule:run
```

## 生产环境使用的第二个思路

- 前期将所有使用的 `composer` 包通过 `composer.lock` 都拉下来，与项目的基础框架文件打成一个包，后续更新就是把 工程文件，更新到容器中进行替换。这个模式不错后续可以实践下~