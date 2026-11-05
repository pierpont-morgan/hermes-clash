# 🚀 Hermes Flclash 单客户端全能直连指南（All-in-One 融合分流流）

> **方案定位**：**单客户端全能流（All-in-One）** —— 手机端**仅运行一个 Flclash**，无需安装/运行 Tailscale App，也无需申请公网 IPv6。  
> 通过在 Flclash（Mihomo 内核）中直接集成 Tailscale 节点与规则分流，实现「在外随时直连家里电脑上的 Hermes AI Agent」与「极速出海代理」无缝并存！

---

## 💡 为什么需要这套方案？（解决两大核心痛点）

1. **彻底解决安卓系统“单 VPN 互斥”痛点**：
   * 安卓系统底层强制只允许一个 VPN 生效。过去手机开了 Tailscale 就没法翻墙，开了代理软件就断了家里的内网连接。
   * 本方案将 Tailscale 作为一个普通节点直接内嵌到 Flclash 中，**手机只开一个软件，同时搞定内网与外网**！
2. **彻底解决“切网卡死在 Reconnecting”的顽疾**：
   * 很多网友反馈：手机前台跑 Tailscale 出门时，从 WiFi 切换到 4G/5G 移动蜂窝网络，很容易被安卓省电策略冻结，一直转圈卡在 `Reconnecting`；
   * 由 Flclash 统一接管网络底层，内核级别的连接重试策略极其健壮，切网无感丝滑，秒级恢复连接。

---

## 🏗️ 架构与分流拓扑

```mermaid
flowchart TD
    Phone["📱 手机端 (仅开一个 Flclash App)"]
    
    Phone -->|"访问 Hermes / 家里内网 (100.x.x.x / 192.168.x.x)"| Rule1["内网分流规则"]
    Phone -->|"访问 Google / ChatGPT / YouTube"| Rule2["海外代理规则"]
    Phone -->|"访问 国内微信 / 淘宝 / 百度"| Rule3["大陆直连规则"]
    
    Rule1 -->|"走内置 Tailscale 虚拟网"| HomePC["💻 家里常开电脑<br/>(运行 Hermes Desktop & Gateway)"]
    HomePC --> Hermes["✨ Hermes AI Agent 智能体"]
    
    Rule2 -->|"走代理落地"| Foreign["🌐 纯净 GCP 专线 / 机场落地"]
    Rule3 -->|"直连"| Direct["🚀 国内互联网"]
```

---

## 🛠️ 第一步：获取你的 Tailscale 凭证（无需公网 IPv6）

你只需要在电脑浏览器打开 Tailscale 控制台，拿到一个授权密钥即可：

1. 浏览器打开 [Tailscale Admin Console - Keys](https://login.tailscale.com/admin/settings/keys)；
2. 点击 **Generate auth key**（生成认证密钥）；
3. 勾选 **Reusable**（可复用）和 **Ephemeral**（可选，临时设备），点击生成；
4. 复制生成的密钥备用（格式类似：`tskey-auth-k123456CNTRL-abcdef...`）。

---

## 📱 第二步：在 Flclash 配置文件中写入融合节点（核心）

打开你手机上使用的 Clash/Flclash 配置文件（`.yaml`），加入以下两部分配置：

### 1. 新增 Tailscale 专属节点（加入到 `proxies:` 列表中）
```yaml
proxies:
  # === 1. 直接让 Flclash 接入你的 Tailscale 虚拟网 ===
  - name: "🏡 Tailscale-家里直连"
    type: tailscale
    hostname: "flclash-phone"
    auth-key: "tskey-auth-kXXXXX-XXXXXXXXXXXX"  # 填入你刚才生成的 Tailscale Auth Key
    state-dir: "./tailscale-state"               # 保持连接凭据持久化
    accept-routes: true                         # 自动接收家里电脑广播的路由
    udp: true

  # === 2. 你的原有外网代理节点保持不变 ===
  - name: "🚀 GCP-链式加速"
    type: vmess
    # ... 原有代理节点配置保持不变 ...
```

### 2. 配置顶部精准分流规则（插入到 `rules:` 列表最顶部）
```yaml
rules:
  # 1. 访问家里内网（Hermes、NAS、电脑的 100.x.x.x 网段）走 Tailscale 节点直连
  - IP-CIDR,100.64.0.0/10,🏡 Tailscale-家里直连,no-resolve
  - IP-CIDR,192.168.0.0/16,🏡 Tailscale-家里直连,no-resolve

  # 2. 国内常见应用保持直连
  - GEOIP,CN,DIRECT

  # 3. 兜底所有海外外网流量走你的代理策略组
  - MATCH,PROXIES
```

---

## ⚡ 效果验证与使用

1. 在手机上打开 **Flclash**，导入该配置并启动；
2. 手机浏览器直接访问电脑的 Tailscale 虚拟 IP 与 Hermes 端口（例如 `http://100.113.228.56:8000`），秒开 **Hermes AI Agent** 对话；
3. 同时打开 YouTube / Google / ChatGPT，秒开 4K，流畅自如；
4. **从 WiFi 切换到 5G 移动流量，连接依然秒恢复，永不卡死！**

---

## 🤖 附：让其他 AI 帮配置的专属 Prompt（一键复制）

可以直接将以下提示词发送给 ChatGPT / Claude / Cursor，让 AI 自动化协助修改配置：

```markdown
# 任务目标：在 Flclash (Mihomo) YAML 中集成 Tailscale 节点实现内网外网双分流

## 背景与需求
用户希望在手机端仅运行一个 Flclash 客户端（无需安装/打开 Tailscale App，免公网 IPv6），同时实现：
1. 访问家里运行的 Hermes Agent（Tailscale 100.64.0.0/10 及局域网段）走内嵌 Tailscale 节点；
2. 访问海外网站走用户原有的代理策略组；
3. 彻底解决安卓单 VPN 冲突和切网卡死问题。

## 执行指令：
1. 请读取用户的 Clash YAML 配置文件；
2. 在 proxies 列表中新增 type: tailscale 节点模板（包含 hostname、auth-key、state-dir、accept-routes、udp）；
3. 在 rules 列表最顶部插入 IP-CIDR 分流规则，确保 100.64.0.0/10 和 192.168.0.0/16 定向分流至该 Tailscale 节点；
4. 输出严格采用 UTF-8（无 BOM）编码的修改后完整 YAML 配置。
```

---

## 📄 License
MIT © 2026. 欢迎自由 Fork 与交流！
