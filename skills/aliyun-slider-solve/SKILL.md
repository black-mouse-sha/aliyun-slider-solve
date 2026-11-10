---
name: aliyun-slider-solve
description: >-
  阿里云滑块验证码（aliyunCaptcha）的机制分析与自动化求解。
  当页面出现「滑动验证」拼图、或提交返回 F001／F015 判定码时用：
  先按判定码分流——F015 是位置误差（拼图位移与鼠标行程不是 1:1，要按二次模型反解），
  F001 是环境被判机器（先覆盖 navigator.webdriver 再谈轨迹）。
  只对阿里云有效，其他厂商协议完全不同。
  不用于：代理与指纹配置（另用 proxy-fingerprint-layering / clash-chain-proxy）。
---
# 阿里云滑块：先看判定码，再动手

属于开源包 反检测代理栈（antidetect-proxy-stack）。

## 含什么

三个判定码的含义与对应改法、一个二次响应模型与反解公式、一套从识别到注入的完整流程、以及一份「哪些验证码厂商这套办法不适用」的清单。

## 适合谁

- 自动化流程卡在阿里云滑块上，反复失败但不知道失败在哪一步的人
- 看到 `F001`／`F015`／`T001` 想知道含义的人
- 已经能拖动滑块，但拼图总差几十像素对不上的人
- 想搞清楚「为什么轨迹做得再像也被拦」的人

## 怎么用

1. 第 1 节先读判定码。**不看判定码就开始调参数，等于蒙着眼睛开车。**
2. `F015` → 查第 2 节二次模型，你的鼠标行程算错了。
3. `F001` → 查第 3 节环境指纹，跟轨迹无关。
4. `T001` → 通过。
5. 第 4 节是完整流程，第 6 节是不适用范围。

## 依赖

- Playwright（或任何能发 CDP `Input.dispatchMouseEvent` 的自动化框架）
- Python + `opencv` + `numpy`
- 缺口识别模型：`captcha-recognizer`（YOLO，自带权重）
- 人形轨迹生成器：见 `../NOTICE` 所列上游项目

## 示例

完整一次成功记录（含失败样本与参数）见包根 `examples/bitbrowser-download/README.md` 的第四节。

## 验证到哪

**已实测成功**：在比特浏览器官网下载页，用本文参数一次性通过，取得含签名的下载链接并成功下载 111.65 MB 安装包（判定码 `T001`）。

**已实测失败**：同一套参数在同一站点的中文域名下连续 3 次 `F001`，未通过；说明风控强度按站点／域名不同，**本文不构成「通用能过」的结论**。

**未验证**：更换分辨率、更换浏览器内核、更换会话后 `A_CDP`／`B_CDP` 这两个系数是否需要重新标定（上游文档说需要，本文没有足够样本复核）。

---

一句话：**F015 改算法，F001 改环境，先分清再动手。**

## 0. 为什么这个滑块特别难

两个独立难点，很多人只解决了一个：

```
难点一（算法）：拼图块的位移 与 鼠标行程 不是 1:1，是二次关系
难点二（环境）：navigator.webdriver 为 true 时，轨迹再像也直接判机器
```

只解决难点一 → 位置对了，判 `F001`。
只解决难点二 → 环境干净了，位置总差几十像素，判 `F015`。

## 1. 三个判定码

阿里云在 `-verify` 接口的响应里返回结果，格式大致是：

```json
{"Result": {"VerifyCode": "T001"}}
```

| 码 | 含义 | 该改什么 |
|---|---|---|
| **T001** | 通过 | 无 |
| **F001** | 风控拦截（行为／环境被判机器） | 第 3 节：环境指纹 + 轨迹形态 |
| **F015** | 位置误差（拼图没落到缺口上） | 第 2 节：二次模型反解 |

**怎么做**：在 Playwright 里挂一个响应监听，把所有命中 `-verify` 的响应体打出来。这是唯一可靠的进度反馈，比看「验证码窗口消失了没有」准得多。

## 2. F015：拼图位移是二次的

这是整套方案里最值钱的发现。

**实测关系（在该站点标定）：**

```
piece.left = A_CDP × m + B_CDP × m²

A_CDP = 0.0769
B_CDP = 0.003552

m         = 鼠标总行程（CSS px）
piece.left = 拼图块的最终 left 值
```

**反解——给定缺口位置，求鼠标该走多远：**

```python
import math

def solve_mouse_travel(target_left, A=0.0769, B=0.003552):
    """target_left: 拼图块需要到达的 left 值；返回鼠标应走的总像素"""
    disc = A * A + 4 * B * target_left
    return (-A + math.sqrt(disc)) / (2 * B)
```

**实测对照：**

| 缺口位置 target_left | 按 1:1 拖（错） | 二次反解（对） | 差 |
|---|---|---|---|
| 216 | 拼图只到 **182** | 鼠标走 **236** | **34 px** |

**34 像素就是这个滑块的门槛。** 按 1:1 拖，落点误差稳定在 30 像素以上，判 `F015`，无论轨迹多像人都没用。

### 落点应该「故意偏一点」

上游的做法是：85% 的概率让落点偏正 2～4 px（拼图多走一点）。实测偏负时 `F001` 率明显更高。

```python
err_piece = random.uniform(2.0, 4.0)          # 想偏多少像素
slope_end = A + 2 * B * D                      # 该点处的斜率
bias_px   = err_piece / slope_end              # 换算成鼠标多走多少
```

## 3. F001：环境比轨迹更致命

如果判定码是 `F001`，先别调轨迹参数，**先确认这一项**：

```javascript
navigator.webdriver   // 必须是 false
```

Playwright 默认是 `true`。必须用 `addInitScript` 在任何页面脚本之前覆盖：

```javascript
await page.addInitScript(() => {
  Object.defineProperty(navigator, "webdriver", { get: () => false, configurable: true });
  if (!window.chrome) window.chrome = {};
  if (!window.chrome.runtime) window.chrome.runtime = { connect: () => {}, sendMessage: () => {} };
  const fp = [
    { name: "PDF Viewer", filename: "internal-pdf-viewer", description: "Portable Document Format" },
    { name: "Chrome PDF Viewer", filename: "internal-pdf-viewer", description: "Portable Document Format" },
    { name: "Chromium PDF Viewer", filename: "internal-pdf-viewer", description: "Portable Document Format" },
  ];
  Object.defineProperty(navigator, "plugins", { get: () => fp, configurable: true });
  Object.defineProperty(navigator, "languages", { get: () => ["en-US", "en"], configurable: true });
  const gl = HTMLCanvasElement.prototype.getContext;
  HTMLCanvasElement.prototype.getContext = function (type, ...rest) {
    const ctx = gl.call(this, type, ...rest);
    if (ctx && (type === "webgl" || type === "experimental-webgl") && !ctx.__patched) {
      ctx.__patched = true;
      const gp = ctx.getParameter.bind(ctx);
      ctx.getParameter = function (p) {
        if (p === 37445) return "Intel Inc.";
        if (p === 37446) return "Intel Iris OpenGL Engine";
        return gp(p);
      };
    }
    return ctx;
  };
});
```

**实测效果**：加上这段之后，同一套轨迹从连续 `F001` 变为 `T001`。

### 轨迹形态也要像人

真人和合成轨迹的差别不在「像不像曲线」，而在速度分布：

| 特征 | 真人 | 平滑曲线 |
|---|---|---|
| 峰值速度位置 | 行程 75% 附近（末端甩鞭） | 5～20%（一起步就冲） |
| 峰值速度 | 很高（约 9700 px/s） | 很低（约 1400 px/s） |
| y 方向漂移 | 大（手臂自然偏移） | 几乎为 0 |
| 完全静止帧 | 有 | 没有 |

上游项目用 30 条真人轨迹训练分类器，能把真人与合成轨迹完全分开。**照着「慢逼近 → 谷底犹豫 → 末端甩鞭 → 收尾」重构，是这套方案能过的关键之一。**

## 4. 完整流程

```
① 打开页面，触发验证码
② 用 YOLO 识别缺口 box_left（captcha-recognizer，置信度通常 0.95+）
③ 换算到页面坐标：
      scale       = 图片显示宽度 / naturalWidth
      target_left = box_left × scale − ex0
      ex0         = 拼图块 alpha 通道的左边缘偏移（实测约 2）
④ 二次反解，得到鼠标总行程 D
⑤ 生成人形轨迹 [(dx, dy, dt_ms), ...]
⑥ 逐帧注入鼠标事件（CDP 或 page.mouse），按累计时间调度
⑦ 松手，等 -verify 响应，读 VerifyCode
```

### 别用 CDP 串行发大量 move

实测：用 `cdp.send("Input.dispatchMouseEvent")` 逐帧 `await`，在 170 帧左右会挂死（程序不返回、不报错）。换成 Playwright 的 `page.mouse.move()` 就正常。

**建议**：降采样（每 2 帧发一次）+ 每次调用包一层超时。

```javascript
const withTimeout = (p, ms) => Promise.race([
  Promise.resolve(p).catch(() => null),
  new Promise(r => setTimeout(() => r(null), ms)),
]);
await withTimeout(page.mouse.move(x, y), 1500);
```

## 5. 常见失败对照

| 症状 | 原因 | 改法 |
|---|---|---|
| 稳定 `F015`，落点差 30+ px | 按 1:1 算了鼠标行程 | 用第 2 节反解公式 |
| 稳定 `F001` | `navigator.webdriver` 为 true | 第 3 节 initScript |
| 环境已干净仍 `F001` | 轨迹形态（峰值位置／y 漂移） | 换人形轨迹变体 |
| 程序挂死不返回 | CDP 逐帧串行堆压 | 换 `page.mouse` + 降采样 + 超时 |
| 点击后验证码不弹 | 按钮在视口外，或被广告层挡住 | 先 `scrollIntoViewIfNeeded`，必要时 `force: true` |
| 同站换域名就过不去 | 风控强度按域名不同 | 本文实测：同一站 `.cn` 域名比 `.net` 严得多 |

## 6. 这套办法不适用于

| 厂商 | 能否照搬 |
|---|---|
| 阿里云（aliyunCaptcha） | 可以 |
| 极验 GeeTest | 不行，`w` 参数是 AES+RSA 加密 |
| 腾讯 TCaptcha | 不行，TDC 指纹 + POW 工作量证明 |
| 网易易盾 | 不行，IR SDK + 独立 AES |
| 数美 | 不行，12 个参数各自 DES-ECB |
| 自研滑块 | 得重新标定，第 2 节那套流程可复用 |

**换厂商就是换协议，没有任何参数可以直接沿用。**

## 7. 最后一句实话

这套东西解决的是「机器能不能通过一次图形校验」，不是「绕过平台规则」。

判定码能告诉你失败在哪一层——这是它最大的价值。至于拿它去做什么，本文不做判断，也不保证任何站点的通过率。上游项目自报单次 90.9%，本文在另一站点实测是 1 次通过、3 次失败。
