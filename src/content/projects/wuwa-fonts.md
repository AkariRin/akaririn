---
title: "Wuwa Fonts"
published: 2026-06-26
description: "鸣潮字体库"
status: "published"
tags:
  - "CSS"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/wuwa-fonts"
  - label: "npm"
    icon: "simple-icons:npm"
    value: "https://www.npmjs.com/package/@aemeath-projects/wuwa-fonts"
---

鸣潮字体库

<div style="display:flex;flex-wrap:wrap;align-items:center;gap:0.5rem;margin:1rem 0">
  <img src="https://img.shields.io/npm/v/%40aemeath-projects%2Fwuwa-fonts?style=for-the-badge" alt="NPM Version" style="margin:0" />
  <img src="https://img.shields.io/github/license/aemeath-projects/wuwa-fonts?style=for-the-badge" alt="License" style="margin:0" />
</div>

## 快速安装

```bash
npm install @aemeath-projects/wuwa-fonts
```

在应用入口文件或站点组件中导入即可。

```js
import "@aemeath-projects/wuwa-fonts";                             // 默认 — Solaris Universal, weight 400
import "@aemeath-projects/wuwa-fonts/solaris-universal.css";       // 仅 Solaris Universal
import "@aemeath-projects/wuwa-fonts/digital-tube.css";            // 仅 Digital Tube
import "@aemeath-projects/wuwa-fonts/rinascita-ribbon.css";        // 仅 Rinascita Ribbon
import "@aemeath-projects/wuwa-fonts/old-font.css";                // 仅 Old Font
import "@aemeath-projects/wuwa-fonts/lahai-roi.css";               // 仅 Lahai-Roi
import "@aemeath-projects/wuwa-fonts/ragunna.css";                 // 仅 Ragunna
import "@aemeath-projects/wuwa-fonts/septimont.css";               // 仅 Septimont
import "@aemeath-projects/wuwa-fonts/400.css";                     // 全部 7 款, weight 400
import "@aemeath-projects/wuwa-fonts/latin-lahai-roi.css";         // 单家族, 带 unicode-range
```

## 支持的字体家族

所有家族均为 `400 normal`，子集桶名为 `latin`。字符覆盖以字体文件的实际字形为准：
对每个 `cmap` 条目逐字形检查轮廓、宽度和重复指纹，只排除空字形（空格、NBSP）与 `.notdef`，
其余真实字形全部写入 `unicode-range`。Wiki 对照表用于交叉核对，不再作为唯一裁剪依据。

| 字体家族                 | ID                 | 字符覆盖（实际字形）                                          |
|--------------------------|--------------------|----------------------------------------------------------------|
| Solaris Universal（默认）| `solaris-universal`| `0-9`、`A-Z`、`a-z`、`,` `.` `/` `:` `;` `(` `)` `[` `]`      |
| Digital Tube             | `digital-tube`     | `0-9`                                                          |
| Rinascita Ribbon         | `rinascita-ribbon` | `0-9`、`A-Z`、`a-z`                                            |
| Old Font                 | `old-font`         | `A-Z`、`a-z`、`"` `'` `,` `-` `.` `:`                          |
| Lahai-Roi                | `lahai-roi`        | `0-9`、`A-Z`、`a-z`、`%`、`.`、`Δ/δ Θ/θ Μ/μ Π/π Σ/σ Φ/φ Ψ/ψ`、`∂`、`𝑋`、`𝑥` |
| Ragunna                  | `ragunna`          | `A-Z`、`a-z`                                                   |
| Septimont                | `septimont`        | `A-Z`、`a-z`（含 J、Z）                                        |

精确 Unicode 范围见 [`unicode.json`](https://github.com/aemeath-projects/wuwa-fonts/blob/master/unicode.json)，结构与家族 ID 一一对应。

Septimont 的 J/Z 与 Old Font 的 a-z 均为字体文件内的真实字形，已纳入 `unicode-range`；
其中 Old Font 的大小写共用同一套轮廓。

## CSS 用法

导入后即可在 CSS 样式表、CSS Module 或 CSS-in-JS 中引用字体名称。

```css
body {
  font-family: "Solaris Universal", sans-serif;
}

.digital {
  font-family: "Digital Tube", monospace;
}

h1 {
  font-family: "Ragunna", cursive;
}

code {
  font-family: "Old Font", monospace;
}
```

## 许可与版权

本仓库中由 Aemeath Projects 原创的构建配置、脚本与文档以
[MIT License](https://github.com/aemeath-projects/wuwa-fonts/blob/master/LICENSE) 发布。

**字体文件的版权归库洛游戏（Kuro Games）所有，上述 MIT 许可不适用于字体文件。**
本项目仅做 TTF → WOFF/WOFF2 的格式转换与 npm 打包，不对字形主张任何权利，
也不代表权利人授予字体本身的使用或再分发许可。

原始字体未随附授权文件：`name` 表版权字段为 `Typeface © (TRI). 2024. All Rights Reserved`
（Old Font 为 FontCreator 模板默认值 `(your company)`），也没有 OFL 声明或 `LICENSE` 表。
如需在项目中使用这些字体，请先向权利人确认授权范围。

## 字体来源

来源页面：[鸣潮 WIKI 文字对照表](https://wuwa.huijiwiki.com/wiki/%E6%96%87%E5%AD%97%E5%AF%B9%E7%85%A7%E8%A1%A8)

感谢 B 站分享者：[言语关系](https://space.bilibili.com/379335206)、[鸟居枫](https://space.bilibili.com/10599)。

| 字体家族           | 上游文件                             |
|--------------------|--------------------------------------|
| Solaris Universal  | `WuWaSolarisUniversal-260927.ttf`    |
| Digital Tube       | `WuWaDigitalTube-Regular.ttf`        |
| Rinascita Ribbon   | `WuWaRinascitaRibbon-Regular.ttf`    |
| Old Font           | `WWOldFont-Regular.ttf`              |
| Lahai-Roi          | `WuWaLahai-RoiRegularNewXx.ttf`      |
| Ragunna            | `WuWa_Ragunna_Decorated.ttf`         |
| Septimont          | `WuWa_Septimont_Digital.ttf`         |

本项目仅进行 TTF 到 WOFF/WOFF2 的格式转换和 npm 打包，不修改字形。本项目与库洛游戏及原始字体作者无关。
