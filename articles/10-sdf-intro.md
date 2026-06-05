# SDF入门：让切割不再只能走直线

第8篇我们用 `discard` + 平面方程把模型切开了。第9篇我们给切面上了色。

但有个问题不知道你试过没有——平面切割只能直着切。`dot(world_pos, normal) - distance` 描述的永远是一个无限大的平面，一刀下去横平竖直。

如果你想在模型上挖一个球形的洞呢？想啃出一个带圆角的缺口呢？平面方程做不到——它不知道"圆"是什么。

这时候该 **SDF（Signed Distance Field，有向距离场）** 上场了。

一句话概括 SDF：**给你空间中的任意一点，它告诉你这个点到最近表面的距离——以及你是在表面外面还是里面。**

## 先"看见"SDF

在理解它之前，我们直接把它画出来。球体的 SDF 公式就一行：

```glsl
float sdf_sphere(vec3 p, float r) {
    return length(p) - r;
}
```

`length(p)` 是点到原点的距离。减去半径 `r`：刚好在球面上返回 `0`，在球外返回**正数**，在球内返回**负数**——正负号告诉你方向，这就是"有向"的意思。

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

    // d=0 映射为 gray=0.5（灰色=球面）
    float gray = clamp(d * 0.5 + 0.5, 0.0, 1.0);

    ALBEDO = vec3(gray);
}
```

把这个 shader 贴到一个 Cube 上，你会看到：球心位置是黑的，球面位置是灰色的，球外逐渐变白。空间中每个像素到球面的距离，平滑地从黑过渡到白——这就是"场"。

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

盒子比球体复杂，但用法一样——传入位置，返回有向距离。不用死记硬背，当工具函数用。

把可视化换成盒子验证：

```glsl
float d = sdf_box(world_pos, vec3(0.0), vec3(1.5));
float gray = d * 0.3 + 0.5;
gray = clamp(gray, 0.0, 1.0);
ALBEDO = vec3(gray);
```

你会看到一个**圆角**矩形的渐变区域——SDF 天生带"圆润感"，等距线自动变圆角。更多形状（圆柱、圆环、锥体）留给后面的文章，今天球体+盒子够用了。

## 实战：球形切割

有了 SDF，把平面切割改成球形切割，只换一行代码。

平面切割：

```glsl
float clip = dot(world_pos, normalize(plane_normal)) - plane_distance;
```

换成球体 SDF：

```glsl
float clip = length(world_pos - sphere_center) - sphere_radius;
```

完整 shader，复用第9篇的截面着色方案：

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
    // 核心变化：用球体 SDF 替代平面方程
    float clip = length(world_pos - sphere_center) - sphere_radius;

    vec3 base = texture(TEXTURE, UV).rgb;
    vec3 normal = normalize(NORMAL);
    vec3 view = normalize(VIEW);

    // 背面判断（跟第9篇完全一样）
    bool is_back = dot(normal, view) > 0.0;

    if (clip < 0.0) {
        // 像素在球体内部
        if (is_back) {
            ALBEDO = section_color;
        } else {
            discard;
        }
    } else {
        // 像素在球体外部，正常显示 + 边缘发光
        ALBEDO = base;
        float edge = smoothstep(edge_width, 0.0, clip);
        EMISSION = edge_color * edge * 2.0;
    }
}
```

模型被球形区域"挖空"，截面橙色，边缘发光。**换一个距离函数，就换一种切割形状**。

SDF 还有一个优势：`clip` 值本身就是平滑的。平面方程线性变化，软化等宽；SDF 距离场跟随形状，同样的 `smoothstep` 软化，切出来的效果更自然。

## 对比

| 对比维度 | 平面方程 | SDF |
|---------|---------|-----|
| 切割形状 | 只能无限平面 | 球、盒、圆柱……任意形状 |
| 边缘软化 | 线性过渡，等宽 | 跟随形状，更自然 |
| 多形状组合 | 不支持 | 可叠加（第11篇讲） |
| 公式复杂度 | 简单 | 稍复杂，但复制粘贴即可 |

平面方程不是没用，横切竖切它简单高效。但需求变成"挖球洞"或"圆角剖面"时，SDF 就是正确答案。

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

3 秒内球从半径 0 膨胀到 5.0——动态的"解剖展示"。

## 这篇你学到了

- SDF 是什么：空间中每一点到最近表面的有向距离（正=外，负=内，零=表面）
- 球体 SDF：`length(p - center) - radius`
- 盒子 SDF：拆解成 `q = abs(p - center) - half_size` 再套公式
- 用 SDF 替代平面方程做切割，一行改动实现球形挖洞
- SDF 切割的软边天然跟随形状，比平面切更自然

## 下篇预告

现在只能用单个形状切割。但 SDF 真正厉害的地方在于：**多个 SDF 可以叠加、相交、相减**——两个球取并集、一个盒子减去一个球……

这叫 **SDF 布尔运算**，第11篇我们专门讲 union（并）、intersection（交）、subtraction（差）。三行代码，无限组合。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲 SDF 布尔运算，组合出你想要的任何形状。
