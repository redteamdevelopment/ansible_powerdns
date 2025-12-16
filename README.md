# PowerDNS Recursor 自动化部署项目

[![Ansible](https://img.shields.io/badge/Ansible-2.20+-green.svg)](https://www.ansible.com/)
[![Debian](https://img.shields.io/badge/Debian-12-blue.svg)](https://www.debian.org/)
[![PowerDNS](https://img.shields.io/badge/PowerDNS-4.8.8-orange.svg)](https://www.powerdns.com/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

> **使用 Ansible 自动化部署 PowerDNS Recursor 集群，支持差异化 DNS 转发、Lua 域名劫持、Web API 监控和多云架构**




mac 刷新dns
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

scutil --dns


---

## 📋 目录

- [项目概述](#项目概述)
- [功能特性](#功能特性)
- [系统要求](#系统要求)
- [快速开始](#快速开始)
- [项目结构](#项目结构)
- [配置说明](#配置说明)
- [部署流程](#部署流程)
- [验证测试](#验证测试)
- [故障排查](#故障排查)
- [高级配置](#高级配置)
- [生产环境建议](#生产环境建议)
- [常见问题](#常见问题)
- [维护指南](#维护指南)
- [技术架构](#技术架构)
- [贡献指南](#贡献指南)
- [许可证](#许可证)

---

## 项目概述

本项目提供了一套完整的 Ansible 自动化方案，用于在多台 Debian 12 服务器上快速部署 PowerDNS Recursor DNS 递归解析服务。

### 适用场景

- **企业级 DNS 基础设施**: 为内部网络或客户提供可靠的 DNS 解析服务
- **多地域 DNS 集群**: 在全球多个数据中心部署 DNS 节点，实现就近访问
- **智能 DNS 转发**: 根据域名后缀（如 `.cn`, `.us`）使用不同的上游 DNS 服务器
- **域名劫持/重定向**: 通过 Lua 脚本实现特定域名的自定义解析（内容过滤、本地服务重定向等）
- **路由器/物联网设备上游 DNS**: 为大量设备提供统一的 DNS 解析服务

### 项目背景

该项目源于实际生产需求：为柬埔寨 400 台路由器提供稳定、高效的 DNS 解析服务。经过实践验证，已成功部署在 DigitalOcean、AWS、Vultr、Azure 等多个云平台。

---

## 功能特性

### ✅ 核心功能

- **🚀 一键自动化部署**: 单条命令完成从系统检测到服务验证的全流程
- **🎯 差异化 DNS 转发**: 根据域名后缀智能选择上游 DNS（支持 `.cn`, `.us`, 默认等规则）
- **🔧 Lua 域名劫持**: 灵活的域名重写和劫持能力（适用于广告过滤、本地服务等）
- **📊 Web API 监控**: 内置 RESTful API 接口，支持实时查询统计数据
- **🛡️ DigitalOcean 优化**: 解决 DO 平台 DNS 端口过滤问题，确保公网可访问
- **🔄 高可用设计**: 支持多节点部署，自动负载均衡
- **✅ 自动化验证**: 部署后自动测试所有核心功能，输出详细报告

### 🔒 安全特性

- **随机 API 密钥生成**: 每次部署自动生成高强度密钥
- **systemd-resolved 冲突处理**: 自动禁用系统默认 DNS 服务，释放 53 端口
- **配置文件权限管理**: 严格控制配置文件访问权限
- **备份机制**: 自动备份原始系统配置

### 📈 性能优化

- **多线程处理**: 默认 2 线程，可根据服务器配置调整
- **缓存优化**: 
  - 200 万条记录缓存
  - 50 万条包缓存
  - 零 SERVFAIL 缓存时间
- **根提示文件**: 使用本地根服务器列表，加速根域名解析

---

## 系统要求

### 控制节点（执行 Ansible 的机器）

| 组件 | 最低版本 | 推荐版本 |
|------|---------|---------|
| Ansible | 2.9+ | 2.20+ |
| Python | 3.6+ | 3.10+ |
| SSH Client | OpenSSH 7.0+ | OpenSSH 8.0+ |

### 目标节点（DNS 服务器）

| 组件 | 要求 |
|------|------|
| **操作系统** | Debian 12 (Bookworm) |
| **CPU** | 1 核心（推荐 2 核心+） |
| **内存** | 512MB（推荐 1GB+） |
| **硬盘** | 5GB 可用空间 |
| **网络** | 公网 IP，UDP/TCP 53 端口可访问 |
| **权限** | root 或具有 sudo 权限的用户 |

### 网络要求

- 控制节点能够 SSH 连接到所有目标节点
- 目标节点可以访问互联网（用于安装软件包）
- UDP/TCP 53 端口对外开放（根据安全策略配置防火墙）
- TCP 8082 端口（Web API，可选，建议仅内网访问）

---

## 快速开始

### 1. 克隆项目

```bash
git clone https://github.com/your-repo/powerdns-recursor-ansible.git
cd powerdns-recursor-ansible
```

### 2. 安装 Ansible（如未安装）

**macOS:**
```bash
brew install ansible
```

**Debian/Ubuntu:**
```bash
sudo apt update
sudo apt install ansible -y
```

**验证安装:**
```bash
ansible --version
# 期望输出: ansible [core 2.20+]
```

### 3. 配置目标服务器

编辑 `inventory.ini` 文件，添加你的服务器信息：

```ini
[dns_servers]
recursor1 ansible_host=152.42.186.194 ansible_ssh_pass=your_password
recursor2 ansible_host=203.0.113.10 ansible_ssh_pass=your_password
recursor3 ansible_host=198.51.100.20 ansible_ssh_pass=your_password

[dns_servers:vars]
ansible_user=root
ansible_python_interpreter=/usr/bin/python3
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

> **安全提示**: 生产环境推荐使用 SSH 密钥认证替代密码，删除 `ansible_ssh_pass` 配置。

### 4. 执行部署

```bash
ansible-playbook -i inventory.ini playbook.yml
```

### 5. 验证部署

部署完成后，Ansible 会自动输出部署摘要。你也可以手动验证：

```bash
# 测试 DNS 查询
dig @152.42.186.194 google.com

# 测试 Lua 劫持
dig @152.42.186.194 www.baidu.com
# 期望返回: 10.0.0.8

# 测试 Web API（从服务器执行）
ssh root@152.42.186.194
curl -H "X-API-Key: YOUR_API_KEY" http://127.0.0.1:8082/api/v1/servers/localhost/statistics
```

---

## 项目结构

```
.
├── README.md                    # 本文档
├── inventory.ini                # Ansible 主机清单（包含目标服务器列表）
├── playbook.yml                 # Ansible 主 playbook（部署逻辑）
├── gemini.md                    # 开发规范和协作指南
├── dnssetup.md                  # 技术文档和避坑指南
├── plan.md                      # 项目规划和需求分析
├── deployment-notes.md          # 部署改进记录
├── problem.md                   # Bug 记录（可选，如遇到问题）
└── templates/
    ├── recursor.conf.j2         # PowerDNS Recursor 配置模板
    └── override.lua.j2          # Lua 域名劫持脚本模板
```

### 文件说明

#### `inventory.ini`
定义目标服务器列表和连接参数。支持分组、变量继承等高级功能。

#### `playbook.yml`
核心部署脚本，包含以下任务流程：
1. 系统版本检测
2. systemd-resolved 处理
3. 软件包安装
4. 配置文件部署
5. 服务启动
6. 功能验证

#### `templates/recursor.conf.j2`
PowerDNS Recursor 配置文件模板，使用 Jinja2 语法支持变量化配置。

#### `templates/override.lua.j2`
Lua 脚本模板，用于实现域名劫持和自定义解析逻辑。

---

## 配置说明

### Playbook 变量

在 `playbook.yml` 中定义的关键变量：

```yaml
vars:
  pdns_allow_from: "0.0.0.0/0"  # 允许查询的来源 IP 段
  pdns_threads: 2                # 工作线程数（建议与 CPU 核心数相同）
  pdns_api_key: "随机生成的密钥"  # Web API 访问密钥
```

#### `pdns_allow_from`
**重要**: DigitalOcean 平台必须设置为 `0.0.0.0/0`，否则公网 DNS 查询会被过滤。其他平台可以根据安全策略调整。

**示例配置**:
```yaml
# 只允许内网访问
pdns_allow_from: "10.0.0.0/8,192.168.0.0/16,172.16.0.0/12"

# 只允许特定 IP 段
pdns_allow_from: "203.0.113.0/24,198.51.100.0/24"
```

#### `pdns_threads`
根据服务器 CPU 核心数调整：
- 1 核心: `threads: 1`
- 2 核心: `threads: 2`
- 4 核心及以上: `threads: 4`

#### `pdns_api_key`
部署时自动生成，格式为 Base64 编码的 32 字节随机数。可手动修改：

```bash
# 生成新密钥
openssl rand -base64 32
```

### 差异化 DNS 转发配置

在 `templates/recursor.conf.j2` 中定义转发规则：

```conf
# .cn 域名使用国内 DNS
forward-zones-recurse=cn.=114.114.114.114;223.5.5.5;119.29.29.29

# .us 域名使用 Google DNS
forward-zones-recurse+=us.=8.8.8.8

# 默认域名使用 Cloudflare + Google + Quad9
forward-zones-recurse+=.=1.1.1.1;8.8.8.8;9.9.9.9
```

**匹配规则**:
- 按照配置顺序匹配
- 第一条必须不带 `+=`（定义默认规则）
- 支持多个上游 DNS，用 `;` 分隔
- 支持任意域名后缀（如 `.local`, `.corp` 等）

**添加新规则示例**:
```conf
# 添加 .jp 域名使用日本 DNS
forward-zones-recurse+=jp.=203.0.113.1;203.0.113.2

# 添加 .local 域名使用内网 DNS
forward-zones-recurse+=local.=10.0.0.1
```

### Lua 域名劫持配置

在 `templates/override.lua.j2` 中定义劫持规则：

```lua
local overrides = {
    ["www.baidu.com."] = "10.0.0.8",
    ["ad.example.com."] = "127.0.0.1",  -- 广告域名返回本地
    ["internal.company."] = "192.168.1.100",  -- 内部服务重定向
}

function preresolve(dq)
    local qname = dq.qname:toString():lower()
    local ip = overrides[qname]

    if ip and dq.qtype == pdns.A then
        dq:addAnswer(pdns.A, ip)
        dq.rcode = 0
        return true
    end

    return false
end
```

**注意事项**:
- 域名必须以 `.` 结尾（FQDN 格式）
- 仅支持 A 记录劫持（IPv4）
- Lua 劫持优先级高于 `forward-zones-recurse`
- 修改后需重启服务: `systemctl restart pdns-recursor`

---

## 部署流程

### 完整部署流程图

```
┌─────────────────────────────────────────────────────────────┐
│ 1. 系统检测                                                   │
│    ✓ 验证 Debian 12                                          │
│    ✗ 非 Debian 12 → 中断部署                                 │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. 环境准备                                                   │
│    ✓ 检测 systemd-resolved 状态                              │
│    ✓ 备份 /etc/resolv.conf                                   │
│    ✓ 禁用 systemd-resolved                                   │
│    ✓ 配置外部 DNS (1.1.1.1, 8.8.8.8)                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. 软件安装                                                   │
│    ✓ apt update                                              │
│    ✓ 安装 pdns-recursor                                      │
│    ✓ 安装 dnsutils (dig)                                     │
│    ✓ 安装 curl, jq, tcpdump                                  │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 配置部署                                                   │
│    ✓ 创建 /etc/powerdns 目录                                 │
│    ✓ 部署 recursor.conf                                      │
│    ✓ 部署 override.lua                                       │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. 服务启动                                                   │
│    ✓ 启用 pdns-recursor (systemctl enable)                   │
│    ✓ 启动服务 (systemctl start)                              │
│    ✓ 强制重启（确保配置生效）                                 │
│    ✓ 等待端口 53 就绪                                         │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. 功能验证                                                   │
│    ✓ 验证端口监听 (0.0.0.0:53)                               │
│    ✓ 测试 Lua 劫持 (www.baidu.com → 10.0.0.8)                │
│    ✓ 测试 DNS 转发 (google.com)                              │
│    ✓ 测试 Web API (127.0.0.1:8082)                           │
└─────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. 输出报告                                                   │
│    📊 部署摘要                                                │
│    - 服务器信息                                               │
│    - 系统版本                                                 │
│    - 服务状态                                                 │
│    - 功能测试结果                                             │
│    - API 密钥                                                 │
│    - 配置文件路径                                             │
└─────────────────────────────────────────────────────────────┘
```

### 部署命令详解

#### 基础部署
```bash
ansible-playbook -i inventory.ini playbook.yml
```

#### 仅部署到特定节点
```bash
ansible-playbook -i inventory.ini playbook.yml --limit recursor1
```

#### 并行部署（加速）
```bash
# 同时在 4 台服务器上并行执行
ansible-playbook -i inventory.ini playbook.yml -f 4
```

#### 检查模式（Dry Run）
```bash
# 不实际执行，仅显示将要执行的操作
ansible-playbook -i inventory.ini playbook.yml --check
```

#### 详细输出模式
```bash
# 显示详细的执行日志
ansible-playbook -i inventory.ini playbook.yml -vvv
```

---

## 验证测试

### 自动化验证

部署完成后，playbook 会自动执行以下验证任务：

| 测试项 | 验证内容 | 预期结果 |
|--------|---------|---------|
| **端口监听** | `ss -nlup \| grep :53` | `0.0.0.0:53` 存在 |
| **Lua 劫持** | `dig @127.0.0.1 www.baidu.com` | 返回 `10.0.0.8` |
| **DNS 转发** | `dig @127.0.0.1 google.com` | 返回真实 IP 列表 |
| **Web API** | `curl http://127.0.0.1:8082/api/...` | HTTP 200 状态码 |

### 手动验证

#### 1. 检查服务状态
```bash
ssh root@YOUR_SERVER_IP
systemctl status pdns-recursor

# 期望输出:
# ● pdns-recursor.service - PowerDNS Recursor
#      Loaded: loaded
#      Active: active (running)
```

#### 2. 测试公网访问
```bash
# 从本地测试（替换为你的服务器 IP）
dig @152.42.186.194 google.com

# 期望输出：
# ;; ANSWER SECTION:
# google.com.    300    IN    A    142.250.xxx.xxx
```

#### 3. 测试 Lua 劫持
```bash
dig @152.42.186.194 www.baidu.com

# 期望输出：
# ;; ANSWER SECTION:
# www.baidu.com.    3600    IN    A    10.0.0.8
```

#### 4. 测试差异化转发
```bash
# 测试 .cn 域名（可能超时，正常现象）
dig @152.42.186.194 www.qq.cn +time=5

# 测试 .us 域名
dig @152.42.186.194 example.us +short
```

#### 5. 查看 Web API 统计
```bash
ssh root@152.42.186.194

# 替换为你的实际 API 密钥
curl -H "X-API-Key: 50GQacjn5x0NUTC3U7SDwV1PUpt5JERRJf2Eb346pOc=" \
  http://127.0.0.1:8082/api/v1/servers/localhost/statistics | jq .

# 输出示例：
# [
#   {"name": "questions", "type": "StatisticItem", "value": "1234"},
#   {"name": "cache-hits", "type": "StatisticItem", "value": "567"},
#   ...
# ]
```

---

## 故障排查

### 常见问题及解决方案

#### ❌ 公网 DNS 查询超时

**现象**:
```bash
dig @152.42.186.194 google.com
# 超时无响应
```

**原因**:
1. DigitalOcean 云防火墙阻止了 53 端口
2. `allow-from` 配置不正确
3. 服务未监听在 `0.0.0.0`

**解决方案**:

```bash
# 1. 检查端口监听状态
ssh root@152.42.186.194
ss -nlup | grep :53

# 应该显示: 0.0.0.0:53
# 如果显示: 127.0.0.1:53，则需要重启服务

systemctl restart pdns-recursor
ss -nlup | grep :53  # 再次检查

# 2. 检查配置文件
cat /etc/powerdns/recursor.conf | grep -E 'local-address|allow-from'

# 应该显示:
# local-address=0.0.0.0
# allow-from=0.0.0.0/0

# 3. 检查 DigitalOcean 防火墙
# 登录 DO 控制台 → Networking → Firewalls
# 添加规则: UDP/TCP 53，来源: All IPv4 或你的特定 IP 段
```

---

#### ❌ Ansible 连接失败

**现象**:
```
fatal: [recursor1]: FAILED! => {"msg": "Using a SSH password instead of a key is not possible..."}
```

**解决方案**:
```bash
# 方法 1: 添加 SSH 密钥免密登录
ssh-keygen -t rsa -b 4096
ssh-copy-id root@152.42.186.194

# 删除 inventory.ini 中的 ansible_ssh_pass 行

# 方法 2: 已在配置中添加，确保有以下行
# ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

---

#### ❌ systemd-resolved 未被禁用

**现象**:
```
bind: Address already in use
```

**解决方案**:
```bash
ssh root@152.42.186.194

# 手动禁用
systemctl stop systemd-resolved
systemctl disable systemd-resolved

# 修改 resolv.conf
echo "nameserver 1.1.1.1" > /etc/resolv.conf
echo "nameserver 8.8.8.8" >> /etc/resolv.conf

# 重新运行 playbook
```

---

#### ❌ Lua 脚本不生效

**现象**:
```bash
dig @127.0.0.1 www.baidu.com
# 返回真实 IP，而不是 10.0.0.8
```

**解决方案**:
```bash
ssh root@152.42.186.194

# 检查 Lua 脚本是否存在
cat /etc/powerdns/override.lua

# 检查配置文件引用
grep lua-dns-script /etc/powerdns/recursor.conf
# 应该显示: lua-dns-script=/etc/powerdns/override.lua

# 检查日志是否有 Lua 错误
journalctl -u pdns-recursor -f

# 重启服务
systemctl restart pdns-recursor

# 再次测试
dig @127.0.0.1 www.baidu.com +short
# 应返回: 10.0.0.8
```

---

#### ❌ Web API 无法访问

**现象**:
```bash
curl http://152.42.186.194:8082/api/...
# Connection refused 或 Empty reply
```

**解决方案**:
```bash
# 1. 从服务器本地测试
ssh root@152.42.186.194
curl -H "X-API-Key: YOUR_KEY" http://127.0.0.1:8082/api/v1/servers/localhost/statistics

# 如果本地可访问，说明防火墙阻止了 8082 端口
# 建议: 仅在内网访问 API，不要对公网开放

# 2. 检查配置
grep webserver /etc/powerdns/recursor.conf
# 应显示:
# webserver=yes
# webserver-address=0.0.0.0
# webserver-port=8082
# api-key=YOUR_KEY
```

---

#### ❌ .cn 域名解析超时

**现象**:
```bash
dig @152.42.186.194 www.qq.cn
# 超时
```

**原因**: 从境外服务器访问国内 DNS（114.114.114.114 等）存在网络延迟和丢包。

**解决方案**:

**方案 1**: 移除 .cn 转发规则（推荐）
```conf
# 编辑 templates/recursor.conf.j2
# 删除或注释掉这一行:
# forward-zones-recurse=cn.=114.114.114.114;223.5.5.5;119.29.29.29

# 重新部署
ansible-playbook -i inventory.ini playbook.yml
```

**方案 2**: 使用更稳定的国内 DNS
```conf
# 使用 DNSPod 或阿里云 DNS
forward-zones-recurse=cn.=119.29.29.29;223.5.5.5
```

**方案 3**: 针对特定地区部署
```yaml
# 国内节点使用国内 DNS
# 国外节点不配置 .cn 转发
# 在 inventory.ini 中使用 host_vars 实现
```

---

### 日志查看

#### 实时查看日志
```bash
ssh root@152.42.186.194
journalctl -u pdns-recursor -f
```

#### 查看最近 100 条日志
```bash
journalctl -u pdns-recursor -n 100
```

#### 查看特定时间段日志
```bash
journalctl -u pdns-recursor --since "2025-12-14 10:00:00" --until "2025-12-14 12:00:00"
```

#### 查询特定域名的解析日志
```bash
# 开启追踪
rec_control trace-regex 'baidu'

# 执行查询
dig @127.0.0.1 www.baidu.com

# 查看日志
journalctl -u pdns-recursor -f

# 关闭追踪
rec_control trace-regex
```

---

## 高级配置

### 多环境部署

使用 Ansible 的 `group_vars` 和 `host_vars` 实现不同环境的配置。

#### 目录结构
```
.
├── inventory/
│   ├── production.ini       # 生产环境主机
│   ├── staging.ini          # 测试环境主机
│   ├── group_vars/
│   │   ├── production.yml   # 生产环境变量
│   │   └── staging.yml      # 测试环境变量
│   └── host_vars/
│       ├── recursor1.yml    # 节点 1 特定变量
│       └── recursor2.yml    # 节点 2 特定变量
```

#### 生产环境配置 (`group_vars/production.yml`)
```yaml
pdns_allow_from: "203.0.113.0/24,198.51.100.0/24"  # 仅允许特定 IP 段
pdns_threads: 4
pdns_api_key: "{{ vault_pdns_api_key }}"  # 使用 Ansible Vault 加密
```

#### 测试环境配置 (`group_vars/staging.yml`)
```yaml
pdns_allow_from: "0.0.0.0/0"  # 测试环境允许所有 IP
pdns_threads: 2
pdns_api_key: "test-key-123"
```

#### 部署到特定环境
```bash
# 部署到生产环境
ansible-playbook -i inventory/production.ini playbook.yml

# 部署到测试环境
ansible-playbook -i inventory/staging.ini playbook.yml
```

---

### 使用 Ansible Vault 加密敏感信息

#### 创建加密文件
```bash
ansible-vault create group_vars/production.yml
```

#### 编辑加密文件
```bash
ansible-vault edit group_vars/production.yml
```

#### 部署时提供密码
```bash
ansible-playbook -i inventory.ini playbook.yml --ask-vault-pass
```

---

### 添加自定义上游 DNS

编辑 `templates/recursor.conf.j2`:

```conf
# 添加公司内部域名解析
forward-zones-recurse+=company.internal.=10.0.0.53

# 添加特定国家/地区 DNS
forward-zones-recurse+=jp.=203.0.113.1;203.0.113.2
forward-zones-recurse+=uk.=198.51.100.1

# 添加私有网络域名
forward-zones-recurse+=lan.=192.168.1.1
```

---

### 性能调优

#### 高并发场景（QPS > 10000）
```conf
# templates/recursor.conf.j2

# 增加线程数（建议与 CPU 核心数相同）
threads={{ pdns_threads }}  # 设置为 4 或 8

# 增加缓存容量
max-cache-entries=5000000
max-packetcache-entries=1000000

# 增加并发连接数
max-mthreads=4096
max-tcp-clients=1024

# 启用 TCP Fast Open
tcp-fast-open=100
```

#### 内存受限场景（< 1GB RAM）
```conf
# 减少缓存容量
max-cache-entries=500000
max-packetcache-entries=100000

# 减少线程数
threads=1
```

#### 启用 DNSSEC 验证
```conf
# 增强安全性，但会增加解析延迟
dnssec=validate
```

---

### 集成监控

#### Prometheus Exporter

安装 PowerDNS Recursor Exporter:

```bash
# 在 playbook.yml 中添加任务
- name: Install PowerDNS Recursor Exporter
  shell: |
    wget https://github.com/janeczku/powerdns_exporter/releases/download/v1.0.0/powerdns_exporter
    chmod +x powerdns_exporter
    mv powerdns_exporter /usr/local/bin/
    
    # 创建 systemd 服务
    cat > /etc/systemd/system/pdns-exporter.service <<EOF
    [Unit]
    Description=PowerDNS Recursor Exporter
    After=network.target

    [Service]
    Type=simple
    ExecStart=/usr/local/bin/powerdns_exporter --api-url=http://127.0.0.1:8082/api/v1 --api-key={{ pdns_api_key }}
    Restart=always

    [Install]
    WantedBy=multi-user.target
    EOF
    
    systemctl daemon-reload
    systemctl enable pdns-exporter
    systemctl start pdns-exporter
```

Prometheus 配置:
```yaml
scrape_configs:
  - job_name: 'powerdns-recursor'
    static_configs:
      - targets: ['152.42.186.194:9120']
```

---

## 生产环境建议

### 1. 高可用架构

#### 多节点部署（推荐 4+ 节点）

```ini
# inventory.ini
[dns_servers]
recursor1 ansible_host=152.42.186.194   # 新加坡 DigitalOcean
recursor2 ansible_host=203.0.113.10     # 东京 Vultr
recursor3 ansible_host=198.51.100.20    # 新加坡 AWS
recursor4 ansible_host=192.0.2.30       # 香港 Azure
```

#### 客户端配置（路由器示例）

将路由器 DNS 设置为多个节点，实现自动故障转移：

```
Primary DNS:   152.42.186.194
Secondary DNS: 203.0.113.10
Tertiary DNS:  198.51.100.20
```

#### 负载均衡策略

将 400 台路由器分为 4 组，每组使用不同的节点顺序：

| 路由器组 | DNS 1 | DNS 2 | DNS 3 |
|---------|-------|-------|-------|
| Group A (100台) | Rec1 | Rec2 | Rec3 |
| Group B (100台) | Rec2 | Rec3 | Rec4 |
| Group C (100台) | Rec3 | Rec4 | Rec1 |
| Group D (100台) | Rec4 | Rec1 | Rec2 |

---

### 2. 安全加固

#### 防火墙配置（DigitalOcean 示例）

```bash
# 创建 Cloud Firewall 规则

# Inbound Rules:
UDP 53: 203.0.113.0/24  # 仅允许你的路由器公网 IP 段
TCP 53: 203.0.113.0/24
TCP 22: YOUR_ADMIN_IP   # SSH 仅允许管理员 IP

# Outbound Rules:
All Traffic: All IPv4   # 允许访问上游 DNS
```

#### 定期更新 API 密钥

```bash
# 生成新密钥
NEW_KEY=$(openssl rand -base64 32)

# 更新 playbook.yml
vim playbook.yml
# 修改 pdns_api_key: "NEW_KEY"

# 重新部署
ansible-playbook -i inventory.ini playbook.yml
```

#### 启用速率限制

在 `templates/recursor.conf.j2` 中添加：

```conf
# 限制每个 IP 每秒最多 100 次查询
max-qperq=100

# 限制每个 IP 每秒最多 1000 次查询（全局）
queries-per-ip=1000
```

---

### 3. 备份与恢复

#### 备份配置文件

```bash
# 在 playbook 中添加备份任务
- name: Backup configuration files
  archive:
    path:
      - /etc/powerdns/recursor.conf
      - /etc/powerdns/override.lua
    dest: /root/pdns-backup-{{ ansible_date_time.date }}.tar.gz
```

#### 恢复配置

```bash
ssh root@152.42.186.194
tar -xzf /root/pdns-backup-2025-12-14.tar.gz -C /
systemctl restart pdns-recursor
```

---

### 4. 监控告警

推荐监控指标：

| 指标 | 告警阈值 | 说明 |
|------|---------|------|
| 服务状态 | 非 active | 服务异常 |
| QPS | < 10 或 > 50000 | 流量异常 |
| 缓存命中率 | < 80% | 缓存效率低 |
| 响应时间 | > 100ms | 解析延迟高 |
| CPU 使用率 | > 80% | 需要扩容 |
| 内存使用率 | > 90% | 内存不足 |

---

## 常见问题

### Q1: 为什么必须使用 Debian 12？

**A**: 本项目经过在 Debian 12 环境下充分测试，其他发行版可能存在以下问题：
- 软件包版本不一致
- systemd 配置差异
- 文件路径不同

如需支持其他发行版，需要修改 playbook 中的系统检测和软件包安装逻辑。

---

### Q2: 可以在 Docker 中运行吗？

**A**: 可以，但需要修改部署方式：

```dockerfile
# Dockerfile
FROM debian:12
RUN apt update && apt install -y pdns-recursor
COPY recursor.conf /etc/powerdns/recursor.conf
COPY override.lua /etc/powerdns/override.lua
EXPOSE 53/udp 53/tcp 8082/tcp
CMD ["pdns_recursor", "--daemon=no"]
```

```bash
# 构建镜像
docker build -t powerdns-recursor .

# 运行容器
docker run -d -p 53:53/udp -p 53:53/tcp --name dns powerdns-recursor
```

---

### Q3: 如何实现 Anycast DNS？

**A**: Anycast 需要 BGP 路由支持，步骤如下：

1. 申请 ASN 和 IP 段
2. 在多个地理位置部署相同 IP 的 DNS 服务器
3. 通过 BGP 宣告路由
4. 客户端自动连接到最近的节点

推荐使用云服务商的 Anycast 方案（如 Cloudflare、Google Cloud）。

---

### Q4: 支持 IPv6 吗？

**A**: 支持，需要修改配置：

```conf
# templates/recursor.conf.j2
local-address=0.0.0.0,[::]  # 同时监听 IPv4 和 IPv6
```

---

### Q5: 如何限制特定域名的查询？

**A**: 使用 Lua 脚本：

```lua
-- templates/override.lua.j2
local blocked_domains = {
    ["ad.example.com."] = true,
    ["malware.test."] = true,
}

function preresolve(dq)
    local qname = dq.qname:toString():lower()
    
    if blocked_domains[qname] then
        dq.rcode = pdns.NXDOMAIN  -- 返回域名不存在
        return true
    end
    
    return false
end
```

---

## 维护指南

### 日常维护任务

#### 每周
- 检查服务运行状态
- 查看日志是否有异常
- 监控 QPS 和缓存命中率

#### 每月
- 更新系统软件包: `apt update && apt upgrade`
- 备份配置文件
- 审查防火墙规则

#### 每季度
- 更新 PowerDNS Recursor 到最新稳定版
- 审查 Lua 劫持规则，删除过时配置
- 性能调优和容量规划

---

### 升级 PowerDNS Recursor

```bash
ssh root@152.42.186.194

# 备份配置
cp /etc/powerdns/recursor.conf /root/recursor.conf.backup
cp /etc/powerdns/override.lua /root/override.lua.backup

# 更新软件包
apt update
apt upgrade pdns-recursor -y

# 重启服务
systemctl restart pdns-recursor

# 验证版本
pdns_recursor --version
```

---

### 扩容新节点

```bash
# 1. 在 inventory.ini 中添加新节点
vim inventory.ini
# 添加: recursor5 ansible_host=NEW_IP

# 2. 仅部署到新节点
ansible-playbook -i inventory.ini playbook.yml --limit recursor5

# 3. 验证新节点
dig @NEW_IP google.com
```

---

### 回滚配置

```bash
# 1. SSH 到目标服务器
ssh root@152.42.186.194

# 2. 恢复备份
cp /root/recursor.conf.backup /etc/powerdns/recursor.conf
cp /root/override.lua.backup /etc/powerdns/override.lua

# 3. 重启服务
systemctl restart pdns-recursor
```

---

## 技术架构

### 系统架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端（400 台路由器）                      │
│                 DNS Query: google.com / www.baidu.com            │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│                      负载均衡（Round Robin）                       │
│          Primary: Rec1 → Secondary: Rec2 → Tertiary: Rec3        │
└────────────┬──────────────┬─────────────────┬───────────────────┘
             │              │                 │
    ┌────────┴────┐  ┌─────┴──────┐  ┌──────┴────────┐
    │   Recursor1  │  │  Recursor2  │  │   Recursor3   │
    │  Singapore   │  │    Tokyo    │  │   Hong Kong   │
    │  DO/AWS/Vultr│  │   Vultr     │  │     Azure      │
    └────────┬────┘  └─────┬──────┘  └──────┬────────┘
             │              │                 │
    ┌────────┴──────────────┴─────────────────┴────────┐
    │         PowerDNS Recursor 核心处理流程             │
    │                                                    │
    │  1. Lua preresolve (域名劫持)                      │
    │     ↓ www.baidu.com → 10.0.0.8                    │
    │                                                    │
    │  2. Forward Zones (差异化转发)                     │
    │     ↓ .cn → 114.114.114.114                       │
    │     ↓ .us → 8.8.8.8                               │
    │     ↓ .   → 1.1.1.1/8.8.8.8/9.9.9.9               │
    │                                                    │
    │  3. Cache (缓存)                                   │
    │     ↓ 200万条记录 + 50万包缓存                      │
    │                                                    │
    │  4. Recursive Resolution (递归解析)                │
    │     ↓ 根服务器 → TLD → 权威 NS                      │
    └────────────────────────┬───────────────────────────┘
                             │
             ┌───────────────┴────────────────┐
             │                                │
    ┌────────▼─────────┐          ┌──────────▼──────────┐
    │  上游 DNS         │          │   根服务器           │
    │                  │          │   TLD 服务器         │
    │ • Cloudflare     │          │   权威 NS            │
    │   1.1.1.1        │          │                     │
    │ • Google         │          │   (仅当无转发规则时) │
    │   8.8.8.8        │          │                     │
    │ • Quad9          │          │                     │
    │   9.9.9.9        │          │                     │
    │ • 114 DNS        │          │                     │
    └──────────────────┘          └─────────────────────┘
```

### 数据流示例

#### 场景 1: 查询 www.baidu.com（Lua 劫持）

```
Client → Recursor → Lua preresolve → 返回 10.0.0.8
```

**流程**:
1. 客户端发起查询: `dig www.baidu.com`
2. Recursor 执行 Lua 脚本
3. 匹配到劫持规则: `["www.baidu.com."] = "10.0.0.8"`
4. 直接返回: `10.0.0.8`
5. 不查询上游 DNS

---

#### 场景 2: 查询 google.com（默认转发）

```
Client → Recursor → 检查缓存 → 转发到 1.1.1.1 → 返回结果 → 缓存
```

**流程**:
1. 客户端发起查询: `dig google.com`
2. Lua 脚本未匹配
3. 检查缓存，未命中
4. 匹配 `.=1.1.1.1;8.8.8.8;9.9.9.9` 规则
5. 转发到 Cloudflare (1.1.1.1)
6. 接收结果: `142.250.xxx.xxx`
7. 写入缓存
8. 返回给客户端

---

#### 场景 3: 查询 www.qq.cn（.cn 转发）

```
Client → Recursor → 转发到 114.114.114.114 → 超时/返回结果
```

**流程**:
1. 客户端发起查询: `dig www.qq.cn`
2. 匹配 `cn.=114.114.114.114;223.5.5.5;119.29.29.29` 规则
3. 转发到 114.114.114.114
4. **可能超时**（境外访问国内 DNS 不稳定）
5. 失败后尝试 223.5.5.5
6. 最终返回结果或 SERVFAIL

---

### 技术栈

| 组件 | 版本 | 作用 |
|------|------|------|
| **Ansible** | 2.20+ | 自动化部署工具 |
| **PowerDNS Recursor** | 4.8.8 | DNS 递归解析服务 |
| **Lua** | 5.1+ | 域名劫持脚本语言 |
| **Debian** | 12 (Bookworm) | 操作系统 |
| **systemd** | 252 | 服务管理 |
| **Jinja2** | 3.0+ | 配置模板引擎 |

---

## 贡献指南

欢迎贡献代码、文档和问题反馈！

### 贡献流程

1. **Fork 项目**
   ```bash
   git clone https://github.com/your-username/powerdns-recursor-ansible.git
   cd powerdns-recursor-ansible
   ```

2. **创建分支**
   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **提交修改**
   ```bash
   git add .
   git commit -m "Add: your feature description"
   ```

4. **推送分支**
   ```bash
   git push origin feature/your-feature-name
   ```

5. **创建 Pull Request**
   - 访问 GitHub 仓库
   - 点击 "New Pull Request"
   - 填写详细的修改说明

### 代码规范

遵循 `gemini.md` 中定义的开发规范：

- ✅ 所有代码必须添加注释
- ✅ 日志必须使用英文
- ✅ 采用渐进式开发策略
- ✅ 提交前运行测试验证

### 提交 Bug

创建 Issue 时请包含：

- **操作系统**: Debian 12 / Ubuntu 22.04 等
- **Ansible 版本**: `ansible --version` 输出
- **PowerDNS 版本**: `pdns_recursor --version` 输出
- **错误信息**: 完整的日志输出
- **复现步骤**: 详细的操作步骤

### 功能建议

欢迎提出新功能建议，请在 Issue 中详细描述：

- **使用场景**: 什么情况下需要此功能
- **预期效果**: 功能应该如何工作
- **技术方案**: 你的初步实现思路（可选）

---

## 许可证

本项目采用 [MIT License](LICENSE)。

```
MIT License

Copyright (c) 2025 PowerDNS Recursor Ansible Project

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

## 联系方式

- **项目主页**: https://github.com/your-repo/powerdns-recursor-ansible
- **Issues**: https://github.com/your-repo/powerdns-recursor-ansible/issues
- **Email**: your-email@example.com
- **文档**: https://your-docs-site.com

---

## 致谢

感谢以下项目和资源：

- [PowerDNS](https://www.powerdns.com/) - 优秀的开源 DNS 软件
- [Ansible](https://www.ansible.com/) - 强大的自动化工具
- [DigitalOcean](https://www.digitalocean.com/) - 稳定的云平台
- [Debian](https://www.debian.org/) - 可靠的操作系统

---

## 更新日志

### v1.0.0 (2025-12-14)

**初始版本发布**

- ✅ 支持 Debian 12 自动化部署
- ✅ 差异化 DNS 转发（.cn/.us/默认）
- ✅ Lua 域名劫持功能
- ✅ Web API 监控
- ✅ DigitalOcean 平台优化
- ✅ 自动化验证和测试
- ✅ 完整文档和故障排查指南

---

**项目维护者**: GitHub Copilot & Community Contributors  
**最后更新**: 2025-12-14  
**文档版本**: 1.0.0
