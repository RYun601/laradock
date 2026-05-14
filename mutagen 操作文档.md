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
