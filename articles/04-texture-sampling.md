# texture()函数：Shader里最重要的那行代码

前两章我们学了UV和坐标系。现在你已经能做出渐变、波纹、基于世界坐标的效果了。

但有个问题：这些效果都是纯色，或者用数学公式生成的图案。现实中的物体——石头、木头、金属——你不可能用几行代码算出它们的表面细节。

这就需要一个东西：**纹理**。

## 为什么需要纹理

假设你要做一个砖墙。用纯色肯定不行，砖墙的纹理太复杂了。

你想到的第一个方案可能是脚本控制：

```s3
# 在i3dAct脚本里
func iStart():
    mesh_render.get_surface_override_material(0).albedo_texture = load("res://brick.png")
```

这样确实能贴图，但纹理只是"静态贴上去"，shader里没法操作它。

如果你想让砖墙：
- 表面有凹凸感（不需要额外几何体）
- 随时间风化、变色
- 被技能击中时局部溶解

这些都需要在shader里读取纹理，然后对颜色进行处理。

## texture()函数：从纹理里"采样"

`texture()` 是shader里读取纹理的唯一方式。它的用法很简单：

```glsl
vec4 color = texture(纹理, UV坐标);
```

上一篇我们其实已经用过一次了：

```glsl
shader_type spatial;

void fragment() {
    ALBEDO = texture(TEXTURE, UV).rgb;
}
```

这行代码做的事情：
1. `TEXTURE` —— 引擎自动传入的当前材质贴图
2. `UV` —— 当前像素对应的纹理坐标（0-1范围）
3. `texture()` —— 根据UV坐标，从贴图里读取颜色
4. `.rgb` —— 只取RGB颜色，忽略Alpha通道

结果：模型正常显示贴图，跟不用shader时一样。

### texture()返回什么

`texture()` 返回一个 `vec4`：
- `.r` —— 红色通道（0.0到1.0）
- `.g` —— 绿色通道
- `.b` —— 蓝色通道
- `.a` —— Alpha通道（透明度）

如果你只想要颜色，就取 `.rgb`。如果你需要透明度（比如做UI或溶解效果），就取 `.rgba` 或者直接赋值给 `ALBEDO`（引擎会自动处理）。

## 用uniform传入自定义纹理

`TEXTURE` 只能读取当前材质的主贴图。如果你想用多张不同的纹理，需要用 `uniform` 声明：

```glsl
shader_type spatial;

uniform sampler2D my_noise;  // 声明一个纹理变量

void fragment() {
    vec4 noise_color = texture(my_noise, UV);
    ALBEDO = noise_color.rgb;
}
```

在i3dAct编辑器里，你会在Inspector看到这个 `my_noise` 属性，可以直接把纹理拖进去。

## UV扭曲：让贴图"动起来"

静态纹理太死板了。在传给 `texture()` 之前修改UV，就能实现各种动态效果。

### 基础UV偏移

```glsl
shader_type spatial;

uniform float offset = 0.0;

void fragment() {
    vec2 moved_uv = UV + vec2(offset, 0.0);  // 水平偏移
    ALBEDO = texture(TEXTURE, moved_uv).rgb;
}
```

在脚本里改 `offset`，贴图就会左右移动：

```s3
func Update():
    material.set_shader_parameter("offset", TIME * 0.1)
```

这就是最简单的UV动画——滚动贴图，常用于水流、传送带效果。

### 用sin做波浪扭曲

```glsl
shader_type spatial;

void fragment() {
    vec2 wave_uv = UV;
    wave_uv.x += sin(UV.y * 20.0 + TIME * 3.0) * 0.02;

    ALBEDO = texture(TEXTURE, wave_uv).rgb;
}
```

效果：贴图像热浪一样扭曲。`UV.y * 20.0` 控制波纹密度，`TIME * 3.0` 让波纹移动，`0.02` 控制扭曲幅度。

如果你把这段代码挂到一个"熔岩"或者"水面"的材质上，效果立竿见影。

### 径向扭曲

```glsl
shader_type spatial;

void fragment() {
    vec2 center = vec2(0.5, 0.5);
    vec2 dir = UV - center;
    float dist = length(dir);
    float angle = atan(dir.y, dir.x);

    // 旋转+缩放，制造漩涡
    float twist = dist * 3.0 + TIME * 2.0;
    vec2 twisted_uv = center + vec2(
        cos(angle + twist) * dist,
        sin(angle + twist) * dist
    );

    ALBEDO = texture(TEXTURE, twisted_uv).rgb;
}
```

这个效果在UV的中心制造了一个漩涡，适合能量球、黑洞、传送门之类的视觉。

## 多纹理混合

一张纹理不够用。实战中经常需要把多张纹理混合在一起。

### 两张纹理的简单混合

```glsl
shader_type spatial;

uniform sampler2D texture_rocks;
uniform sampler2D texture_grass;

void fragment() {
    vec4 rocks = texture(texture_rocks, UV);
    vec4 grass = texture(texture_grass, UV);

    // 混合：0=全rock，1=全grass
    ALBEDO = mix(rocks.rgb, grass.rgb, 0.5).rgb;
}
```

`mix(a, b, t)` 是线性插值：
- t=0.0：结果是a
- t=1.0：结果是b
- t=0.5：a和b各一半

这里固定写了0.5，实际项目中可以用一张灰度图或者顶点颜色来控制混合比例。

### 用混合贴图控制过渡

```glsl
shader_type spatial;

uniform sampler2D texture_snow;
uniform sampler2D texture_ground;
uniform sampler2D blend_mask;  // 灰度图，白色=雪，黑色=地面

void fragment() {
    vec4 snow = texture(texture_snow, UV);
    vec4 ground = texture(texture_ground, UV);
    float mask = texture(blend_mask, UV).r;  // 取红色通道当混合系数

    ALBEDO = mix(ground.rgb, snow.rgb, mask);
}
```

这是地形渲染的标准做法。一张混合贴图控制不同材质在模型表面的分布，不需要对模型切分。

### 贴图切换过渡效果

让两张纹理之间做一个渐变切换，配合脚本里的进度值：

```glsl
shader_type spatial;

uniform sampler2D tex_before;
uniform sampler2D tex_after;
uniform float progress : hint_range(0.0, 1.0) = 0.0;

void fragment() {
    vec4 before = texture(tex_before, UV);
    vec4 after = texture(tex_after, UV);

    ALBEDO = mix(before.rgb, after.rgb, progress);
}
```

脚本里控制进度：

```s3
@export var dissolve_material: Material
var timer = 0.0

func Update():
    timer += get_delta_time()
    dissolve_material.set_shader_parameter("progress", smoothstep(0.0, 2.0, timer))
```

2秒内从旧纹理平滑过渡到新纹理。`smoothstep` 让过渡有缓动效果，而不是生硬的线性变化。

## 纹理的重复和采样模式

有个细节需要注意：当UV超出0-1范围时，texture()会怎么采样？

默认行为是**重复（Repeat）**：
- UV=0.5 采样贴图中间
- UV=1.5 也是贴图中间（重复一次）
- UV=-0.3 相当于 0.7

这在做平铺纹理时很有用。比如你想让砖墙更密集：

```glsl
void fragment() {
    vec2 tiled_uv = UV * 4.0;  // 0-1变成0-4，重复4次
    ALBEDO = texture(TEXTURE, tiled_uv).rgb;
}
```

一张512x512的贴图，平铺4次，效果相当于2048x2048，但内存只占用一张小图。

如果你想限制UV不越界：

```glsl
vec2 clamped_uv = clamp(UV, 0.0, 1.0);
ALBEDO = texture(TEXTURE, clamped_uv).rgb;
```

或者用 `fract()` 只取小数部分（强制重复）：

```glsl
vec2 repeated_uv = fract(UV * 4.0);
```

## 常见陷阱

### 陷阱1：纹理过滤导致边缘模糊

当你把纹理缩小显示时，`texture()` 默认会做线性过滤（bilinear filtering）。这在大部分情况下是对的，但当你需要像素级精确的效果（比如像素风游戏），过滤会让边缘模糊。

i3dAct的贴图导入设置里可以改过滤模式。对像素风游戏，用Nearest（最近邻）过滤。

### 陷阱2：在shader里直接修改纹理

你不能在shader里"修改"纹理本身。shader只能读取纹理、输出颜色。如果你想持久化修改，需要在脚本里改纹理数据，或者用Render Texture（渲染到纹理）。

### 陷阱3：纹理坐标和屏幕坐标混淆

```glsl
// 错误：用FRAGCOORD采样纹理
vec4 color = texture(my_texture, FRAGCOORD.xy);
```

`FRAGCOORD` 是像素坐标（比如 0 到 1920），而纹理采样需要0-1的UV。如果要根据屏幕位置采样，需要归一化：

```glsl
vec2 screen_uv = FRAGCOORD.xy / VIEWPORT_SIZE;
vec4 color = texture(my_texture, screen_uv);
```

### 陷阱4：忘记检查纹理是否已加载

如果 `uniform sampler2D` 没有赋值，`texture()` 会返回默认颜色（通常是黑色或粉色）。在i3dAct编辑器里拖入纹理后，记得检查材质预览。

## 实战：做一个流动的岩浆表面

综合运用我们学过的内容：

```glsl
shader_type spatial;

uniform sampler2D lava_texture;
uniform sampler2D noise_texture;
uniform float flow_speed = 1.0;
uniform float distortion_strength = 0.1;

void fragment() {
    // 用noise扭曲UV
    vec4 noise = texture(noise_texture, UV + vec2(TIME * flow_speed * 0.1, 0.0));
    vec2 distortion = (noise.rg - 0.5) * distortion_strength;

    // 主纹理做两个方向相反的流动，叠加出复杂效果
    vec2 flow_uv_1 = UV * 2.0 + vec2(TIME * flow_speed, 0.0) + distortion;
    vec2 flow_uv_2 = UV * 2.0 + vec2(-TIME * flow_speed * 0.7, TIME * flow_speed * 0.3) - distortion;

    vec4 lava_1 = texture(lava_texture, flow_uv_1);
    vec4 lava_2 = texture(lava_texture, flow_uv_2);

    // 用multiply混合，让暗部更暗，亮部更亮
    vec3 final_color = lava_1.rgb * lava_2.rgb * 2.0;

    // 加一点发光感
    EMISSION = final_color * 0.3;
    ALBEDO = final_color;
}
```

这个shader做了几件事：
1. 用噪声纹理扭曲UV，让流动不那么规律
2. 两个方向相反、速度不同的UV流动
3. 用乘法混合两张采样结果，产生更有机的明暗变化
4. 把一部分颜色输出到 `EMISSION`，让岩浆有自发光效果

效果比你单贴一张滚动贴图强得多。

## 这篇你学到了什么

1. `texture(纹理, UV)` 是读取纹理的唯一方式
2. `uniform sampler2D` 可以传入自定义纹理到shader
3. 修改UV再传给texture()，能实现扭曲、流动、漩涡等效果
4. `mix()` 可以混合两张纹理的颜色
5. 用混合贴图（mask）可以控制不同纹理在表面的分布
6. UV超出0-1会重复采样，适合做平铺效果

## 什么时候用什么

| 需求 | 做法 |
|------|------|
| 显示材质的基础贴图 | `texture(TEXTURE, UV)` |
| 让贴图滚动 | `UV + vec2(TIME * speed, 0)` |
| 让贴图扭曲 | `UV + sin/cos偏移` |
| 两张纹理渐变切换 | `mix(tex1, tex2, progress)` |
| 地形多种材质混合 | `mix()` + 混合遮罩贴图 |
| 增加纹理细节密度 | `UV * scale` 然后采样 |

## 下篇预告

你现在能用纹理做出丰富的表面效果了。但混合、过渡、限制范围这些操作，每次都手写公式有点累。

下一篇，我们来学shader里的三个核心数学函数：`smoothstep`、`mix`、`clamp`。这三个函数几乎是每个shader都会用的基础工具，掌握之后你能更轻松地控制各种过渡效果。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲smoothstep、mix、clamp这三剑客，内容更实用。
