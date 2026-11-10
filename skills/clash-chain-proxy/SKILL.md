---
name: clash-chain-proxy
description: >-
  把一条「大陆连不上」的代理接进 Clash，并给它开一个独立端口。
  当代理返回 403 forbidden ip=、SOCKS5 握手返回非 0500、
  Clash 里加完节点却 502 Bad Gateway、或者发现主流量偷偷走了按流量计费的节点时用：
  先用 dialer-proxy 让机场节点做跳板出海，再用 listeners 把这条代理隔离到独立端口，
  最后用 append 而不是 prepend，避免 select 组默认选中它。
  不用于：判断该买哪种代理（另见正文第 0 节）；验证码拦截（另用 aliyun-slider-solve）。
---
# 链式代理与端口隔离

属于开源包 反检测代理栈（antidetect-proxy-stack）。

## 含什么

一套四步接法（判、接、隔、验）、三个必踩的坑（type 错、引号缺、prepend 插队）、一个端口隔离的架构图、以及从症状反推原因的对照表。

## 适合谁

- 买的代理服务商拒绝中国大陆来源 IP（表现为 `403 forbidden ip=<你的IP> not supported`）
- 想让命令行工具单独走一条代理，又不想影响整台电脑上网
- 加完节点跑不通，只会看到 `502 Bad Gateway` 不知道从哪查的人
- 用按流量计费的代理，担心主流量误走它烧钱的人

## 怎么用

1. 第 0 节先判断「要不要中转」——不是所有代理都需要机场。
2. 第 1 节探明协议类型，这一步错了后面全白费。
3. 第 2～3 节改两个文件：节点定义 + 端口监听。
4. 第 4 节验证，第 5 节按症状查错。

## 依赖

- Clash Verge Rev（内核 Mihomo v1.18+，需要 `dialer-proxy` 与 `listeners`）
- 一条出口代理（HTTP 或 SOCKS5，带账密）
- 一个能出海的机场节点（仅当代理拒绝大陆 IP 时需要）
- Windows PowerShell 用于验证；Linux/macOS 把路径换成 `~/.config/clash-verge/`

## 示例

完整的一次「加节点 → 开端口 → 验出口」记录见包根 `examples/bitbrowser-download/README.md` 的第三节。

## 验证到哪

已在本机实测通过：代理 `us.<provider>.io:3000`（HTTP 类型，账密认证）经机场节点中转后，独立端口 `7920` 出口为 `66.65.70.187 / New York City / Charter Communications / America/New_York`，同时 `7897` 主端口仍为机场节点，两条线互不影响。

已实测的边界：**同一个代理用 SOCKS5 配置会失败**（服务端返回 `48 54`，即 ASCII `HT`，说明它是 HTTP 代理）；**`listeners` 的 `proxy` 字段填代理组名同样有效**（不仅是节点名）。

**未验证**：代理服务商给的 `host:port` 在更换地区／会话参数后的行为；`dialer-proxy` 指向自动测速组时的稳定性。这两项只写了机制，没有足够样本。

---

一句话：**能直连就直连，非直连才加跳板；加了跳板，就用独立端口把它关起来。**

## 0. 先判断：到底要不要中转

中转不是标配。先花两分钟判断，能省掉一层故障点。

```powershell
# 直接测这条代理能不能用（把 host:port 换成你的）
Test-NetConnection 你的代理主机 -Port 你的端口
```

能在 200ms 内 `TcpTestSucceeded : True`，**且实际请求不被拒**，就不用中转，指纹浏览器里直接填原始账密即可。

**只有下面这种才需要中转：**

```
HTTP/1.1 403 Forbidden
msg: forbidden ip=你的本地公网IP not supported
```

意思是「服务端不欢迎你现在的来源 IP」。中转只是换个来源，不改变你最终的身份——网站看到的仍然是最后一跳的 IP。

**代价要知道**：每多一层，延迟加一截、故障点加一个、排查难度翻倍。

## 1. 第一步：探明协议类型（最容易错的一步）

服务商给的 `host:port:user:pass` 长得一样，**但协议可能是 HTTP 或 SOCKS5，填错就永远连不上**。

一条命令判出来：

```powershell
# 向代理端口发 SOCKS5 握手前两个字节
$c = New-Object Net.Sockets.TcpClient('代理主机', 端口)
$s = $c.GetStream()
$s.Write([byte[]](0x05,0x01,0x02), 0, 3)
$buf = New-Object byte[] 2
$s.Read($buf, 0, 2) | Out-Null
Write-Host ($buf | ForEach-Object { $_.ToString('x2') }) -Separator ' '
$c.Close()
```

| 回应 | 含义 | Clash 里 type 填 |
|---|---|---|
| `05 02` | 真 SOCKS5，要求账密 | `socks5` |
| `48 54` | ASCII 的 `HT`，**它是 HTTP 代理** | `http` |

> 实测踩过：某服务商 `:3000` 端口是 HTTP 代理，按 SOCKS5 填，浏览器直接报 `ERR_SOCKS_CONNECTION_FAILED`。换 `type: http` 立刻通。

## 2. 第二步：写节点（用 append，不要 prepend）

配置文件位置（Clash Verge Rev）：

```
%APPDATA%\io.github.clash-verge-rev.clash-verge-rev\profiles\
```

这里有一堆随机名的 yaml。**要改的是「当前订阅对应的 proxies 覆盖文件」**，判定方法：

打开同目录上一级的 `profiles.yaml`，找 `current: <uid>`，再往下找 `uid: <同一个uid>` 那段里的 `proxies: <文件名>`。

内容写成：

```yaml
append:
  - name: "我的代理-01"
    type: http                      # ← 第 1 步探明，别猜
    server: your.proxy.host
    port: 3000
    username: "user-with space-and-dash"   # ← 有空格或特殊字符必须加引号
    password: "yourpass"
    dialer-proxy: "机场节点名"       # ← 仅当中转时才写
    udp: false

prepend: []
delete: []
```

### 为什么必须是 append

**`select` 类型的代理组默认选中第一个成员。**

用 `prepend` 把节点插到列表最前面，它就会成为该组的默认选项——主流量在你毫不知情的情况下全部走这条代理。

实测症状：主端口 `7897` 的出口从机场节点变成了按 GB 计费的住宅 IP，**每个小时都在扣钱**。

用 `append` 放到最后，默认选中的仍是机场节点。

## 3. 第三步：开独立端口

改 `Merge.yaml`（同目录，名字固定）：

```yaml
profile:
  store-selected: true

listeners:
  - name: cli-proxy
    type: mixed
    port: 7920
    proxy: "我的代理组名"     # 可以填节点名，也可以填组名（实测都支持）
```

- `port` 选一个没被占用的，避开 `7897`
- `proxy` 填组名时，在界面切换组会让这个端口跟着变——这就是「开关」

### 想让端口跟着界面开关走

建一个只含两项的组（在 groups 覆盖文件里）：

```yaml
prepend:
  - name: "🅿️ 专用出口"
    type: select
    proxies:
      - DIRECT                 # ← 放第一位 = 默认关闭
      - 我的代理-01
```

**DIRECT 排第一位**是关键：每次 Clash 重载，默认回到「关闭」状态，不会偷偷跑流量。要用时在界面上点一下。

## 4. 第四步：验证（三个都要看）

```powershell
# ① 端口起来了吗
Get-NetTCPConnection -State Listen | Where-Object { $_.LocalPort -eq 7920 }

# ② 这个端口走到哪
Invoke-RestMethod 'https://ipinfo.io/json' -Proxy 'http://127.0.0.1:7920'

# ③ 主端口有没有被带跑（这一步最容易漏）
Invoke-RestMethod 'https://ipinfo.io/json' -Proxy 'http://127.0.0.1:7897'
```

**②③ 的结果必须不同。** 如果 ③ 也变成了代理的 IP，说明你的节点被判进主组并且被选中了，回第 2 节改用 `append`。

## 5. 按症状查错

| 症状 | 原因 | 改法 |
|---|---|---|
| `502 Bad Gateway`，**0ms 立即返回** | 节点没合入配置，或 `dialer-proxy` 指向的节点名不存在 | 检查节点名拼写；确认改的是「当前订阅」对应的覆盖文件 |
| `407` | 帐号密码错 | 核对；注意用户名里的空格和特殊字符 |
| `403 forbidden ip=...` | 没走 `dialer-proxy`，直连出去了 | 加上 `dialer-proxy`，确认跳板节点可用 |
| 端口没监听 | YAML 语法错 | 只能空格缩进，不能 Tab；冒号后要有空格 |
| 主端口出口变了 | 用了 `prepend` | 改成 `append`，并把组里 DIRECT 放第一位 |
| 能连但很慢 | 两跳叠加 | 正常现象；换更近的跳板，或改用能直连的代理 |

## 6. 一个坑：全局 Merge 不支持 prepend-proxies

Clash Verge 有两处「扩展配置」：

| 文件 | 支持什么 | 不支持什么 |
|---|---|---|
| `Merge.yaml`（全局） | `listeners`、`profile` 等**顶层键** | `prepend-proxies` 不生效 |
| 当前订阅的 proxies 覆盖文件 | `prepend` / `append` / `delete` | 顶层 `listeners` |

**节点定义放覆盖文件，端口定义放 Merge.yaml。** 放错地方的表现是：端口起来了，但节点不存在，于是 `502`。

## 7. 如果你要的是「全电脑都走这条代理」

不是改这个端口，而是：

```
Clash Verge → 代理 → 「🚀 节点选择」→ 选中这条代理
```

效果是所有经过 Clash 的流量都走它。**代价**：按流量计费的代理会烧得很快，看视频一小时几 GB。真要长期这样，用流量不限的静态 ISP，不要用动态住宅。

（其余软件不走系统代理的，还得开 TUN 模式才能接管，那不是本文范围。）
