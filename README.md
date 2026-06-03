# AI-BolaScan — IDOR越权自动化检测工具

## 功能概览

支持根据域名、关键字、漏洞状态、标记状态的筛选

![image-20260603180403497](https://photoscloud.oss-cn-shanghai.aliyuncs.com/image-20260603180403497.png)

### 核心能力

| 模块 | 说明 |
|:---|:---|
| **主动 IDOR 扫描** | 自动识别 URL/Body/Header 中的资源 ID 参数，生成请求并对比响应差异 |
| **未授权访问检测 (NoAuth)** | 剥离认证头后重放请求，在同一 API 结果中展示未授权访问风险 |
| **AI 辅助研判** | 对接 Claude / DeepSeek / GPT / GLM / Qwen 等模型，自动分析响应差异是否为真实越权 |
| **误报过滤** | 内置 20 条规则 + 自定义 YAML，过滤签名失败、权限拒绝、空响应、限流等噪音 |
| **PII 检测** | 自动识别响应中的手机号、身份证、邮箱等敏感信息，评估数据泄露风险 |

### 流量处理

| 模块 | 说明 |
|:---|:---|
| **MITM HTTPS 代理** | 自动签发 CA 证书，解密 HTTPS 流量，支持上游代理级联 |
| **智能过滤** | 自动跳过 JS/CSS/图片/字体等静态资源，过滤攻击 payload 和扫描器噪音 |
| **域名白名单** | 仅对指定域名执行变异测试，避免干扰非目标站点 |
| **域名黑名单** | 正则表达式匹配，屏蔽 Google/Microsoft/Apple/CDN 等非业务域名 |
| **URL 归一化** | 自动去重相似请求，识别 RESTful 路径模式，合并重复 API |

### 报告与可视化

| 模块 | 说明 |
|:---|:---|
| **Web Dashboard** | 实时展示扫描发现，支持按严重程度、域名、API 路径、NoAuth 结果筛选 |
| **JSONL 持久化** | 所有发现写入 `findings.jsonl`，支持离线分析和 CI 集成 |
| **响应 Diff** | 并排对比原始响应与变异响应，高亮差异字段 |

---

## 快速启动

```bash
# macOS / Linux
./AI-BolaScan-mac-arm64

# Windows
AI-BolaScan-windows-amd64.exe
```

启动后：

- HTTPS 代理：`127.0.0.1:8888`
- Dashboard：`http://127.0.0.1:8090/dashboard`

---

## 代理链路配置

### 浏览器 → AI-BolaScan（独立使用）

```
浏览器代理 → 127.0.0.1:8888
```

1. 浏览器设置代理为 `127.0.0.1:8888`
2. 安装 MITM 证书（首次使用）：
   - **macOS**：双击 `data/bolascan-ca.pem` → 钥匙串 → 设置为"始终信任"
   - **Windows**：双击 `data/bolascan-ca.pem` → 安装到"受信任的根证书颁发机构"
   - **Firefox**：设置 → 隐私与安全 → 证书 → 导入

---

### 浏览器 → Burp → AI-BolaScan（推荐）

```
浏览器代理 → Burp (8080) → AI-BolaScan (8888) → 目标
```

**Burp 上游代理设置**：Settings → Network → Connections → Upstream Proxy Servers → Add：

| 字段 | 填写 |
|:---|:---|
| Destination host | `*` |
| Proxy host | `127.0.0.1` |
| Proxy port | `8888` |

**导入 AI-BolaScan CA 到 Burp**：

macOS：

```bash
"/Applications/Burp Suite Professional.app/Contents/Resources/jre.bundle/Contents/Home/bin/keytool" \
  -import -trustcacerts -alias bolascan-ca \
  -file data/bolascan-ca.pem \
  -keystore "/Applications/Burp Suite Professional.app/Contents/Resources/jre.bundle/Contents/Home/lib/security/cacerts" \
  -storepass changeit -noprompt
```

Windows（管理员）：

```cmd
"C:\Program Files\BurpSuiteProfessional\jre\bin\keytool.exe" ^
  -import -trustcacerts -alias bolascan-ca ^
  -file data\bolascan-ca.pem ^
  -keystore "C:\Program Files\BurpSuiteProfessional\jre\lib\security\cacerts" ^
  -storepass changeit -noprompt
```

浏览器代理设为 Burp（`127.0.0.1:8080`），重启 Burp 生效。

### Burp history 记录导入 AI-BolaScan（离线审计）

Burp history 右键 `save item`，保存为xml文件，选择导入则可以自动进行离线审计

![image-20260603183123110](https://photoscloud.oss-cn-shanghai.aliyuncs.com/image-20260603183123110.png)

每次扫描结果会生成一个带时间戳的 json 文件，可以在右上角进行选择

![image-20260603183342464](https://photoscloud.oss-cn-shanghai.aliyuncs.com/image-20260603183342464.png)

## 配置文件

### config/ai.yaml — AI 研判

`enabled: true` 时，可疑差异自动提交 AI 二次研判。取消注释对应块切换模型：

```yaml
# DeepSeek（默认）
default:
  provider: deepseek
  model: deepseek-v4-pro
  api_key: ${DEEPSEEK_API_KEY}
  base_url: https://api.deepseek.com/chat/completions
  max_tokens: 512
  reasoning_effort: high

# Claude 官方 API
# default:
#   provider: claude
#   model: claude-sonnet-4-5-20251001
#   api_key: ${ANTHROPIC_API_KEY}
#   base_url: https://api.anthropic.com/v1/messages
#   max_tokens: 512
```

### config/domain.yaml — 域名白名单

仅列表内域名执行测试，留空测试所有。支持通配符 `*.example.com`。

### config/domain_blacklist.yaml — 域名黑名单

每行一条正则，匹配的域名跳过不扫描：

```yaml
(^|\.)google(apis|usercontent|analytics)?\.com$
(^|\.)gstatic\.com$
(^|\.)jsdelivr\.net$
```

### config/false_positive_rules.yaml — 误报规则

YAML 格式，`text` / `regex` 匹配 `request` / `response` / `additional_notes`。内置 20 条规则

![image-20260603184236728](https://photoscloud.oss-cn-shanghai.aliyuncs.com/image-20260603184236728.png)

## 启动参数

不带任何参数直接运行时，默认行为：

- 代理监听 `127.0.0.1:8888`，Dashboard `127.0.0.1:8090`
- HTTPS MITM 代理 + IDOR 主动扫描 + NoAuth 未授权检测同时开启
- 变异前自动重放原始请求作为基线对比
- 对 GET/POST/PUT/DELETE 等全部方法执行变异
- 跳过上游 TLS 证书校验

常用可选参数：

| 参数 | 默认值 | 说明 |
|:---|:---|:---|
| `-upstream-proxy` | — | 上游代理，如 `http://127.0.0.1:8080` |
| `-workers` | 10 | 并发 worker 数 |
| `-max-mutations` | 10 | 每请求最大变异数 |
| `-proxy-addr` | `127.0.0.1:8888` | 代理监听地址 |
| `-api-addr` | `127.0.0.1:8090` | Dashboard 地址 |
| `-domain-list` | `config/domain.yaml` | 白名单路径 |
| `-domain-blacklist` | `config/domain_blacklist.yaml` | 黑名单路径 |
| `-fp-rules` | `config/false_positive_rules.yaml` | 误报规则路径 |
| `-ai-config` | `config/ai.yaml` | AI 配置路径 |

## 声明
仅限用于技术研究和获得正式授权的攻防项目，请使用者遵守《中华人民共和国网络安全法》，切勿用于任何非法活动，若将工具做其他用途，由使用者承担全部法律及连带责任，作者及发布者不承担任何法律及连带责任！

使用前先按照文档步骤一步一步来，报错问题自行AI调试解决，类似issue不予回复，感谢理解！