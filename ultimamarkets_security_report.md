# ultimamarkets.com 安全评估报告

**评估日期**: 2026-06-08  
**评估范围**: ultimamarkets.com 及所有已发现子域名  
**评估类型**: 被动侦察 + 配置安全检查  
**授权状态**: 网站所有者授权

---

## 执行摘要

本次对 ultimamarkets.com（一家持牌CFD交易经纪商）的安全评估共发现 **2项高危漏洞**、**3项中危漏洞**、**2项低危问题**。最严重的问题是三个子域名存在**子域名接管风险**，以及**DMARC邮件策略未启用强制执行**，均会对品牌声誉和用户安全造成重大威胁。

| 风险等级 | 数量 |
|---------|------|
| 严重 (Critical) | 0 |
| 高危 (High) | 2 |
| 中危 (Medium) | 3 |
| 低危 (Low) | 2 |
| 信息 (Info) | 3 |

---

## 发现的漏洞详情

### [HIGH-1] 子域名接管风险 — 三个悬空 CNAME

**风险等级**: 高危  
**受影响域名**:
- `client.ultimamarkets.com` → CNAME → `ultimaco.match-trade.com` (**NXDOMAIN**)
- `m-platform.ultimamarkets.com` → CNAME → `ultimamtr.match-trade.com` (**NXDOMAIN**)
- `platform.ultimamarkets.com` → CNAME → `ultimamtr.match-trade.com` (**NXDOMAIN**)

**问题描述**:  
上述三个子域名的 DNS CNAME 记录指向 `match-trade.com`（一家白标交易平台服务商）下的子域名，但这些目标子域名目前已**不存在**（返回 NXDOMAIN）。父域名 `match-trade.com` 本身可以正常解析（87.98.239.3），但 `ultimaco.match-trade.com` 和 `ultimamtr.match-trade.com` 均未配置 A 记录。

**攻击场景**:  
攻击者若能在 `match-trade.com` 侧创建上述子域名（例如通过 match-trade 客户门户或社会工程），即可托管钓鱼页面、恶意内容，并以 `*.ultimamarkets.com` 官方域名呈现给用户。由于域名归属于 ultimamarkets.com，用户浏览器会显示合法的 HTTPS 证书，极具欺骗性。

**修复建议**:
1. **立即删除**这三条悬空 CNAME DNS 记录（如平台已停用）
2. 如仍需使用这些子域名，联系 match-trade.com 恢复对应的 DNS 配置
3. 定期审查所有 CNAME 记录，确保目标均可正常解析

```
# 验证命令
dig CNAME client.ultimamarkets.com
dig A ultimaco.match-trade.com   # 应返回 NXDOMAIN，说明目标不存在
```

---

### [HIGH-2] DMARC 策略未启用强制执行 (p=none)

**风险等级**: 高危  
**当前 DMARC 配置**:
```
v=DMARC1; p=none;
```

**问题描述**:  
DMARC 策略设置为 `p=none`，意味着即使邮件未通过 SPF/DKIM 验证，邮件服务器也不会对其采取任何强制措施（隔离或拒绝）。此外，未配置 `rua`（聚合报告）和 `ruf`（失败报告）地址，导致无法监控潜在的邮件滥用情况。

对于金融类平台，这意味着攻击者可以伪造 `@ultimamarkets.com` 邮件地址向用户发送钓鱼邮件，难以与真实邮件区分。

**注意**: SPF 记录配置正确（`-all` 强制拒绝），但 DMARC `p=none` 使 SPF 失效。

**修复建议**:
```
# 推荐逐步迁移路径:
# 1. 阶段一: 添加报告收件箱，开始收集数据
v=DMARC1; p=none; rua=mailto:dmarc-reports@ultimamarkets.com; ruf=mailto:dmarc-failures@ultimamarkets.com

# 2. 阶段二: 启用隔离策略 (评估2-4周后)
v=DMARC1; p=quarantine; pct=25; rua=mailto:dmarc-reports@ultimamarkets.com

# 3. 阶段三: 完全拒绝 (推荐最终目标)
v=DMARC1; p=reject; rua=mailto:dmarc-reports@ultimamarkets.com; ruf=mailto:dmarc-failures@ultimamarkets.com
```

---

### [MEDIUM-1] 主域名及多个子域名缺少 HSTS 响应头

**风险等级**: 中危  
**受影响域名**:
- `ultimamarkets.com` — 无 HSTS
- `email.ultimamarkets.com` — 无 HSTS
- `new.ultimamarkets.com` — 无 HSTS
- `promotion.ultimamarkets.com` — 无 HSTS
- `trustpilot.ultimamarkets.com` — 无 HSTS

**已正确配置 HSTS 的域名**（供参考）:
- `afm.ultimamarkets.com`: `max-age=31536000`
- `go.ultimamarkets.com`: `max-age=2592000; includeSubDomains`
- `partners.ultimamarkets.com`: `max-age=31536000`
- `webtrader.ultimamarkets.com`: `max-age=31536000`

**问题描述**:  
HTTP Strict Transport Security (HSTS) 标头缺失，使网站易受协议降级攻击（SSL Stripping）影响。攻击者可在中间人场景下将 HTTPS 连接降级为 HTTP，窃取用户会话 Cookie 和数据。

**修复建议**（在 Cloudflare 或 Web 服务器中添加）:
```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

---

### [MEDIUM-2] Content Security Policy (CSP) 弱配置

**风险等级**: 中危  
**当前 CSP**:
```
default-src 'none'; 
script-src 'nonce-...' 'unsafe-eval' https://challenges.cloudflare.com; 
style-src 'unsafe-inline'; 
form-action http: https:;
```

**问题列表**:

| 指令 | 问题 | 风险 |
|------|------|------|
| `script-src 'unsafe-eval'` | 允许 JavaScript eval() 执行 | XSS 载荷可通过 eval() 执行 |
| `style-src 'unsafe-inline'` | 允许内联 CSS | CSS 注入攻击 |
| `form-action http: https:` | 允许表单提交到任意 URL | 开放重定向、钓鱼风险 |

**注意**: `email.ultimamarkets.com` 的 CSP 配置较好（使用 hash-based）：
```
script-src 'self' 'sha256-J+Y4l+yfxXd4cYzH9LhXUSHSb7zZu2bgddfCumVZJMo=';
form-action 'none';
```

**修复建议**:
```
# 移除 unsafe-eval（需代码改造）
script-src 'nonce-{随机值}' https://challenges.cloudflare.com;

# 限制 form-action 到已知域
form-action 'self' https://ultimamarkets.com;

# 对 style-src 使用 nonce 或 hash
style-src 'nonce-{随机值}';
```

---

### [MEDIUM-3] 缺少 CAA DNS 记录

**风险等级**: 中危  
**当前状态**: ultimamarkets.com 无任何 CAA 记录

**问题描述**:  
Certificate Authority Authorization (CAA) DNS 记录用于限制哪些证书颁发机构（CA）可以为该域名颁发 SSL/TLS 证书。缺少 CAA 记录意味着任何 CA 均可颁发证书，增加了因 CA 失误或被攻击导致的证书误发风险（如 DigiCert 或 Let's Encrypt 攻击）。

**修复建议**（添加到 DNS）:
```
; 仅允许 Let's Encrypt 和 Cloudflare 颁发证书
ultimamarkets.com. 300 IN CAA 0 issue "letsencrypt.org"
ultimamarkets.com. 300 IN CAA 0 issue "comodoca.com"
ultimamarkets.com. 300 IN CAA 0 issuewild "letsencrypt.org"
ultimamarkets.com. 300 IN CAA 0 iodef "mailto:security@ultimamarkets.com"
```

---

### [LOW-1] afm.ultimamarkets.com 暴露服务器技术信息

**风险等级**: 低危  
**响应头**: `server: Apache`  

**问题描述**:  
`afm.ultimamarkets.com` 不在 Cloudflare 保护之后，直接暴露 Apache 服务器标识。虽然未泄露版本号（较好），但服务器类型信息仍可帮助攻击者针对性地选择攻击向量。

**附加发现**: 该子域名对所有路径返回 HTTP 400，可能为内部服务或已废弃服务，建议确认是否仍需对外暴露。

**修复建议**:
```apache
# Apache 配置中隐藏服务器标识
ServerTokens Prod
ServerSignature Off
```
或将其纳入 Cloudflare 代理保护。

---

### [LOW-2] newsletter.ultimamarkets.com DNS 记录无对应服务

**风险等级**: 低危  
**状态**: DNS 记录存在但无 CNAME 目标，服务不可访问

**问题描述**:  
`newsletter.ultimamarkets.com` 在证书透明度日志中有记录，但当前 DNS 查询返回 NXDOMAIN（无 A 记录，无 CNAME）。如果该域名曾经活跃，遗留的 SSL 证书可能存在未清理情况。

**修复建议**: 确认该子域名是否已停用，如是则清理相关 DNS 记录和 SSL 证书。

---

## 信息性发现（无直接风险）

### [INFO-1] 邮件基础设施

- **MX 记录**: Microsoft 365 Exchange Online (`ultimamarkets-com.mail.protection.outlook.com`)
- **SPF**: 正确配置，包含 Outlook、Mandrill、SendCloud、Zendesk、Marketo，末尾使用 `-all`（严格模式）✅
- **使用的邮件服务商**: Mandrill（事务邮件）、SendCloud、Marketo（营销自动化）

### [INFO-2] 网络基础设施

- **Cloudflare 代理**: 主域名和大多数子域名均通过 Cloudflare 代理
- **IP 地址**: 104.18.33.86, 172.64.154.170（均为 Cloudflare Anycast IP）
- **IPv6**: 已启用（2606:4700:4404::ac40:9aaa, 2606:4700:440a::6812:2156）
- **命名服务器**: jake.ns.cloudflare.com, lady.ns.cloudflare.com

### [INFO-3] 第三方服务暴露（TXT 记录）

通过 DNS TXT 记录可识别的第三方集成:
- Google Analytics / Search Console
- LinkedIn 验证
- Microsoft 365
- Mandrill (Mailchimp)
- SendCloud
- Zendesk（客服支持）
- Marketo（营销自动化）

---

## 已实施的安全措施（良好实践）

| 安全措施 | 状态 |
|---------|------|
| HTTPS 强制跳转 (HTTP → HTTPS 301) | ✅ |
| TLS 1.0/1.1 已禁用 | ✅ |
| TLS 1.2/1.3 支持 | ✅ |
| Cloudflare Bot 防护 | ✅ |
| X-Content-Type-Options: nosniff | ✅ |
| X-Frame-Options: SAMEORIGIN（防点击劫持）| ✅ |
| Permissions-Policy 配置 | ✅ |
| Cross-Origin 头部（COEP/COOP/CORP） | ✅ |
| Referrer-Policy | ✅ |
| 仅开放必要端口 (80, 443) | ✅ |
| 无暴露的数据库端口 | ✅ |
| SPF `-all` 严格配置 | ✅ |
| Server 版本不泄露（Cloudflare 代理） | ✅ |

---

## 优先修复路线图

| 优先级 | 操作 | 预计工时 |
|--------|------|---------|
| P0 - 立即 | 删除三个悬空 CNAME 记录 (client/m-platform/platform) | 15 分钟 |
| P1 - 本周 | 将 DMARC 升级至 `p=quarantine` 并添加报告地址 | 30 分钟 |
| P1 - 本周 | 为主域名启用 HSTS | 30 分钟 |
| P2 - 本月 | 修复所有子域名的 HSTS 缺失 | 2 小时 |
| P2 - 本月 | 添加 CAA DNS 记录 | 30 分钟 |
| P3 - 季度 | 移除 CSP 中的 `unsafe-eval` | 需代码审查 |
| P3 - 季度 | 将 DMARC 升级至 `p=reject` | 在观察期后 |

---

## 附录：扫描方法

本次评估使用的技术：
- DNS 记录枚举（A, MX, TXT, CNAME, CAA, NS, AAAA）
- 证书透明度日志查询（crt.sh）
- HTTP 响应头分析
- SSL/TLS 协议版本测试
- 端口扫描（常见服务端口）
- 子域名接管检测（CNAME 悬空分析）
- 路径探测（常见敏感路径）
- DMARC/SPF/DKIM 邮件安全评估

*本报告仅包含被动侦察和配置检查结果，未进行任何主动漏洞利用或渗透测试。*
