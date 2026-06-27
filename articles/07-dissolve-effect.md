# 把前面学的串起来：做一个帅气的溶解特效

前六篇我们分别学了 Shader 基础、UV、坐标系、纹理采样、数学函数、Noise 纹理。每个知识点单独看好像都会了，但真要做效果的时候，怎么把它们串起来？

这篇就是来解决这个问题的。我们从零开始，一步步搭出一个**可以在项目里直接用**的溶解特效。

先看最终效果：一个模型从下往上逐渐消散，消散边缘有火焰般的发光，颜色从橙色渐变到白色，整个过程可以用脚本控制播放。

没有成品图的话，先脑补一下《蜘蛛侠：平行宇宙》里角色被传送走的那种像素消散效果——我们要做的就是类似的东西。

## 溶解特效拆解

把一个溶解特效拆开来看，它其实是这几个层的叠加：

```
溶解特效 = 基础贴图 + 溶解遮罩 + 边缘发光 + 定向控制 + 噪声动画
```

对应到我们已经学过的知识：

| 层 | 用到的技术 | 在哪学的 |
|---|---|---|
| 基础贴图 | `texture(TEXTURE, UV)` | 第4篇 |
| 溶解遮罩 | `smoothstep` + noise 采样 | 第5篇 + 第6篇 |
| 边缘发光 | `smoothstep` 相减提取边界 | 第5篇 |
| 定向控制 | UV 坐标运算 | 第2篇 |
| 噪声动画 | `TIME` + UV 偏移 | 第2篇 + 第6篇 |

先把每个部分单独实现，最后拼成完整效果。

## Step 1：基础溶解框架

溶解的核心逻辑一句话：**每个像素取一个噪声值，噪声值小于阈值的像素被"溶解掉"**。

```glsl
shader_type spatial;

uniform sampler2D dissolve_noise;
uniform float progress : hint_range(0.0, 1.0) = 0.0;

void fragment() {
    vec3 base_color = texture(TEXTURE, UV).rgb;
    float noise = texture(dissolve_noise, UV).r;

    // progress 越大，被溶解的像素越多
    float dissolve = smoothstep(progress - 0.05, progress + 0.05, noise);

    ALBEDO = base_color * dissolve;
}
```

给 `dissolve_noise` 拖入一张 `NoiseTexture2D`（不会创建的去回顾第6篇开头），然后拖动 `progress` 滑条：

- `progress = 0.0`：整个模型完整显示
- `progress = 0.5`：约一半像素消失
- `progress = 1.0`：全部溶解

`smoothstep` 让消失的边缘有一点过渡，不会一刀切。但这个版本太简陋了——边缘就是灰扑扑的，没有任何视觉冲击力。

## Step 2：边缘发光

溶解特效的灵魂在于**边缘发光**。角色消散时，身体边缘应该有一圈亮色，暗示能量正在作用。

边缘怎么提取？用两个 `smoothstep` 相减：

```glsl
// 宽的过渡带
float outer = smoothstep(progress - 0.05, progress + 0.05, noise);
// 窄的过渡带
float inner = smoothstep(progress - 0.02, progress + 0.02, noise);
// 相减得到边缘窄条
float edge = outer - inner;
```

画个示意图（脑补版）：

```
noise 值 →

 外层 smoothstep：  ___________/‾‾‾‾‾‾‾‾‾‾‾
 内层 smoothstep：  _______________/‾‾‾‾‾‾‾
 相减得到边缘带：   ___________/‾‾‾\________
                              ↑
                          发光区域
```

把这个边缘带叠加上去：

```glsl
shader_type spatial;

uniform sampler2D dissolve_noise;
uniform float progress : hint_range(0.0, 1.0) = 0.0;
uniform float edge_width : hint_range(0.0, 0.2) = 0.05;
uniform vec3 edge_color : source_color = vec3(1.0, 0.4, 0.0);

void fragment() {
    vec3 base_color = texture(TEXTURE, UV).rgb;
    float noise = texture(dissolve_noise, UV).r;

    // 主体遮罩
    float dissolve = smoothstep(progress - edge_width, progress + edge_width, noise);

    // 边缘提取
    float edge = smoothstep(progress - edge_width * 0.3, progress + edge_width * 0.3, noise)
               - smoothstep(progress - edge_width, progress + edge_width, noise);

    // 组合
    vec3 color = base_color * dissolve;
    color += edge_color * edge * 2.0;  // 2.0 是亮度系数

    ALBEDO = color;
}
```

现在溶解边缘有了一圈橙色发光。但有两个问题：

1. 发光亮度均匀，没有"最亮处发白"的层次感
2. 溶解方向是纯随机的，缺乏方向性——角色是整片随机消失，而不是从脚到头逐渐消散

## Step 3：定向溶解

真实的溶解特效通常有方向性：角色从脚底开始消散，逐渐到头顶。这需要我们在噪声值上叠加一个 UV 方向的偏移：

```glsl
// UV.y 接近 1（模型顶部）→ bias 为正 → 更难被溶解
// UV.y 接近 0（模型底部）→ bias 为负 → 更容易被溶解
float directional_bias = (UV.y - 0.5) * 1.0;
float threshold = noise + directional_bias;
```

为什么用 `UV.y - 0.5`？因为 UV.y 范围是 0-1，减 0.5 后范围变成 -0.5 到 +0.5。这样模型顶部（UV.y > 0.5）的像素拿到正偏置，相当于"更难达到溶解阈值"；底部（UV.y < 0.5）拿负偏置，更容易被溶解。

把这个融入前面的代码：

```glsl
shader_type spatial;

uniform sampler2D dissolve_noise;
uniform float progress : hint_range(0.0, 1.0) = 0.0;
uniform float edge_width : hint_range(0.0, 0.2) = 0.05;
uniform vec3 edge_color : source_color = vec3(1.0, 0.4, 0.0);
uniform float direction_strength : hint_range(0.0, 2.0) = 1.0;

void fragment() {
    vec3 base_color = texture(TEXTURE, UV).rgb;
    float noise = texture(dissolve_noise, UV).r;

    // 定向偏置：从底部开始溶解
    float bias = (UV.y - 0.5) * direction_strength;
    float threshold = noise + bias;

    float dissolve = smoothstep(progress - edge_width, progress + edge_width, threshold);

    float edge = smoothstep(progress - edge_width * 0.3, progress + edge_width * 0.3, threshold)
               - smoothstep(progress - edge_width, progress + edge_width, threshold);

    vec3 color = base_color * dissolve + edge_color * edge * 2.0;

    ALBEDO = color;
}
```

现在调 `direction_strength` 可以控制方向性的强弱。设为 0 时退化为纯随机溶解，设为 1.5 时方向感很强。

如果你想从其他方向溶解，把 `UV.y` 换成 `UV.x`（从左到右）、`1.0 - UV.y`（从顶到底）、或者任意组合。

## Step 4：给边缘加层次

现在边缘发光只有一个颜色，看起来有点扁平。好的溶解特效在边缘上有**颜色梯度**：

- 最外层（刚被溶解的）：深色，接近烟雾感
- 中间层：主题色，火焰橙/魔法紫
- 最内层（即将消失的）：亮白色，高能量感

用两层边缘叠加：

```glsl
// 外层：宽一点的暗色边缘
float outer_edge = smoothstep(progress - edge_width * 1.5, progress + edge_width * 1.5, threshold)
                 - smoothstep(progress - edge_width * 0.8, progress + edge_width * 0.8, threshold);

// 内层：窄一点的亮色边缘
float inner_edge = smoothstep(progress - edge_width * 0.5, progress + edge_width * 0.5, threshold)
                 - smoothstep(progress - edge_width, progress + edge_width, threshold);

// 芯：最窄的白热边缘
float core_edge = smoothstep(progress - edge_width * 0.2, progress + edge_width * 0.2, threshold)
                - smoothstep(progress - edge_width * 0.5, progress + edge_width * 0.5, threshold);

// 组合发光
vec3 edge_glow = vec3(0.0);
edge_glow += edge_color * 0.5 * outer_edge;       // 外层暗
edge_glow += edge_color * 1.5 * inner_edge;       // 中层亮
edge_glow += vec3(1.0, 0.9, 0.7) * 3.0 * core_edge; // 芯白热
```

外层、中层、内层三个环带，颜色从暗到亮，就有了层次。你可以根据自己的效果主题改颜色：冰冻溶解用蓝色+白色，毒雾溶解用绿色+暗紫色，等等。

## Step 5：让噪声"活着"

现在把 `progress` 从 0 拉到 1，效果就出来了。但有一个问题：在 `progress` 不变的时候（比如停在 0.5），溶解边缘是**完全静止**的。

真实的效果即使在静止时，边缘也应该有微弱的"躁动"——就像火焰边缘一直在抖动一样。

解决方式：给噪声的 UV 加上 TIME 偏移，让它始终在流动：

```glsl
// 噪声随时间缓慢流动
vec2 noise_uv = UV + vec2(TIME * 0.02, TIME * 0.03);
float noise = texture(dissolve_noise, noise_uv).r;
```

两个方向的偏移速度不同（0.02 vs 0.03），让流动方向是斜的，看起来更自然。

再把这种流动做得更有机一点——多加一层不同频率的噪声：

```glsl
// 主噪声：低频，控制溶解主体
float noise1 = texture(dissolve_noise, UV + vec2(TIME * 0.02, TIME * 0.02)).r;

// 细节噪声：高频，给边缘增加细碎感
float noise2 = texture(dissolve_noise, UV * 2.5 + vec2(-TIME * 0.05, TIME * 0.03)).r;

// 混合两层噪声
float noise = noise1 * 0.7 + noise2 * 0.3;
```

主噪声决定溶解的大轮廓，细节噪声给边缘添加细碎的侵蚀感。两层叠加的效果比单层丰富得多。

## 完整代码

把所有步骤组合在一起，这就是最终的溶解 shader：

```glsl
shader_type spatial;

// 纹理
uniform sampler2D dissolve_noise;

// 溶解控制
uniform float progress : hint_range(0.0, 1.0) = 0.0;
uniform float edge_width : hint_range(0.01, 0.3) = 0.06;
uniform vec3 edge_color : source_color = vec3(1.0, 0.35, 0.05);

// 方向控制
uniform float direction_y : hint_range(-1.0, 1.0) = 1.0;
uniform float direction_strength : hint_range(0.0, 2.0) = 1.0;

// 发光强度
uniform float glow_intensity : hint_range(0.0, 5.0) = 2.0;

// 噪声动画速度
uniform float noise_speed : hint_range(0.0, 0.2) = 0.03;

void fragment() {
    vec3 base_color = texture(TEXTURE, UV).rgb;

    // 双层噪声，边缘更细碎
    vec2 flow = vec2(TIME * noise_speed, TIME * noise_speed * 1.3);
    float noise1 = texture(dissolve_noise, UV + flow).r;
    float noise2 = texture(dissolve_noise, UV * 2.5 - flow * 1.5).r;
    float noise = noise1 * 0.7 + noise2 * 0.3;

    // 定向偏置
    float bias = (UV.y - 0.5) * direction_strength * direction_y;
    float threshold = clamp(noise + bias, 0.0, 1.0);

    // 外层暗色边缘
    float outer = smoothstep(progress - edge_width * 1.5, progress + edge_width * 1.5, threshold)
                - smoothstep(progress - edge_width * 0.8, progress + edge_width * 0.8, threshold);

    // 内层主题色边缘
    float inner = smoothstep(progress - edge_width * 0.5, progress + edge_width * 0.5, threshold)
                - smoothstep(progress - edge_width, progress + edge_width, threshold);

    // 白色核心边缘
    float core = smoothstep(progress - edge_width * 0.2, progress + edge_width * 0.2, threshold)
               - smoothstep(progress - edge_width * 0.5, progress + edge_width * 0.5, threshold);

    // 溶解主体遮罩
    float dissolve = smoothstep(progress - edge_width, progress + edge_width, threshold);

    // 三层发光叠加
    vec3 glow = vec3(0.0);
    glow += edge_color * 0.4 * outer;
    glow += edge_color * 1.2 * inner;
    glow += vec3(1.0, 0.9, 0.7) * 2.5 * core;
    glow *= glow_intensity;

    // 最终颜色
    vec3 color = base_color * dissolve + glow;

    ALBEDO = color;
    EMISSION = glow;
    ALPHA = dissolve;
}
```

使用前记得把材质的 **Transparent** 打开（在 Inspector 里勾选），否则 `ALPHA` 不会生效，溶解掉的区域会显示黑色而不是透明。

参数说明：

| 参数 | 作用 | 推荐范围 |
|------|------|---------|
| `progress` | 溶解进度，脚本驱动 | 0.0 - 1.0 |
| `edge_width` | 边缘发光的宽度 | 0.03 - 0.1 |
| `edge_color` | 边缘发光主题色 | 橙/蓝/紫 |
| `direction_y` | 溶解方向，正数=从下往上 | -1.0 - 1.0 |
| `direction_strength` | 方向性强弱，0=无方向 | 0.5 - 1.5 |
| `glow_intensity` | 发光整体亮度 | 1.0 - 3.0 |
| `noise_speed` | 噪声流动速度，0=静止 | 0.01 - 0.05 |

## 脚本控制

shader 写好了，但 `progress` 需要脚本来驱动。以下是控制脚本：

```s3
@export var dissolve_material: Material
var dissolve_tween: Tween

func play_dissolve(duration: float = 2.0):
    dissolve_tween = create_tween()
    dissolve_tween.tween_method(set_progress, 0.0, 1.0, duration)

func play_appear(duration: float = 2.0):
    dissolve_tween = create_tween()
    dissolve_tween.tween_method(set_progress, 1.0, 0.0, duration)

func set_progress(value: float):
    dissolve_material.set_shader_parameter("progress", value)

func reset_dissolve():
    if dissolve_tween:
        dissolve_tween.kill()
    dissolve_material.set_shader_parameter("progress", 0.0)
```

用法：在需要触发溶解的地方调用 `play_dissolve()`。比如角色死亡时：

```s3
func on_death():
    play_dissolve(1.5)
    await get_tree().create_timer(1.5).timeout
    queue_free()
```

`play_appear()` 是反向的——从完全溶解状态恢复到实体，适合角色复活或者道具生成。

## 调参指南

不同的游戏风格需要不同的溶解参数。这里给几个参考配置：

### 火焰溶解（适合战斗击杀）

```
edge_color = (1.0, 0.3, 0.05)    # 深橙红
edge_width = 0.06
direction_strength = 1.0
glow_intensity = 2.5
noise_speed = 0.04
```

### 冰冻碎裂（适合冰系技能）

```
edge_color = (0.3, 0.7, 1.0)     # 冰蓝
edge_width = 0.04
direction_strength = 0.3          # 少方向性，更像碎裂
glow_intensity = 2.0
noise_speed = 0.01                # 慢速流动
```

### 暗影传送（适合瞬移/闪现）

```
edge_color = (0.6, 0.2, 0.9)     # 暗紫色
edge_width = 0.08
direction_strength = 0.0          # 无方向性，从四周向中心
glow_intensity = 3.0
noise_speed = 0.06
```

### 金色升天（适合道具拾取/升级）

```
edge_color = (1.0, 0.8, 0.1)     # 金色
edge_width = 0.05
direction_strength = 1.5          # 从下往上
glow_intensity = 3.0
noise_speed = 0.03
```

调参的优先级建议：先调 `edge_width` 确定边缘粗细 → 再调 `edge_color` 确定色调 → 然后调 `direction_strength` 定方向 → 最后调 `glow_intensity` 和 `noise_speed` 微调氛围。

## 常见问题

### 溶解效果看不到边缘发光

大概率是 `glow_intensity` 太低。先把它调到 5.0 看看有没有效果，确认边缘提取逻辑是对的，再往下调到合适的值。

### 边缘发光颜色不对

检查 `ALBEDO` 和 `EMISSION` 的写法。`EMISSION` 不受场景光照影响，是你看到的"纯自发光"。如果场景里有 DirectionalLight 且很亮，`ALBEDO` 上的发光会被冲淡。把发光主要写到 `EMISSION` 里。

### 模型边缘溶解后是黑色而不是透明

两个原因：
1. 材质的 Transparent 模式没开——在 Inspector 里找到材质的 `Transparent` 属性，设为 `true`
2. shader 里没写 `ALPHA = dissolve`

两个都检查一下。

### 溶解从奇怪的方向开始

`direction_y` 的符号决定了方向：
- 正数（1.0）：底部先消失，顶部后消失
- 负数（-1.0）：顶部先消失，底部后消失
- 调小 `direction_strength` 减弱方向性

如果你的模型 UV 展开不标准（比如 UV.y=0 在头顶而不是脚底），方向会不符合预期。这时候需要根据实际情况调整偏置的计算方式。

## 延伸：不只是溶解

这个 shader 框架改几个参数就能变成完全不同的效果：

**燃烧效果**：`direction_y = 1.0`，`edge_color` 设橙色，`progress` 从底部往上走。可以再加一张灰烬纹理，在溶解区域内显示烧焦的颜色：

```glsl
// 在被溶解区域显示烧焦色
vec3 burnt = mix(base_color, vec3(0.1, 0.05, 0.0), 1.0 - dissolve);
color = mix(burnt, base_color, dissolve) + glow;
```

**传送门出现**：`direction_y = 0.0`（无方向性），`edge_color` 设亮蓝色，使用 `play_appear()` 让角色从中间向外围显现。

**冰冻霜化**：把 `edge_color` 设冰蓝色，同时在溶解区域叠加一层白色霜冻纹理，用 `mix` 混合。

思路都是一样的：噪声生成遮罩 → smoothstep 柔化 → 提取边缘叠加发光 → 换颜色和方向就是新效果。

## 这篇你学到了什么

1. **溶解特效的完整实现**：双层噪声 + 定向偏置 + 三层边缘发光
2. **如何把所学知识串联**：UV（定向控制）+ texture（噪声采样）+ smoothstep/mix/clamp（遮罩和发光）+ TIME（动态流动）
3. **参数调优思路**：不同风格需要不同的颜色、宽度、方向配置
4. **脚本控制**：用 Tween 驱动 shader 参数，做平滑的溶解动画
5. **如何触类旁通**：一套框架，改颜色和参数就能变成燃烧、冰冻、传送等多种效果

## 下篇预告

到目前为止，我们所有的效果都还是"表面功夫"——在模型表面上玩纹理和颜色。下一篇我们进入一个新领域：**SDF（Signed Distance Field，有向距离场）**。

用 SDF 你可以做到：用一个平面把模型切成两半、显示模型的截面内部、甚至做出 CT 扫描一样的动态剖切效果。只需要一行 `discard` 加几行数学公式。

下一篇见。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲 SDF 模型剖切，内容更有意思。
