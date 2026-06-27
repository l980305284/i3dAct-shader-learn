# smoothstep、mix、clamp：Shader三剑客

前四篇我们学了UV、坐标系、纹理采样。现在你已经能做出渐变、流动、扭曲效果了。

但有个问题：这些效果的边缘都很"硬"。要么完全显示，要么完全消失，中间没有过渡。

现实中很多效果需要**平滑过渡**：角色溶解时的边缘发光、技能范围的柔软边界、两张贴图的渐变混合...

这就需要三个核心函数：**smoothstep**、**mix**、**clamp**。它们几乎是每个shader都会用的基础工具。

## smoothstep：让硬边缘变软

假设你要做一个圆形区域，区域内显示红色，区域外显示黑色。最直觉的写法是：

```glsl
shader_type spatial;

void fragment() {
    float dist = distance(UV, vec2(0.5));
    if (dist < 0.3) {
        ALBEDO = vec3(1.0, 0.0, 0.0);  // 红
    } else {
        ALBEDO = vec3(0.0);  // 黑
    }
}
```

效果：一个边缘锐利的红圆，像用剪刀剪出来的一样。

但现实中的圆形边缘很少有绝对锐利的。灯光照亮范围、火焰燃烧区域、雾气浓度——这些都有"渐变边界"。

### smoothstep 的用法

```glsl
float result = smoothstep(边缘起点, 边缘终点, 值);
```

它返回一个 0 到 1 之间的值：

- 当"值" ≤ 边缘起点，返回 0
- 当"值" ≥ 边缘终点，返回 1
- 中间用**平滑曲线**过渡（不是直线）

把上面的圆形改成平滑边缘：

```glsl
shader_type spatial;

void fragment() {
    float dist = distance(UV, vec2(0.5));
    float circle = smoothstep(0.3, 0.2, dist);
    ALBEDO = vec3(circle, 0.0, 0.0);
}
```

注意参数顺序：`smoothstep(0.3, 0.2, dist)`。

- dist = 0.3（圆边缘外侧）→ 返回 0 → 黑色
- dist = 0.2（更靠近中心）→ 返回 1 → 红色
- dist 在 0.2 到 0.3 之间 → 平滑过渡

这里我故意把参数写成 `(大, 小, 值)`，因为 dist 越**小**越靠近中心，我想要"靠近中心时返回1"。如果写成 `(0.2, 0.3, dist)`，效果会反过来——中心黑、边缘红。

### smoothstep vs 线性过渡

为什么不用线性插值？看看对比：

```glsl
// 线性过渡：边缘很生硬，能看到明显的"条带"
float linear = clamp((0.3 - dist) / (0.3 - 0.2), 0.0, 1.0);

// smoothstep：边缘自然柔和
float smooth = smoothstep(0.3, 0.2, dist);
```

smoothstep 在边界处变化慢，中间变化快。这种"S型曲线"更接近人眼对自然过渡的感知，不容易出现色带（banding）。

### 实战：扫光效果

让一道光带从模型上扫过，边缘柔软：

```glsl
shader_type spatial;

uniform float sweep_position : hint_range(-1.0, 2.0) = 0.5;
uniform float sweep_width : hint_range(0.01, 0.5) = 0.1;

void fragment() {
    vec3 base_color = texture(TEXTURE, UV).rgb;

    // 计算当前像素到扫光中心的距离
    float dist = abs(UV.x - sweep_position);

    // 扫光强度：中心最亮，向两侧衰减
    float glow = smoothstep(sweep_width, 0.0, dist);

    // 叠加发光色
    vec3 glow_color = vec3(0.5, 0.8, 1.0) * glow * 2.0;

    ALBEDO = base_color + glow_color;
    EMISSION = glow_color;
}
```

脚本里让扫光移动：

```s3
@export var material: Material

func Update():
    var pos = sin(TIME * 2.0) * 0.5 + 0.5
    material.set_shader_parameter("sweep_position", pos)
```

这个效果在装备展示、科技风UI里很常见。没有 smoothstep，光带边缘会很刺眼。

## mix：不只是混合颜色

第4篇我们用过 `mix` 混合两张纹理：

```glsl
vec3 result = mix(color_a, color_b, 0.5);
```

它的完整定义是：`mix(a, b, t) = a * (1.0 - t) + b * t`

- t=0：完全用 a
- t=1：完全用 b
- t=0.5：a 和 b 各一半

### 混合任何值

mix 不只可以混合颜色，任何 vec2、vec3、float 都能混：

```glsl
// 混合两个UV坐标
vec2 mixed_uv = mix(uv_a, uv_b, progress);

// 混合两个坐标位置
vec3 mixed_pos = mix(pos_a, pos_b, blend);

// 混合两个透明度
float mixed_alpha = mix(alpha_a, alpha_b, mask);
```

### 实战：基于高度的颜色混合

做地形时，低处是泥土色，高处是雪白色，中间自然过渡：

```glsl
shader_type spatial;

uniform sampler2D dirt_texture;
uniform sampler2D snow_texture;
uniform float snow_line : hint_range(0.0, 1.0) = 0.6;
uniform float blend_width : hint_range(0.0, 0.2) = 0.05;

varying float height;

void vertex() {
    // 在顶点着色器里计算世界空间高度
    vec3 world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
    height = world_pos.y;
}

void fragment() {
    // 把高度归一化到0-1（假设地形高度在0到10之间）
    float h = clamp(height / 10.0, 0.0, 1.0);

    // 计算混合系数：雪线以下全泥土，雪线以上全雪，中间过渡
    float t = smoothstep(snow_line - blend_width, snow_line + blend_width, h);

    vec3 dirt = texture(dirt_texture, UV).rgb;
    vec3 snow = texture(snow_texture, UV).rgb;

    ALBEDO = mix(dirt, snow, t);
}
```

这里 `smoothstep` 和 `mix` 配合：smoothstep 生成 0 到 1 的柔和过渡值，mix 用这值混合两种颜色。

没有 smoothstep 的话，雪线会是一条明显的锯齿线。

### 用 mix 做 UV 动画过渡

两个 UV 动画效果，用 mix 在它们之间切换：

```glsl
shader_type spatial;

uniform sampler2D tex;
uniform float effect_blend : hint_range(0.0, 1.0) = 0.0;

void fragment() {
    // 效果A：水平滚动
    vec2 uv_a = UV + vec2(TIME * 0.1, 0.0);

    // 效果B：漩涡扭曲
    vec2 center = UV - vec2(0.5);
    float angle = length(center) * 4.0 - TIME;
    vec2 uv_b = vec2(
        cos(angle) * center.x - sin(angle) * center.y,
        sin(angle) * center.x + cos(angle) * center.y
    ) + vec2(0.5);

    // 混合两个UV，然后采样
    vec2 final_uv = mix(uv_a, uv_b, effect_blend);
    ALBEDO = texture(tex, final_uv).rgb;
}
```

这个技巧在转场特效里很有用：两个完全不同的动态效果，用 mix 平滑切换。

## clamp：防止值越界

shader 里的值经常"失控"：

```glsl
float bright = base + extra_light;  // 可能超过1.0
vec2 big_uv = UV * 100.0;           // 可能超出0-1范围
```

颜色超过 1.0 可能导致 HDR 溢出问题。UV 越界可能采样到不该采样的区域。

`clamp(x, min, max)` 把 x 限制在 [min, max] 范围内：

```glsl
// 限制颜色强度
vec3 safe_color = clamp(base_color + glow, 0.0, 1.0);

// 限制UV不越界
vec2 safe_uv = clamp(UV, 0.0, 1.0);

// 限制高度值
float valid_height = clamp(height, 0.0, 10.0);
```

### clamp 和 saturate 的区别

有些引擎有 `saturate(x)`，等价于 `clamp(x, 0.0, 1.0)`。GLSL 标准里没有 saturate，用 clamp 就行。

### 实战：限制发光强度

```glsl
shader_type spatial;

uniform float glow_intensity : hint_range(0.0, 5.0) = 1.0;

void fragment() {
    vec3 base = texture(TEXTURE, UV).rgb;

    // 不加限制的话，glow_intensity 调太高会全白失真
    vec3 glow = base * glow_intensity;
    glow = clamp(glow, 0.0, 2.0);  // 最多亮到2倍

    ALBEDO = base;
    EMISSION = glow;
}
```

这里 clamp 保证即使调参过头，也不会把整个画面炸成白色。

## 组合拳：简易溶解效果

现在把三个函数组合起来，做一个溶解（Dissolve）效果。这是第7篇的预热，先用简单版本来理解原理。

溶解的核心逻辑：

1. 用一张噪声纹理生成随机值
2. 用 smoothstep 控制"哪些像素该消失"
3. 用 mix 控制溶解进度
4. 用 clamp 限制边缘发光强度

```glsl
shader_type spatial;

uniform sampler2D dissolve_noise;
uniform float progress : hint_range(0.0, 1.0) = 0.0;
uniform float edge_width : hint_range(0.0, 0.2) = 0.05;
uniform vec3 edge_color : source_color = vec3(1.0, 0.5, 0.0);

void fragment() {
    vec3 base_color = texture(TEXTURE, UV).rgb;

    // 采样噪声纹理
    float noise = texture(dissolve_noise, UV).r;

    // smoothstep 生成溶解遮罩
    // progress=0时全保留，progress=1时全溶解
    float dissolve = smoothstep(progress, progress - edge_width, noise);

    // 计算边缘发光区域（刚好在溶解边界上）
    float edge = smoothstep(progress, progress - edge_width * 0.5, noise)
               - smoothstep(progress, progress - edge_width, noise);
    edge = clamp(edge, 0.0, 1.0);

    // 基础颜色 * 溶解遮罩 + 边缘发光
    vec3 final_color = base_color * dissolve + edge_color * edge * 3.0;

    ALBEDO = final_color;
    EMISSION = edge_color * edge * 2.0;
}
```

在 i3dAct 编辑器里拖入一张噪声纹理到 `dissolve_noise`，然后调 `progress` 就能看到溶解效果。

代码拆解：

1. `texture(dissolve_noise, UV).r` —— 每个像素拿到一个随机灰度值
2. `smoothstep(progress, progress - edge_width, noise)` —— noise 小于 progress 的像素消失
3. 两个 smoothstep 相减 —— 提取出刚好在边界上的窄条区域
4. `clamp(edge, 0.0, 1.0)` —— 确保边缘值不会异常
5. `mix` 的替代写法 —— `base * dissolve + edge * color`，本质是 mix 的数学展开

### 脚本控制溶解

```s3
@export var dissolve_material: Material

func play_dissolve():
    var tween = create_tween()
    tween.tween_method(set_dissolve, 0.0, 1.0, 2.0)

func set_dissolve(value: float):
    dissolve_material.set_shader_parameter("progress", value)
```

2秒内从实体完全溶解。smoothstep 让溶解边缘有自然的侵蚀感，不是整齐地一行一行消失。

## step：smoothstep 的硬边兄弟

有时候你确实需要硬边缘。这时候用 `step`：

```glsl
float result = step(阈值, 值);
```

- 值 < 阈值 → 返回 0
- 值 ≥ 阈值 → 返回 1

没有过渡，一刀两断。

### 什么时候用 step，什么时候用 smoothstep

| 场景         | 用哪个     | 原因                 |
| ------------ | ---------- | -------------------- |
| 像素风游戏   | step       | 像素风需要锐利边缘   |
| UI 遮罩      | step       | 按钮点击区域要精确   |
| 溶解效果     | smoothstep | 需要柔软侵蚀边缘     |
| 光照范围     | smoothstep | 真实光照没有锐利边界 |
| 卡通渲染描边 | step       | 风格化需要硬边       |

### step 的坑：抗锯齿

step 的硬边在屏幕上会出现锯齿（aliasing）。因为像素只有"是"或"否"两种状态，没有中间过渡。

如果你的效果出现锯齿，考虑换成 smoothstep 并给一个很小的过渡范围：

```glsl
// 可能有锯齿
float mask = step(0.5, UV.x);

// 抗锯齿版本，过渡范围 1 像素宽
float mask_aa = smoothstep(0.5 - fwidth(UV.x), 0.5 + fwidth(UV.x), UV.x);
```

`fwidth` 返回当前像素在屏幕上的变化率，用来动态调整过渡宽度。

## 实战：能量护盾效果

综合运用三个函数，做一个受击时闪烁的能量护盾：

```glsl
shader_type spatial;

uniform float hit_time = -100.0;  // 上次受击时间，默认很久以前
uniform vec3 shield_color : source_color = vec3(0.2, 0.6, 1.0);
uniform float shield_power : hint_range(0.0, 1.0) = 0.3;

void fragment() {
    // 时间差：距离上次受击多久了
    float time_since_hit = TIME - hit_time;

    // 受击闪烁：刚被击中时强度高，随时间衰减
    float hit_flash = clamp(1.0 - time_since_hit * 3.0, 0.0, 1.0);
    hit_flash = pow(hit_flash, 2.0);  // 让衰减更自然

    // Fresnel 边缘光：视角越平行表面，越亮
    vec3 view_dir = normalize(VIEW);
    vec3 normal = normalize(NORMAL);
    float fresnel = 1.0 - abs(dot(view_dir, normal));
    fresnel = pow(fresnel, 3.0);

    // 基础护盾强度 + 受击闪烁
    float intensity = mix(shield_power, 1.0, hit_flash);

    // 最终颜色
    vec3 emission = shield_color * fresnel * intensity * 2.0;

    // 透明度：边缘不透明，中心半透明
    float alpha = mix(0.1, 0.6, fresnel);
    alpha = clamp(alpha + hit_flash * 0.3, 0.0, 1.0);

    ALBEDO = shield_color * 0.1;
    EMISSION = emission;
    ALPHA = alpha;
}
```

脚本里触发受击：

```s3
@export var shield_material: Material

func on_hit():
    shield_material.set_shader_parameter("hit_time", Time.time)
```

这里用到了：

- `clamp` 限制受击闪光的时间和透明度
- `mix` 在基础强度和受击强度之间切换
- `smoothstep` 的亲戚 `pow` 来调整曲线形态（fresnel 的 pow 也是在做边缘软化）

## 这篇你学到了什么

1. `smoothstep(a, b, x)` —— 让值在 a 和 b 之间平滑过渡，返回 0 到 1
2. `mix(a, b, t)` —— 按 t 的比例混合 a 和 b，不只是颜色
3. `clamp(x, min, max)` —— 把值限制在范围内，防止越界
4. `step(阈值, x)` —— 硬边缘判断，需要锐利边界时用
5. 组合拳：smoothstep 做遮罩 → mix/clamp 控制输出 → 实现溶解、扫光、混合等效果

## 三个函数速查

| 函数                      | 作用       | 典型用法                       |
| ------------------------- | ---------- | ------------------------------ |
| `smoothstep(e0, e1, x)` | 平滑过渡   | 边缘柔化、溶解遮罩、范围衰减   |
| `mix(a, b, t)`          | 线性混合   | 颜色混合、UV切换、位置插值     |
| `clamp(x, min, max)`    | 限制范围   | 防止颜色溢出、限制UV、限制参数 |
| `step(t, x)`            | 硬边缘判断 | 像素风、精确遮罩、风格化渲染   |

## 下篇预告

你现在掌握了 shader 最核心的数学工具。但 dissolve 效果里的噪声纹理是怎么来的？自己画太麻烦，用代码生成又太复杂。

下一篇，我们来学 **Noise 纹理**：i3dAct 内置的 NoiseTexture2D 怎么用，怎么让噪声动起来，以及怎么叠加多层噪声做出火焰、水面、云层的效果。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲 Noise 纹理，内容更有意思。
