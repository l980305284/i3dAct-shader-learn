# Article 13 Fullscreen Shader Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Write article 13, 《全屏 Shader 入门：把整张画面当成一张纹理来处理》, and update writing progress.

**Architecture:** This is a documentation-only change. Create one Markdown article with one complete main shader block, then explain that block in prose instead of scattering many small snippets. Update `PROGRESS.md` after the article passes the writing checks.

**Tech Stack:** Markdown, GLSL-style i3dAct `canvas_item` shader, PowerShell/git verification commands.

---

## File Structure

- Create: `articles/13-fullscreen-shader-intro.md`
  - Responsibility: final reader-facing article for chapter 13.
  - Shape: opening, one complete main shader code block, explanation, mode walkthrough, pitfalls, summary, next-article preview.
- Modify: `PROGRESS.md`
  - Responsibility: mark article 13 as complete and link the new file.
- Read for reference only:
  - `docs/superpowers/specs/2026-06-27-article-13-fullscreen-shader-design.md`
  - `articles/12-clipping-animation.md`
  - `articles/03-coordinate-spaces.md`
  - `articles/04-texture-sampling.md`

---

### Task 1: Create Article Shell And Opening

**Files:**
- Create: `articles/13-fullscreen-shader-intro.md`

- [ ] **Step 1: Create the article with title and opening**

Create `articles/13-fullscreen-shader-intro.md` with this initial content:

```markdown
# 全屏 Shader 入门：把整张画面当成一张纹理来处理

第12篇我们把剖切效果动起来了。扫描、旋转、脉冲、路径移动，都是在模型表面上做文章。

从这篇开始，玩法换一下。

不再问“这个模型表面应该是什么颜色”，而是问：

**整张屏幕已经画好了，我能不能把它再处理一遍？**

这就是全屏 Shader，也就是后处理效果的基本思路。颜色滤镜、屏幕扭曲、暗角、像素化、转场，本质上都绕不开这件事：

```text
读取屏幕原颜色 -> 修改颜色 -> 输出到当前像素
```

这一篇先不做复杂效果。我们先搭一个最小但完整的全屏 Shader：它能显示原图、灰度、反色、调亮度、调对比度。

代码先放完整的，后面再拆开讲。
```

- [ ] **Step 2: Verify the shell**

Run:

```powershell
[Console]::OutputEncoding=[System.Text.Encoding]::UTF8
Get-Content -LiteralPath 'articles\13-fullscreen-shader-intro.md' -Encoding UTF8 -Raw
```

Expected: The file exists and the opening renders as readable Chinese, with the title exactly `# 全屏 Shader 入门：把整张画面当成一张纹理来处理`.

---

### Task 2: Add The Single Main Shader Block

**Files:**
- Modify: `articles/13-fullscreen-shader-intro.md`

- [ ] **Step 1: Append the complete main shader section**

Append this content. Keep it as one code block; do not split each mode into separate snippets.

````markdown
## 完整代码

把下面这个 shader 挂到一个覆盖全屏的 2D 节点上。你可以先用 `effect_mode` 切模式，看画面怎么变。

```glsl
shader_type canvas_item;

uniform int effect_mode : hint_range(0, 4) = 0;
uniform float brightness : hint_range(-1.0, 1.0) = 0.0;
uniform float contrast : hint_range(0.0, 2.0) = 1.0;

void fragment() {
    // 1. 读取当前屏幕这一点原本的颜色
    vec4 screen_color = texture(SCREEN_TEXTURE, SCREEN_UV);
    vec3 color = screen_color.rgb;

    // 2. 准备几种处理结果
    float gray_value = dot(color, vec3(0.299, 0.587, 0.114));
    vec3 gray_color = vec3(gray_value);
    vec3 invert_color = vec3(1.0) - color;
    vec3 brightness_color = color + vec3(brightness);
    vec3 contrast_color = (color - vec3(0.5)) * contrast + vec3(0.5);

    // 3. 根据模式选择最终输出
    vec3 final_color = color;

    if (effect_mode == 1) {
        final_color = gray_color;
    } else if (effect_mode == 2) {
        final_color = invert_color;
    } else if (effect_mode == 3) {
        final_color = brightness_color;
    } else if (effect_mode == 4) {
        final_color = contrast_color;
    }

    // 4. 限制到合法颜色范围，再写回屏幕
    final_color = clamp(final_color, 0.0, 1.0);
    COLOR = vec4(final_color, screen_color.a);
}
```

如果 `effect_mode = 0`，画面应该和原来一模一样。这个模式很重要，它能帮你确认：屏幕采样本身是通的。
````

- [ ] **Step 2: Check the main shader constraints**

Manually verify:

1. There is exactly one `glsl` code block in the article so far.
2. The block includes `shader_type canvas_item`.
3. The block uses `texture(SCREEN_TEXTURE, SCREEN_UV)`.
4. The block outputs through `COLOR`.
5. Modes `0` through `4` are represented.

---

### Task 3: Explain The Core Mental Model

**Files:**
- Modify: `articles/13-fullscreen-shader-intro.md`

- [ ] **Step 1: Append the explanation section**

Append this prose. Do not add new shader snippets here; refer back to the main code by variable name.

```markdown
## 这段代码到底在处理什么

前面写模型 shader 的时候，我们经常这样采样：

`texture(TEXTURE, UV)`

意思是：用模型表面的 `UV`，去读当前材质的贴图。

全屏 Shader 换了一组对象：

`texture(SCREEN_TEXTURE, SCREEN_UV)`

意思是：用屏幕上的 `SCREEN_UV`，去读已经渲染好的屏幕画面。

这里有两个关键变量。

`SCREEN_TEXTURE` 是当前屏幕画面。你可以把它理解成引擎已经帮你截好的一张图，里面是摄像机渲染出来的结果。

`SCREEN_UV` 是当前像素在屏幕上的位置。左下角接近 `(0, 0)`，右上角接近 `(1, 1)`。它和模型 `UV` 很像，但对象完全不同：模型 `UV` 贴在模型表面，`SCREEN_UV` 贴在整个屏幕上。

所以全屏后处理最基础的结构就是：

1. 读原来的屏幕颜色
2. 对这个颜色做一点数学处理
3. 把处理后的颜色写到 `COLOR`

你看，事情没有变玄。只是 `texture()` 读的对象，从“模型贴图”换成了“屏幕画面”。
```

- [ ] **Step 2: Verify continuity with earlier articles**

Check that this section explicitly connects to:

- `texture(TEXTURE, UV)` from article 4.
- `SCREEN_UV` from article 3.
- The new full-screen mental model from the approved spec.

---

### Task 4: Explain The Five Modes And Common Pitfalls

**Files:**
- Modify: `articles/13-fullscreen-shader-intro.md`

- [ ] **Step 1: Append mode walkthrough**

Append this content. Keep the explanations prose-first and avoid adding extra code blocks.

```markdown
## 五种模式分别在做什么

`effect_mode = 0` 是原图。它什么都不改，直接把 `screen_color.rgb` 输出回去。写全屏 Shader 的第一步最好永远是原图模式：如果原图都显示不出来，后面的灰度、反色、扭曲都不用看了。

`effect_mode = 1` 是灰度。这里没有偷懒写 `(r + g + b) / 3.0`，而是用了 `0.299, 0.587, 0.114` 这组三个权重。人的眼睛对绿色更敏感，对蓝色没那么敏感，所以绿色权重大一些，蓝色权重小一些。这样转出来的灰度更接近真实观感。

`effect_mode = 2` 是反色。颜色范围是 0 到 1，所以 `1.0 - color` 就能把亮的变暗，暗的变亮。红色 `(1, 0, 0)` 会变成青色 `(0, 1, 1)`。

`effect_mode = 3` 是亮度。它直接给 RGB 加上一个值。`brightness` 是正数，画面变亮；是负数，画面变暗。这个模式很直观，但也最容易越界，所以最后必须 `clamp` 一下。

`effect_mode = 4` 是对比度。它先把颜色减去 `0.5`，让颜色围绕中灰点计算；乘上 `contrast` 后，再加回 `0.5`。`contrast` 大于 1，亮的更亮、暗的更暗；小于 1，画面会变灰、变平。

## 常见坑

第一个坑：把 `UV` 和 `SCREEN_UV` 混着用。

模型 shader 里常用 `UV`，因为你在处理模型表面。全屏 Shader 处理的是屏幕，所以要用 `SCREEN_UV`。如果你拿模型 `UV` 去采样屏幕，结果通常会很奇怪。

第二个坑：一上来就写复杂效果。

全屏 Shader 很容易让人想马上做模糊、扭曲、故障艺术。但第一步应该永远是原图采样。先让 `effect_mode = 0` 正常显示，再慢慢加效果。这样出问题时，你知道问题在新效果里，而不是基础采样没搭好。

第三个坑：灰度直接取红色通道。

`vec3(color.r)` 确实能让画面变成黑白，但它只看红色信息。一个很亮的蓝色物体，红色通道可能很低，结果会被压得很暗。用加权灰度更稳。

第四个坑：忘记限制颜色范围。

亮度和对比度都可能把颜色算到 0 以下或 1 以上。最后统一 `clamp(final_color, 0.0, 1.0)`，是个好习惯。
```

- [ ] **Step 2: Confirm no extra code blocks were introduced**

Run:

```powershell
[Console]::OutputEncoding=[System.Text.Encoding]::UTF8
(Select-String -Path 'articles\13-fullscreen-shader-intro.md' -Pattern '```glsl').Count
```

Expected: `1`

---

### Task 5: Add Summary, Preview, And Progress Update

**Files:**
- Modify: `articles/13-fullscreen-shader-intro.md`
- Modify: `PROGRESS.md`

- [ ] **Step 1: Append summary and ending**

Append this ending:

```markdown
## 这篇你学到了

- 全屏 Shader 的核心流程：读屏幕、改颜色、输出
- `SCREEN_TEXTURE` 是当前屏幕画面
- `SCREEN_UV` 是屏幕上的 0-1 坐标
- `texture(SCREEN_TEXTURE, SCREEN_UV)` 就是在读取当前像素原本的颜色
- 原图模式很重要，它能帮你确认全屏采样框架是否正常
- 灰度、反色、亮度、对比度，本质上都是对屏幕颜色做数学处理

## 下篇预告

这一篇我们只是搭好了框架，做了几个最基础的颜色处理。

下一篇开始，我们正式讲 **颜色滤镜**。不只是灰度和反色，而是更可控地调整画面气氛：偏冷、偏暖、压暗阴影、提亮高光，甚至做出回忆、受伤、夜视这样的整体画面风格。

全屏 Shader 真正好玩的地方，从这里才开始。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲颜色滤镜，让整张画面的气氛跟着变。
```

- [ ] **Step 2: Update `PROGRESS.md`**

Change the article 13 row from:

```markdown
| 第13篇 | 全屏Shader入门 | ⏳ 待写 |
```

to:

```markdown
| 第13篇 | 全屏Shader入门 | [13-fullscreen-shader-intro.md](articles/13-fullscreen-shader-intro.md) | ✅ 完成 |
```

If the table has only three columns in the series-three section, update its header and remaining pending rows to keep the table consistent:

```markdown
| 篇数 | 标题方向 | 文件 | 状态 |
|------|---------|------|------|
| 第13篇 | 全屏Shader入门 | [13-fullscreen-shader-intro.md](articles/13-fullscreen-shader-intro.md) | ✅ 完成 |
| 第14篇 | 颜色滤镜 | - | ⏳ 待写 |
| 第15篇 | 屏幕扭曲 | - | ⏳ 待写 |
| 第16篇 | 暗角+色差 | - | ⏳ 待写 |
| 第17篇 | 像素化/复古风 | - | ⏳ 待写 |
| 第18篇 | 屏幕转场 | - | ⏳ 待写 |
```

---

### Task 6: Verify And Commit

**Files:**
- Review: `articles/13-fullscreen-shader-intro.md`
- Review: `PROGRESS.md`

- [ ] **Step 1: Check line count**

Run:

```powershell
(Get-Content -LiteralPath 'articles\13-fullscreen-shader-intro.md' -Encoding UTF8).Count
```

Expected: Between `150` and `220` lines. The approved spec targets 150-200 lines, but this article may slightly exceed 200 because the main shader is intentionally kept as one large block.

- [ ] **Step 2: Check main-code-block rule**

Run:

```powershell
(Select-String -Path 'articles\13-fullscreen-shader-intro.md' -Pattern '```glsl').Count
```

Expected: `1`

- [ ] **Step 3: Check forbidden engine names and old naming**

Run:

```powershell
[Console]::OutputEncoding=[System.Text.Encoding]::UTF8
rg -n "Godot|Unity|Unreal|_ready|_process|MeshInstance3D|Node3D|\\.gd\\b|\\.tscn\\b" articles\13-fullscreen-shader-intro.md
```

Expected: no output.

- [ ] **Step 4: Check forbidden writing phrases**

Run:

```powershell
[Console]::OutputEncoding=[System.Text.Encoding]::UTF8
rg -n "众所周知|值得注意的是|在今天的游戏开发|革命性的|游戏规则改变" articles\13-fullscreen-shader-intro.md
```

Expected: no output.

- [ ] **Step 5: Check required concepts**

Run:

```powershell
[Console]::OutputEncoding=[System.Text.Encoding]::UTF8
rg -n "SCREEN_TEXTURE|SCREEN_UV|texture\\(SCREEN_TEXTURE, SCREEN_UV\\)|COLOR|effect_mode|颜色滤镜" articles\13-fullscreen-shader-intro.md
```

Expected: matches for every required concept.

- [ ] **Step 6: Review git diff**

Run:

```powershell
git diff -- articles/13-fullscreen-shader-intro.md PROGRESS.md
```

Expected:

- New article contains one main shader block and the approved structure.
- `PROGRESS.md` only changes the series-three table for article 13.
- No unrelated files are changed.

- [ ] **Step 7: Commit article and progress**

Run:

```powershell
git add -- articles/13-fullscreen-shader-intro.md PROGRESS.md
git commit -m "feat: add article 13 - fullscreen shader intro"
```

Expected: commit succeeds.

---

## Self-Review Notes

- Spec coverage: article creation, one complete main shader, explanation of `SCREEN_TEXTURE`/`SCREEN_UV`, five modes, pitfalls, summary, next preview, progress update, and verification are all covered.
- Placeholder scan: no unfinished placeholder markers or vague “write tests” steps are present.
- Scope check: this plan is focused on one documentation article and one progress-table update; it does not include article 14 or any unrelated refactor.
