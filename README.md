# 渗透测试环境一键部署（Ansible）

本项目是一个基于 **Ansible** 的渗透测试环境自动化部署脚本，主要面向：

- 内网/攻防演练实验环境初始化
- 渗透测试人员常用工具与开发环境快速铺设
- ARL 资产侦察灯塔系统的 Docker 化部署

控制端建议使用 **Linux / WSL（如 Ubuntu-24.04）**，被控端为常见的 **Debian / Ubuntu / Kali** 系发行版。

---

## 功能总览

- **基础系统与时间同步**
  - 统一设置时区（默认为 `Asia/Shanghai`）
  - 安装并启用 `chrony`，进行时间同步
  - 安装一批常用基础工具：`curl`、`wget`、`git`、`vim`、`tmux`、`htop`、`tcpdump`、`net-tools`、`tree` 等
  - 配置系统级 `pip` 国内源（默认清华源）
  - 安装并配置 `ufw` 防火墙，默认开放常用端口

- **系统与网络优化**
  - 通过 `sysctl` 自动应用内核与网络参数（最大文件句柄、连接队列、端口范围、TCP 复用等）
  - 可开关控制是否启用优化
  - **BBR 拥塞控制算法**：自动检测内核版本，支持 Linux 4.9+ 内核

- **开发环境安装**
  - C/C++ 构建工具链：`build-essential`、`pkg-config`
  - Python：`python3`、`python3-pip`、`python3-venv`、`python3-dev`
  - Python 工具：`pipx`（并确保 PATH 设置）
  - 其他语言/工具：`golang`、`nodejs`、`npm`

- **渗透测试工具集**
  - 通用工具（Debian/Ubuntu）：
    - 扫描类：`nmap`、`masscan`
    - Web 类：`nikto`、`sqlmap`、`gobuster`、`dirb`、`whatweb`
    - 密码爆破/破解：`hydra`、`john`、`hashcat`
    - 抓包与协议：`wireshark-common`、`tshark`
    - 通信与隧道：`netcat-traditional`、`socat`
    - 字典：`seclists`（如 apt 无包则可通过 git 安装）
  - **GitHub 最新工具**：自动从 GitHub Releases 获取最新版本
    - `nuclei`：现代漏洞扫描器
    - `naabu`：快速端口扫描器
    - `subfinder`：子域名发现工具
    - `observer_ward`：Web 应用指纹识别
    - `afrog`：高性能漏洞扫描器
    - `httpx`：多用途 HTTP 工具包
  - Kali 特性：
    - 可选安装 Kali 元包（默认 `kali-tools-top10`）

- **Docker 环境与优化**
  - 自动安装 Docker 及相关组件：
    - Kali：使用发行版仓库安装 `docker.io`、`docker-compose`
    - Debian/Ubuntu：可使用 Docker 官方仓库安装 `docker-ce` 等组件
  - 支持配置：
    - Docker 用户组：将指定用户加入 `docker` 组
    - `daemon.json`：包括 registry mirror、额外参数等
    - **禁用 iptables 接管**：与 UFW 防火墙协同工作

- **ARL 资产灯塔 Docker 部署**
  - 在指定目录（默认 `/opt/pentest-stack`）生成 `docker compose` 配置
  - 使用 ARL 官方维护的 Docker 镜像（变量可自定义）：
    - 默认镜像：`honmashironeko/arl-docker-portion`
    - 容器名：`arl`
    - 端口映射：`5003:5003`（可配置）
  - 支持 `docker compose` 与旧版 `docker-compose` 两种调用方式
  - Ansible `--check` 模式下不会实际拉取或启动容器，仅生成配置

- **UFW 防火墙配置**
  - 自动安装并配置 UFW 防火墙
  - 默认拒绝所有入站连接，仅开放指定端口
  - 支持端口范围配置
  - 与 Docker 协同工作，确保容器端口映射正常

---

## 项目结构

- 配置与入口
  - `ansible.cfg`：Ansible 配置，指定 inventory、回显格式等
  - `site.yml`：主入口 Playbook，串联所有角色
  - `group_vars/all.yml`：全局变量配置（时区、sysctl、工具列表、Docker 配置等）
  - `inventories/hosts.ini`：示例远程主机清单
  - `inventories/local.ini`：本机（localhost）测试清单

- 角色（roles）
  - `roles/common`：基础环境与时间同步
  - `roles/system_tune`：系统与网络参数调优（sysctl）
  - `roles/dev_env`：开发环境安装与 pipx 配置
  - `roles/pentest_tools`：渗透测试工具集安装（适配 Debian/Ubuntu/Kali）
  - `roles/docker_engine`：Docker / Docker Compose 安装、配置与服务管理
  - `roles/docker_stack`：ARL Docker 栈 compose 文件生成与启动

---

## 角色功能详情

### 1. common：基础环境与时间同步

- 校验系统家族，当前支持：
  - `Debian` 系（Debian / Ubuntu / Kali）
  - `RedHat` 系预留（可按需扩展）
- 安装基础软件包（可通过 `base_packages` 自定义）
- 设置系统时区：
  - 写入 `/etc/timezone`（Debian 系）
  - 链接 `/etc/localtime` 到相应时区文件
- 安装并启用 `chrony` 用于 NTP 时间同步
- 配置 `pip` 全局源为国内镜像（如清华）

### 2. system_tune：系统与网络优化

- 使用 `ansible.posix.sysctl`：
  - 将 `group_vars/all.yml` 中的 `sysctl_tune` 字典项批量写入系统
  - 自动 `sysctl -p` 应用
- 典型优化包含：
  - 文件句柄数、TCP 连接队列、端口范围、FIN 超时、TIME_WAIT 重用等
- 通过 `enable_system_tune` 开关控制是否启用

### 3. dev_env：开发环境

- 针对 Debian 系：
  - 安装 C/C++ 工具链：`build-essential`、`pkg-config`
  - 安装 Python 相关：`python3`、`python3-pip`、`python3-venv`、`python3-dev`
  - 安装 `pipx`，并执行 `pipx ensurepath --global` 确保 PATH
  - 安装 `golang`、`nodejs`、`npm`
- 依赖包列表可在 `dev_packages_debian` 变量中调整

### 4. pentest_tools：渗透测试工具集

- 在 Debian/Ubuntu 上安装常见安全工具：
  - 端口与服务扫描：`nmap`、`masscan`
  - Web 安全：`nikto`、`sqlmap`、`whatweb`、`gobuster`、`dirb`
  - 爆破/破解：`hydra`、`john`、`hashcat`
  - 抓包与协议分析：`wireshark-common`、`tshark`
  - 通信与隧道：`netcat-traditional`、`socat`
  - 字典：`seclists`（如 apt 无包，则支持 git 克隆方式安装到 `pentest_seclists_path`）
- 针对 Kali：
  - 可选安装 Kali 元包，如 `kali-tools-top10`
  - 通过 `pentest_install_kali_meta` 和 `kali_meta_packages` 控制

### 5. docker_engine：Docker 环境

- 平台校验：仅在 `Debian` 系系统上执行
- 安装 Docker 依赖：
  - `ca-certificates`、`curl`、`gnupg` 等
- 安装策略：
  - **Kali**：
    - 直接使用发行版仓库安装 `docker.io`、`docker-compose`
  - **Debian / Ubuntu**：
    - 获取 `dpkg --print-architecture` 结果作为架构
    - 创建 `/etc/apt/keyrings`，下载 Docker 仓库 GPG key
    - 添加 Docker 官方 apt 仓库（URL 可通过变量覆盖）
    - 安装 `docker-ce`、`docker-ce-cli`、`containerd.io`、`docker-buildx-plugin`、`docker-compose-plugin`
- 其他配置：
  - 生成 `/etc/docker/daemon.json`（支持 registry mirror 与附加配置）
  - 启用并启动 `docker` systemd 服务
  - 创建 `docker` 用户组，并将指定用户加入该组

### 6. docker_stack：ARL 容器栈

- 在 `docker_stack_dir`（默认 `/opt/pentest-stack`）生成 `compose.yml`
- 使用参数：
  - `arl_docker_image`：默认 `honmashironeko/arl-docker-portion`
  - `arl_container_name`：默认 `arl`
  - `arl_web_port`：默认 `5003:5003`
- 支持自动判断 `docker compose` / `docker-compose`：
  - 优先检测 `docker compose`
  - 不存在则回退到 `docker-compose`
- 运行逻辑：
  - 在真实执行时（非 `--check` 模式）调用 `up -d` 启动 ARL 栈
  - 在 `--check` 模式下仅生成文件，不拉取镜像、不启动容器

---

## 使用方法

### 1. 准备控制端（推荐 WSL Ubuntu）

在 Windows 上建议通过 WSL 运行 Ansible：

```bash
# 进入项目目录（示例为 D 盘）
cd /mnt/d/AI-code/base/sh

# 确认 ansible 与 ansible-lint 已安装
ansible --version
ansible-lint --version
```

### 2. 修改 Inventory

编辑 `inventories/hosts.ini`，将示例主机替换为你的目标服务器：

```ini
[pentest]
target ansible_host=192.168.56.101 ansible_user=root

[pentest:vars]
ansible_become=true
```

如需在本机上测试 playbook 逻辑，可使用 `inventories/local.ini`：

```ini
[pentest]
localhost ansible_connection=local
```

### 3. 调整全局配置

根据需要修改 `group_vars/all.yml` 中的变量，例如：

- `timezone_name`：统一时区
- `enable_system_tune` / `sysctl_tune`：系统优化开关与参数
- `base_packages`、`dev_packages_debian`：基础包与开发包
- `pentest_packages_debian`、`pentest_install_kali_meta`：渗透工具列表
- `docker_users`：允许免 sudo 使用 Docker 的用户
- `docker_registry_mirrors`、`docker_daemon_options`：Docker 守护进程配置
- `docker_stack_dir`、`arl_*`：ARL Docker 部署相关配置

### 4. 语法与 dry-run 检查（推荐）

先在 WSL 中执行语法检查与演练：

```bash
cd /mnt/d/AI-code/base/sh

# playbook 语法检查
ANSIBLE_CONFIG=ansible.cfg ansible-playbook site.yml -i inventories/local.ini --syntax-check

# dry-run，查看将要修改的内容
ANSIBLE_CONFIG=ansible.cfg ansible-playbook site.yml -i inventories/local.ini --check --diff
```

### 5. 正式部署

确认无问题后，针对目标环境执行：

```bash
cd /mnt/d/AI-code/base/sh
ANSIBLE_CONFIG=ansible.cfg ansible-playbook site.yml -i inventories/hosts.ini
```

部署完成后：

- 可以在目标机上通过 `docker ps | grep arl` 确认 ARL 容器
- 浏览器访问 `https://目标IP:5003/` 打开 ARL Web 控制台

根据 ARL Docker 镜像说明，首次进入容器可能还需要执行额外初始化脚本（例如 `/root/arl/set.sh`），请参考对应镜像文档。

---

## 简明使用教程

### 快速开始（5分钟部署）

#### 1. 准备控制端
```bash
# 在 Ubuntu/WSL 上安装 Ansible
sudo apt update && sudo apt install -y ansible ansible-lint

# 克隆项目（如果未克隆）
git clone <项目地址>
cd /path/to/project
```

#### 2. 配置目标主机
```bash
# 编辑主机清单
nano inventories/hosts.ini

# 添加目标主机（示例）
[pentest]
target ansible_host=192.168.1.100 ansible_user=root
```

#### 3. 执行部署
```bash
# 语法检查（推荐）
ansible-playbook site.yml -i inventories/hosts.ini --syntax-check

# 演练模式（查看将要修改的内容）
ansible-playbook site.yml -i inventories/hosts.ini --check --diff

# 正式部署
ansible-playbook site.yml -i inventories/hosts.ini
```

#### 4. 验证部署
```bash
# 检查系统状态
ssh root@192.168.1.100 "
  # 检查 BBR 是否启用
  sysctl net.ipv4.tcp_congestion_control
  
  # 检查工具安装
  nuclei --version && naabu --version && subfinder --version
  
  # 检查 Docker 和 ARL
  docker ps | grep arl
  
  # 检查 UFW 状态
  ufw status
"
```

### 常用配置示例

#### 1. 修改工具列表
```yaml
# 在 group_vars/all.yml 中添加/删除工具
github_tools:
  - name: nuclei
    enabled: true  # 启用
  - name: naabu
    enabled: false  # 禁用
```

#### 2. 修改开放端口
```yaml
# 在 group_vars/all.yml 中修改端口ufw_allow_ports:
  - "22"     # SSH
  - "80"     # HTTP
  - "443"    # HTTPS
  - "8080"   # 自定义端口
  - "9000:9100"  # 端口范围
```

#### 3. 禁用 BBR
```yaml
# 在 group_vars/all.yml 中
enable_bbr: false
```

### 单角色部署

```bash
# 仅部署基础环境和时间同步
ansible-playbook site.yml -i inventories/hosts.ini --tags common

# 仅部署渗透测试工具
ansible-playbook site.yml -i inventories/hosts.ini --tags pentest

# 仅部署 Docker 环境
ansible-playbook site.yml -i inventories/hosts.ini --tags docker
```

---

## 注意事项

- 本项目假设被控端是类 Unix 系统（Debian / Ubuntu / Kali），不直接支持 Windows 作为被控端。
- 在网络受限环境中，Docker 官方仓库与镜像拉取可能失败：
  - 可以通过修改 `docker_gpg_url`、`docker_repo_base_url`、`docker_registry_mirrors` 等变量指向国内镜像。
- 当前 ARL 部署采用 Docker 方式：
  - 如需改为源码安装，可在现有基础上新增角色或扩展 `docker_stack` 角色，将官方 `setup-arl.sh` 等脚本纳入自动化流程。
- GitHub API 有访问限制，如需部署大量主机，建议设置 `GITHUB_TOKEN` 环境变量。

