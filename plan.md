# PowerDNS Recursor 自动化部署规划文档

> **创建时间:** 2025-12-14  
> **项目目标:** 使用 Ansible 自动化部署 PowerDNS Recursor 集群，支持差异化转发、Lua 域名劫持、多节点架构

---

## 📋 需求分析

根据 `dnssetup.md` 和用户需求，本次需要完成以下工作：

### 1️⃣ **配置文件完善**
- **现状分析:**
  - 当前 `ecursor.conf.j2` 模板已包含核心配置
  - 与 `dnssetup.md` 中的生产级配置基本一致
  - 使用变量化管理关键参数（`pdns_allow_from`, `pdns_threads`, `pdns_api_key`）

- **对比结果:**
  | 配置项 | ecursor.conf.j2 当前状态 | dnssetup.md 推荐配置 | 是否需要修改 |
  |--------|-------------------------|---------------------|-------------|
  | `local-address` | ✅ `0.0.0.0` | ✅ `0.0.0.0` | ❌ 无需修改 |
  | `local-port` | ✅ `53` | ✅ `53` | ❌ 无需修改 |
  | `allow-from` | ✅ `{{ pdns_allow_from }}` | ✅ `0.0.0.0/0` | ⚠️ 需调整变量默认值 |
  | `threads` | ✅ `{{ pdns_threads }}` | ✅ `2` | ❌ 无需修改 |
  | `processes` | ✅ `1` | ✅ `1` | ❌ 无需修改 |
  | `max-cache-entries` | ✅ `2000000` | ✅ `2000000` | ❌ 无需修改 |
  | `max-packetcache-entries` | ✅ `500000` | ✅ `500000` | ❌ 无需修改 |
  | `packetcache-servfail-ttl` | ✅ `0` | ✅ `0` | ❌ 无需修改 |
  | `forward-zones-recurse` | ✅ 完整配置 | ✅ 完整配置 | ❌ 无需修改 |
  | `lua-dns-script` | ✅ `/etc/powerdns/override.lua` | ✅ `/etc/powerdns/override.lua` | ❌ 无需修改 |
  | `webserver` | ✅ `yes` | ✅ `yes` | ❌ 无需修改 |
  | `webserver-address` | ✅ `0.0.0.0` | ✅ `0.0.0.0` | ❌ 无需修改 |
  | `webserver-port` | ✅ `8082` | ✅ `8082` | ❌ 无需修改 |
  | `api-key` | ✅ `{{ pdns_api_key }}` | ✅ `CHANGEME123` | ❌ 无需修改 |
  | `hint-file` | ✅ `/usr/share/dns/root.hints` | ✅ `/usr/share/dns/root.hints` | ❌ 无需修改 |
  | `quiet` | ✅ `yes` | ✅ `yes` | ❌ 无需修改 |

- **结论:** 
  - ✅ **配置文件本身无需修改**
  - ⚠️ **需要调整 `playbook.yml` 中的变量默认值**（见后续方案）

---

### 2️⃣ **系统版本检测**
- **需求:** 确保目标机器运行 Debian 12
- **实现方式:**
  - 在 Ansible playbook 中添加系统版本检测任务
  - 如果版本不符，暂停执行并提示用户
  - 不自动升级系统（避免破坏性操作）

---

### 3️⃣ **禁用 systemd-resolved**
- **背景:** 
  - Debian 12 默认启用 `systemd-resolved` 占用 53 端口
  - PowerDNS Recursor 需要使用 53 端口，存在冲突
- **解决方案:**
  - 在安装 PowerDNS Recursor **之前**执行 `systemctl disable --now systemd-resolved`
  - 确保 53 端口被释放

---

## 🎯 实施方案

### 方案 A: playbook.yml 改进（推荐）

在现有 `playbook.yml` 基础上添加以下任务：

#### 1. **系统版本检测（第一步）**
```yaml
- name: Check Debian version
  assert:
    that:
      - ansible_distribution == "Debian"
      - ansible_distribution_major_version == "12"
    fail_msg: "This playbook requires Debian 12. Current system: {{ ansible_distribution }} {{ ansible_distribution_version }}"
    success_msg: "System check passed: Debian {{ ansible_distribution_version }}"
```

#### 2. **禁用 systemd-resolved（第二步）**
```yaml
- name: Disable systemd-resolved to free port 53
  systemd:
    name: systemd-resolved
    enabled: no
    state: stopped
  ignore_errors: yes  # 某些系统可能未安装此服务
  register: resolved_disabled

- name: Log systemd-resolved status
  debug:
    msg: "systemd-resolved disabled: {{ resolved_disabled.changed }}"
```

#### 3. **调整变量默认值**
```yaml
vars:
  pdns_allow_from: "0.0.0.0/0"  # DigitalOcean 关键配置
  pdns_threads: 2
  pdns_api_key: "CHANGEME123"
```

#### 4. **任务执行顺序**
```
1. 检测系统版本 (assert)
2. 禁用 systemd-resolved
3. 安装 PowerDNS Recursor 和工具
4. 创建配置目录
5. 部署 recursor.conf
6. 部署 override.lua
7. 启动并启用服务
```

---

### 方案 B: 添加预检脚本（可选）

创建独立的预检脚本 `pre-check.yml`：

```yaml
---
- name: Pre-deployment checks
  hosts: dns_servers
  become: yes
  tasks:
    - name: Verify Debian 12
      assert:
        that:
          - ansible_distribution == "Debian"
          - ansible_distribution_major_version == "12"
        fail_msg: "Requires Debian 12"

    - name: Check if port 53 is available
      wait_for:
        port: 53
        state: stopped
        timeout: 1
      ignore_errors: yes
      register: port_check

    - name: Disable systemd-resolved if port 53 is occupied
      systemd:
        name: systemd-resolved
        enabled: no
        state: stopped
      when: port_check.failed
```

执行方式：
```bash
ansible-playbook -i inventory.ini pre-check.yml
ansible-playbook -i inventory.ini playbook.yml
```

---

## 📝 待讨论问题

### 问题 1: 变量默认值设置
**当前配置:**
```yaml
vars:
  pdns_allow_from: "0.0.0.0/0"
  pdns_threads: 2
  pdns_api_key: "CHANGEME123"
```

**问题:**
- ✅ `allow-from=0.0.0.0/0` 符合 DigitalOcean 要求
- ⚠️ 但在生产环境中是否需要：
  - 通过 `inventory.ini` 的 `host_vars` 或 `group_vars` 覆盖不同节点的配置？
  - 例如某些节点只允许特定 IP 段？

**建议:**
- 保持 `playbook.yml` 中的默认值为 `0.0.0.0/0`
- 在 `inventory.ini` 中为不同节点设置不同的 ACL（如需要）

---

### 问题 2: 系统版本检测失败处理
**场景:** 如果目标机器不是 Debian 12，应该：
- ❌ 直接中断执行（当前方案）
- ✅ 提示用户手动升级系统
- ⚠️ 尝试自动兼容其他 Debian 版本？

**推荐:**
- 严格要求 Debian 12（符合 `dnssetup.md` 验证环境）
- 不自动升级系统（避免风险）

---

### 问题 3: systemd-resolved 禁用确认
**风险评估:**
- 禁用 `systemd-resolved` 后，系统本地 DNS 解析会受影响
- 如果机器本身需要通过 127.0.0.53 解析域名，可能导致问题

**建议:**
- 在禁用前检查 `/etc/resolv.conf` 是否指向 `127.0.0.53`
- 如果是，修改为使用外部 DNS（如 `1.1.1.1` 或 `8.8.8.8`）

**实现代码:**
```yaml
- name: Check if resolv.conf uses systemd-resolved
  shell: grep -q "127.0.0.53" /etc/resolv.conf
  register: uses_resolved
  ignore_errors: yes

- name: Update resolv.conf to use external DNS
  copy:
    content: |
      nameserver 1.1.1.1
      nameserver 8.8.8.8
    dest: /etc/resolv.conf
  when: uses_resolved.rc == 0

- name: Disable systemd-resolved
  systemd:
    name: systemd-resolved
    enabled: no
    state: stopped
```

---

### 问题 4: 文件名称修正
**发现问题:**
- 当前模板文件名为 `ecursor.conf.j2`
- 正确应为 `recursor.conf.j2`

**影响范围:**
- `playbook.yml` 中引用的是 `recursor.conf.j2`
- 但实际文件名为 `ecursor.conf.j2`
- **需要重命名文件或修改 playbook 引用**

**建议方案:**
```bash
# 重命名模板文件（推荐）
mv templates/ecursor.conf.j2 templates/recursor.conf.j2
```

或者修改 `playbook.yml`:
```yaml
- name: Deploy recursor.conf
  template:
    src: ecursor.conf.j2  # 改为实际文件名
    dest: /etc/powerdns/recursor.conf
```

---

## 🚀 执行计划

### 阶段 1: 方案确认（当前阶段）
- [ ] 用户确认变量默认值设置
- [ ] 用户确认系统版本检测策略
- [ ] 用户确认 systemd-resolved 禁用方案
- [ ] 用户确认文件名称修正方式

### 阶段 2: 代码实施
- [ ] 修改 `playbook.yml` 添加预检任务
- [ ] 调整变量默认值
- [ ] 修正模板文件名（如需要）
- [ ] 添加详细日志输出（符合 `gemini.md` 规范）

### 阶段 3: 测试验证
- [ ] 在单台测试机器上执行 playbook
- [ ] 验证系统版本检测功能
- [ ] 验证 systemd-resolved 禁用功能
- [ ] 验证 PowerDNS Recursor 正常启动
- [ ] 验证差异化转发功能
- [ ] 验证 Lua 域名劫持功能

### 阶段 4: 生产部署
- [ ] 更新 `inventory.ini` 中的真实 IP
- [ ] 批量部署到 4 台节点
- [ ] 配置 DigitalOcean 防火墙规则
- [ ] 配置路由器 DNS 设置

---

## 📌 注意事项

1. **符合 gemini.md 规范:**
   - ✅ 所有代码添加详细注释
   - ✅ 所有日志使用英文
   - ✅ 采用渐进式部署策略（先测试后生产）

2. **安全建议:**
   - ⚠️ 修改 `pdns_api_key` 默认值（不要使用 `CHANGEME123`）
   - ⚠️ 在 DigitalOcean 控制台配置防火墙规则
   - ⚠️ 限制 Web API 访问来源（仅允许管理 IP）

3. **备份建议:**
   - 在执行 playbook 前备份目标机器配置
   - 保留原始 `/etc/resolv.conf`

---

## 🤔 待用户反馈

请确认以下问题后，我将开始实施修改：

1. **变量默认值:** 是否保持 `pdns_allow_from: "0.0.0.0/0"`？
2. **系统版本:** 是否严格要求 Debian 12？
3. **resolv.conf:** 是否需要自动修改系统 DNS 配置？
4. **文件命名:** 是将 `ecursor.conf.j2` 改为 `recursor.conf.j2`，还是修改 playbook 引用？
5. **API 密钥:** 是否需要生成随机密钥替换 `CHANGEME123`？

---

**文档编写完成，等待用户反馈。**
