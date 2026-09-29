# SQLBot 离线部署指南

## 适用环境

- **目标系统**：Oracle Linux 8.x / RHEL 8.x / CentOS 8.x（x86_64）
- **容器运行时**：Podman（自动离线安装）
- **Python 版本**：Python 3.6+（Podman compose 需要 Python 3.9+，脚本会自动检测）

---

## 目录结构

```
deploy/
── DEPLOY.md          # 本文档
├── package-offline.sh # 离线打包脚本（在有网机器上运行）
├── install.sh         # 安装脚本（在目标机器上运行）
└── sctl               # 服务控制脚本
```

---

## 一、打包（在有外网的机器上）

### 前置条件

**方案 A：使用 WSL2（推荐 Windows 用户）**

1. 安装 WSL2 和 Ubuntu 发行版
2. 在 WSL2 中安装 Podman：
   ```bash
   sudo apt update && sudo apt install -y podman
   ```
3. 安装 Python 3.9+ 和 pip3：
   ```bash
   sudo apt install -y python3 python3-pip
   ```
4. 将项目代码复制到 WSL2 原生文件系统（避免 /mnt 跨文件系统性能问题）：
   ```bash
   cp -r /mnt/d/smartSQL/SQLBot ~/SQLBot
   cd ~/SQLBot
   ```

**方案 B：使用任意 Linux 系统**

1. 已安装 **Podman** 或 **Docker**
2. 已安装 **Python 3.9+** 和 **pip3**
3. 项目代码已在本地

### 打包步骤

```bash
# 1. 进入项目根目录
cd /path/to/SQLBot

# 2. 复制打包脚本到根目录
cp deploy/package-offline.sh ./

# 3. 执行打包
bash package-offline.sh
```

### 打包过程说明

脚本会自动完成以下步骤：

| 步骤 | 内容 | 耗时 |
|------|------|------|
| 1 | 检查环境（Podman/Docker） | 几秒 |
| 2 | 构建 SQLBot 镜像（首次需拉取基础镜像） | 20-40 分钟 |
| 3 | 创建离线包目录结构 | 几秒 |
| 4 | 复制安装脚本和配置 | 几秒 |
| 5 | 导出镜像为 tar 文件 | 2-5 分钟 |
| 6 | 从 Oracle Linux 8 仓库下载 Podman RPM 包（~34MB） | 1-3 分钟 |
| 7 | 从 PyPI 下载 podman-compose wheel 包（~1MB） | 几秒 |
| 8 | 打包为 tar.gz | 1-2 分钟 |

### 输出文件

```
sqlbot-offline-<版本>-x86_64-podman.tar.gz
```

大小约 **2-3 GB**（主要取决于镜像大小）。

### 注意事项

- **不要在 `/mnt/` 目录下构建**（WSL2 用户）：跨文件系统会导致构建极慢
- **镜像构建失败**：如果因网络问题失败，重新运行脚本即可，已完成的层会缓存
- **RPM 下载失败**：Oracle Linux 8 仓库是公共仓库，一般可访问；如失败可手动下载放入 `podman/rpms/`
- **Python 版本**：podman-compose 需要 Python 3.8+，脚本会自动检测并使用 python3.9 或 python3

---

## 二、部署（在目标机器上）

### 前置条件

- Oracle Linux 8.x / RHEL 8.x / CentOS 8.x（x86_64）
- 有 root 或 sudo 权限
- 磁盘空间 ≥ 10GB（镜像 + 数据）
- 内存 ≥ 4GB（推荐 8GB+）

### 步骤

```bash
# 1. 上传离线包到目标机器
scp sqlbot-offline-*.tar.gz user@目标机器:/tmp/

# 2. SSH 登录目标机器
ssh user@目标机器

# 3. 解压
cd /tmp
tar -xzf sqlbot-offline-*.tar.gz
cd sqlbot-offline-package

# 4. （可选）修改配置
vim install.conf
# 主要配置项：
#   SQLBOT_WEB_PORT=8000        # Web 端口
#   SQLBOT_MCP_PORT=8001        # MCP 端口
#   SQLBOT_DB_PASSWORD=xxx      # 数据库密码
#   SQLBOT_DEFAULT_PWD=xxx      # 默认用户密码

# 5. 执行安装（需要 sudo）
sudo bash install.sh
```

### 安装过程说明

脚本会自动完成：

| 步骤 | 内容 |
|------|------|
| 1 | 检查环境并初始化配置 |
| 2 | 设置运行目录（默认 `/opt/sqlbot`） |
| 3 | 复制安装文件到运行目录 |
| 4 | 安装 sctl 命令行工具 |
| 5 | 关闭 SELinux（如果开启） |
| 6 | 离线安装 Podman（从 RPM 包） |
| 7 | 安装 podman-compose（从 Python wheel） |
| 8 | 加载 SQLBot 镜像 |
| 9 | 启动服务 |

### 安装后配置

```bash
# 修改 SQLBot 配置中的服务器 IP（用于图片服务）
vim /opt/sqlbot/conf/sqlbot.conf
# 修改 SQLBOT_SERVER_IMAGE_HOST 为实际 IP：
# SQLBOT_SERVER_IMAGE_HOST=http://实际IP:8001/images/

# 重启服务
sctl restart
```

### 访问服务

```
http://服务器IP:8000
```

默认账号：
- 用户名：`admin`
- 密码：`SQLBot@123456`（或 install.conf 中配置的 SQLBOT_DEFAULT_PWD）

---

## 三、日常运维

### 服务管理

```bash
# 查看状态
sctl status

# 启动
sctl start

# 停止
sctl stop

# 重启
sctl restart

# 查看版本
sctl version

# 清理旧镜像
sctl clear-images

# 清理日志
sctl clear-logs
```

### 日志位置

```
/opt/sqlbot/data/sqlbot/logs/
── info.log
├── debug.log
── error.log
```

### 数据备份

```bash
# 备份数据库（PostgreSQL）
pg_dump -h localhost -U root sqlbot > sqlbot_backup.sql

# 备份数据目录
tar -czf sqlbot-data-backup.tar.gz /opt/sqlbot/data/
```

---

## 四、升级

```bash
# 1. 在有网机器上重新打包新版本
bash package-offline.sh

# 2. 上传新版本离线包到目标机器
scp sqlbot-offline-<新版本>.tar.gz user@目标机器:/tmp/

# 3. 在目标机器上执行安装（会自动识别为升级）
cd /tmp
tar -xzf sqlbot-offline-*.tar.gz
cd sqlbot-offline-package
sudo bash install.sh
```

升级会自动：
- 停止旧版本服务
- 加载新镜像
- 保留数据（数据库、配置文件等）
- 启动新版本

---

## 五、常见问题

### Q1: Podman 安装失败，提示缺少依赖

**A**: 确保 RPM 包完整。如果缺少依赖，从 Oracle Linux 8 AppStream/BaseOS 仓库下载对应 RPM 并放入 `podman/rpms/` 目录重新安装。

### Q2: podman-compose 找不到命令

**A**: 检查 Python 版本。podman-compose 需要 Python 3.8+。如果系统 Python 版本过低（如 3.6），需要安装 Python 3.9+：

```bash
# Oracle Linux 8
dnf module install python39 -y
```

然后重新安装 podman-compose：

```bash
python3.9 -m pip install --target /opt/podman-compose-env --no-index /path/to/wheels/*.whl
```

### Q3: 容器启动失败

**A**: 检查日志：

```bash
podman logs sqlbot
# 或
cat /opt/sqlbot/data/sqlbot/logs/error.log
```

常见原因：
- 端口被占用：修改 install.conf 中的端口配置
- 磁盘空间不足：清理旧数据或扩容
- 内存不足：增加系统内存或 swap

### Q4: 镜像加载失败

**A**: 确保镜像 tar 文件完整。重新打包或重新传输。

---

## 六、技术细节

### 依赖清单

#### Podman RPM 包（从 Oracle Linux 8 AppStream）

| 包名 | 说明 |
|------|------|
| podman | Podman 主程序 |
| podman-docker | Docker 兼容层 |
| podman-plugins | Podman 插件 |
| podman-catatonit | Podman 初始化进程 |
| podman-gvproxy | 网络代理 |
| conmon | 容器监控 |
| containers-common | 容器通用配置 |
| runc | OCI 运行时 |
| crun | 轻量级 OCI 运行时 |
| slirp4netns | 用户态网络 |
| fuse-overlayfs | 用户态文件系统 |
| container-selinux | SELinux 策略 |
| containernetworking-plugins | 网络插件 |
| netavark | 网络栈 |
| aardvark-dns | DNS 服务器 |
| dnsmasq | DNS 转发 |
| libslirp | SLIRP 库 |
| yajl | JSON 库 |

#### Podman RPM 包（从 Oracle Linux 8 BaseOS）

| 包名 | 说明 |
|------|------|
| shadow-utils | 用户管理工具 |
| shadow-utils-subid | 子 ID 管理 |

#### Python Wheel 包（podman-compose 依赖）

| 包名 | 说明 |
|------|------|
| podman-compose | Docker Compose 的 Podman 实现 |
| python-dotenv | 环境变量管理 |
| pyyaml | YAML 解析 |

### 镜像信息

- **基础镜像**：registry.cn-qingdao.aliyuncs.com/dataease/sqlbot-python-pg:latest
- **构建镜像**：registry.cn-qingdao.aliyuncs.com/dataease/sqlbot-base:latest
- **向量模型**：ghcr.io/1panel-dev/maxkb-vector-model:v1.0.1
- **运行环境**：Python 3.11 + Node.js 18 + PostgreSQL 17.6

---

## 七、版本历史

| 日期 | 版本 | 说明 |
|------|------|------|
| 2026-08-25 | v1.0 | 初始版本，支持 Oracle Linux 8 离线部署 |

---

## 八、联系与支持

如有问题，请查看项目文档或联系运维团队。
