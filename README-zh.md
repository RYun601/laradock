# Laradock

Laradock 能够帮你在 **Docker** 上快速搭建 **Laravel** 应用（也适用于其他 PHP 项目）。

## 目录
- [依赖](#依赖)
- [安装](#安装)
- [使用](#使用)
- [容器管理](#容器管理)
- [PHP 配置](#php-配置)
- [Laravel 集成](#laravel-集成)
- [常见问题](#常见问题)

<a name="依赖"></a>
## 依赖

- [Git](https://git-scm.com/downloads)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)

<a name="安装"></a>
## 安装

1 - 克隆 `Laradock` 仓库到你的项目目录：

```bash
git clone https://github.com/laradock/laradock.git
```

2 - 进入 Laradock 目录，复制配置文件：

```bash
cd laradock
cp .env.example .env
```

> **国内网络优化：** 如果你在中国内地，修改 `.env` 中的以下配置来加速构建：
> ```env
> CHANGE_SOURCE=true
> WORKSPACE_COMPOSER_REPO_PACKAGIST=https://mirrors.aliyun.com/composer/
> WORKSPACE_NVM_NODEJS_ORG_MIRROR=https://npmmirror.com/mirrors/node
> WORKSPACE_NPM_REGISTRY=https://registry.npmmirror.com
> ```
> 同时建议配置 [DockerHub 镜像加速](https://www.runoob.com/docker/docker-mirror-acceleration.html)。

<a name="使用"></a>
## 使用

1 - 运行容器（在 `laradock` 目录中执行）：

```bash
docker-compose up -d nginx mysql redis
```

可用的容器：`nginx`, `php-fpm`, `mysql`, `redis`, `postgres`, `mariadb`, `mongo`, `apache2`, `caddy`, `memcached`, `beanstalkd`, `workspace`

> `workspace` 和 `php-fpm` 会在大部分实例中自动运行。

2 - 进入 Workspace 容器执行命令（Artisan, Composer, PHPUnit 等）：

```bash
docker-compose exec workspace bash
```

> 使用 `--user=laradock` 可以以主机用户身份创建文件。

3 - 配置 Laravel 的 `.env` 文件：

```env
DB_HOST=mysql
REDIS_HOST=redis
```

4 - 打开浏览器访问 `http://localhost`。

<a name="容器管理"></a>
## 容器管理

### 查看运行中的容器
```bash
docker ps
docker-compose ps
```

### 停止容器
```bash
docker-compose stop              # 停止所有
docker-compose stop {容器名称}    # 停止指定容器
```

### 删除容器
```bash
docker-compose down
```

### 进入容器
```bash
docker-compose exec {container-name} bash
```

### 重建容器
修改 `Dockerfile` 后重新构建：

```bash
docker-compose build                   # 重建所有
docker-compose build {container-name}  # 重建指定容器
docker-compose build --no-cache {container-name}  # 无缓存重建
```

### 查看日志
```bash
docker logs {container-name}
```
Nginx 日志在 `logs/nginx` 目录。

<a name="php-配置"></a>
## PHP 配置

### 切换 PHP 版本

打开 `docker-compose.yml`，修改 `php-fpm` 的 `dockerfile` 字段：

```yaml
php-fpm:
    build:
        context: ./php-fpm
        dockerfile: Dockerfile-74    # 改为目标版本的 Dockerfile
```

然后重建容器：

```bash
docker-compose build php-fpm
```

### 安装 PHP 扩展

- PHP-FPM 扩展在 `php-fpm/Dockerfile-XX` 中配置
- PHP-CLI 扩展在 `workspace/Dockerfile` 中配置

### 安装 Xdebug

在 `docker-compose.yml` 中为 `workspace` 和 `php-fpm` 设置 `INSTALL_XDEBUG=true`，然后重建容器：

```bash
docker-compose build workspace php-fpm
```

<a name="laravel-集成"></a>
## Laravel 集成

### 安装 Laravel

```bash
# 进入 workspace 容器后
composer create-project laravel/laravel my-cool-app
```

### 运行 Artisan 命令

进入 workspace 容器后执行：

```bash
php artisan
composer update
phpunit
```

### 使用 Redis

`.env` 配置：
```env
REDIS_HOST=redis
CACHE_DRIVER=redis
SESSION_DRIVER=redis
```

安装依赖：
```bash
composer require predis/predis:^1.0
```

### 使用 Mongo

1. 在 `docker-compose.yml` 中为 `workspace` 和 `php-fpm` 设置 `INSTALL_MONGO=true`
2. 重建容器：`docker-compose build workspace php-fpm`
3. 启动 Mongo：`docker-compose up -d mongo`
4. 配置 `config/database.php` 添加 MongoDB 连接
5. 安装 `jenssegers/mongodb` 包

<a name="常见问题"></a>
## 常见问题

### 访问报错：Failed to connect to xxx port 80

在 Docker 环境中，PHP 容器通过域名访问其他容器时可能解析失败，报错：

```
Failed to connect to api.your-project.test port 80 after 3 ms: Couldn't connect to server
```

**解决方案 1（推荐）：在 docker-compose.yml 中为 Nginx 容器配置网络别名**

```yaml
### NGINX Server #########################################
    nginx:
        # ... 其他配置保持不变 ...
        networks:
            frontend:
            backend:
                aliases:
                  - api.your-project.test
                  - admin.your-project.test
        extra_hosts:
          - "host.docker.internal:host-gateway"
```

然后重建nginx:

```bash
docker-compose up -d nginx
```

这样所有连接到 backend 网络的容器都可以通过域名访问 Nginx。

**解决方案 2（临时）：手动修改 php-fpm 容器的 hosts 文件**

```bash
# 1. 查看 nginx 容器的 IP 地址
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' nginx

# 2. 进入 php-fpm 容器
docker-compose exec php-fpm bash

# 3. 编辑 /etc/hosts，添加（IP 替换为上一步查到的地址）：
# 172.19.0.1 api.your-project.test
# 172.19.0.1 admin.your-project.test
```

> 注意：方案 2 在容器重启后会失效，推荐使用方案 1。

### 自定义域名

在 `/etc/hosts` 中添加：
```
127.0.0.1    laravel.test
```

### 空白页问题

在 Laravel 根目录执行：
```bash
sudo chmod -R 777 storage bootstrap/cache
```

### 端口冲突

确保 80、3306 等端口没有被其他程序（Apache、MySQL 等）占用。

### 安装 Node + NVM

在 `docker-compose.yml` 中设置 `INSTALL_NODE=true`，然后重建 `workspace` 容器。

---

## 帮助 & 问题

- [Gitter 社区](https://gitter.im/Laradock/laradock)
- [GitHub Issues](https://github.com/laradock/laradock/issues)

## 许可证

[MIT License](https://github.com/laradock/laradock/blob/master/LICENSE)
