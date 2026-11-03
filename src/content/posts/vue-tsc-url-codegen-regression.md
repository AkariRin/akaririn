---
title: "96 个类型错误，根因是 URL 里的双斜杠：一次 vue-tsc codegen 回归排查实录"
published: 2026-11-03T02:22:59+08:00
description: "一次依赖升级后，vue-tsc 从 0 个错误变成 96 个，报错却全指向无关变量。顺着虚拟代码往下挖，根因是 @vue/language-core 在 3.3.12 引入的一行双斜杠转义正则误伤了 URL 字面量。本文记录完整的定位过程、最小复现，以及截至 2026 年 11 月的上游修复进展。"
tags: [Vue, vue-tsc, TypeScript, 前端, 排错]
category: "前端"
image: ""
draft: false
comment: true
---

升级依赖本来是最普通的维护动作。某天把前端依赖整体升了一版，`pnpm type-check` 从 0 个错误变成 96 个。报错内容大致是：

```text
src/views/jrlp/JrlpView.vue(67,42): error TS2304: Cannot find name 'item'.
src/views/user/AdminView.vue(35,57): error TS18047: 'currentAdmin.value' is possibly 'null'.
src/components/UserInfoCard.vue(88,31): error TS2304: Cannot find name 'props'.
```

`item` 是 `v-for` 里刚声明的循环变量，`currentAdmin` 在 `v-if` 里已经判空，`props` 是 `<script setup>` 的宏参数。这些名字在源码里明明存在，错误却像从另一个文件里飘过来的。更怪的是，报错文件几乎都含 `https://`。

这篇文章记录完整的定位过程，以及截至 2026-11-03 的上游状态。

## 一、先把"全是误报"这个结论推翻

第一反应是"vue-tsc 又变严了，全是误报"。我们甚至一度在项目记录里写下"96 个错误逐条核对全部为误报，暂缓处理"。这个结论是错的，原因有两个：

1. 它没有解释为什么偏偏是这些文件报错；
2. 它没有解释为什么报错变量恰好是模板作用域里的那一批。

真正有用的观察是**错误的分布形态**：同一个文件里，`v-for` 的循环变量、`v-slot` 的插槽变量、`v-if` 收窄后的值，会成片地报"找不到"或"可能为 null"。这不是类型检查变严，而更像是 TypeScript 根本没有建立这个模板的作用域。

做了一次版本矩阵对照后，问题就清楚了：

| vue-tsc | 错误数 | 说明 |
|---|---|---|
| 3.3.10 | 0 | 正常 |
| 3.3.11 | 1 | 真实缺陷：`:key` 传了对象 |
| 3.3.12 | 96 | 48 × TS2304 + 46 × TS18047 + 1 × TS7006 + 1 × TS2322 |
| 3.3.13 | 96 | 与 3.3.12 相同 |

也就是说，3.3.12 把 1 个真实错误放大成了 96 个。其中 48 个 `Cannot find name`、46 个 `possibly null`，以及那个 `implicitly has an 'any' type`（模板作用域丢失后，回调参数失去了推断来源），都是同一处 codegen 故障的级联产物。只有 1 个 TS2322 是真正需要改代码的问题。

当错误呈现出"整片作用域变量同时消失"的形态时，不要继续逐条核对类型，先怀疑解析。

## 二、最小复现：组件属性里的 `//`

把问题缩减到最小复现后，触发条件简单得有点意外：**组件属性值里出现 `//`**。

```vue
<!-- Child.vue -->
<script setup lang="ts">
defineProps<{ label?: string }>()
</script>

<template>
  <span>{{ label }}</span>
</template>
```

```vue
<!-- App.vue -->
<script setup lang="ts">
import { ref } from "vue"
import Child from "./Child.vue"

const items = ref([{ id: 1, name: "a" }])
const enabled = ref(true)
</script>

<template>
  <div v-if="enabled">
    <div v-for="item in items" :key="item.id">
      <Child :label="`https://example.com/${item.id}`" />
      <p>{{ item.name }} {{ enabled.value ? "on" : "off" }}</p>
    </div>
  </div>
</template>
```

用两个版本分别跑 `vue-tsc --noEmit`（复现环境：Vue 3.5.44、TypeScript 6.0.3、Node 26.12.1）：

```text
# vue-tsc 3.3.11
exit 0

# vue-tsc 3.3.14
src/App.vue(12,45): error TS2552: Cannot find name 'item'. Did you mean 'items'?
src/App.vue(13,13): error TS2552: Cannot find name 'item'. Did you mean 'items'?
src/App.vue(13,37): error TS2551: Property 'value' does not exist on type 'true'. Did you mean 'valueOf'?
```

几个边界值得记下来：

- 静态属性 `label="https://example.com"` 和绑定 `` :label="`https://example.com/${item.id}`" `` 都会触发。
- 触发点不必和报错变量在同一个元素上。同一文件里只要有一个组件属性含 `//`，整个模板作用域的绑定都可能失效。
- 原生元素 `<img src="https://example.com">` 不触发，`<template v-for>` 也不触发。只有组件的 props 会进入出问题的那段 codegen。

## 三、根因：把注释转义，却误伤了字符串

问题出在 `@vue/language-core` 的模板代码生成。3.3.12 之前，组件 props 是直接按原样 yield 的：

```js
yield propsStr;
```

3.3.12 引入了一项改动（上游 PR #6230，2026-09-29 合并），为了让 props 字符串保持单行，同时不丢失 `//` 行注释的语义，改成了这样：

```diff
- yield propsStr;
+ yield `// @ts-ignore${newLine}`;
+ yield propsStr
+   .replace(/\/\/(.*)$/gm, (_, body) => `/*${body.replaceAll('*/', '*\\/')} */`)
+   .replace(/\n/g, ' ');
```

意图是合理的：props 字符串里可能含 `//` 行注释，而后面会把换行压成空格，行注释会吞掉同一行后面的代码，所以先把 `//` 注释改写成 `/* ... */` 块注释。

问题在于，这个正则**只做纯文本匹配，不认识字符串字面量**。它会把 prop 值里的 `//` 也当成注释，而 URL 里恰好到处都是 `//`。

`<Child label="https://example.com" />` 生成的虚拟 TypeScript 会从：

```ts
label: (`https://example.com`),
```

变成：

```ts
label: (`https:/*example.com`), */
```

`/*` 落在字符串字面量内部，只是普通文本，无害；但配对插入的 `*/` 落在了字符串外部。孤立的 `*/` 没有真正开启注释的 `/*` 与之配对，TypeScript 解析器在这里直接报错，整个表达式被截断，虚拟文件因此语法非法。

今天用 3.3.14 实际 dump 出来的生成代码是这样的（两个 `label` 分别来自 props 类型推断副本和实际调用）：

```ts
const __VLS_2 = __VLS_asFunctionalComponent1(__VLS_1, new __VLS_1({
// @ts-ignore
label: (`https:/*example.com/${item.id}`), */ }));
const __VLS_3 = __VLS_2({
label: (`https://example.com/${item.id}`),
}, ...__VLS_functionalComponentArgsRest(__VLS_2));
```

第二份调用其实是对的，但前一份推断副本已经足以让整个虚拟文件解析失败。

TypeScript 解析中断后，模板作用域从未被正确绑定，于是该文件里所有模板变量都变成"找不到"，所有 `v-if` 收窄都失效。报错位置和 URL 毫无关系，诊断信息里也不会出现 `https`，这就是它最难查的地方。

:::note[一处需要纠正的旧记录]

早期排查记录把出问题的位置写成了 `generateElement`。实际是 `generateComponent`（3.3.14 的 `lib/codegen/template/element.js:185`）。这个区别很重要：原生元素由 `generateElement` 处理，不经过这段转义，所以不受影响。

:::

## 四、把虚拟代码挖出来

`vue-tsc` 输出的诊断已经映射回 `.vue` 源码，光看报错很难定位。更直接的办法是绕过映射，dump `@vue/language-core` 生成的虚拟 TypeScript。

```js
import ts from "typescript"
import fs from "node:fs"
import path from "node:path"
import {
  createVueLanguagePlugin,
  getDefaultCompilerOptions,
} from "@vue/language-core"

const fileName = path.resolve("src/App.vue")
const snapshot = ts.ScriptSnapshot.fromString(
  fs.readFileSync(fileName, "utf8"),
)

const vueCompilerOptions = {
  ...getDefaultCompilerOptions(),
  extensions: [".vue"],
  plugins: [],
  vitePressExtensions: [],
  petiteVueExtensions: [],
}

const plugin = createVueLanguagePlugin(ts, {}, vueCompilerOptions, (id) => id)
const code = plugin.createVirtualCode(fileName, "vue", snapshot)

// 3.3.14 的 script_ts 嵌在 root_tags 下，需要递归收集
const all = []
const walk = (c) => {
  all.push(c)
  for (const child of c.embeddedCodes ?? []) walk(child)
}
walk(code)

for (const embedded of all) {
  fs.writeFileSync(
    `virtual-${embedded.id}.txt`,
    embedded.snapshot.getText(0, embedded.snapshot.getLength()),
  )
}
```

然后在输出里搜 URL：

```text
$ grep -n "example.com" virtual-script_ts.txt
33:label: (`https:/*example.com/${item.id}`), */ }));
35:label: (`https://example.com/${item.id}`),
```

一眼就能看到第 33 行被插入了 `/*` 和 `*/`。把这份 dump 写成 `.ts`，用普通 `tsc` 编译，会得到 `TS1003` / `TS1109` / `TS1128` 之类的语法错误，而不是类型错误——这就确认了是解析中断，不是类型不匹配。

几个容易踩的坑：

- 3.3.14 里 `script_ts` 不再直接挂在顶层 `embeddedCodes` 上，而是嵌在 `root_tags` 下面，需要递归。
- `resolveEmbeddedCode` 内部会 `catch` 异常并打到 stderr。看到 dump 出来是空内容，先查 stderr，不要以为文件本来就没有内容。
- 如果只想快速判断是不是这个问题，不必写 dump 脚本：把报错文件里组件属性中的 `//` 全部改成 script 常量，错误成片消失，基本就可以确认。

## 五、修复与规避

### 方案 A：精确 pin 3.3.11（我们的选择）

```json
{
  "devDependencies": {
    "vue-tsc": "3.3.11"
  }
}
```

注意**不要加 `^`**。`^3.3.11` 会在下一次 `pnpm update` 时解析到 3.3.14，把 bug 原样带回来。`vue-tsc@3.3.11` 对 `@vue/language-core` 的依赖本身就是精确版本，因此不需要额外写 `overrides`。

### 方案 B：把 URL 提到 script 里

另一个项目（Vue + Vuetify）选择了不降级，而是让模板里不出现含 `//` 的字面量：

```vue
<script setup lang="ts">
const githubUrl = "https://github.com/AkariRin/bookshelf"
</script>

<template>
  <v-btn :href="githubUrl" icon="mdi-github" target="_blank" />
</template>
```

这个办法在 3.3.12 / 3.3.13 下确实有效，因为生成的 props 代码里不再有 `//`。代价是要长期遵守一条纪律：模板里的组件属性永远不能直接写 URL。一旦复发，报错仍然是与 URL 无关的 `Cannot find name`，排查成本很高。

### 方案 C：用 pnpm patch 改正则

如果既不想降级、又不想改业务代码，可以打补丁，把匹配范围限制到行首注释：

```js
.replace(
  /^([ \t]*)\/\/(.*)$/gm,
  (_, indent, body) => `${indent}/*${body.replaceAll('*/', '*\\/')} */`,
)
```

在原始案例里，这一改动让错误从 96 个降到 1 个。但它需要你自己维护补丁，直到上游修复。

### 顺手修掉真问题

`MessageSegmentRenderer.vue` 里原本写的是：

```vue
<template v-for="seg in segments" :key="seg">
```

`MessageSegment` 是对象，没有唯一 id。把对象传给 `:key`，运行时会被 `String()` 成 `"[object Object]"`，同一消息的各段 key 全部重复，diff 复用会出错。改成索引：

```vue
<template v-for="(seg, segIndex) in segments" :key="segIndex">
```

这个问题是 3.3.11 新增的 `<template>` props 检查（#6155）暴露出来的。3.3.10 及以前不检查 `<template>` 上的 props，所以它不是 codegen 回归产生的，在 3.3.11 上会直接暴露出来。

当时的最终验证：`pnpm type-check` 零错误，547 项测试全过，`pnpm lint` 干净。

## 六、上游进展（2026-11-03 核查）

| 日期 | 事件 |
|---|---|
| 2026-09-29 | PR #6230 合并，引入 props 单行化与 `//` 转义 |
| 2026-10-02 | vue-tsc 3.3.12 发布，回归出现 |
| 2026-10-03 | Issue #6240 提交，含最小复现 |
| 2026-10-10 | vue-tsc 3.3.13 发布，仍含 bug |
| 2026-10-13 | 我们提交的 #6264 被关闭，标记为 #6240 的 duplicate |
| 2026-10-20 | PR #6266 关闭且未合并，修复回到 #6241 / #6277 两条线 |
| 2026-10-29 | vue-tsc 3.3.14 发布，正则仍在 `element.js:185` |
| 2026-10-30 | Issue #6322 提交，记录 3.3.14 的新症状 |

截至今天：

- **#6240 仍 open**，#6241 仍 open。后者是社区提交的修复，`mergeable_state` 为 `clean`，把正则改成了能识别字符串字面量的版本，并补了 #6240 / #6322 两个 tsc fixture，但尚未合并。
- **npm 上的 latest 仍是 3.3.14，问题也还在**。今天用最小复现实测，3.3.11 退出码 0，3.3.14 报 `TS2552` 与 `TS2551`。
- **症状发生了变化**。3.3.14 把模板虚拟代码与 setup 控制流隔离后，损坏的推断副本还会让 `.value` 收窄失效，于是除了 `Cannot find name`，还会出现 `Property 'value' does not exist`（见 #6322）。旧的报错文本不能作为唯一判据。
- **4.0.0-alpha.3 不包含这段正则**。4.0 的 codegen 已重构，相关代码在 `lib/template/elements.js`，但它的 peer dependency 要求 TypeScript ≥ 7.1.0-beta，目前不适合放进生产项目。

结论仍然是：**继续精确 pin `vue-tsc@3.3.11`**。升级前至少确认一件事：`node_modules/@vue/language-core/lib/codegen/template/element.js` 里那段 `.replace(/\/\/(.*)$/gm, ...)` 已经消失，或者 #6241 已合并并随新版本发布。不要只看版本号。

## 七、复盘

1. **解析错误会伪装成类型错误。** 当同一文件的模板作用域成片消失时，先怀疑 codegen 或 parse，而不是类型系统变严。
2. **"全是误报"是一个需要被证伪的结论。** 最小复现是分界线。没有最小复现之前，"误报"只是猜测。
3. **编译器类工具适合精确 pin。** caret 范围会把已知 bug 悄悄带回来，而且是在一次看起来无关的 `pnpm update` 里。
4. **跟踪上游要跟 duplicate target。** 我们最初开的 #6264 被关闭，真正有效的是更早的 #6240。重复开 issue 只会分散信息。
5. **症状会随上游重构漂移。** 3.3.12 的 `Cannot find name` 和 3.3.14 的 `Property 'value' does not exist` 是同一个根因。判断依据应该是最小复现，不是错误文本。
6. **给上游报 bug 时，版本矩阵、最小复现、生成的虚拟代码**比大段文字描述有用得多。

如果你也在升级 `vue-tsc` 后看到成片的 `Cannot find name` 或 `possibly null`，先搜一下模板里的 `https://`。大概率不是你的代码变差了，而是 props 里的 `//` 被当成了注释。
