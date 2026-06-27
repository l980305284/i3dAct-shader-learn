# 一行 discard，切开你的模型

假设你做了一个机器人模型，想展示它内部的机械结构。或者你做了一个建筑，想让玩家看到房间布局。总之——你需要把模型"切开"。

最暴力的办法：shader 里有一个关键字 `discard`，它能让 GPU 直接跳过当前像素。配合一个平面方程，就能做到"平面一侧显示，另一侧消失"。

先看代码，再解释：

```glsl
shader_type spatial;

uniform vec3 plane_normal = vec3(0.0, 1.0, 0.0);
uniform float plane_distance = 0.0;

varying vec3 world_pos;

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    // 计算当前像素到切平面的距离
    float clip = dot(world_pos, normalize(plane_normal)) - plane_distance;

    // 在平面"下方"的像素，直接丢弃
    if (clip < 0.0) discard;

    vec3 base = texture(TEXTURE, UV).rgb;
    ALBEDO = base;
}
```

把这段代码保存为 `.s3shader` 文件，拖到一个 `MeshRender` 的材质上。你会看到模型从某个位置被"削掉"了一截。

注意：切面内部是透明的，你能看到模型的背面。这是因为 `discard` 只是跳过了当前像素——背面的像素还在正常渲染。下一篇我们解决这个问题，让截面看起来像个实心剖面。

## 那行代码在干什么

核心就两行：

```glsl
float clip = dot(world_pos, normalize(plane_normal)) - plane_distance;
if (clip < 0.0) discard;
```

`dot(world_pos, plane_normal)` 是点积。不用管它的数学定义，你只需要知道它在干什么：**它告诉你一个点在平面的哪一侧**。

想象一个水平面（normal = `(0, 1, 0)`，即朝上）：

- 点在平面上方 → dot 结果 > `plane_distance` → `clip > 0` → 保留
- 点在平面下方 → dot 结果 < `plane_distance` → `clip < 0` → 丢弃

`plane_distance` 控制平面离模型原点多远。调大它，切面往上移；调小它，切面往下移。

用其他法线方向：

- `(0, 1, 0)` —— 水平切，从上往下削
- `(1, 0, 0)` —— 竖直切，从右往左削
- `(0, 0, 1)` —— 前后切，从前往后削
- 任何方向都行，比如 `(0.7, 0.7, 0)` —— 斜着切

## varying 是什么

你可能注意到了 `varying vec3 world_pos` 这一行。这是 vertex shader 和 fragment shader 之间的"传话通道"：

```glsl
// vertex() 负责算，放到 varying 里
void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

// fragment() 通过 varying 拿到值
void fragment() {
    float clip = dot(world_pos, ...);
    // world_pos 在这里可以直接用
}
```

为什么要在 `vertex()` 里算世界坐标？因为 `vertex()` 每个顶点跑一次，算好世界坐标后 GPU 自动插值给每个像素。如果在 `fragment()` 里算，每个像素都要重复一遍矩阵乘法，浪费。

`varying` 就是用来在 vertex 和 fragment 之间传递数据的。后面你会经常用到它。

## 稍微改进一下

硬边切有点假。加点 `smoothstep` 让切割边缘有一点点过渡（一个像素宽），视觉上会舒服很多：

```glsl
shader_type spatial;

uniform vec3 plane_normal = vec3(0.0, 1.0, 0.0);
uniform float plane_distance = 0.0;
uniform float edge_softness : hint_range(0.0, 0.01) = 0.001;

varying vec3 world_pos;

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    float clip = dot(world_pos, normalize(plane_normal)) - plane_distance;

    // 用 smoothstep 做柔和边缘，避免像素锯齿
    float alpha = smoothstep(0.0, edge_softness, clip);
    if (alpha < 0.01) discard;

    vec3 base = texture(TEXTURE, UV).rgb;
    ALBEDO = base;
    ALPHA = alpha;
}
```

`edge_softness` 默认 0.001，几乎看不出来软边，但能消掉锯齿。记得在材质 Inspector 里把 `Transparent` 打开，否则 `ALPHA` 不生效。

## 用脚本控制切面

在实际项目里，你不太可能手调 `plane_normal` 和 `plane_distance`。更常见的做法是用脚本驱动：

```s3
@export var clip_material: Material
@export var clip_target: Item3D

func Update():
    if clip_target:
        var y = clip_target.global_position.y
        clip_material.set_shader_parameter("plane_distance", y)
```

切面位置会自动跟随目标物体的 Y 坐标，你可以用另一个 Item3D 做"切割手柄"。

## 这篇你学到了

- `discard` 让像素直接消失，GPU 跳过后续所有计算
- 平面方程 `dot(pos, normal) - distance` 判断像素在哪一侧
- `varying` 在 vertex 和 fragment 之间传递数据
- 脚本驱动 shader 参数做动态切割

## 局限

切开后里面是空的——你看到的是模型的"背面"。切面本身没有颜色，就是个透明的空洞。

下一篇解决这个问题：**用 `NORMAL` 判断正反面，给切断面填上颜色**。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲给剖切面上色，效果更实用。
