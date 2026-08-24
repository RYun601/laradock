# Laradock PHP 5.6 多版本运行时设计

## 目标

在现有 PHP 8.3 与 PHP 7.4 服务之外，增加 PHP 5.6 的 CLI 与 PHP-FPM 运行时。三个版本必须可同时启动，并可由 Nginx 按站点选择 PHP-FPM 服务。

## 范围

本次只增加与现有 PHP 7.4 服务对等的 PHP 5.6 服务和环境变量：

- `workspace-php-56`：提供 PHP 5.6 CLI、Composer 和开发命令。
- `php-fpm-56`：监听容器网络内的 `9000` 端口，处理 PHP 5.6 的 Web 请求。
- `.env`：提供 `PHP56_VERSION=5.6`，并为 PHP 5.6 Workspace 分配不冲突的宿主机端口。
- `nginx/sites/php56.conf`：提供一个可直接加载的 PHP 5.6 Laravel 风格虚拟主机。

不修改默认 `workspace`、`php-fpm` 或 Nginx 的全局 upstream。因此现有 PHP 8.3 的默认站点和 PHP 7.4 服务均保持原行为。

## 服务设计

`workspace-php-56` 复制 `workspace-php-74` 的构建参数、代码挂载、网络、环境变量和 Docker-in-Docker 连接，仅将 PHP 版本变量替换为 `${PHP56_VERSION}`。其宿主机端口采用现有 7.4 编号的下一组：

| 用途 | 环境变量 | 宿主机端口 |
| --- | --- | --- |
| SSH | `WORKSPACE_56_SSH_PORT` | `2226` |
| BrowserSync | `WORKSPACE_56_BROWSERSYNC_HOST_PORT` | `3004` |
| BrowserSync UI | `WORKSPACE_56_BROWSERSYNC_UI_HOST_PORT` | `3005` |
| Vue 开发服务 | `WORKSPACE_56_VUE_CLI_SERVE_HOST_PORT` | `8084` |
| Vue UI | `WORKSPACE_56_VUE_CLI_UI_HOST_PORT` | `8004` |
| Angular 开发服务 | `WORKSPACE_56_ANGULAR_CLI_SERVE_HOST_PORT` | `4204` |
| Vite | `WORKSPACE_56_VITE_PORT` | `5177` |

`php-fpm-56` 复制 `php-fpm-74` 的构建参数、代码挂载、网络和环境变量，仅替换版本变量为 `${PHP56_VERSION}`，并挂载已有的 `./php-fpm/php5.6.ini` 到容器的 `/usr/local/etc/php/php.ini`。服务仅通过 Docker backend 网络暴露端口 `9000`，不新增宿主机端口映射。

## Nginx 路由

Nginx 当前构建时生成的 `php-upstream` 仍指向 `.env` 中的 `NGINX_PHP_UPSTREAM_CONTAINER=php-fpm`，即默认 PHP 8.3 服务。不得把该变量改为 `php-fpm-56`，否则全部使用 `php-upstream` 的站点都会切换到 PHP 5.6。

新增的 `nginx/sites/php56.conf` 以现有 Laravel 站点格式为基线，使用 `server_name php56.test` 与 `root /var/www/php56/public`。其 PHP location 直接指定 PHP 5.6 服务：

```nginx
location ~ \.php$ {
    fastcgi_pass php-fpm-56:9000;
}
```

该配置由 `NGINX_SITES_PATH=./nginx/sites/` 作为运行时 bind mount 挂载，并以 `.conf` 后缀被 Nginx 自动加载。其应用目录必须位于 `${APP_CODE_PATH_HOST}/php56`，使容器内路径为 `/var/www/php56/public`。PHP 7.4 站点同理使用 `fastcgi_pass php-fpm-74:9000`；默认站点继续使用 `fastcgi_pass php-upstream`。这使每个站点独立选择解释器，且不会要求创建额外 Nginx 容器或占用新的 HTTP/HTTPS 端口。

## 启动与验证

实施后使用以下方式验证：

1. 使用 `docker compose -p laradock-php56 config -q` 解析配置，且 `docker compose -p laradock-php56 config --services` 包含 `workspace-php-56` 与 `php-fpm-56`。
2. 构建并启动 `workspace-php-56` 和 `php-fpm-56` 后，以 `php -r 'if (PHP_VERSION_ID < 50600 || PHP_VERSION_ID >= 50700) { fwrite(STDERR, PHP_VERSION . PHP_EOL); exit(1); } echo PHP_VERSION, PHP_EOL;'` 分别检查两个容器，确认版本保持 `>= 5.6.0` 且 `< 5.7.0`。
3. Nginx 验证只构建 `nginx`。`nginx/sites` 是运行时 bind mount，因此启动时会加载 `php56.conf`，无需构建默认 `php-fpm` 镜像。为使默认 `php-upstream` 同时能够解析，启动一个隔离 backend 网络中的临时 DNS 提供者：`docker run -d --rm --name laradock-php56-nginx-default-fpm --network laradock-php56_backend --network-alias php-fpm laradock-php-fpm:latest`。然后以临时端口 `8088`、`8443`、`8188` 执行 `docker compose -p laradock-php56 up -d nginx --no-deps` 和 `docker compose -p laradock-php56 exec -T nginx nginx -t`，确认默认上游和 `php-fpm-56:9000` 都可被 Nginx 加载。

验证在隔离工作树中每条运行时 Docker Compose 命令必须显式使用 `-p laradock-php56`，因为 `.env` 的常规项目名为 `laradock`，而该名称正在被当前开发环境使用。清理顺序必须先执行 `docker compose -p laradock-php56 down --remove-orphans`，再停止仍存在的 `laradock-php56-nginx-default-fpm` 辅助容器；若 `docker network inspect laradock-php56_backend --format '{{json .Containers}}'` 返回 `{}`，再执行 `docker network rm laradock-php56_backend`。最后清除三个临时端口环境变量。该流程不删除镜像或卷，也不影响 `laradock-*` 主项目容器。

## 兼容性与约束

PHP 5.6 已停止维护，仅用于兼容遗留应用。构建参数沿用现有 PHP 7.4 服务，实际启用的扩展必须与 PHP 5.6 兼容；若某个已开启的扩展不支持 PHP 5.6，构建将明确失败并需要在后续单独调整该扩展开关。本次不改变全局扩展配置，避免影响现有 8.3 和 7.4 服务。
