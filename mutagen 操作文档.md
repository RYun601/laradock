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

# 重置项目中所有会话（清空同步状态，解决冲突）
mutagen project reset

# 强制刷新所有待同步变更
mutagen project flush
```

## 其他常用命令（针对单个会话）

```bash
# 单独暂停某个会话（如 JKT-SYSTEM)
mutagen sync pause JKT-SYSTEM

# 单独恢复某个会话
mutagen sync resume JKT-SYSTEM

# 单独终止某个会话
mutagen sync terminate JKT-SYSTEM

# 查看单个会话详细状态
mutagen sync list JKT-SYSTEM
```

## 让 daemon 随 WSL 自动启动，这样重启后不需要手动操作：
```bash
# 在 WSL 的 ~/.bashrc 末尾添加：
# mutagen daemon 自动启动
if ! mutagen daemon status &>/dev/null; then
    mutagen daemon start 2>/dev/null &
fi

```

## 单个会话重建（如修改 mutagen.yml 配置后需要重建生效）
```bash
# 终止指定会话
mutagen sync terminate JKT-SYSTEM

# 重建（需指定完整路径，配置项继承 mutagen.yml 的 defaults）
mutagen sync create --name JKT-SYSTEM /mnt/d/work/JKT_SYSTEM /home/ryun/projects/JKT_SYSTEM
```

## 全部会话批量重建（一键脚本）

```bash
mutagen sync terminate laradock
mutagen sync terminate JKT-SYSTEM
mutagen sync terminate drpeixun-cn

mutagen sync create --name laradock /mnt/d/work/laradock /home/ryun/projects/laradock
mutagen sync create --name JKT-SYSTEM /mnt/d/work/JKT_SYSTEM /home/ryun/projects/JKT_SYSTEM
mutagen sync create --name drpeixun-cn /mnt/d/work/drpeixun_cn /home/ryun/projects/drpeixun_cn
```

## 日志同步说明

由于 `.gitignore` 里有 `/logs`，Mutagen 默认使用 `syntax: "git"` 会尊重 `.gitignore` 规则，因此 `logs/` 下的日志文件不会被同步。
如需同步日志文件，两个方案：

1. 从 `.gitignore` 移除 `/logs`，在 `mutagen.yml` 里用 `logs/**` 忽略
2. `mutagen.yml` 设置 `ignore.syntax: "docker"` 不再读取 `.gitignore`，手动列出所有 ignore 项
