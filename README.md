# GD-V2ray-writemyself
## 开发者：xian xichun

使用v2ray在云服务器上搭建VPS翻墙服务器，来实现科学上网，使用文档说明十分清楚，使用前需要购买境外云服务器，并且确保服务器可以与境外的网站通信
# Xray VPS 单节点自动化部署与 Clash 订阅生成指南

本项目提供了一套完整、干净、易维护的 Linux VPS 单节点部署方案。基于 **Xray-core** 与 **HTTP 静态 Web 服务**，一键生成属于自己的 Clash / Mihomo 订阅链接，并包含开机自动更新动态 IP 的守护服务。

---

## 🌟 项目特性

- **跨发行版支持**：完美兼容 Ubuntu / Debian / RHEL 9 / Rocky Linux / AlmaLinux / CentOS Stream 等主流 Linux 系统。
- **自动参数渲染**：搭建时自动生成全局唯一 UUID、公私钥对及公网 IP，无需手动修改 JSON 字符串。
- **自动化 Clash 订阅**：同步生成 `/var/www/html/clash.yaml`，客户端直接拉取订阅地址即可使用。
- **动态 IP 守护（自启修复）**：内置 Systemd 服务，VPS 重启变动公网 IP 后自动修正订阅配置文件中的地址。

---

## 📋 前置准备

1. **境外 VPS 一台**（如华为云海外节点、AWS、DigitalOcean 等），系统推荐选择 Ubuntu 20.04+ 或 Rocky Linux 9 / RHEL 9[cite: 48]。
2. **云服务器安全组放行**（入站规则）：
   - `22/tcp`（SSH 远程连接）[cite: 47]
   - `80/tcp`（HTTP 订阅服务）[cite: 47]
   - `443/tcp`（Xray 节点端口）[cite: 47]

---

## 🚀 快速部署步骤

> ⚠️ **重要注意事项**：
> 在执行 **步骤三（生成环境变量参数）** 后，请确保在**同一个终端会话内连贯完成**后续步骤[cite: 46]。若中途断开 SSH 重新连接，需要重新运行步骤三以刷新环境变量[cite: 46]。

### 1. 基础环境安装与防火墙放行

请根据您的 Linux 系统选择对应的执行命令：

#### 🔹 Ubuntu / Debian
```bash
apt update -y
apt install -y curl wget unzip tar openssl apache2 ufw
systemctl enable --now apache2

# 防火墙配置
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable

# 安装官方最新版 Xray-core
bash -c "$(curl -L [https://github.com/XTLS/Xray-install/raw/main/install-release.sh](https://github.com/XTLS/Xray-install/raw/main/install-release.sh))" @ install



🔹 RHEL 9 / Rocky Linux / AlmaLinux / CentOS Stream
dnf update -y
dnf install -y curl wget unzip tar openssl httpd firewalld
systemctl enable --now httpd
systemctl enable --now firewalld

# 防火墙配置
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload

# 安装官方最新版 Xray-core
bash -c "$(curl -L [https://github.com/XTLS/Xray-install/raw/main/install-release.sh](https://github.com/XTLS/Xray-install/raw/main/install-release.sh))" @ install
2. 生成自动化部署参数
在终端中连贯运行以下命令，系统将自动计算生成 UUID 及密钥参数：
export MY_UUID=$(/usr/local/bin/xray uuid)
export KEYS=$(/usr/local/bin/xray x25519)
export MY_PRIVATE_KEY=$(echo "$KEYS" | grep -i "Private" | awk -F': ' '{print $2}' | tr -d ' ')
export MY_PUBLIC_KEY=$(echo "$KEYS" | grep -i "Public" | awk -F': ' '{print $2}' | tr -d ' ')
export MY_SHORT_ID=$(openssl rand -hex 8)
export MY_IP=$(curl -s [https://api.ipify.org](https://api.ipify.org) || curl -s [https://ifconfig.me](https://ifconfig.me))

echo "=========================================="
echo "服务器公网IP : $MY_IP"
echo "UUID         : $MY_UUID"
echo "Private Key  : $MY_PRIVATE_KEY"
echo "Public Key   : $MY_PUBLIC_KEY"
echo "Short ID     : $MY_SHORT_ID"
echo "=========================================="

3. 写入 Xray 配置文件并启动服务
写入节点配置文件并启动 Xray 服务[cite: 47, 48]：

mkdir -p /usr/local/etc/xray

cat > /usr/local/etc/xray/config.json << EOF
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "port": 443,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "${MY_UUID}",
            "alterId": 0
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
          "path": "/ray"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
EOF

# 校验配置文件格式是否正确
/usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json

# 启动并设置开机自启
systemctl daemon-reload
systemctl enable --now xray
systemctl restart xray
systemctl status xray
4. 生成 Clash 订阅文件
生成符合 Clash / Mihomo 规范的订阅配置文件至 Web 根目录[cite: 47, 48]：
cat > /var/www/html/clash.yaml << EOF
port: 7890
socks-port: 7891
mixed-port: 7890
allow-lan: true
mode: rule
log-level: info
ipv6: false
dns:
  enable: true
  enhanced-mode: fake-ip
  nameserver:
    - 223.5.5.5
    - 119.29.29.29
  fallback:
    - [https://1.1.1.1/dns-query](https://1.1.1.1/dns-query)
    - [https://8.8.8.8/dns-query](https://8.8.8.8/dns-query)
proxies:
  - name: "VMESS-节点"
    type: vmess
    server: "${MY_IP}"
    port: 443
    uuid: "${MY_UUID}"
    alterId: 0
    cipher: auto
    network: ws
    ws-opts:
      path: "/ray"
proxy-groups:
  - name: "🚀 节点选择"
    type: select
    proxies:
      - VMESS-节点
      - DIRECT
rules:
  - GEOIP,CN,DIRECT
  - MATCH,🚀 节点选择
EOF

# 设置读取权限及 SELinux 上下文
chmod 644 /var/www/html/clash.yaml
restorecon -v /var/www/html/clash.yaml 2>/dev/null || true
chcon -R -t httpd_sys_content_t /var/www/html/ 2>/dev/null || true

5. 配置开机自动更新 IP 守护脚本
为防止按量计费的 VPS 重启后变动 IP 导致订阅配置失效，部署自动更新脚本[cite: 47, 48]：

# 1. 创建自动检测与更新 IP 脚本
cat > /usr/local/bin/update-clash-ip.sh << 'EOF'
#!/bin/bash
NEW_IP=""
# 重试获取公网 IP（最多 30 次）
for i in {1..30}; do
    NEW_IP=$(curl -s --connect-timeout 5 [https://api.ipify.org](https://api.ipify.org) || curl -s --connect-timeout 5 [https://ifconfig.me](https://ifconfig.me))
    if [ -n "$NEW_IP" ]; then
        break
    fi
    sleep 2
done

# 获取成功后替换 clash.yaml 中的静态 IP 字段
if [ -n "$NEW_IP" ] && [ -f "/var/www/html/clash.yaml" ]; then
    sed -i -E "s/(server:[[:space:]]*[\"']).*([\"'])/\1${NEW_IP}\2/g; t; s/(server:[[:space:]]*)[^\"'].*/\1${NEW_IP}/g" /var/www/html/clash.yaml
fi
EOF

chmod +x /usr/local/bin/update-clash-ip.sh

# 2. 注册为开机执行一次的 systemd 服务
cat > /etc/systemd/system/update-clash-ip.service << 'EOF'
[Unit]
Description=Auto Update Clash IP on Boot
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/update-clash-ip.sh

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable update-clash-ip.service
📡 客户端接入指南
搭建完成后，你的专属 Clash 订阅地址为[cite: 47, 48]：
http://你的服务器公网IP/clash.yaml
打开 Clash Verge Rev、FlClash、Clash Nyanpasu 或 Clash Meta for Android 等客户端，导入上述订阅 URL 即可刷新出节点使用。   💡 服务器变更 IP 后的操作：
当 VPS 重新开机拿到新 IP 时，直接在客户端将原来的订阅链接修改为 http://新IP/clash.yaml 并重新拉取一次，即可继续使用[cite: 47, 48]。
🔍 服务状态校验与测试
搭建完成后可运行以下命令验证各项服务是否正常运转[cite: 47, 48]：
# 查看 Xray 运行状态
systemctl status xray

# 查看 80 / 443 端口监听
ss -tlnp | grep -E '80|443'

# 本地测试订阅文件拉取
curl -I [http://127.0.0.1/clash.yaml](http://127.0.0.1/clash.yaml)


