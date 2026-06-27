# 给剖切面上色：让截面不再空洞

上一篇我们用 `discard` 把模型切开了。但切完之后截面是透明的——你看到的是模型的背面，而不是一个实心的剖切面。

这篇做两件事：

1. 给切面填上颜色，让它看起来像真正的"剖面"
2. 在切割边缘加一圈发光

## 核心思路

切开模型后，暴露出来的面其实是**模型的背面**（正常渲染时被背面剔除挡住了）。所以策略很简单：

```
像素在切平面下方 → 如果是背面 → 涂截面颜色
像素在切平面上方 → 正常渲染 + 边缘发光
```

需要一个关键设置：让 shader 同时渲染正面和背面。用 `render_mode cull_disabled`。

```glsl
shader_type spatial;
render_mode cull_disabled;

uniform vec3 plane_normal = vec3(0.0, 1.0, 0.0);
uniform float plane_distance = 0.0;
uniform vec3 section_color : source_color = vec3(0.9, 0.3, 0.0);
uniform float edge_width : hint_range(0.0, 0.1) = 0.02;
uniform vec3 edge_color : source_color = vec3(1.0, 0.6, 0.1);

varying vec3 world_pos;

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    float clip = dot(world_pos, normalize(plane_normal)) - plane_distance;

    vec3 base = texture(TEXTURE, UV).rgb;
    vec3 normal = normalize(NORMAL);
    vec3 view = normalize(VIEW);

    // 判断是不是背面：法线朝相机 → 背面
    bool is_back = dot(normal, view) > 0.0;

    if (clip < 0.0) {
        // 在切平面下方
        if (is_back) {
            ALBEDO = section_color;
        } else {
            discard;
        }
    } else {
        // 在切平面上方，正常显示 + 边缘发光
        ALBEDO = base;
        float edge = smoothstep(edge_width, 0.0, clip);
        EMISSION = edge_color * edge * 2.0;
    }
}
```

效果：模型被水平切开，截面是橙色，切割线有一圈亮橙色发光。

## 逐段解释

### render_mode cull_disabled

默认情况下 GPU 只渲染正面朝相机的三角形（背面剔除），背面被跳过以节省性能。

但剖切效果需要看到"模型内部"——也就是背面。`cull_disabled` 关掉背面剔除，所有面都渲染。只影响当前材质，性能损失可以忽略。

### 背面判断

```glsl
bool is_back = dot(normal, view) > 0.0;
```

记住结论就行：**dot(normal, view) > 0 = 背面**。两个方向向量同向时 dot 为正，法线朝相机就是背面。

### 两段逻辑

`clip < 0.0`：像素在切平面下方。背面 → 涂截面颜色。正面 → discard 跳过（这些是模型外表面，不跳过会显示错误的碎片）。

`clip > 0.0`：像素在切平面上方，正常渲染。靠近切割线（clip 接近 0）时叠加一圈发光。

`EMISSION` 是自发光，不受场景光照影响。切边发光写到 `EMISSION` 里，暗处也能看到。

## 改进：截面加纹理

纯色截面有点单调。用世界坐标贴一张纹理上去：

```glsl
uniform sampler2D section_texture;

// 在 clip < 0.0 的分支里，替换纯色：
if (is_back) {
    // 用世界坐标的 XZ 分量采样（水平切面）
    vec2 section_uv = world_pos.xz * 0.5;
    vec3 tex_sample = texture(section_texture, section_uv).rgb;
    ALBEDO = tex_sample;
}
```

`world_pos.xz` 用世界空间的 XZ 坐标替代 UV。不管模型原始 UV 怎么展开，截面纹理始终均匀。如果不是水平切面，换对应的坐标分量就行。

## 改进：建筑剖面网格线

做建筑剖面时，加上网格线更有"工程图"的感觉：

```glsl
if (is_back) {
    // 基础截面颜色
    vec3 base_section = section_color;

    // 用世界坐标做网格
    vec2 grid_uv = world_pos.xz;
    float grid = abs(sin(grid_uv.x * 10.0) * sin(grid_uv.y * 10.0));
    grid = smoothstep(0.02, 0.05, grid);

    // 暗底色 + 亮网格线
    ALBEDO = mix(base_section * 0.6, base_section, grid);
}
```

`sin` 在两个方向产生周期性波动。两条 sin 相乘，交汇处值为 1，其他位置接近 0。`smoothstep` 把接近 0 的像素变暗，留下亮网格线。

## 脚本驱动

和上一篇一样的模式，用脚本控制切面：

```s3
@export var section_material: Material

func play_section_scan(start_y: float = -2.0, end_y: float = 5.0, duration: float = 3.0):
    var tween = create_tween()
    tween.tween_method(
        func(y): section_material.set_shader_parameter("plane_distance", y),
        start_y, end_y, duration
    )
```

3 秒内切面从底部扫到顶部，像建筑剖面展示动画。

## 这篇你学到了

- `render_mode cull_disabled` 关掉背面剔除，让 GPU 渲染模型内部
- `dot(NORMAL, VIEW) > 0` 判断背面
- 分区逻辑：切面下方给截面颜色、上方正常渲染 + 边缘发光
- 世界坐标替代 UV 做截面纹理采样

## 下篇预告

`discard` 和平面切很直观，但只能做"一刀切"。想要球形切除、圆角切割、两个形状融合——需要 **SDF（Signed Distance Field，有向距离场）**。

下一篇我们讲 SDF 是什么，以及它能做什么。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲 SDF 有向距离场入门。
