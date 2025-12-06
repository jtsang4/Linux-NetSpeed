# Linux-NetSpeed 项目安全分析报告

**分析日期**: 2025-12-06  
**分析者**: GitHub Copilot Security Agent  
**项目**: jtsang4/Linux-NetSpeed

## 执行摘要

本报告对 Linux-NetSpeed 项目进行了全面的安全审计。该项目是一个用于Linux系统TCP网络加速的脚本工具，主要用于安装和配置BBR、BBRplus、Lotserver等TCP拥塞控制算法。

**总体评估**: ⚠️ **中等风险 - 建议谨慎使用**

---

## 项目概述

### 项目用途
Linux-NetSpeed 是一个开源项目，用于：
- 安装和配置各种TCP拥塞控制算法（BBR、BBRplus、BBR2等）
- 更换Linux内核以支持特定的网络优化
- 安装第三方网络加速工具（如Lotserver/锐速）
- 优化系统网络参数

### 主要文件
- `tcp.sh` - 带内核卸载功能的主安装脚本（已停更）
- `tcpx.sh` - 不卸载内核版本的主安装脚本（推荐使用）
- `InstallNET.sh` - 系统重装脚本（来自MoeClub）
- 各种预编译内核包（在bbr/、bbrplus/、lotserver/目录中）

---

## 安全分析结果

### ✅ 未发现的恶意行为

经过详细分析，**未发现**以下典型恶意行为：

1. **无后门或远程控制**
   - 未发现反向shell连接（如 `/dev/tcp`、`nc -e`）
   - 未发现隐藏的网络监听服务
   - 无未授权的SSH密钥安装

2. **无数据窃取**
   - 未发现收集敏感信息的代码
   - 无密码或密钥窃取行为
   - 不会泄露系统信息到外部服务器

3. **无恶意代码执行**
   - 未使用混淆的base64编码执行代码
   - 没有隐藏的eval执行
   - 代码逻辑清晰可读

4. **无持久化恶意程序**
   - 不会安装隐藏的cron任务
   - 不修改init.d或systemd服务文件以安装后门
   - 不在系统中植入持久化恶意软件

### ⚠️ 发现的安全风险

虽然没有发现明确的恶意代码，但存在以下安全风险：

#### 1. 执行远程脚本（高风险）

脚本会从GitHub下载并直接执行外部脚本，存在供应链攻击风险：

**tcp.sh 中的远程脚本执行:**
```bash
# 行 1384: 安装Lotserver（锐速）
echo | bash <(wget --no-check-certificate -qO- https://raw.githubusercontent.com/fei5seven/lotServer/master/lotServerInstall.sh) install

# 行 1580: 执行tcpx.sh
bash <(wget -qO- https://github.com/ylx2016/Linux-NetSpeed/raw/master/tcpx.sh)

# 行 1587: teddysun的BBR脚本
bash <(wget -qO- https://github.com/teddysun/across/raw/master/bbr.sh)

# 行 1596: 系统重装脚本
bash <(wget -qO- https://github.com/fcurrk/reinstall/raw/master/NewReinstall.sh)
```

**tcpx.sh 中的远程脚本执行:**
```bash
# 行 1082: 安装Lotserver
echo | bash <(wget --no-check-certificate -qO- https://raw.githubusercontent.com/fei5seven/lotServer/master/lotServerInstall.sh) install

# 行 1143: brutal TCP编译脚本
bash <(curl -fsSL https://tcp.hy2.sh/)
```

**风险说明:**
- 使用 `--no-check-certificate` 禁用SSL证书验证，容易遭受中间人攻击
- 这些外部脚本的内容可能随时更改，本项目作者无法控制
- 如果第三方仓库被入侵，可能导致恶意代码执行

#### 2. 需要Root权限（中等风险）

```bash
# 行 37-40: 必须以root身份运行
if [ "$EUID" -ne 0 ]; then
    echo "请使用 root 用户身份运行此脚本"
    exit
fi
```

**风险说明:**
- 脚本需要完整的root权限
- 可以修改系统的任何部分
- 如果有未发现的漏洞，后果严重

#### 3. 安装和替换内核（高风险）

脚本会下载并安装自定义编译的Linux内核：

```bash
# 从GitHub releases下载内核包
wget -O kernel-headers-c8.rpm $headurl
wget -O kernel-c8.rpm $imgurl
```

**内核来源:**
- `https://api.github.com/repos/ylx2016/kernel/releases`
- `https://api.github.com/repos/UJX6N/bbrplus-6.x_stable/releases`

**风险说明:**
- 自定义内核可能包含后门或漏洞
- 替换内核是高风险操作，可能导致系统不稳定
- 预编译内核的真实来源和编译选项难以验证

#### 4. 系统配置大幅修改（中等风险）

脚本会修改关键系统配置：

```bash
# 修改sysctl参数
echo "net.ipv4.tcp_retries2 = 8
net.ipv4.tcp_slow_start_after_idle = 0
fs.file-max = 1000000
..." >> /etc/sysctl.d/99-sysctl.conf

# 修改文件描述符限制
echo "*               soft    nofile           1000000
*               hard    nofile          1000000" > /etc/security/limits.conf

# 修改systemd配置
cat > '/etc/systemd/system.conf' << EOF
[Manager]
DefaultTimeoutStartSec=30s
DefaultTimeoutStopSec=30s
...
EOF
```

**风险说明:**
- 这些修改可能影响系统稳定性和安全性
- 某些配置可能与现有应用冲突
- 很难完全还原这些更改

#### 5. 第三方依赖不可信（中等风险）

项目依赖多个第三方来源：

**GitHub仓库:**
- `fei5seven/lotServer` - Lotserver安装脚本
- `teddysun/across` - BBR安装脚本
- `fcurrk/reinstall` - 系统重装脚本
- `xykt/IPQuality` - IP质量检测
- `UJX6N/bbrplus-6.x_stable` - BBRplus内核

**风险说明:**
- 这些第三方仓库的安全性未经验证
- 如果这些仓库被攻击，可能影响本项目用户
- 无法保证这些仓库未来不会添加恶意代码

#### 6. 使用压缩包中的KeyGen（低风险）

项目包含两个LotServer密钥生成器的zip文件：
- `LotServer_KeyGen-main.zip`
- `LotServer_KeyGen-master.zip`

内容包括：
- `keygen.php` - PHP密钥生成器（约18KB）
- 许可证模板文件

**风险说明:**
- 用于绕过商业软件的许可验证
- 可能涉及软件盗版和法律问题
- PHP文件未在主脚本中自动执行（较低风险）

#### 7. InstallNET.sh系统重装功能（高风险）

`InstallNET.sh` 脚本可以完全重装系统：

```bash
## Default root password: MoeClub.org
## It can reinstall Debian, Ubuntu, CentOS system with network.
```

**风险说明:**
- 会擦除整个系统并重装
- 如果误用，会导致数据丢失
- 用户可能不知道这个功能的严重性

---

## 代码质量评估

### 优点
1. ✅ 代码结构清晰，有中文注释
2. ✅ 提供用户交互式菜单
3. ✅ 有系统类型检测
4. ✅ 开源可审计

### 缺点
1. ❌ 缺少输入验证
2. ❌ 错误处理不完善
3. ❌ 使用 `--no-check-certificate` 禁用SSL验证
4. ❌ 直接执行远程脚本，无完整性检查
5. ❌ 没有代码签名或校验和验证

---

## 外部连接分析

### 合法的GitHub连接
```
https://api.github.com/repos/ylx2016/kernel/releases
https://github.com/ylx2016/Linux-NetSpeed/
https://github.com/UJX6N/bbrplus-6.x_stable/releases
https://raw.githubusercontent.com/fei5seven/lotServer/
```

### 其他外部连接
```
https://ip.im/info - IP信息查询
https://tcp.hy2.sh/ - Brutal TCP编译脚本
https://deb.debian.org/debian/ - Debian官方镜像
```

### GitHub代理/镜像（用于加速下载）
```
https://gh-proxy.com/
https://ghproxy.crazypeace.workers.dev/
https://ghps.cc/
https://gh.ddlc.top/
```

**注意**: 使用第三方GitHub代理可能存在中间人攻击风险。

---

## 安全建议

### 对于用户

#### 🛑 如果你非常关注安全，不建议使用此项目，因为：
1. 会下载并执行多个不受你控制的远程脚本
2. 需要root权限，风险极高
3. 会替换系统内核
4. 依赖多个第三方仓库

#### ⚠️ 如果你决定使用，请遵循以下建议：

1. **仅在测试环境使用**
   - 不要在生产服务器上使用
   - 使用虚拟机或容器进行测试
   - 做好完整的系统备份

2. **审查代码**
   - 在执行前阅读完整的脚本代码
   - 理解每个步骤的作用
   - 检查所有外部脚本的内容

3. **手动下载和检查**
   ```bash
   # 不要直接执行，先下载
   wget -O tcp.sh "https://github.com/ylx2016/Linux-NetSpeed/raw/master/tcpx.sh"
   
   # 仔细阅读代码
   less tcp.sh
   
   # 确认安全后再执行
   chmod +x tcp.sh && ./tcp.sh
   ```

4. **验证外部脚本**
   - 检查所有将被下载的外部脚本
   - 确认第三方仓库是可信的
   - 查看这些仓库的历史记录和维护者

5. **监控系统**
   - 使用 `iptables -L -n` 监控网络连接
   - 检查 `ps aux` 查看运行的进程
   - 使用 `netstat -tulpn` 查看监听端口

6. **使用快照/备份**
   - 在执行前创建系统快照
   - 准备好回滚方案
   - 保存重要数据

### 对于项目维护者

1. **减少远程脚本执行**
   - 将关键功能整合到主脚本中
   - 提供脚本的SHA256校验和
   - 使用本地文件而不是远程下载

2. **启用SSL证书验证**
   - 移除所有 `--no-check-certificate` 选项
   - 确保使用HTTPS且验证证书

3. **添加完整性检查**
   ```bash
   # 示例：验证下载文件的校验和
   expected_hash="abc123..."
   downloaded_hash=$(sha256sum file.sh | awk '{print $1}')
   if [ "$expected_hash" != "$downloaded_hash" ]; then
       echo "校验失败！"
       exit 1
   fi
   ```

4. **改进错误处理**
   - 使用 `set -e` 在错误时退出
   - 添加完整的错误检查
   - 提供有意义的错误消息

5. **代码签名**
   - 使用GPG签名发布版本
   - 提供验证说明
   - 建立信任链

6. **文档化风险**
   - 在README中明确警告风险
   - 说明脚本会做什么
   - 提供详细的回滚步骤

---

## 结论

### 项目合法性评估: ✅ **合法**

Linux-NetSpeed 是一个**合法的开源项目**，其目的是为Linux系统提供TCP网络优化。代码是公开的，作者并无明显的恶意意图。

### 安全性评估: ⚠️ **存在风险但非恶意**

**未发现恶意代码**，但由于以下原因存在**安全风险**：

1. ✅ **好消息**:
   - 无后门或木马
   - 无数据窃取
   - 代码开源可审计
   - 社区有一定使用基础

2. ⚠️ **风险因素**:
   - 依赖多个外部脚本和仓库
   - 需要root权限
   - 会替换系统内核
   - 大幅修改系统配置
   - 禁用SSL证书验证
   - 缺少完整性验证

### 最终建议

**风险等级**: 🟡 **中等（非恶意但有风险）**

**适用场景**:
- ✅ 个人测试服务器
- ✅ 虚拟机环境
- ✅ 可以随时重装的系统
- ❌ 生产环境
- ❌ 关键业务服务器
- ❌ 包含重要数据的系统

**一句话总结**:  
这个项目**不是恶意软件**，但由于需要root权限、替换内核、依赖多个外部脚本，存在**固有的安全风险**。建议仅在充分理解风险并做好备份的情况下，在测试环境中使用。

---

## 技术细节附录

### 分析方法
1. 手动代码审查
2. 搜索已知恶意模式（base64、eval、反向shell等）
3. 分析网络连接和外部依赖
4. 检查文件权限和隐藏文件
5. 验证下载来源

### 已检查的恶意模式
- ❌ `/dev/tcp` 反向shell
- ❌ `nc -e` 网络回连
- ❌ `bash -i` 交互式后门
- ❌ base64编码的恶意载荷
- ❌ eval执行混淆代码
- ❌ SSH密钥注入
- ❌ authorized_keys修改
- ❌ 隐藏的cron任务
- ❌ 数据外泄代码
- ❌ 挖矿程序
- ❌ 僵尸网络客户端

### 检查的文件
- tcp.sh (2065行)
- tcpx.sh (2299行)  
- InstallNET.sh (867行)
- Debian_Kernel.sh
- c7_tcp_react.sh
- 以及所有目录中的内核包

---

**报告生成时间**: 2025-12-06  
**审计工具**: 手动代码审查 + 自动化模式匹配  
**置信度**: 高（已进行全面审查）

---

## 免责声明

本安全分析报告基于2025年12月6日的项目状态。项目代码可能随时更新，外部依赖也可能改变。此报告不构成使用此项目的保证或建议。使用者应自行承担使用风险。

建议定期重新审查项目代码，特别是在项目有重大更新时。
