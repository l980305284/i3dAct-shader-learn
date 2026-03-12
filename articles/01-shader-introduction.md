# 别被"着色器"这个名字吓跑——写给程序员的i3dAct Shader入门

假设你接到一个需求：让一个3D球体每秒闪烁一次，从白色变到淡红色，再变回来。

传统做法是这样的：

```s3
# i3dAct脚本 (.s3文件)
var elapsed_time = 0.0

func Update():
    elapsed_time += get_delta_time()
    var flash = 0.5 + 0.5 * sin(elapsed_time * 6.28)
    mesh_render.get_surface_override_material(0).albedo_color = Color(1.0, flash, flash)
```

这需要在每帧里获取材质、修改属性。如果有十个物体要闪烁，就得写十份代码。

但如果我告诉你，shader只要5行：

```glsl
shader_type spatial;

void fragment() {
    ALBEDO = vec3(1.0, 0.5 + 0.5 * sin(TIME), 0.5 + 0.5 * sin(TIME));
}
```

把这个shader挂到材质上，任何使用这个材质的物体都会自动闪烁。十个物体？没问题。一百个？也没问题。

这不是"更简洁的写法"，这是**另一种思维方式**。理解这种思维方式，你就能做出脚本做不到的效果。

## Shader到底是什么

"着色器"这个名字起得很糟糕。它让你以为shader是用来"着色"的，是美术工具。

实际上，shader是一个**在GPU上运行的函数**。

就这么简单。

你写一个函数，输入是"像素的位置"，输出是"这个像素应该是什么颜色"。GPU拿到这个函数，对屏幕上几百万个像素同时执行——这就是shader。

用伪代码表示：

```javascript
// 这是你写的shader
function getColor(uv) {
    return { r: 1.0, g: 0.0, b: 0.0 };  // 红色
}

// GPU做的事（伪代码，实际是并行的）
for (let x = 0; x < screenWidth; x++) {
    for (let y = 0; y < screenHeight; y++) {
        let uv = { x: x / screenWidth, y: y / screenHeight };
        screen[x][y] = getColor(uv);  // 每个像素都调用你的函数
    }
}
```

当然，实际运行时不是串行的for循环，而是几百万个GPU核心同时执行。这就是为什么shader能实时处理复杂效果——并行计算的威力。

## 为什么shader看起来很难学

因为有两个概念混在一起了：

1. **Shader本身**——就是写一个 `getColor(x, y)` 函数，这有什么难的？
2. **图形学知识**——怎么用数学描述颜色、光照、形状？

很多人把第2点当成了shader的难度。其实不是。shader只是个工具，图形学才是知识。

这就好比：学JavaScript不等于学前端开发。JavaScript是工具，HTML/CSS/浏览器原理才是知识。

这个系列聚焦在shader工具本身，图形学知识会在用到时顺便讲。我不会上来就给你讲光照模型、PBR渲染——那是在你需要的时候才讲的东西。

## i3dAct里的Shader类型

新建一个ShaderMaterial，第一行你必须写：

```glsl
shader_type spatial;
```

这告诉引擎你这个shader是用在哪里的。i3dAct支持这些类型：

| 类型 | 用途 | 你现在需要记住的 |
|------|------|-----------------|
| `canvas_item` | 2D UI、精灵等 | 做2D游戏记住这个 |
| `spatial` | 3D物体（MeshRender等） | **记住这个** |
| `particles` | 粒子系统 | 以后再说 |
| `sky` | 天空盒 | 以后再说 |
| `fog` | 雾效 | 以后再说 |

如果你做3D游戏，大部分时候用 `spatial`。就这么简单。

## 你的第一个Shader

别看了，动手写。

1. 在场景里创建一个球体（或任何MeshRender）
2. 在Inspector里找到Material属性
3. 点击下拉箭头，选择"New ShaderMaterial"
4. 点击新创建的ShaderMaterial，找到Shader属性
5. 点击下拉箭头，选择"New Shader"
6. 点击新创建的Shader，会打开shader编辑器

你会看到引擎已经帮你写了：

```glsl
shader_type spatial;

void vertex() {
    // Vertex shader code goes here.
}

void fragment() {
    // Fragment shader code goes here.
}
```

先别管vertex是什么，把vertex函数删掉，只保留：

```glsl
shader_type spatial;

void fragment() {
    ALBEDO = vec3(1.0, 0.0, 0.0);
}
```

运行游戏。

你的球体变成了纯红色。

<!-- [配图：左边是原球体，右边是shader效果后的纯红色球体] -->

## fragment()函数做了什么

`fragment()` 是shader的入口函数，每个像素都会调用一次。

你可以把它理解为：**"GPU问你：这个像素应该显示什么颜色？你的fragment函数就是回答。"**

在`spatial` shader里，`ALBEDO` 是输出变量，代表这个像素的基础颜色。

`vec3(1.0, 0.0, 0.0)` 是一个三维向量，三个分量分别是：
- R = 1.0（红色通道，最大值）
- G = 0.0（绿色通道）
- B = 0.0（蓝色通道）

所以这是纯红色。

你可能会问：那模型原来的贴图呢？我明明有贴图，为什么没了？

因为你把ALBEDO赋值为纯红，**覆盖了贴图的颜色**。贴图的数据还在，只是你没使用它。

如果你想显示贴图本身：

```glsl
shader_type spatial;

void fragment() {
    ALBEDO = texture(TEXTURE, UV).rgb;
}
```

`texture()` 是采样函数，从纹理中读取颜色。
`TEXTURE` 是你的贴图。
`UV` 是像素在模型表面的坐标（下一篇详细讲）。

## 让颜色动起来

回到闪烁效果：

```glsl
shader_type spatial;

void fragment() {
    float flash = 0.5 + 0.5 * sin(TIME * 6.28);  // 每秒一个周期
    ALBEDO = vec3(1.0, flash, flash);
}
```

`TIME` 是引擎提供的内置变量，表示游戏开始后经过的秒数。

`sin(TIME * 6.28)` 在-1到1之间波动（6.28 ≈ 2π，所以每秒完成一个完整周期）。乘以0.5再加0.5，就变成0到1之间波动。

G和B通道同步变化，红色通道保持1.0，所以是从白色（1,1,1）到粉红色（1,0,0）再到白色。

这5行代码做的事，用脚本要更多步骤，而且只能控制一个物体。

## shader vs 传统方式：什么时候用哪个

<!-- [配图：对比图，左边是脚本控制材质的代码，右边是shader代码面板] -->

shader擅长：
- 每帧都需要重新计算的效果（波动、流动、闪烁）
- 需要精确控制每个像素的效果（溶解、扭曲、变形）
- 需要高性能处理的效果（粒子、大量物体）
- 需要统一应用到多个物体的效果（材质复用）

传统方式（脚本控制材质属性）擅长：
- 离散的状态切换（开/关、显示/隐藏）
- 需要逻辑判断的场景（根据血量变色）
- 简单的属性变化（位置、缩放、旋转）
- 需要和其他系统交互的效果（音效、UI）

很多效果两种方式都能做，选择你觉得舒服的。但随着你熟悉shader，你会发现越来越多效果用shader更简单。

## 这篇你学到了什么

1. Shader就是在GPU上运行的函数，输入像素位置，输出像素颜色
2. i3dAct里用 `shader_type spatial` 声明3D shader
3. `fragment()` 函数决定每个像素的颜色
4. `ALBEDO = vec3(r, g, b)` 设置像素颜色
5. `TIME` 变量让效果动起来

## 下篇预告

你注意到 `UV` 这个变量了吗？它决定了贴图上的哪个位置对应模型表面的哪个像素。

下一篇，我们来拆解UV坐标，学会之后你就能做出各种渐变、圆形、自定义形状的效果。

那才是shader真正开始变得有趣的地方。

---

*思考题：如果把ALBEDO改成 `vec3(UV.x, UV.y, 0.0)`，你会看到什么？先猜一下，然后试试看。*

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们聊UV坐标，内容更有意思。
