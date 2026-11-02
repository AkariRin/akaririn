---
title: "stagnant-illusion"
published: 2025-10-29
description: "静滞虚幻"
status: "published"
tags:
  - "TypeScript"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/AkariRin/stagnant-illusion"
  - label: "主页"
    icon: "material-symbols:link"
    value: "https://stagnant-illusion.rbq.dev"
---

为你的头像打上源石封印，使用Vuetify3重构，拥有更好的UI

## 原项目
原项目为[SusieGlitter/prts](https://github.com/SusieGlitter/prts)

## Cloudflare Pages API

### 端点

**POST** `/api`

### 请求

通过 JSON 请求体发送 POST 请求。

#### 请求体参数

```json
{
  "image": "base64_encoded_image",
  "pos-x": 0,
  "pos-y": 0,
  "scale": 0.8,
  "opacity": 0.8,
  "brightness": 100,
  "yellow": false,
  "yellowOpacity": 1
}
```

#### 参数说明

| 参数 | 类型 | 范围 | 默认值 | 说明                              |
|------|------|------|--------|---------------------------------|
| `image` | string | - | - | **必需**，Base64 编码的图像             |
| `pos-x` | number | [-1, 1] | 0 | 覆盖层水平位置（-1 左边，0 中心，1 右边）        |
| `pos-y` | number | [-1, 1] | 0 | 覆盖层垂直位置（-1 上方，0 中心，1 下方）        |
| `scale` | number | [0.1, 3] | 1 | 覆盖层缩放参数                         |
| `opacity` | number | [0, 1] | 0.8 | 覆盖层透明度（0 完全透明，1 完全不透明）          |
| `brightness` | number | [0, 200] | 100 | 图像亮度调整（100 为原始亮度）               |
| `yellow` | boolean | - | false | 是否叠加黄色源石                        |
| `yellowOpacity` | number | [0, 1] | 1 | 黄色源石的透明度（当 `yellow` 为 true 时生效） |

### 响应

#### 成功响应 (200)

返回 PNG 格式的合成图像。

**Headers:**
- `Content-Type: image/png`
- `X-Processing-Time: {毫秒}` - 处理耗时

#### 错误响应

**400 - 请求错误**
- 缺少 `image` 参数
- 参数值超出允许范围

```json
{
  "success": false,
  "message": "Invalid composition options",
  "error": {
    "code": "INVALID_OPTIONS",
    "message": "Options parameters do not meet requirements. Check: pos-x [-1,1], pos-y [-1,1], scale [0.1,3], opacity [0,1], brightness [0,200], yellow (boolean), yellowOpacity [0,1]"
  }
}
```

**405 - 方法不允许**

仅支持 POST 请求。

**500 - 服务器错误**

图像处理失败。

```json
{
  "success": false,
  "message": "Image processing failed",
  "error": {
    "code": "PROCESSING_ERROR",
    "message": "错误详情"
  }
}
```

### 使用示例

#### JavaScript/TypeScript

```typescript
const image = ''; // Base64 encoded image string

const response = await fetch('/api', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    image,
    'pos-x': 0,
    'pos-y': 0,
    'scale': 0.8,
    'opacity': 0.8,
    'brightness': 100,
    'yellow': false,
    'yellowOpacity': 1,
  }),
});

const imageBlob = await response.blob();
```

#### cURL

```bash
curl -X POST https://example.com/api \
  -H "Content-Type: application/json" \
  -d '{
    "image": "base64_encoded_image_here",
    "pos-x": 0,
    "pos-y": 0,
    "scale": 0.8,
    "opacity": 0.8,
    "brightness": 100,
    "yellow": false,
    "yellowOpacity": 1
  }'
```
