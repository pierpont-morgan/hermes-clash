# 🚀 Flclash 内嵌 Tailscale 实战指南：单 App 搞定内网穿透与外网代理

> **核心特性**：**手机端仅需运行一个 Flclash (Mihomo)**！  
> 无需安装/后台挂着 Tailscale App，无需公网 IPv6，无需改动路由器。直接在 Clash 配置文件中内嵌 Tailscale 节点，实现「出门在外随时直连家里电脑上的 Hermes AI Agent」与「极速外网代理」完美并存分流。

---

## 💡 为什么需要这个方案？

许多安卓用户在搭配使用 Tailscale 与代理软件时，都会遇到以下痛点：
1. **安卓单 VPN 机制冲突**：系统强制只能运行一个 VPN。开启 Tailscale 连家里内网，翻墙代理就会被踢下线；开翻墙代理，家里的 Hermes 就断连；
2. **切网重连痛点（出门即断）**：正如推特网友反馈，Tailscale 官方 App 对移动网络切换较为敏感，从家里的 Wi-Fi 切换到室外 4G/5G 流量时，容易陷入持续 `reconnecting` 假死状态；
3. **Flclash 单 App 终极解法**：Flclash（基于现代化 Mihomo 内核）本身就支持将 Tailscale 作为节点接入。将 Tailscale 虚拟网直接做成一个普通的底层节点，由 Flclash 统一接管流量与路由分流，**一台手机只需常驻一个客户端**！

---

## 🏗️ 架构与分流逻辑

```mermaid
flowchart TD
    Phone["📱 手机端 (仅开 Flclash 一个应用)"]
    
    Phone -->|"访问 Hermes / 家里设备 (100.x.x.x / 192.168.x.x)"| Rule1["命中 Tailscale 内网规则"]
    Phone -->|"访问 Google / ChatGPT / YouTube"| Rule2["命中海外代理规则 (PROXIES)"]
    Phone -->|"访问 国内应用 (微信/淘宝/B站)"| Rule3["命中大陆直连规则 (DIRECT)"]
    
    Rule1 -->|"Flclash 内嵌 Tailscale 引擎打洞直连"| HomePC["💻 家里常开电脑 / NAS<br/>(运行 Hermes Desktop & Agent)"]
    HomePC --> Hermes["✨ Hermes AI Agent 服务"]
    
    Rule2 -->|"走专线加密隧道"| Foreign["🌐 纯净 GCP / 机场落地节点"]
    Rule3 -->|"直连"| Direct["🚀 国内互联网"]
```

---

## 🛠️ 第一步：获取 Tailscale 认证密钥（Auth Key）

为了让手机端的 Flclash 无需弹出登录页面即可自动接入你的私有 Tailnet 网络：

1. 用浏览器登录 [Tailscale 控制台密钥管理页](https://login.tailscale.com/admin/settings/keys)；
2. 点击 **`Generate auth key...`**（生成认证密钥）；
3. 选项建议：
   * **Reusable**（可重复使用）：勾选（方便多设备或配置重载）；
   * **Expiration**（有效期）：可根据个人需求设置（如 90 天或无需频繁过期的 Key）；
4. 点击生成并复制保存该密钥（格式如 `tskey-auth-kXXXXX-XXXXXXXXXXXX`）。

---

## 📱 第二步：在 Flclash 的 YAML 配置文件中集成

把你的 Tailscale 虚拟节点写入到手机 Flclash 所使用的配置文件（`.yaml`）中：

### 1. 节点定义（在 `proxies:` 列表中加入）
```yaml
proxies:
  # === 1. 直接让 Flclash 作为节点加入你的 Tailscale 虚拟网 ===
  - name: "🏡 Tailscale-家里直连"
    type: tailscale
    hostname: "flclash-phone"                    # 在 Tailscale 后台显示的手机节点名
    auth-key: "tskey-auth-kXXXXX-XXXXXXXXXXXX"  # 填入你刚才生成的 Tailscale Auth Key
    state-dir: "./tailscale-state"               # 持久化存储节点状态，避免重启后生成重复设备
    accept-routes: true                         # 接收家里电脑广播的局域网子网路由
    udp: true

  # === 2. 你的原有海外代理节点（如 GCP 链式代理） ===
  - name: "🚀 GCP-链式加速"
    type: vmess
    server: 35.212.xxx.xxx
    # ... 其他原有配置保持不变 ...
```

### 2. 分流规则定义（在 `rules:` 列表中最顶部插入）
```yaml
rules:
  # 1. 访问家里电脑与 Hermes 服务（Tailscale 100.x 网段及家庭局域网）走 Tailscale 节点
  - IP-CIDR,100.64.0.0/10,🏡 Tailscale-家里直连,no-resolve
  - IP-CIDR,192.168.0.0/16,🏡 Tailscale-家里直连,no-resolve

  # 2. 国内应用直连
  - GEOIP,CN,DIRECT

  # 3. 兜底所有海外流量走外网代理组
  - MATCH,PROXIES
```

---

## ✨ 优势与实测体验

1. **彻底告别双 VPN 冲突**：手机后台清清爽爽，只有一个 Flclash 小钥匙图标；
2. **切网毫秒级平滑**：Mihomo 内核在底层处理移动网络与 Wi-Fi 切换时的重连极其稳健，彻底解决官方 App 的频繁断连；
3. **免公网 IP、免改路由器**：所有打洞与握手均由 Tailscale 官方中继体系全自动维护，安全又省心。

---

## 📄 License
MIT © 2026. 欢迎自由 Fork 与交流！
