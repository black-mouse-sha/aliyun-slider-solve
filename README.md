# 阿里云滑块求解

仓名：`aliyun-slider-solve`
副标题：**Aliyun slider captcha — mechanism and solver.**
中文说明：阿里云滑块验证码（aliyunCaptcha）的机制拆解与自动化求解。
署：**black-mouse-sha**

> 文档状态：**v0.1**，一次真实折腾的记录。
> 已实测通过 1 次（取得签名下载链接，成功下载 111.65 MB 安装包，判定码 `T001`）。
> 已实测失败 3 次（同一站点的中文域名，全部 `F001`，未通过）。
> **失败样本原样留在正文与 [`examples/`](examples/aliyun-slider/README.md) 里，不做修剪。**
> 不承诺任何站点的通过率；换厂商就换协议，参数无法沿用。

## 这是什么

一个把「滑块过不去」拆成可定位问题的技能。

阿里云滑块有两个独立难点，很多人只解决了一个，于是要么卡在位置、要么卡在风控：

| 难点 | 表现 | 判定码 | 改哪里 |
|---|---|---|---|
| **算法**：拼图位移与鼠标行程不是 1:1，是二次关系 | 落点总差 30+ 像素 | `F015` | 二次模型反解 |
| **环境**：`navigator.webdriver` 为 true 时轨迹再像也判机器 | 位置对了仍被拦 | `F001` | initScript 覆盖 |

**先看判定码，再动手。** 不看码就调参数，等于蒙着眼睛开车。

## 对我有什么用

- **失败能定位。** 每次失败都有判定码，`F015` 和 `F001` 通向完全不同的修法，不用靠猜。
- **省掉几小时的盲目调参。** 二次模型这一条就能把落点误差从 30+ px 降到 1～3 px；环境覆盖这一条能让同一套轨迹从连续失败变为通过。
- **知道边界在哪。** 正文写了「这套办法不适用于极验／腾讯／网易／数美」，也写了中文站实测失败。不吹。

## 核心结论（最值钱的两条）

**一、拼图位移是二次的**

```
piece.left = 0.0769 × m + 0.003552 × m²

鼠标走 216 px  →  拼图只到 182 px   （差 34 px，判 F015）
鼠标走 236 px  →  拼图正好到 216 px （对）
```

反解：

```python
disc = A*A + 4*B*target_left
D = (-A + math.sqrt(disc)) / (2*B)      # A=0.0769, B=0.003552
```

**二、F001 跟轨迹无关**

`navigator.webdriver` 必须为 `false`，要在任何页面脚本之前用 `addInitScript` 覆盖，并连带补 `chrome.runtime`、`plugins`、`languages`、WebGL vendor。实测：加之前连续 `F001`，加之后同一套轨迹一次 `T001`。

## 仓库结构

```
skills/aliyun-slider-solve/SKILL.md   判定码 + 二次模型 + 轨迹 + 环境覆盖
examples/aliyun-slider/README.md      一次完整记录（1 次成功 / 3 次失败）
```

## 来源

核心方法来自 [`kangleyao/slider-captcha-lab`](https://github.com/kangleyao/slider-captcha-lab)：二次位移模型、人形轨迹生成、CDP 注入思路。本仓做的是在另一个页面上复现与补充实测。见 [`NOTICE`](NOTICE)。

## 说明

技术记录，不是教程书。所有结论标了验证程度：「已实测」的是真跑过的，「机制说明」的是没跑通的。有误欢迎提 issue。
