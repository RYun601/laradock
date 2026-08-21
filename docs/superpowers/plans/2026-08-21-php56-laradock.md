# Laradock PHP 5.6 多版本运行时 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在现有 PHP 8.3 和 PHP 7.4 服务之外，增加可并行运行的 PHP 5.6 Workspace 与 PHP-FPM 服务。

**Architecture:** `.env` 为 PHP 5.6 定义独立版本变量和宿主机端口。`docker-compose.yml` 完整镜像已有 PHP 7.4 的两个服务，分别命名为 `workspace-php-56` 与 `php-fpm-56`，仅替换服务名、版本变量、Workspace 端口变量和 PHP ini 挂载。Nginx 默认 upstream 继续指向 PHP 8.3；遗留站点在自身 FastCGI location 直连 `php-fpm-56:9000`。

**Tech Stack:** Docker Compose、Laradock `workspace`/`php-fpm` 镜像、Nginx FastCGI。

## Global Constraints

- `PHP56_VERSION` 的精确值为 `5.6`。
- 服务名称必须是 `workspace-php-56` 和 `php-fpm-56`。
- 5.6 Workspace 端口必须依次使用 `2226`、`3004`、`3005`、`8084`、`8004`、`4204`、`5177`。
- `php-fpm-56` 必须挂载已有的 `./php-fpm/php5.6.ini`。
- `nginx/sites/php56.conf` 必须以 `server_name php56.test`、`root /var/www/php56/public` 和 `fastcgi_pass php-fpm-56:9000` 定义 PHP 5.6 虚拟主机。
- 不得修改 `PHP_VERSION=8.3`、`PHP74_VERSION=7.4`、`NGINX_PHP_UPSTREAM_CONTAINER=php-fpm`，或默认 Nginx 站点的 `fastcgi_pass php-upstream`。
- `docker-compose.yml` 当前有用户未提交的 Nginx 域名别名修改；暂存和提交时只能选择本计划新增的 PHP 5.6 hunks。
- PHP 5.6 仅用于遗留应用兼容；构建中发现不兼容扩展时，先报告具体扩展，不修改共用的全局扩展开关。
- 用户已确认 `.env` 使用一次性静态配置契约检查；服务的实际行为以 Task 2 的 Compose 解析和 Task 3 的运行时验证为准。
- 隔离工作树的每条运行时 Docker Compose 命令必须显式使用 `-p laradock-php56`，不得使用 `.env` 中会指向用户现有容器的 `COMPOSE_PROJECT_NAME=laradock`；验证后必须执行 `docker compose -p laradock-php56 down --remove-orphans`，并清理为验证创建的辅助容器和空网络。

---

## File Structure

- Modify: `.env`
  - 保存 PHP 5.6 的版本选择和 `workspace-php-56` 的宿主机端口映射值。
- Modify: `docker-compose.yml`
  - 声明 `workspace-php-56` 和 `php-fpm-56`，复用 7.4 服务的完整构建参数与运行时连接。
- Create: `nginx/sites/php56.conf`
  - 定义可由 Nginx 自动加载的 PHP 5.6 Laravel 风格虚拟主机。
- Reuse without modification: `php-fpm/php5.6.ini`
  - 作为 `php-fpm-56` 的运行时 php.ini；该文件已存在。
- Reuse without modification: `php-fpm/Dockerfile`
  - 已包含 PHP 5.6 的镜像兼容逻辑。

### Task 1: 定义 PHP 5.6 版本与 Workspace 端口

**Files:**
- Modify: `.env:44-52`
- Test: PowerShell 环境变量契约断言（仅执行，不创建测试文件）

**Interfaces:**
- Consumes: `.env` 当前的 `PHP74_VERSION` 和 `WORKSPACE_74_*` 端口变量。
- Produces: `PHP56_VERSION` 与七个 `WORKSPACE_56_*` 变量，供 `docker-compose.yml` 插值。

- [ ] **Step 1: 写出会失败的环境变量契约检查**

```powershell
$required = @(
  'PHP56_VERSION=5.6',
  'WORKSPACE_56_SSH_PORT=2226',
  'WORKSPACE_56_BROWSERSYNC_HOST_PORT=3004',
  'WORKSPACE_56_BROWSERSYNC_UI_HOST_PORT=3005',
  'WORKSPACE_56_VUE_CLI_SERVE_HOST_PORT=8084',
  'WORKSPACE_56_VUE_CLI_UI_HOST_PORT=8004',
  'WORKSPACE_56_ANGULAR_CLI_SERVE_HOST_PORT=4204',
  'WORKSPACE_56_VITE_PORT=5177'
)
$content = Get-Content -Raw '.env'
$missing = $required | Where-Object { $content -notmatch [regex]::Escape($_) }
if ($missing) { throw "Missing .env entries: $($missing -join ', ')" }
```

- [ ] **Step 2: 运行检查并确认它失败**

Run: 在仓库根目录运行 Step 1 的 PowerShell 代码。

Expected: 抛出 `Missing .env entries`，其中包含全部八个 `PHP56_VERSION` / `WORKSPACE_56_*` 条目。

- [ ] **Step 3: 在 PHP 7.4 变量组后加入精确配置**

在 `.env` 的 `PHP74_VERSION=7.4` 与现有 `WORKSPACE_74_*` 端口组之后，加入：

```dotenv
PHP56_VERSION=5.6
WORKSPACE_56_SSH_PORT=2226
WORKSPACE_56_BROWSERSYNC_HOST_PORT=3004
WORKSPACE_56_BROWSERSYNC_UI_HOST_PORT=3005
WORKSPACE_56_VUE_CLI_SERVE_HOST_PORT=8084
WORKSPACE_56_VUE_CLI_UI_HOST_PORT=8004
WORKSPACE_56_ANGULAR_CLI_SERVE_HOST_PORT=4204
WORKSPACE_56_VITE_PORT=5177
```

- [ ] **Step 4: 重跑环境变量契约检查**

Run: 再次执行 Step 1 的 PowerShell 代码。

Expected: 命令成功结束且不输出内容。

- [ ] **Step 5: 提交独立的环境变量变更**

```powershell
git add -- .env
git diff --cached --check
git commit -m "config: add PHP 5.6 workspace variables"
```

Expected: 缓存区仅包含 `.env` 的八行 PHP 5.6 变量，不包含用户已有的 Compose 或 Nginx 修改。

### Task 2: 添加 PHP 5.6 Workspace 与 PHP-FPM 服务

**Files:**
- Modify: `docker-compose.yml:219-379`
- Modify: `docker-compose.yml:502-627`
- Test: Docker Compose 服务声明与配置解析断言（仅执行，不创建测试文件）

**Interfaces:**
- Consumes: Task 1 的 `${PHP56_VERSION}` 与 `WORKSPACE_56_*` 变量；已有 `php-fpm/php5.6.ini`；Docker Compose 的 `backend`、`frontend` 和 `docker-in-docker` 服务。
- Produces: 可由 `docker compose build` 和 `docker compose up` 操作的 `workspace-php-56` 与 `php-fpm-56` 服务；二者分别提供 PHP 5.6 CLI 和网络端口 9000。

- [ ] **Step 1: 写出会失败的 Compose 服务契约检查**

```powershell
$services = @(docker compose config --services)
$required = @('workspace-php-56', 'php-fpm-56')
$missing = $required | Where-Object { $_ -notin $services }
if ($missing) { throw "Missing Compose services: $($missing -join ', ')" }
```

- [ ] **Step 2: 运行检查并确认它失败**

Run: 在仓库根目录运行 Step 1 的 PowerShell 代码。

Expected: 抛出 `Missing Compose services: workspace-php-56, php-fpm-56`。

- [ ] **Step 3: 添加 `workspace-php-56` 完整服务块**

在 `docker-compose.yml` 中 `workspace-php-74` 服务块结束、`### PHP-FPM` 注释之前，插入当前 `workspace-php-74` 的完整副本，并应用以下精确替换：

```text
### WORKSPACE PHP 7.4  ->  ### WORKSPACE PHP 5.6
workspace-php-74      ->  workspace-php-56
${PHP74_VERSION}      ->  ${PHP56_VERSION}
WORKSPACE_74_         ->  WORKSPACE_56_
```

保留其余全部 build args、`volumes`、`extra_hosts`、`tty`、environment、frontend/backend networks 和 `docker-in-docker` link，不增删任何项目或端口映射。生成的服务必须含有：

```yaml
    workspace-php-56:
      restart: always
      build:
        context: ./workspace
        args:
          - LARADOCK_PHP_VERSION=${PHP56_VERSION}
      ports:
        - "${WORKSPACE_56_SSH_PORT}:22"
        - "${WORKSPACE_56_BROWSERSYNC_HOST_PORT}:3000"
        - "${WORKSPACE_56_BROWSERSYNC_UI_HOST_PORT}:3001"
        - "${WORKSPACE_56_VUE_CLI_SERVE_HOST_PORT}:8080"
        - "${WORKSPACE_56_VUE_CLI_UI_HOST_PORT}:8000"
        - "${WORKSPACE_56_ANGULAR_CLI_SERVE_HOST_PORT}:4200"
        - "${WORKSPACE_56_VITE_PORT}:5173"
```

- [ ] **Step 4: 添加 `php-fpm-56` 完整服务块**

在 `php-fpm-74` 服务块结束、`### PHP Worker` 注释之前，插入当前 `php-fpm-74` 的完整副本，并应用以下精确替换：

```text
### PHP-FPM 7.4        ->  ### PHP-FPM 5.6
php-fpm-74             ->  php-fpm-56
${PHP74_VERSION}       ->  ${PHP56_VERSION}
./php-fpm/php${PHP74_VERSION}.ini  ->  ./php-fpm/php${PHP56_VERSION}.ini
```

保留其余全部 build args、应用代码与 Docker 证书 volumes、xdebug.ini 挂载、`expose: "9000"`、backend network、environment 和 `docker-in-docker` link。生成的关键部分必须为：

```yaml
    php-fpm-56:
      restart: always
      build:
        context: ./php-fpm
        args:
          - LARADOCK_PHP_VERSION=${PHP56_VERSION}
      volumes:
        - ./php-fpm/php${PHP56_VERSION}.ini:/usr/local/etc/php/php.ini
      expose:
        - "9000"
      networks:
        - backend
```

- [ ] **Step 5: 运行 Compose 契约、解析与静态引用检查**

```powershell
$services = @(docker compose config --services)
$required = @('workspace-php-56', 'php-fpm-56')
$missing = $required | Where-Object { $_ -notin $services }
if ($missing) { throw "Missing Compose services: $($missing -join ', ')" }
docker compose config -q
rg -n "workspace-php-56|php-fpm-56|PHP56_VERSION|WORKSPACE_56_|php\$\{PHP56_VERSION\}\.ini" docker-compose.yml .env
```

Expected: 两个服务都在输出中；`docker compose config -q` 返回退出码 0；`php-fpm-56` 的 ini 挂载解析为 `php5.6.ini`；默认 Nginx upstream 仍引用 `php-fpm`。

- [ ] **Step 6: 提交仅包含 PHP 5.6 Compose hunks 的变更**

```powershell
git add -p -- docker-compose.yml
git diff --cached --check
git diff --cached -- docker-compose.yml
git commit -m "config: add PHP 5.6 workspace and fpm services"
```

在 `git add -p` 中只接受 `workspace-php-56` 和 `php-fpm-56` 的新增 hunks，拒绝 Nginx 域名 aliases 的用户修改。提交前的 diff 必须不包含 `admin.drs.cn`、`adm.imed.loc` 或 `www.imed.loc`。

### Task 3: 构建运行时并验证按站点选择 PHP 5.6

**Files:**
- Verify: `docker-compose.yml`
- Verify: `php-fpm/php5.6.ini`
- Create: `nginx/sites/php56.conf`
- Do not modify: `nginx/sites/default.conf`

**Interfaces:**
- Consumes: Task 2 的两个 Compose 服务和现有 Docker 镜像构建逻辑。
- Produces: 已启动的 PHP 5.6 CLI/FPM 容器；供遗留站点使用的 `php-fpm-56:9000` FastCGI 目标。

- [ ] **Step 1: 构建 PHP 5.6 两个服务**

```powershell
docker compose -p laradock-php56 build workspace-php-56 php-fpm-56
```

Expected: 两个镜像均构建成功。若构建失败，记录第一个失败的 PHP 扩展或 Dockerfile 命令；不要为规避失败而修改 8.3/7.4 共用扩展变量。

- [ ] **Step 2: 启动两个 PHP 5.6 服务**

```powershell
docker compose -p laradock-php56 up -d workspace-php-56 php-fpm-56
docker compose -p laradock-php56 ps workspace-php-56 php-fpm-56
```

Expected: 两个服务状态为 `running`，且 `php-fpm-56` 显示容器内 `9000/tcp` 端口。

- [ ] **Step 3: 对两个容器执行 PHP 版本冒烟检查**

```powershell
docker compose -p laradock-php56 exec -T workspace-php-56 php -r 'if (PHP_VERSION_ID < 50600 || PHP_VERSION_ID >= 50700) { fwrite(STDERR, PHP_VERSION . PHP_EOL); exit(1); } echo PHP_VERSION, PHP_EOL;'
docker compose -p laradock-php56 exec -T php-fpm-56 php -r 'if (PHP_VERSION_ID < 50600 || PHP_VERSION_ID >= 50700) { fwrite(STDERR, PHP_VERSION . PHP_EOL); exit(1); } echo PHP_VERSION, PHP_EOL;'
```

Expected: 两条命令均输出 `5.6.x` 并返回退出码 0。

- [ ] **Step 4: 创建自动加载的 PHP 5.6 站点配置，而不改默认 upstream**

新建 `nginx/sites/php56.conf`，完整内容如下：

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name php56.test;
    root /var/www/php56/public;
    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        try_files $uri /index.php =404;
        fastcgi_pass php-fpm-56:9000;
        fastcgi_index index.php;
        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 600;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/letsencrypt/;
        log_not_found off;
    }

    error_log /var/log/nginx/php56_error.log;
    access_log /var/log/nginx/php56_access.log;
}
```

文件必须使用 `.conf` 后缀，才能被 Nginx 的 `include /etc/nginx/sites-available/*.conf` 自动加载。不得修改 `nginx/sites/default.conf` 的 `fastcgi_pass php-upstream`，也不得把 `.env` 的 `NGINX_PHP_UPSTREAM_CONTAINER` 改为 `php-fpm-56`。

- [ ] **Step 5: 验证 PHP 5.6 站点配置与默认 upstream**

```powershell
rg --no-ignore -n "server_name php56\.test;|root /var/www/php56/public;|fastcgi_pass (php-fpm-56:9000|php-upstream);|NGINX_PHP_UPSTREAM_CONTAINER=php-fpm" nginx\sites .env
docker compose -p laradock-php56 build nginx
$env:NGINX_HOST_HTTP_PORT = '8088'
$env:NGINX_HOST_HTTPS_PORT = '8443'
$env:VARNISH_BACKEND_PORT = '8188'
try {
    docker run -d --rm --name laradock-php56-nginx-default-fpm --network laradock-php56_backend --network-alias php-fpm laradock-php-fpm:latest
    if ($LASTEXITCODE -ne 0) { throw "Default FPM DNS provider failed with exit code $LASTEXITCODE" }

    docker compose -p laradock-php56 up -d nginx --no-deps
    if ($LASTEXITCODE -ne 0) { throw "Nginx startup failed with exit code $LASTEXITCODE" }

    docker compose -p laradock-php56 exec -T nginx nginx -t
    if ($LASTEXITCODE -ne 0) { throw "nginx -t failed with exit code $LASTEXITCODE" }
}
finally {
    docker compose -p laradock-php56 down --remove-orphans

    docker container inspect laradock-php56-nginx-default-fpm *> $null
    if ($LASTEXITCODE -eq 0) {
        docker stop laradock-php56-nginx-default-fpm
    }

    $backendContainers = docker network inspect laradock-php56_backend --format '{{json .Containers}}' 2>$null
    if ($LASTEXITCODE -eq 0 -and $backendContainers -eq '{}') {
        docker network rm laradock-php56_backend
    }

    Remove-Item Env:NGINX_HOST_HTTP_PORT -ErrorAction SilentlyContinue
    Remove-Item Env:NGINX_HOST_HTTPS_PORT -ErrorAction SilentlyContinue
    Remove-Item Env:VARNISH_BACKEND_PORT -ErrorAction SilentlyContinue
}
```

Expected: `nginx/sites/php56.conf` 包含 `php56.test`、PHP 5.6 应用根目录和 `fastcgi_pass php-fpm-56:9000`；默认站点和 `.env` 仍包含 `php-upstream` / `php-fpm`；Nginx 配置检查成功。`nginx/sites` 是 Nginx 服务的运行时 bind mount，因此站点配置会在容器启动时加载；本验证只构建 `nginx`，不冷构建默认 `php-fpm` 镜像。临时辅助容器仅在隔离 backend 网络中提供默认 `php-upstream -> php-fpm` 的 DNS 解析；PHP 5.6 上游仍由 `php-fpm-56` 服务名解析。临时使用 `8088`、`8443`、`8188`，防止隔离工作树的 Nginx 与当前工作目录中已有的容器发生端口冲突；所有运行时 Compose 命令使用 `-p laradock-php56`，finally 块先清理 Compose 项目，再停止仍存在的辅助容器、删除空 backend 网络并清除临时端口变量，因此不影响主项目。

- [ ] **Step 6: 记录验证结果并完成最终状态检查**

```powershell
docker compose -p laradock-php56 config -q
docker compose -p laradock-php56 ps --all
git status --short
```

Expected: Compose 配置有效；运行时验证创建的临时项目容器均已清理；状态输出中不包含意外新增或暂存的用户已有 Nginx 删除、Nginx 域名 aliases 修改或其他无关文件。
