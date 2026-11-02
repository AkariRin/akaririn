---
title: "Tesseract"
published: 2025-12-03
description: "超维立方"
status: "published"
tags:
  - "Java"
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/AkariRin/Tesseract"
---

适用于 Minecraft 1.20.1 的 Forge 模组，依赖 Mekanism。

## 功能

- **超维立方方块**：具有末地传送门视觉效果的特殊方块，亮度 15，硬度极高，不可被活塞推动
- **驱动器阵列 GUI**：与 Mekanism 存储系统集成的驱动器阵列管理界面
- **仪表盘 GUI**：超维立方状态与控制面板

## 使用方式

### 获取方块

在创造模式"超维立方"标签页中找到**超维立方**，或使用指令：
```
/give @s tesseract:tesseract
```

### 放置与交互

放置超维立方后，有两种交互方式：

| 操作 | 效果 |
|------|------|
| **右键**（站立） | 打开 QIO 仪表板（物品搜索、查看、合成） |
| **Shift + 右键**（蹲下） | 打开驱动器阵列（插入/取出 QIO 驱动器） |

### QIO 存储使用流程

1. **Shift + 右键**打开驱动器阵列，插入 Mekanism QIO 驱动器
2. 在驱动器阵列界面配置 QIO 频率（用于多方块共享存储）
3. **右键**打开仪表板，搜索和提取物品，或使用 QIO 合成工作台

### 方块特性

- 亮度 15，可作为光源
- 硬度 100 / 爆炸抗性 3,600,000，需合适工具破坏
- 不可被活塞推动
- 支持 Mekanism 安全系统（放置者自动成为所有者）

## 依赖

| 依赖 | 版本 |
|------|------|
| Minecraft | 1.20.1 |
| Forge | 47.4.10 |
| Mekanism | 1.20.1-10.4.16.80 |
| Java | 17 |

## 构建

```bash
./gradlew build
```

产物位于 `build/libs/`。

## 开发运行

```bash
# 客户端
./gradlew runClient

# 服务端
./gradlew runServer
```

## Mod 信息

- **Mod ID**：`tesseract`
- **包名**：`dev.rbq.tesseract`
