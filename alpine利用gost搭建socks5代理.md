# Alpine Linux 部署 GOST (v3) SOCKS5 代理完整指南

本文档介绍如何在 **Alpine Linux** 环境（含 LXC/LXD 容器、轻量云服务器或 VPS）中，从零搭建并运行基于 [GOST (v3)](https://github.com/go-gost/gost) 的带认证 SOCKS5 代理服务。

---

## 目录
- [一、环境准备](#一环境准备)
- [二、下载与安装 GOST](#二下载与安装-gost)
- [三、配置 OpenRC 服务与开机自启](#三配置-openrc-服务与开机自启)
- [四、连通性测试与客户端配置](#四连通性测试与客户端配置)
- [五、日常运维与服务管理](#五日常运维与服务管理)
- [六、内存与资源监控](#六内存与资源监控)
- [七、踩坑记录与排障指南（FAQ）](#七踩坑记录与排障指南faq)
- [八、彻底卸载与清理](#八彻底卸载与清理)

---

## 一、环境准备

* **操作系统**：Alpine Linux（支持 v3.x+）
* **权限要求**：`root` 权限
* **网络依赖**：安装基础下载与证书工具

执行以下命令安装依赖：
```bash
apk update
apk add curl tar ca-certificates
```

---

## 二、下载与安装 GOST

GOST 采用 Go 语言静态编译，直接下载对应架构的二进制文件即可运行。

1. **自动识别 CPU 架构并下载最新发布版**：
   ```bash
   [ "$(uname -m)" = "x86_64" ] && ARCH="amd64" || ARCH="arm64"
   curl -sSL -o /tmp/gost.tar.gz "https://github.com/go-gost/gost/releases/download/v3.0.0/gost_3.0.0_linux_${ARCH}.tar.gz"
   ```

2. **解压并移动至系统路径**：
   ```bash
   tar -zxvf /tmp/gost.tar.gz -C /usr/local/bin gost
   chmod +x /usr/local/bin/gost
   rm -f /tmp/gost.tar.gz
   ```

3. **验证安装**：
   ```bash
   gost -V
   ```
   若输出版本号（如 `gost 3.0.0`），说明二进制安装成功。

---

## 三、配置 OpenRC 服务与开机自启

Alpine 默认使用 **OpenRC** 替代 systemd。为确保服务稳定常驻、异常退出可排查，建议配置标准服务脚本及日志输出。

> [!IMPORTANT]
> 请在下述配置中，将 `10086`、`admin`、`your_password` 替换为你自定义的端口和账号密码。

### 1. 写入 OpenRC 服务脚本

直接复制以下整段命令到终端执行：

```bash
echo '#!/sbin/openrc-run
name="gost"
command="/usr/local/bin/gost"

# 格式：-L socks5://用户名:密码@:端口
command_args="-L socks5://admin:your_password@:10086"

command_background="yes"
pidfile="/run/gost.pid"
output_log="/var/log/gost.log"
error_log="/var/log/gost.err"

depend() {
    need net
}' > /etc/init.d/gost

chmod +x /etc/init.d/gost
```

### 2. 注册并启动服务

```bash
rc-update add gost default
rc-service gost restart
rc-service gost status
```

* 正常状态输出应为：`* status: started`

---

## 四、连通性测试与客户端配置

### 1. 本机回环测试（验证服务本身）

在 Alpine 终端内执行：
```bash
curl -x socks5://admin:your_password@127.0.0.1:10086 https://ipinfo.io
```
* **成功响应**：返回包含当前服务器外网 IP 及位置的 JSON 数据。

### 2. 外网客户端连接配置

在外网设备（Windows / macOS / 手机）或代理工具（SwitchyOmega、Telegram、Proxifier）中配置：

| 配置项 | 参数值 |
| :--- | :--- |
| **代理协议** | `SOCKS5` |
| **服务器地址** | 你的服务器公网 IP |
| **服务端口** | `10086`（或自定义端口） |
| **用户名** | `admin`（或自定义账号） |
| **密码** | `your_password`（或自定义密码） |

> [!NOTE]
> 如果本机测试通过但外部无法连接，请登录云服务商控制台（阿里云/腾讯云/AWS/甲骨文等），在 **防火墙 / 安全组** 中放行该端口的 **TCP 与 UDP** 流量。

---

## 五、日常运维与服务管理

| 操作 | 命令 |
| :--- | :--- |
| 查看服务运行状态 | `rc-service gost status` |
| 重启服务 | `rc-service gost restart` |
| 停止服务 | `rc-service gost stop` |
| 启动服务 | `rc-service gost start` |
| 查看正常运行日志 | `cat /var/log/gost.log` |
| 查看错误异常日志 | `cat /var/log/gost.err` |
| 查看端口监听情况 | `netstat -tlpn \| grep 10086` |

---

## 六、内存与资源监控

GOST 内存开销极小，正常空载仅占用约 **10MB ~ 25MB** 物理内存。

* **精确查看实际物理内存（自动换算为 MB）**：
  ```bash
  awk '/VmRSS/{printf "gost 实际物理内存占用: %.2f MB\n", $2/1024}' /proc/$(pidof gost)/status
  ```
* **查看详细内存指标（RSS 与 VmSize）**：
  ```bash
  grep -E 'VmRSS|VmSize' /proc/$(pidof gost)/status
  ```
* **实时动态监测**：
  ```bash
  top -p $(pidof gost)
  ```

---

## 七、踩坑记录与排障指南（FAQ）

### 1. 启动报 `Exec format error`
* **原因**：下载的二进制架构与 CPU 不符（如 ARM 机器下载了 amd64 二进制），或脚本文件被混入 Windows 换行符（CRLF）。
* **排查与解决**：
  1. 运行 `uname -m` 确认实际架构（`x86_64` 对应 `amd64`，`aarch64` 对应 `arm64`）。
  2. 运行 `apk add dos2unix && dos2unix /etc/init.d/gost` 清除潜在的 Windows 换行符。

### 2. 状态显示 `* status: crashed`
* **原因**：OpenRC 虽然拉起了进程，但 gost 启动后立刻退出，或 OpenRC 记录的 PID 文件与实际进程脱节。
* **排查流程**：
  1. 查看错误日志：`cat /var/log/gost.err`。
  2. 如果日志显示 `listening on ...` 且 `netstat -tlpn | grep <端口>` 显示端口正常在监听，说明只是 PID 失步，运行以下命令即可恢复：
     ```bash
     pidof gost > /run/gost.pid
     rc-service gost status
     ```

### 3. 日志提示 `bind: address already in use`
* **原因**：指定的端口已经被占用，或之前启动过旧的 gost 实例成为孤儿进程挂在后台。
* **解决办法**：
  ```bash
  # 强制清理旧实例
  killall -9 gost
  # 重新启动 OpenRC 服务
  rc-service gost restart
  ```

### 4. 终端粘贴长命令时 URL 截断或报语法错误
* **原因**：网页端或部分远程终端控制台存在**单行字符宽度截断**限制，粘贴超长一键命令会被强行切入换行符。
* **防范**：避免使用超过 100 字符的超长单行，采用多步短命令或 `echo '...' > 文件` 形式写入。

---

## 八、彻底卸载与清理

如需从服务器上完全删除 gost 服务及残留文件，执行以下一键清理脚本：

```bash
# 1. 停止并杀死后台进程
rc-service gost stop 2>/dev/null
killall -9 gost 2>/dev/null

# 2. 注销并删除 OpenRC 开机自启服务
rc-update del gost default 2>/dev/null
rm -f /etc/init.d/gost
rm -f /run/gost.pid

# 3. 删除二进制程序与配置目录
rm -f /usr/local/bin/gost
rm -rf /etc/gost ~/.gost

# 4. 删除产生的日志和临时文件
rm -f /var/log/gost.log /var/log/gost.err
rm -f /tmp/gost*
```

**验证是否卸载干净**：
```bash
which gost              # 应提示 not found
netstat -tlpn | grep 10086 # 应无任何输出
```
