---
title: "trpg"
published: 2026-07-25
description: "TRPG 骰娘规则引擎：无状态掷骰求值、CoC 7/6 版与 D&D 5e 规则判定、人物生成公式、日志状态机等纯函数集合"
status: "published"
tags:
  - "TypeScript"
  - "coc"
  - "dice-roller"
  - "dnd"
  - "trpg"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/aemeath-projects/trpg"
  - label: "npm"
    icon: "material-symbols:package-2"
    value: "https://www.npmjs.com/package/@aemeath-projects/trpg"
---

骰娘规则引擎——掷骰、CoC/D&D 判定、人物生成、日志状态机，纯函数，双端可用。

![NPM Version](https://img.shields.io/npm/v/%40aemeath-projects%2Ftrpg?style=for-the-badge)
![CI Status](https://img.shields.io/github/actions/workflow/status/aemeath-projects/trpg/publish.yml?style=for-the-badge)
![Codecov](https://img.shields.io/codecov/c/github/aemeath-projects/trpg?style=for-the-badge)
![License](https://img.shields.io/github/license/aemeath-projects/trpg?style=for-the-badge)

## 这是什么

一个给骰娘 Bot 用的规则计算库。它不做任何 I/O——不连数据库、不发网络请求、也不绑定任何 Bot 框架。你把数据喂进来，它算完还给你，怎么存是你的事。

所有模块都可以按 subpath 单独导入，打包工具会自动 tree-shake 掉不用的部分：

```ts
import { rollExpression } from '@aemeath-projects/trpg/dice'
import { generateCharacter } from '@aemeath-projects/trpg/coc'
```

除了 `story`（归档编解码）依赖 `msgpackr` + `fflate` 两个纯 JS 包，其余模块零依赖。不导入 `story` 就不会把这些依赖打进去。

Node 和浏览器行为一致——加密走 WebCrypto，没用 `node:crypto`。

## 安装

```bash
pnpm add @aemeath-projects/trpg
```

需要 Node.js ≥ 24.17。

## 模块

| 模块 | 导入路径 | 主要导出 |
|---|---|---|
| **dice** | `.../trpg/dice` | `rollExpression` · `maxOfExpression` · `rollDie` |
| **coc** | `.../trpg/coc` | `generateCharacter` · `judgeCoCResult` · `applySanityCheck` · `applyGrowthCheck` · `rollBonusPenaltyD100` |
| **dnd** | `.../trpg/dnd` | `computeSkillModifier` · `rollNamedAbilityScoreSets` · `applyDeathSaveRoll` · `initSlots` · `restSlots` · `generateName` |
| **decks** | `.../trpg/decks` | `draw` |
| **stats** | `.../trpg/stats` | `summarize` |
| **log** | `.../trpg/log` | `start` · `pause` · `resume` · `end` · `computeDuration` |
| **story** | `.../trpg/story` | `packStory` · `unpackStory` · `packStoryEncrypted` · `unpackStoryEncrypted` · `readStoryHeader` · `isStoryEncrypted` |
| **types** | `.../trpg/types` | `RandomSource` · `CheckResult` |

## 快速上手

### 掷骰

```ts
import { rollExpression, maxOfExpression } from '@aemeath-projects/trpg/dice'

const r = rollExpression('4d6kh3')   // D&D 属性掷骰：4d6 取最高 3 个
r.total   // → 15
r.detail  // → "4D6=[5,3,6,2]=>(kh3)[5,3,6]=14"

maxOfExpression('3d6+2')            // → 20，不掷骰，纯理论上限
```

支持 `NdM`、`d%`、四则运算、`khN` / `klN`（取高/取低）、嵌套括号。

### CoC 人物生成与检定

```ts
import { generateCharacter, judgeCoCResult } from '@aemeath-projects/trpg/coc'

const char = generateCharacter('coc7')  // 7 版人物，含八围 + 派生属性

const outcome = judgeCoCResult({ roll: 45, skill: 60, rule: 'coc7' })
outcome.result       // → 'success'
outcome.description  // → 中文描述
```

`judgeCoCResult` 内置 6 版和 7 版共 8 种房规策略。

#### 理智检定与幕间成长

```ts
import { applySanityCheck, applyGrowthCheck } from '@aemeath-projects/trpg/coc'

// san check：传入当前 san，返回损失量和疯狂状态
const outcome = applySanityCheck({ currentSanity: 42, maxSanity: 99, lossExpr: '1d6/1d20' })

// 幕间成长
const grown = applyGrowthCheck({ skillName: '图书馆使用', currentValue: 45 })
```

### D&D 5e

```ts
import {
  rollNamedAbilityScoreSets,
  computeSkillModifier,
  applyDeathSaveRoll,
  initSlots,
  generateName,
} from '@aemeath-projects/trpg/dnd'

// 6 属性掷骰（4d6kh3）
const scores = rollNamedAbilityScoreSets()

// 技能调整值
const mod = computeSkillModifier({ attributeMod: 3, isProficient: true, proficiencyBonus: 2 })
mod.total   // → 5

// 死亡豁免状态机
const state = applyDeathSaveRoll(prevState, { roll: 15 })
state.successes  // → 1

// 法术位
const slots = initSlots({ level: 3, slots: { 1: 4, 2: 2 } })

// 随机姓名（按种族音节拼接）
const name = generateName('elf')
```

### 抽取 & 统计

```ts
import { draw } from '@aemeath-projects/trpg/decks'
import { summarize } from '@aemeath-projects/trpg/stats'

draw(['A', 'B', 'C', 'D'], 2)  // → ['C', 'A']（无放回）

const summary = summarize([{ result: 'success' }, { result: 'great_success' }, { result: 'fail' }])
summary.successRate        // → 0.666...
summary.greatSuccessCount  // → 1
```

### 日志状态机

```ts
import { start, pause, resume, end, computeDuration } from '@aemeath-projects/trpg/log'

let log = start(Date.now())           // → recording
log = pause(log, Date.now())          // → paused
log = resume(log, Date.now())         // → recording
log = end(log, Date.now())            // → ended

computeDuration(log, Date.now())      // → 累计有效毫秒（已扣除暂停）
```

状态转换：`recording` ⇄ `paused` → `ended`。

### `.story` 跑团日志归档

`.story` 是跑团记录的交换格式——8 字节头 + msgpack 载荷（默认 gzip，可选口令加密）。Bot 导出它，在线查看器打开它。

```ts
import { packStory, unpackStory, STORY_EXTENSION } from '@aemeath-projects/trpg/story'
import type { StoryDocument } from '@aemeath-projects/trpg/story'

const doc: StoryDocument = {
  meta: {
    logName: '深海余烬',
    createdBy: '10001',
    startedAt: 1769472000000,
    endedAt: 1769486400000,
    durationMs: 12600000,
    entryCount: 2,
    exportedAt: Date.now(),
    source: 'My Dice Bot',
  },
  actors: [
    { id: '10001', name: 'KP老王' },
    { id: '10002', name: '调查员小李', characterName: '陈默',
      avatar: 'https://example.com/avatar/10002.png' },
  ],
  characters: [
    { name: '陈默', ruleset: 'coc7', owner: 1,
      attributes: { 力量: 50, 侦查: 60, san: 55 } },
  ],
  entries: [
    { offset: 0, actor: 0, type: 'message', text: '你们推开了灯塔的门。' },
    { offset: 22400, actor: 1, type: 'roll', character: 0,
      text: '陈默 进行 侦查 检定：D100=23/60 困难成功',
      roll: { expression: '1d100', total: 23, detail: 'D100=23', rolls: [23],
              skill: '侦查', target: 60, outcome: 'hard_success' } },
  ],
}

const bytes = packStory(doc)
const restored = unpackStory(bytes)
```

文档四段：`meta` → `actors` → `characters` → `entries`，段之间用下标引用。`offset` 是相对 `startedAt` 的毫秒偏移，`text` 只存纯文本（图片/语音压成占位符），`characters` 独立成段不挂在 actor 下。`unpackStory` 内部按版本号分派解析器，你拿到的永远是同一种 `StoryDocument`。

```ts
import { readStoryHeader, isSupportedStoryVersion } from '@aemeath-projects/trpg/story'

const header = readStoryHeader(bytes)   // 只读 8 字节头
if (!isSupportedStoryVersion(header.version)) { /* 格式太新或已废弃 */ }
```

#### 口令加密

```ts
import { packStoryEncrypted, unpackStoryEncrypted, isStoryEncrypted } from '@aemeath-projects/trpg/story'

const bytes = await packStoryEncrypted(doc, 'a very long passphrase')
isStoryEncrypted(bytes)   // → true
const restored = await unpackStoryEncrypted(bytes, 'a very long passphrase')
```

AES-256-GCM + PBKDF2-HMAC-SHA256，走 WebCrypto。先压缩后加密。口令丢了归档永久不可读。

### 可复现随机

所有接受随机数的函数都支持注入 `RandomSource`：

```ts
import seedrandom from 'seedrandom'
import { rollExpression } from '@aemeath-projects/trpg/dice'

const rng = seedrandom('fixed-seed')
rollExpression('3d6', rng)  // 每次相同输入返回相同结果
```
