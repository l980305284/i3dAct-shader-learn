# 第10篇 SDF 入门文章 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 撰写第10篇 Shader 教程文章《SDF入门：让切割不再只能走直线》，输出到 `articles/10-sdf-intro.md`

**Architecture:** 单文件 Markdown 文章，6个内容段 + 结尾。前端回顾第8-9篇的平面切割局限，中部用可视化手法引入 SDF 概念，后端实战球形切割。写作风格延续已完成的第6-9篇：口语化中文、代码先行后解释、i3dAct 命名规范。每步写完后追加到文件。

**Tech Stack:** Markdown、GLSL（i3dAct spatial shader）、i3dAct Script（.s3）

---

### Task 1: 撰写开篇段落（"一刀切"的局限 → 引入 SDF）

**Files:**
- Create: `articles/10-sdf-intro.md`

- [ ] **Step 1: 写入开篇段落**

写入以下内容作为文章开头：

```markdown
# SDF入门：让切割不再只能走直线

第8篇我们用 `discard` + 平面方程把模型切开了。第9篇我们给切面上了色。

但有个问题不知道你试过没有——平面切割只能直着切。`dot(world_pos, normal) - distance` 描述的永远是一个无限大的平面，一刀下去横平竖直。

如果你想在模型上挖一个球形的洞呢？想啃出一个带圆角的缺口呢？平面方程做不到——它不知道"圆"是什么。

这时候该 **SDF（Signed Distance Field，有向距离场）** 上场了。

一句话概括 SDF：**给你空间中的任意一点，它告诉你这个点到最近表面的距离——以及你是在表面外面还是里面。**

听起来抽象？几行代码就懂了。
```

- [ ] **Step 2: 追加写入第二部分开头——可视化 SDF**

写入以下内容（衔接上文）：

```markdown
## 先"看见"SDF

在理解 SDF 之前，我们直接把它画出来。球体的 SDF 公式就一行：

```glsl
float sdf_sphere(vec3 p, float r) {
    return length(p) - r;
}
```

`length(p)` 是点到原点的距离。减去半径 `r`，结果就是：

- `p` 刚好在球面上 → 距离 = `r` → 返回 `0`
- `p` 在球外面 → 距离 > `r` → 返回 **正数**
- `p` 在球里面 → 距离 < `r` → 返回 **负数**

这就是"有向"的意思——正负号告诉你方向。

### 用颜色把 SDF 画出来

把这个函数的结果当颜色输出，你能直接"看见"距离场：

```glsl
shader_type spatial;

varying vec3 world_pos;

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    // 以世界原点为中心，半径2.0的球体 SDF
    float d = length(world_pos) - 2.0;

    // 把 SDF 值映射为灰度色
    // d > 0（球外）→ 偏白，d < 0（球内）→ 偏黑，d ≈ 0 → 中灰
    float gray = d * 0.5 + 0.5;
    gray = clamp(gray, 0.0, 1.0);

    ALBEDO = vec3(gray);
}
```

把这个 shader 贴到一个足够大的平面上（或者一个 Cube），你会看到：

- 一个黑色圆球区域（SDF < 0，在里面）
- 球边缘是灰色（SDF ≈ 0，在表面上）
- 球外围逐渐变白（SDF > 0，在外面，越来越远）

这就是 SDF 的"场"——空间中每个像素到球面的距离，平滑地从黑过渡到白。

如果你用平面方程也这么画，它永远只能画出"一刀切"的直线渐变。而 SDF 画出了曲线——这就是它能做曲线切割的原因。
```

**预计行数**: ~70行（含此段）

---

### Task 2: 撰写基础 SDF 形状段落

**Files:**
- Modify: `articles/10-sdf-intro.md`（追加）

- [ ] **Step 1: 追加球体 SDF 完整写法 + 盒子 SDF**

写入以下内容：

```markdown
## 两种最常用的 SDF

球体的 SDF 我们刚才见过了。加上圆心偏移的完整版：

```glsl
// 球体 SDF
float sdf_sphere(vec3 p, vec3 center, float radius) {
    return length(p - center) - radius;
}
```

第二个必学的 SDF 是**盒子**：

```glsl
// 盒子 SDF
float sdf_box(vec3 p, vec3 center, vec3 half_size) {
    vec3 q = abs(p - center) - half_size;
    return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}
```

盒子比球体复杂一些，但用法完全一样——返回"点到盒子表面的有向距离"。

不用死记硬背这个公式。把它们当工具函数用就行：传入位置，得到距离。

### 验证一下

把球体 SDF 换成盒子 SDF 输出颜色：

```glsl
float d = sdf_box(world_pos, vec3(0.0), vec3(1.5));
float gray = d * 0.3 + 0.5;
gray = clamp(gray, 0.0, 1.0);
ALBEDO = vec3(gray);
```

你会看到一个圆角矩形的渐变区域，不是横平竖直的直线——SDF 天生带"圆润感"。

更多的形状（圆柱、圆环、锥体）我们留给第11篇。今天只讲球体和盒子，够做切割了。
```

- [ ] **Step 2: 验证——确认 SDF 公式正确、代码可运行**

检查要点：
1. `sdf_sphere` 公式正确（`length(p - center) - radius`）
2. `sdf_box` 公式正确（标准盒子 SDF）
3. 可视化代码中的 `clamp` 范围合理

---

### Task 3: 撰写球形切割实战段落

**Files:**
- Modify: `articles/10-sdf-intro.md`（追加）

- [ ] **Step 1: 追加球形切割完整 shader**

写入以下内容：

```markdown
## 实战：球形切割

有了 SDF，把第8-9篇的平面切割改成球形切割，只需要换一行代码。

原来平面切割的判断是：
```glsl
float clip = dot(world_pos, normalize(plane_normal)) - plane_distance;
```

换成球体 SDF：
```glsl
float clip = length(world_pos - sphere_center) - sphere_radius;
```

完整 shader，直接复用第9篇的截面着色方案：

```glsl
shader_type spatial;
render_mode cull_disabled;

uniform vec3 sphere_center = vec3(0.0, 0.0, 0.0);
uniform float sphere_radius = 2.0;
uniform vec3 section_color : source_color = vec3(0.9, 0.3, 0.0);
uniform float edge_width : hint_range(0.0, 0.1) = 0.02;
uniform vec3 edge_color : source_color = vec3(1.0, 0.6, 0.1);

varying vec3 world_pos;

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    // 用球体 SDF 替代平面方程
    float clip = length(world_pos - sphere_center) - sphere_radius;

    vec3 base = texture(TEXTURE, UV).rgb;
    vec3 normal = normalize(NORMAL);
    vec3 view = normalize(VIEW);

    // 背面判断（跟第9篇一样）
    bool is_back = dot(normal, view) > 0.0;

    if (clip < 0.0) {
        // 球体内
        if (is_back) {
            ALBEDO = section_color;
        } else {
            discard;
        }
    } else {
        // 球体外，正常显示 + 边缘发光
        ALBEDO = base;
        float edge = smoothstep(edge_width, 0.0, clip);
        EMISSION = edge_color * edge * 2.0;
    }
}
```

效果：模型被一个球形区域"挖空"，球内部暴露出的截面是橙色，球边缘有一圈亮橙色发光。

对比平面切割，核心差异只有一行：`clip` 的计算方式从平面方程换成了 SDF。其余逻辑（背面判断、截面着色、边缘发光）完全不变。
```

- [ ] **Step 2: 追加切线边缘软化段落**

写入以下内容：

```markdown
### 边缘软化

SDF 做切割还有一个天然优势：`clip` 值本身就是"到表面的平滑距离"，不像平面方程那样线性变化。

```glsl
float clip = length(world_pos - sphere_center) - sphere_radius;
float alpha = smoothstep(-0.1, 0.1, clip);
```

`clip` 从球心到球面再到球外，是连续过渡的。用 `smoothstep` 做边缘软化比平面切割更自然——因为 SDF 值的变化本身就是"圆润的"。

对比平面切割：`dot(world_pos, normal) - distance` 是一个线性函数，过渡区域是等宽的。而 SDF 的过渡区域跟随形状走——这就是为什么 SDF 切割的软边看起来更舒服。
```

**预计行数**: ~100行（含此段）

---

### Task 4: 撰写 SDF 优势 + 脚本控制 + 结尾

**Files:**
- Modify: `articles/10-sdf-intro.md`（追加）

- [ ] **Step 1: 追加"为什么 SDF 更好"段落**

写入以下内容：

```markdown
## 平面切 vs SDF 切

| 对比维度 | 平面方程 | SDF |
|---------|---------|-----|
| 切割形状 | 只能无限平面 | 球、盒、圆柱……任意形状 |
| 边缘软化 | 线性过渡，等宽 | 跟随形状，更自然 |
| 多形状组合 | 不支持 | 可以叠加多个 SDF（第11篇讲） |
| 公式复杂度 | 简单 | 稍复杂，但复制粘贴即可 |

平面方程不是没用。它简单、性能好，横切竖切完全够用。但当你的需求变成"在模型上挖洞"或"做个圆角剖面"时，SDF 就是正确答案。

## 用脚本控制

和第8-9篇一样的模式，用脚本驱动 SDF 参数：

```s3
@export var clip_material: Material
@export var scan_target: Item3D

func Update():
    if scan_target:
        var pos = scan_target.global_position
        clip_material.set_shader_parameter("sphere_center", pos)
```

或者做一个扫描动画——球从小变大，"解剖"模型：

```s3
func play_scan(duration: float = 3.0):
    var tween = create_tween()
    tween.tween_method(
        func(r): clip_material.set_shader_parameter("sphere_radius", r),
        0.0, 5.0, duration
    )
```

球从半径 0 开始，3 秒内膨胀到 5.0，切割区域越来越大——一个动态的"解剖展示"效果。
```

- [ ] **Step 2: 追加结尾段落**

写入以下内容：

```markdown
## 这篇你学到了

- SDF 是什么：空间中每一点到最近表面的有向距离（正=外，负=内，零=表面）
- 球体 SDF：`length(p - center) - radius`
- 盒子 SDF：拆解成 `q = abs(p - center) - half_size` 再套公式
- 用 SDF 替代平面方程做切割，一行改动实现球形挖洞
- SDF 切割的软边天然跟随形状，比平面切更自然

## 下篇预告

现在只能用一个形状切割。但 SDF 真正厉害的地方在于：**多个 SDF 可以像图层一样叠加、相交、相减**。

两个球体 SDF 的"并集"→ 两个重叠的球形切割。  
一个盒子 SDF "减去"一个球体 SDF → 方孔里挖一个圆槽。

这叫 **SDF 布尔运算**——下一篇我们讲 union（并）、intersection（交）、subtraction（差）。三行代码，无限组合。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲 SDF 布尔运算，组合出你想要的任何形状。
```

---

### Task 5: 全文复核 + 提交

**Files:**
- Review: `articles/10-sdf-intro.md`

- [ ] **Step 1: 篇幅检查**

运行以下命令检查行数：

```bash
wc -l articles/10-sdf-intro.md
```

预期: 150-200 行。如果不足，在"边缘软化"段落后补充参数调优说明。如果超出，检查代码块是否有空行冗余，可适当压缩。

- [ ] **Step 2: i3dAct 命名规范检查**

用 grep 搜索禁用语，确认文章内未出现：
- `Godot`
- `Unity`
- `Unreal`
- `_ready`
- `_process`
- `MeshInstance3D`
- `Node3D`
- `.gd`
- `.tscn`

同时确认使用了正确名称：
- `MeshRender`（出现：Inspector 引用）
- `Item3D`（出现：脚本示例）
- `.s3`（出现：脚本示例）
- `.s3shader`（出现：shader 文件）
- `varying`（出现：shader 代码）

```bash
grep -in 'godot\|unity\|unreal\|_ready\|_process\|meshinstance\|node3d\|\.gd\b\|\.tscn\b' articles/10-sdf-intro.md
```
预期: 无输出（无匹配）

- [ ] **Step 3: 禁用短语检查**

搜索 AI 模板短语：
```bash
grep -in '众所周知\|值得注意的是\|在今天的游戏开发\|革命性的\|游戏规则改变' articles/10-sdf-intro.md
```
预期: 无输出

- [ ] **Step 4: 代码可执行性检查**

通读全文所有代码块，确认：
1. `sdf_sphere` 和 `sdf_box` 函数签名一致
2. 球形切割 shader 中的 `varying` 变量与 vertex/fragment 中的使用一致
3. 脚本控制示例的 `.s3` 代码语法正确（`func`、`var`、`tween`）
4. 所有 `uniform` 声明格式正确

- [ ] **Step 5: 结尾格式检查**

确认文章以以下内容结束：
```
---
（空行）
如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲 SDF 布尔运算，组合出你想要的任何形状。
```
确认包含分隔线 `---`、求点赞语、下篇预告。

- [ ] **Step 6: 提交**

```bash
git add articles/10-sdf-intro.md
git commit -m "feat: add article 10 - SDF intro (spherical cutting)"
```

- [ ] **Step 7: 更新 PROGRESS.md**

将第10篇的状态从 `⏳ 待写` 改为 `✅ 完成`。

修改 `PROGRESS.md` 第25行：
```
| 第10篇 | SDF是什么：有向距离场入门 | [10-sdf-intro.md](articles/10-sdf-intro.md) | ✅ 完成 |
```

然后提交：
```bash
git add PROGRESS.md
git commit -m "docs: update progress - article 10 complete"
```

---

## 验收核对表

| 检查项 | 预期 | 检查方式 |
|--------|------|---------|
| 篇幅 | 150-200行 | `wc -l` |
| i3dAct 命名 | 无其他引擎名 | grep |
| 禁用短语 | 0处 | grep |
| 求点赞语 | 存在 | 目视 |
| 下篇预告 | 提到布尔运算 | 目视 |
| 代码可运行 | 无语法错误 | 目视 |
| PROGRESS.md | 已更新 | 目视 |
