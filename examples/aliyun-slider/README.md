# 一次真实的滑块记录：1 次通过，3 次失败

目标页：BitBrowser 官网下载页的阿里云滑块（`aliyunCaptcha`）。
任务：点「Download」→ 过滑块 → 拿到带签名的安装包链接。

**成功 1 次，失败 3 次。全记在这里。**

---

## 一、先说我为什么失败了十几次

前 15 次的判定码全部是 `F015`（位置误差），落点稳定差 30 多像素。

我以为是自己算错了缺口位置，于是反复换识别方法：阈值检测、列直方图、连通域……**越换越乱**。

真相跟识别精度无关，是位移关系搞错了。

---

## 二、卡点 1：拼图位移是二次的

我一开始按 1:1 处理：缺口在 x=216，就让鼠标拖 216 px。

结果拼图只到 **182 px**，差 34 px。

**实测关系：**

```
piece.left = 0.0769 × m + 0.003552 × m²

m = 216  →  left = 0.0769×216 + 0.003552×216² = 16.6 + 165.8 = 182.4
```

反解——给定缺口位置，求鼠标该走多远：

```python
import math

A, B = 0.0769, 0.003552

def solve_mouse_travel(target_left):
    disc = A*A + 4*B*target_left
    return (-A + math.sqrt(disc)) / (2*B)

solve_mouse_travel(216)   # → 236.0
```

**改成 236 之后，落点误差从 30+ px 降到 1～3 px。** 判定码变成 `F001`（不再是位置问题了）。

> 这两个系数在某些分辨率／站点下需要重新标定。上游项目提供了标定脚本，我这次是直接复用了发布值，在另一个页面上复现成功。

---

## 三、卡点 2：F001 跟轨迹无关

位置准了之后，判定变成 `F001`（风控拦截）。

我以为是轨迹不够像人，于是：

- 换轨迹变体（`baseline` / `combo` / `peak_late` / `feedback`）
- 调总时长（0.7s ~ 1.6s 都试过）
- 加 y 轴漂移
- 降采样、改事件间隔

**全部无效，稳定 `F001`。**

真相是 `navigator.webdriver` 为 `true`。Playwright 默认就是 `true`。

加上这段（必须在任何页面脚本之前执行）：

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

**加完之后，同一套轨迹一次通过 `T001`。**

---

## 四、成功那一次的完整数据

| 项目 | 值 |
|---|---|
| 判定码 | `T001` |
| YOLO 置信度 | 0.949 |
| 缺口位置 target_left | 236.11 |
| 鼠标行程 D | 247.22 |
| 实际落点 | 237.43 |
| 落点误差 | **−1.32 px** |
| 轨迹变体 | `feedback` |
| 事件数 | 130 |
| 总时长 | 793 ms |
| 结果 | 取得签名链接，下载 **111.65 MB** |

判定码是这么读到的（挂在 Playwright 的响应监听上）：

```javascript
page.on("response", async (resp) => {
  if (/-verify/.test(resp.url())) {
    const d = await resp.json();
    let code = "";
    const r = d.Result;
    if (r && typeof r === "object") code = r.VerifyCode || "";
    else if (typeof r === "string") code = r;
    console.log("判定码:", code);
  }
});
```

**这个监听比什么都重要。** 没有它，你只知道「好像没通过」，不知道是位置错了还是被判机器。

---

## 五、失败的 3 次：同一套代码，换个域名就挂了

同一站点，把 `.net` 换成 `.cn`，同一套参数：

| 次数 | 判定码 | 落点误差 | 轨迹变体 |
|---|---|---|---|
| 1 | `F001` | −1.1 px | feedback |
| 2 | 超时（未取到判定） | — | baseline |
| 3 | `F001` | −1.74 px | feedback_peak_late |

**位置是准的（误差 1～2 px），环境也做了同样的覆盖，但依然判机器。**

推测是中文站风控更严（可能加了额外的环境检测或行为模型），**但没有验证**。

这条留在这里，就是为了说明：**同一套方案跨域名可能直接失效，别以为过了 A 站就能过 B 站。**

---

## 六、另外两个跟滑块无关、但一样耗时间的坑

留个记录，省得下次再踩：

| 现象 | 原因 | 解法 |
|---|---|---|
| 程序里逐帧发 CDP `Input.dispatchMouseEvent`，到 170 帧左右挂死（不返回、不报错） | 串行 `await` 堆压 | 换 `page.mouse.move()`，降采样（每 2 帧一次），每次调用包超时 |
| 拿到签名下载链接，命令行直接 GET 返回 `403` | 链接绑定浏览器会话 | 用 `context.request.get(url)`（共享 cookie jar） |

---

## 七、如果重来一次

| 顺序 | 动作 | 省多少 |
|---|---|---|
| 1 | 先挂 `-verify` 响应监听，看判定码 | 省掉全部盲目调参（这次是最后才想到的） |
| 2 | 见到 `F015` → 直接上二次反解 | 省掉十几轮换识别算法 |
| 3 | 见到 `F001` → 先查 `navigator.webdriver` | 省掉一轮换轨迹 |
| 4 | 换域名后从零重试，别假设能复用 | 省掉 3 次无效尝试 |

---

## 八、这份记录能证明什么、不能证明什么

**能证明：**

- 二次模型在另一个页面上复现成功（系数与上游发布值一致）
- `navigator.webdriver` 覆盖对判定码有决定性影响（同一轨迹：覆盖前 `F001`，覆盖后 `T001`）
- 判定码监听是唯一可靠的反馈通道

**不能证明：**

- 这套方案对任何站点都能过（中文站 3 次全败）
- 上游自报的单次通过率适用于你的目标站
- 换分辨率后系数不用重标（未测）
