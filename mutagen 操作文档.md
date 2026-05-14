# Mutagen 常用命令（基于 mutagen.yml 项目配置）

## 守护进程管理
```bash
# 启动守护进程
mutagen daemon start

# 停止守护进程
mutagen daemon stop

# 查看系统中所有 Mutagen 同步会话（不限于当前项目）
mutagen sync list
```

## 项目操作（需在 mutagen.yml 所在目录执行）

```bash
# 启动/创建项目中所有同步会话
mutagen project start

# 查看项目状态（列出所有会话及状态）
mutagen project list

# 暂停项目中所有会话
mutagen project pause

# 恢复项目中所有会话
mutagen project resume

# 终止并清理项目中所有会话
mutagen project terminate

# 终止系统中所有同步会话（不限于当前项目）
mutagen sync terminate --all

# 重置项目中所有会话（清空同步状态，解决冲突）
mutagen project reset

# 强制刷新所有待同步变更
mutagen project flush
```

## 其他常用命令（针对单个会话）

```bash
# 单独暂停某个会话（如 JKT-ADM)
mutagen sync pause JKT-ADM

# 单独恢复某个会话
mutagen sync resume JKT-ADM

# 单独终止某个会话
mutagen sync terminate JKT-ADM

# 查看单个会话详细状态（含冲突详情）
mutagen sync list JKT-ADM --long

# 实时监控单个会话
mutagen sync monitor JKT-ADM
```

## 修改配置后重建会话（推荐方式）

修改 mutagen.yml 后，用项目命令重建所有会话：

```bash
mutagen sync terminate --all
sleep 5
mutagen sync list          # 确认已清空
mutagen project start
```

仅重建单个会话（如 JKT-ADM）：

```bash
mutagen sync terminate JKT-ADM
mutagen project start      # 只会创建 YAML 中有但未运行的会话
```

## 让 daemon 随 WSL 自动启动

```bash
# 在 WSL 的 ~/.bashrc 末尾添加：
# mutagen daemon 自动启动
if ! mutagen daemon status &>/dev/null; then
    mutagen daemon start 2>/dev/null &
fi

```

## 当前配置的项目列表

| Session 名称        | Windows 路径                         | Linux 路径                                |
|---------------------|--------------------------------------|-------------------------------------------|
| laradock            | /mnt/d/work/laradock                 | /home/ryun/projects/laradock              |
| JKT-ADM             | /mnt/d/work/JKT_SYSTEM/ADM           | /home/ryun/projects/JKT_SYSTEM/ADM        |
| JKT-CRM             | /mnt/d/work/JKT_SYSTEM/CRM           | /home/ryun/projects/JKT_SYSTEM/CRM        |
| JKT-OA              | /mnt/d/work/JKT_SYSTEM/OA            | /home/ryun/projects/JKT_SYSTEM/OA         |
| JKT-hppi-idr-cn     | /mnt/d/work/JKT_SYSTEM/hppi_idr_cn   | /home/ryun/projects/JKT_SYSTEM/hppi_idr_cn|
| JKT-projm           | /mnt/d/work/JKT_SYSTEM/projm         | /home/ryun/projects/JKT_SYSTEM/projm      |
| drpeixun-cn         | /mnt/d/work/drpeixun_cn              | /home/ryun/projects/drpeixun_cn           |

## 日志同步说明

由于 `.gitignore` 里有 `/logs`，Mutagen 默认使用 `syntax: "git"` 会尊重 `.gitignore` 规则，因此 `logs/` 下的日志文件不会被同步。
如需同步日志文件，两个方案：

1. 从 `.gitignore` 移除 `/logs`，在 `mutagen.yml` 里用 `logs/**` 忽略
2. `mutagen.yml` 设置 `ignore.syntax: "docker"` 不再读取 `.gitignore`，手动列出所有 ignore 项：

   ```yaml
   sync:
     defaults:
       mode: two-way-resolved
       ignore:
         syntax: "docker"          # 不再自动读取 .gitignore
         paths:
           - "node_modules/**"
           - ".git/**"
           - ".idea/**"
           - "logs/**"            # 由 mutagen 层面忽略（这样不依赖 .gitignore）
           - "data/**"
     laradock:
       alpha: "/mnt/d/work/laradock"
       beta: "/home/ryun/projects/laradock"
     # ... 其他会话
   ```

   **注意**：改语法后 `.gitignore` 里原有的所有规则都会失效，需要把 `.gitignore` 中你仍然想忽略的目录/文件手动搬到 `mutagen.yml` 里。
