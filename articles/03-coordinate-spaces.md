# 坐标系大乱斗：Model、World、View、Screen都是啥？

写shader的时候，你会遇到一堆跟坐标相关的变量：

- `VERTEX` - 顶点坐标
- `MODEL_MATRIX` - 模型变换矩阵
- `MODELVIEW_MATRIX` - 模型视图矩阵
- `SCREEN_UV` - 屏幕UV
- ...

它们都跟位置有关，但"在哪"和"指向哪"完全不同。搞混了，写出来的效果就跑偏。

这一章，我们来理清这些坐标系。

## 一个例子说明问题

假设你做了一个角色模型，角色手里拿一把剑。

你想写一个shader，让剑尖发出光晕。

问题来了：剑尖在哪？

- 在模型空间里，剑尖可能在 (1.5, 0.0, 0.0) —— 相对于角色中心
- 在世界空间里，剑尖可能在 (100.0, 5.0, 50.0) —— 相对于场景原点
- 在视图空间里，剑尖可能在 (-0.3, 0.1, 10.0) —— 相对于相机

同一个点，在不同坐标系里有不同的数值。你用错了，光晕就跑到别的地方去了。

## 四大坐标系详解

### Model Space（模型空间）

**原点**：模型自己的中心

**用途**：存储模型的顶点数据

**谁在用**：`VERTEX`

模型空间是模型"自己的世界"。一个角色模型，脚底可能在Y=-1，头顶在Y=2，不管这个角色站在场景的哪里，模型空间里的坐标都不变。

```glsl
// 在vertex()函数里，VERTEX一开始就是模型空间坐标
void vertex() {
    // VERTEX 此时是模型空间
    VERTEX.y += 0.5;  // 让整个模型往上移动0.5单位
}
```

### World Space（世界空间）

**原点**：场景原点 (0, 0, 0)

**用途**：游戏逻辑、物体之间的位置关系

**谁在用**：`MODEL_MATRIX * VERTEX`

世界空间是"大家共享的世界"。所有物体的位置、旋转、缩放，最终都转换到世界空间来比较。

```glsl
varying vec3 world_pos;

void vertex() {
    // 模型空间 → 世界空间
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}
```

### View Space（视图空间 / 相机空间）

**原点**：相机位置

**用途**：光照计算、深度判断

**谁在用**：`MODELVIEW_MATRIX * VERTEX`

视图空间是"相机眼中的世界"。原点在相机位置，Z轴指向相机前方。

```glsl
varying vec3 view_pos;

void vertex() {
    // 模型空间 → 视图空间
    view_pos = (MODELVIEW_MATRIX * vec4(VERTEX, 1.0)).xyz;
}
```

### Screen Space（屏幕空间）

**原点**：屏幕左下角

**用途**：最终渲染输出

**谁在用**：`SCREEN_UV`、`FRAGCOORD`

屏幕空间是"最终画在哪"。X从0到屏幕宽度，Y从0到屏幕高度。在shader里通常用归一化的UV形式（0-1）。

```glsl
void fragment() {
    // SCREEN_UV 是屏幕空间的UV坐标
    vec4 screen_color = texture(SCREEN_TEXTURE, SCREEN_UV);
}
```

## 坐标转换流程

```
模型数据 (Model Space)
    │
    │ MODEL_MATRIX（模型的位置、旋转、缩放）
    ▼
世界坐标 (World Space)
    │
    │ VIEW_MATRIX（相机的位置、旋转）
    ▼
视图坐标 (View Space)
    │
    │ PROJECTION_MATRIX（透视/正交投影）
    ▼
裁剪空间 (Clip Space)
    │
    │ 透视除法
    ▼
屏幕坐标 (Screen Space)
```

你不需要背这个流程，但需要知道：**每次转换都是乘一个矩阵**。

## 实战：在世界空间里做效果

假设你想让一个物体的表面颜色随世界位置变化——比如越靠近世界Y=0的位置越蓝。

如果直接用VERTEX.y，效果会跟着物体移动。物体移动到Y=100，效果也跑到Y=100。

这时候需要用世界坐标：

```glsl
shader_type spatial;

varying vec3 world_pos;

void vertex() {
    // 转换到世界空间
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    // 基于世界坐标Y值混合颜色
    float blend = smoothstep(-2.0, 2.0, world_pos.y);  // Y从-2到2，颜色渐变

    vec3 low_color = vec3(0.2, 0.4, 0.8);   // 低处偏蓝
    vec3 high_color = vec3(0.8, 0.6, 0.3);  // 高处偏橙

    ALBEDO = mix(low_color, high_color, blend);
}
```

这样，无论物体移动到哪，颜色渐变都基于世界坐标计算，保持一致。

<!-- [配图：物体在世界坐标中产生一致的渐变效果] -->

## 实战：基于距离相机的远近做效果

假设你想让物体"距离相机越远颜色越淡"。

需要视图空间：

```glsl
shader_type spatial;

varying float view_distance;

void vertex() {
    // 视图空间的Z就是距离相机的深度
    vec4 view_pos = MODELVIEW_MATRIX * vec4(VERTEX, 1.0);
    view_distance = -view_pos.z;  // 注意：Z是负的（相机看向-Z方向）
}

void fragment() {
    // 距离越远，颜色越淡
    float fade = smoothstep(5.0, 20.0, view_distance);

    vec3 base_color = vec3(0.8, 0.4, 0.2);  // 基础颜色
    vec3 far_color = vec3(0.5, 0.5, 0.5);   // 远处颜色

    ALBEDO = mix(base_color, far_color, fade);
}
```

## varying关键字是什么

你可能注意到了，vertex函数里的变量要加 `varying` 才能在fragment函数里用。

```glsl
shader_type spatial;

varying vec3 world_pos;  // 声明varying变量（写在函数外面）

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;  // 在vertex里赋值
}

void fragment() {
    float x = world_pos.x;  // 在fragment里读取
}
```

**varying的作用**：在vertex和fragment之间传递数据。

vertex函数处理顶点，fragment函数处理像素。顶点只有几个到几千个，像素有几百万个。varying变量会被GPU自动插值，从顶点值过渡到每个像素。

**注意**：varying变量要写在函数外面（全局位置），不能写在函数里面。

## 常见陷阱

### 陷阱1：在fragment里直接用VERTEX

```glsl
void fragment() {
    float x = VERTEX.x;  // 错误！
}
```

`VERTEX` 只在vertex函数里有效。fragment函数里要用，必须通过varying传递。

### 陷阱2：搞混坐标空间

```glsl
// 错误：用模型坐标判断是否在水面以下
if (VERTEX.y < 0.0) {
    // 物体移动到世界Y=100时，这个判断就错了
}
```

应该用世界坐标：

```glsl
varying vec3 world_pos;

void vertex() {
    world_pos = (MODEL_MATRIX * vec4(VERTEX, 1.0)).xyz;
}

void fragment() {
    if (world_pos.y < 0.0) {
        // 正确：基于世界坐标判断
    }
}
```

### 陷阱3：法线变换用错矩阵

法线不能直接用MODEL_MATRIX变换，要用 `MODEL_MATRIX` 的逆转置矩阵。

i3dAct提供了 `NORMAL_MATRIX`：

```glsl
void vertex() {
    // 正确的法线变换
    NORMAL = normalize(NORMAL_MATRIX * NORMAL);
}
```

为什么？简单说：如果模型有非均匀缩放（比如x方向放大2倍，y方向不变），直接用MODEL_MATRIX会让法线方向变错，导致光照计算错误。

这个问题更深入的原理涉及线性代数，现在先记住"法线用NORMAL_MATRIX"就够了。

## 这篇你学到了什么

1. 四大坐标系：Model（模型）→ World（世界）→ View（视图）→ Screen（屏幕）
2. `MODEL_MATRIX` 把模型坐标转换到世界坐标
3. `MODELVIEW_MATRIX` 把模型坐标转换到视图坐标
4. `varying` 用于在vertex和fragment之间传递数据
5. 法线变换要用 `NORMAL_MATRIX`，不能直接用MODEL_MATRIX
6. 不同效果需要选择合适的坐标系

## 什么时候用哪个坐标系

| 需求 | 用什么坐标系 |
|------|-------------|
| 模型自带的动画效果（如呼吸、摇摆） | 模型空间（VERTEX） |
| 跟世界位置相关的效果（如水面、地面阴影） | 世界空间（MODEL_MATRIX * VERTEX） |
| 跟相机距离相关的效果（如雾、景深） | 视图空间（MODELVIEW_MATRIX * VERTEX） |
| 后处理效果（如模糊、色彩校正） | 屏幕空间（SCREEN_UV） |

## 下篇预告

现在你知道了坐标系是怎么回事，可以正确地"定位"你的shader效果了。

下一篇，我们来聊聊纹理采样——`texture()` 函数的更多用法，包括多层纹理混合、UV动画等。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲纹理采样，让shader效果更丰富。
