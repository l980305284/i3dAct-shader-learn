# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个**i3dAct引擎Shader教程文章**写作项目。目标是撰写一套8篇的shader入门+实战系列文章。

## 引擎信息

目标引擎是 **i3dAct**，命名规范详见 [I3DAct_Engine_Reference.md](I3DAct_Engine_Reference.md)。

关键命名映射：
- 节点基类: `Node` → `Item`
- 3D基类: `Node3D` → `Item3D`
- 网格实例: `MeshInstance3D` → `MeshRender`
- 脚本文件: `.gd` → `.s3`
- 场景文件: `.tscn` → `.iscn`
- 初始化函数: `_ready()` → `iStart()`
- 更新函数: `_process()` → `Update()`

**重要**：文章中不出现其他引擎名称，只使用i3dAct命名。

## 文章进度

详见 [PROGRESS.md](PROGRESS.md)。

已完成：第1-12篇（Shader入门 → 剖切动画）
待完成：第13-49篇（全屏Shader、风格化渲染、顶点着色器等）

## 写作规范

详见 [ARTICLE_PLAN.md](ARTICLE_PLAN.md)。

### 禁止使用
- "在今天的游戏开发领域..."
- "众所周知..."
- "值得注意的是..."

### 必须做到
- 每个概念先用代码/图示，再解释
- 代码块必须有实际意义
- 文章末尾添加求点赞语
- 使用i3dAct命名规范

### 文章结尾格式
```
---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲XXX，内容更有意思。
```

## 文件结构

```
shader-learn/
├── articles/           # 文章目录
│   ├── 01-shader-introduction.md
│   ├── 02-uv-coordinates.md
│   └── 03-coordinate-spaces.md
├── ARTICLE_PLAN.md     # 文章规划
├── PROGRESS.md         # 写作进度
└── I3DAct_Engine_Reference.md  # 引擎命名参考
```
