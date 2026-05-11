# Sub2API Alpine Linux 部署指南

本指南介绍如何在 Alpine Linux 上部署 Sub2API。

## Alpine Linux 特点

- **轻量级**：最小镜像大小约 5MB
- **包管理器**：使用 `apk` 而不是 `apt` 或 `yum`
- **C 库**：使用 musl libc 而不是 glibc
- **初始化系统**：通常使用 OpenRC，现代版本支持 systemd

## 前置要求

### 系统依赖

Alpine 需要安装以下必要工具：

```bash
# 基本工具
sudo apk add --no-cache curl tar

# 可选但推荐
sudo apk add --no-cache ca-certificates
```

Alpine 默认使用 **OpenRC** 作为初始化系统（无需额外安装）。

### 服务依赖

Sub2API 需要 PostgreSQL 和 Redis：

```bash
# 使用 Alpine 包管理器安装
sudo apk add --no-cache postgresql redis

# 或使用 Docker 容器
docker run -d --name postgres -e POSTGRES_PASSWORD=password postgres:alpine
docker run -d --name redis redis:alpine
```

## 安装方式

### 方式一：自动安装脚本（推荐）

```bash
# 基础安装（最新版本）
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash

# 或指定版本
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash -s -- install -v v0.1.0
```

脚本会自动检测 Alpine Linux 并进行相应配置。

### 方式二：Docker 安装

创建 `Dockerfile.alpine`：

```dockerfile
FROM alpine:latest

# 安装基本工具和依赖
RUN apk add --no-cache \
    curl \
    tar \
    ca-certificates \
    openrc \
    postgresql-client

# 安装 Sub2API
RUN curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | bash

# 暴露端口
EXPOSE 8080

# 启动服务
CMD ["sub2api"]
```

构建和运行：

```bash
docker build -f Dockerfile.alpine -t sub2api:alpine .
docker run -d -p 8080:8080 --name sub2api sub2api:alpine
```

## 配置说明

### 用户和权限

安装脚本会自动创建 `sub2api` 系统用户：

```bash
# 查看用户信息
id sub2api
```

### OpenRC 服务管理

Alpine 使用 **OpenRC** 初始化系统。使用以下命令管理 Sub2API 服务：

```bash
# 启动服务
sudo rc-service sub2api start

# 停止服务
sudo rc-service sub2api stop

# 查看状态
sudo rc-service sub2api status

# 重启服务
sudo rc-service sub2api restart

# 添加到默认运行级别（开机自启）
sudo rc-update add sub2api

# 从默认运行级别移除
sudo rc-update del sub2api

# 查看运行级别配置
sudo rc-config list
```

本地安装时，OpenRC 服务会强制设置 `DATA_DIR=/opt/sub2api`，避免误读 Docker 默认路径 `/app/data`。

如果系统没有启用 syslog，请先启动它，Sub2API 的 Alpine 服务现在通过 syslog 记录输出：

```bash
sudo rc-update add syslog boot
sudo rc-service syslog start
```

#### 服务启动卡住？

如果 `rc-service sub2api restart` 启动卡住，请参考 [TROUBLESHOOT.md](TROUBLESHOOT.md) 的故障排查指南。

**快速修复**（如果遇到启动卡住）：
```bash
# 1. 停止并清理
sudo rc-service sub2api stop
sudo killall sub2api 2>/dev/null || true
sudo rm -f /var/run/sub2api.pid

# 2. 重新生成服务脚本
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash -s -- install

# 3. 启动
sudo rc-service sub2api start
```

### 查看日志

Alpine 的 OpenRC 日志输出到系统日志（syslog）：

```bash
# 查看最新日志
sudo logread -f

# 如果系统写入 /var/log/messages，也可以这样看
sudo tail -f /var/log/messages

# 查找 Sub2API 相关日志
sudo logread -f | grep sub2api
```

## 常见问题

### 问题 1：缺少 glibc 兼容库

Alpine 使用 musl libc，某些预编译的 Go 二进制文件可能需要 glibc。

**解决方案**：
```bash
# 安装 glibc 兼容层
sudo apk add gcompat

# 或使用特定的 Alpine 构建版本
```

### 问题 2：rc-service 命令不工作

OpenRC 需要正确的环境配置。

**解决方案**：
```bash
# 检查 OpenRC 是否安装
sudo rc-status

# 重启 OpenRC
sudo rc-service openrc restart

# 或重启系统
sudo reboot
```

### 问题 3：PostgreSQL 连接问题

Alpine 中的 PostgreSQL 客户端库可能与服务器版本不匹配。

**解决方案**：
```bash
# 确保 PostgreSQL 客户端版本匹配服务器
sudo apk add postgresql-client

# 检查连接
psql -h localhost -U postgres -c "SELECT 1"
```

### 问题 4：配置文件权限错误 (permission denied)

如果遇到 `open ./config.yaml: permission denied` 错误，原因通常是：

1. **文件所有权错误** - 文件由 root 拥有，但应该由 sub2api 拥有
2. **工作目录错误** - 应用程序在错误的目录中启动

**完整修复方案**：
```bash
# 1. 修复文件权限和所有权
sudo chown -R sub2api:sub2api /opt/sub2api
sudo find /opt/sub2api -type f -exec chmod 644 {} \;
sudo find /opt/sub2api -type d -exec chmod 755 {} \;
sudo chmod +x /opt/sub2api/sub2api
sudo chmod 755 /opt/sub2api/data

# 2. 重新生成 OpenRC 脚本（包含正确的工作目录）
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash -s -- install -v $(curl -s https://api.github.com/repos/Wei-Shaw/sub2api/releases/latest | grep -o '"tag_name": "[^"]*' | cut -d'"' -f4)

# 3. 重启服务
sudo rc-service sub2api restart

# 4. 验证工作目录
sudo rc-service sub2api status
ps aux | grep sub2api
```

如果仍然有权限问题，检查：
```bash
# 查看配置文件的所有者和权限
ls -la /opt/sub2api/config.yaml

# 查看应用进程的用户
ps aux | grep sub2api

# 查看进程的工作目录
pwdx $(pgrep sub2api)
```

## 性能优化

### 内存优化

Alpine 默认配置较小的内存限制，可以调整：

```bash
# 查看当前限制
ulimit -a

# 临时增加（需要 root）
sudo ulimit -n 65536
```

### 启动优化

在 Alpine 中启用更快的启动：

```bash
# 使用 systemd 缓存
sudo systemctl cache
```

## 安全加固

### 防火墙配置

```bash
# 启用 Alpine 防火墙（如果安装了）
sudo apk add iptables

# 仅允许必要的端口
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
```

### 文件权限

```bash
# 限制配置文件权限
sudo chmod 600 /opt/sub2api/config.yaml
sudo chown sub2api:sub2api /opt/sub2api/config.yaml
```

## 日志管理

Alpine 使用 OpenRC 的日志输出到系统日志：

```bash
# 查看最新日志（实时）
sudo logread -f

# 如果系统写入 /var/log/messages，也可以查看该文件
sudo tail -f /var/log/messages

# 导出日志到文件
sudo logread > sub2api.log
```

### 配置日志级别

编辑 `/opt/sub2api/config.yaml` 来调整日志级别：

```yaml
log:
  level: info  # debug, info, warn, error
  format: json  # json or text
```

然后重启服务：

```bash
sudo rc-service sub2api restart
```

## 备份和恢复

### 备份

```bash
# 备份安装目录
sudo tar -czf sub2api-backup.tar.gz /opt/sub2api

# 备份配置
sudo tar -czf sub2api-config-backup.tar.gz /etc/sub2api
```

### 恢复

```bash
# 恢复安装
sudo tar -xzf sub2api-backup.tar.gz -C /

# 恢复配置
sudo tar -xzf sub2api-config-backup.tar.gz -C /
```

## 升级

```bash
# 使用安装脚本升级
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash -s -- upgrade

# 或升级到特定版本
curl -sSL https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/install.sh | sudo bash -s -- upgrade -v v0.2.0
```

## 支持

如遇问题，请：

1. 查看日志：`sudo tail -f /var/log/messages | grep sub2api` 或 `sudo logread -f`
1. 确认 syslog 已运行：`sudo rc-service syslog status`
2. 检查服务状态：`sudo rc-service sub2api status`
3. 检查系统要求：`uname -a`
4. 验证依赖：`which curl tar ca-certificates`
5. 提交 Issue：https://github.com/Wei-Shaw/sub2api/issues

## 相关资源

- [Alpine Linux 官方文档](https://wiki.alpinelinux.org/)
- [Alpine Linux 包查询](https://pkgs.alpinelinux.org/)
- [Sub2API 主仓库](https://github.com/Wei-Shaw/sub2api)
