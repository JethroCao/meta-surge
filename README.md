# 🌐 Cloud-Native Network Infrastructure (Mihomo/Clash Meta)

这是基于 Clash Meta (Mihomo) 内核构建的高度自动化分流方案。

# 🏗️ 配置核心逻辑指南

本仓库的配置文件遵循 “解耦 -> 筛选 -> 调度” 的模块化设计方案。

## 1. 节点引入 (Proxy Providers)

通过 `proxy-providers` 实现订阅地址与主配置的解耦，支持自动更新与本地缓存。

配置要点：
- `type: http`: 远程获取订阅。
- `path`: 必须设置，确保断网或机场维护时仍有本地缓存可用。
- `health-check`: 建议开启，用于自动剔除失效节点。

```yaml
proxy-providers:
  provider_a:
    type: http
    url: "你的订阅链接"
    interval: 3600
    path: ./provider_a.yaml
    health-check:
      enable: true
      interval: 600
      url: http://www.gstatic.com/generate_204
```

## 2. 策略分组 (Proxy Groups)

利用正则表达式 (`filter`) 实现节点的自动化清洗，无需手动维护繁琐的节点列表。

配置要点：
- `type: select`: 手动选择，适合归属地敏感业务（Gemini/AI）。
- `type: url-test`: 自动优选，适合大流量业务（视频/下载）。
- `filter`: 使用正则匹配关键字，如 `(?i)美国|US` 自动抓取美国节点。

### `filter` 使用教程（节点名正则）
`filter` 会对“节点名称”做正则匹配，命中即加入该策略组。常用写法：

**1) 基础匹配（国家/地区）**
```yaml
filter: "(?i)香港|HK|Hong Kong"
```

**2) 多条件合并（OR）**
```yaml
filter: "(?i)日本|JP|Tokyo|Osaka"
```

**3) 排除关键词（NOT）**
使用“先全量匹配，再排除”的写法：
```yaml
filter: "(?i)^(?!.*(专线|IPLC)).*(美国|US|USA).*"
```

**4) 绑定业务标签**
```yaml
filter: "(?i)(Netflix|NF|解锁)"
```

**5) 仅匹配特定前缀/后缀**
```yaml
filter: "^(?i)US-"
filter: "(?i)-A$"
```

**小贴士**
- `(?i)` 表示忽略大小写。
- 关键字之间用 `|` 代表“或”。
- `.` `(` `)` `+` `*` `?` 等为正则特殊字符，需匹配本身时用反斜杠转义。
- 建议优先用简单关键字组合，复杂正则仅在必要时使用。

```yaml
proxy-groups:
  - name: "🔍 Google"
    type: select
    use: [provider_a]
    filter: "(?i)美国|US|USA" # 自动过滤美国节点，锁定 Gemini 可用区
```

## 3. 分流调度 (Rules)

遵循 “从上到下，首位命中” 的阶梯式原则。

书写规范：
- 私有流量 (L1)：`private_ip` / `private_domain` 必须置顶，确保局域网与本地开发环境直连。
- 核心服务 (L2)：针对 Google、GitHub、AI 等进行定向分流。
- 区域分流 (L3)：利用 `geolocation-!cn` 自动接管所有未点名的海外流量。
- 兜底放行 (L4)：国内域名 (`cn_domain`) 直连，最后由 `MATCH` 捕获残余流量。

### 怎么改 / 怎么扩展
**1) 调整优先级**
- 规则从上到下依次命中，想“更优先”就往上移。
- `MATCH` 永远放最后。

**2) 新增某个服务**
先在规则集里加（或引入）对应的 `RULE-SET`，再在 rules 里插入一行即可：
```yaml
- RULE-SET,telegram_domain,📨 Telegram
```

**3) 添加自定义直连/代理**
临时规则可以直接写在 rules 里：
```yaml
- DOMAIN-SUFFIX,example.com,DIRECT
- DOMAIN-KEYWORD,openai,🚀 一键代理
```

**4) no-resolve 使用建议**
- IP 类规则（`IP-CIDR` / `RULE-SET` 的 IP 列表）建议加 `no-resolve`，提升性能。
- 域名类规则不需要 `no-resolve`。

```yaml
rules:
  - RULE-SET,private_ip,DIRECT,no-resolve  # 优先级最高：内网放行
  - RULE-SET,google_domain,🔍 Google       # 精准打击：特种业务
  - RULE-SET,geolocation-!cn,🚀 一键代理   # 区域过滤：海外长尾
  - MATCH,🐟 漏网之鱼                      # 最终兜底：万能捕蚊灯
```

# 🛠️ 安装与部署
安装方式（以 macOS 的 ClashX Meta 为例）：
1. `git clone` 本项目。
2. 打开 `combined.yaml`，将 `proxy-providers` 中的订阅链接替换为你自己的。
3. 在 ClashX Meta 中依次点击：配置 → 打开配置文件夹。
4. 将 `combined.yaml` 放入该配置文件夹。
5. 回到 ClashX Meta，选择配置 `combined`。
![alt text](assets/rules-1.png)
6. 在 ClashX Meta 中依次点击：Meta → 代理 Providers → 更新全部。
![alt text](assets/rules-2.png)
7. 在 ClashX Meta 中依次点击：Meta → 规则 Providers → 更新全部。
![alt text](assets/rules-3.png)

Maintenance: 改动 `combined.yaml` 后，请在 UI 中更新 Providers 以触发 MRS 规则集下载。
