# SDF布尔运算：三行代码，无限组合

第10篇我们用单个 SDF 做出了球形切割。但实际项目里很少只用一个形状——你可能想在球旁边再挖个洞，或者把盒子和球叠在一起切。

SDF 最妙的地方就是这个：**组合形状只需要一行数学运算**。不需要建模软件，不需要 CSG 节点，纯 shader 搞定。

## 先"看见"三种运算

布尔运算的核心代码短到离谱：

```glsl
float sdf_union(float d1, float d2)        { return min(d1, d2); }     // 并集
float sdf_intersection(float d1, float d2) { return max(d1, d2); }     // 交集
float sdf_subtraction(float d1, float d2)  { return max(-d1, d2); }    // 差集
```

第10篇讲过，SDF 返回有向距离（负=内，正=外）。布尔运算做的事就是**在距离值上比大小**。

### 并集 Union：取 min

```glsl
float d = min(d_sphere, d_box);
```

为什么是 `min`？空间中任意一点，到两个形状的"最近距离"——当然取更近的那个。`min` 就是"谁离我更近就听谁的"，两个形状自然粘在一起。

用第10篇的可视化 shader 把并集画出来：

```glsl
float d1 = length(world_pos - vec3(-1.5, 0.0, 0.0)) - 1.2;  // 左球
float d2 = sdf_box(world_pos, vec3(1.5, 0.0, 0.0), vec3(1.0)); // 右盒子
float d = min(d1, d2);
float gray = clamp(d * 0.3 + 0.5, 0.0, 1.0);
ALBEDO = vec3(gray);
```

你会看到左边的球和右边的盒子"长"在了一起，没有接缝，就是完整的一体。

### 交集 Intersection：取 max

```glsl
float d = max(d_sphere, d_box);
```

`max` 的逻辑反过来：**两个区域都"在里面"（d 都 < 0），`max` 才 < 0**。只要有一个在外面（d > 0），`max` 就是正数——被切掉了。所以只留下重叠区域。

把上面的 `min` 换成 `max`，你会看到只剩两个形状的交叉部分——像一个被四个平面切过的球。

### 差集 Subtraction：max(-d1, d2)

```glsl
float d = max(-d_sphere, d_box);  // 盒子减去球
```

`-d1` 把里外翻转（原来是负的变正，正的变负），然后和 d2 取 `max`（交集）。效果：从盒子里挖掉球形。

```glsl
float d = max(-d_sphere, d_box);
float gray = clamp(d * 0.3 + 0.5, 0.0, 1.0);
ALBEDO = vec3(gray);
```

盒子表面出现了一个半球形缺口——用球体"啃"出来的。

**差集有方向**：`max(-d1, d2)` 是 d2 减去 d1。反过来 `max(-d2, d1)` 就是 d1 减去 d2，效果不同。

## 实战：组合切割

把布尔运算塞进第10篇的切割框架，`clip` 那行换一下就行：

```glsl
shader_type spatial;
render_mode cull_disabled;

uniform vec3 shape1_pos = vec3(0.0, 0.0, 0.0);
uniform vec3 shape2_pos = vec3(1.0, 0.0, 0.0);
uniform float sphere_radius = 1.2;
uniform vec3 box_half = vec3(1.8);
uniform vec3 section_color : source_color = vec3(0.9, 0.3, 0.0);
uniform float edge_width : hint_range(0.0, 0.1) = 0.02;
uniform vec3 edge_color : source_color = vec3(1.0, 0.6, 0.1);

varying vec3 world_pos;

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    // 盒子 SDF
    vec3 q = abs(world_pos - shape1_pos) - box_half;
    float d_box = length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);

    // 球体 SDF
    float d_sphere = length(world_pos - shape2_pos) - sphere_radius;

    // 布尔运算：盒子减去球 = 方孔里挖圆槽
    float clip = max(-d_sphere, d_box);
    // 试试换成: min(d_sphere, d_box)   → 球+盒子并集
    // 试试换成: max(d_sphere, d_box)   → 球∩盒子交集

    vec3 base = texture(TEXTURE, UV).rgb;
    vec3 normal = normalize(NORMAL);
    vec3 view = normalize(VIEW);
    bool is_back = dot(normal, view) > 0.0;

    if (clip < 0.0) {
        if (is_back) {
            ALBEDO = section_color;
        } else {
            discard;
        }
    } else {
        ALBEDO = base;
        float edge = smoothstep(edge_width, 0.0, clip);
        EMISSION = edge_color * edge * 2.0;
    }
}
```

效果：方形的剖切区域，但角落里有一个球形缺口。换 `clip` 那行就能在并集/交集/差集之间切换。

从第8篇到现在，框架代码一直没变（背面判断、截面着色、边缘发光）。变的只有 `clip` 那一行。**框架不变，距离函数随便换**——这就是 SDF 切割模式的设计价值。

## 平滑混合（可选）

`min`/`max` 在形状交界处是锐利的。想让接缝变平滑，用多项式混合：

```glsl
float sdf_smooth_min(float a, float b, float k) {
    float h = clamp(0.5 + 0.5 * (b - a) / k, 0.0, 1.0);
    return mix(b, a, h) - k * h * (1.0 - h);
}
```

`k` 控制混合范围，越大越软。`smooth_min(d1, d2, 0.3)` 就能让并集的交界面自然过渡。不是所有效果都需要，但做有机形状（黏土、流体）很好用。

## 用脚本控制

两个形状都可以独立驱动：

```s3
@export var clip_material: Material

func Update():
    # shape1 跟随鼠标/目标，shape2 固定
    clip_material.set_shader_parameter("shape2_pos", some_target.global_position)
```

或者做动画——让球在盒子周围转：

```s3
func play_orbit(duration: float = 4.0):
    var tween = create_tween().set_loops()
    tween.tween_method(
        func(angle):
            var x = cos(angle) * 2.0
            var z = sin(angle) * 2.0
            clip_material.set_shader_parameter("shape2_pos", Vector3(x, 0.0, z)),
        0.0, TAU, duration
    )
```

球绕着盒子转，切割区域跟着动态变化。

## 这篇你学到了

- 并集 `min(d1, d2)`：两个形状粘在一起
- 交集 `max(d1, d2)`：只留重叠部分
- 差集 `max(-d1, d2)`：d2 减去 d1，挖洞专用
- 三种运算本质是距离值比大小，不是几何计算
- 框架代码不变，换 `clip` 那行就能切换切割形状

## 下篇预告

形状能组合了，但切面还是静止的。下一篇我们让切面动起来——旋转扫描、节奏脉冲、沿路径移动……让剖切效果真正"活"起来。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲剖切动画，让切割动起来。
