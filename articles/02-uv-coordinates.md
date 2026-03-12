# UV坐标：Shader世界的经纬度

上一章结尾留了个思考题：

> 如果把ALBEDO改成 `vec3(UV.x, UV.y, 0.0)`，你会看到什么？

你试了吗？

你会看到球体变成了这样：左下角是黑色（0,0,0），往右逐渐变红，往上逐渐变绿，右上角是黄色（1,1,0）。

<!-- [配图：球体UV可视化效果，左下黑→右红→上绿→右上黄] -->

这就是UV坐标的可视化。`UV.x` 控制红色通道，`UV.y` 控制绿色通道。每个像素的颜色直接反映了它的UV位置。

这一章，我们来彻底搞懂UV到底是什么。

## UV是什么

UV是一张"地图"。

想象你有一张世界地图贴纸，想把它贴到地球仪上。你怎么贴？

你需要知道：
- 地图上的每个点，对应地球仪上的哪个位置

UV就是这套对应关系。

在3D模型上，每个顶点都有一个UV坐标：
- U = 横向位置（类似经度）
- V = 纵向位置（类似纬度）

UV的范围是 0.0 到 1.0：
- (0, 0) = 贴图的左下角
- (1, 0) = 贴图的右下角
- (0, 1) = 贴图的左上角
- (1, 1) = 贴图的右上角

注意：V轴向上，不是向下。这和很多图片编辑软件的坐标系不同（那些是Y轴向下），但在shader世界里，V就是向上的。

## 为什么叫UV，不叫XY

好问题。

因为在3D空间里，X、Y、Z已经被用了（表示物体的空间位置）。为了避免混淆，贴图坐标就用了U和V。

你可以这么记：
- XYZ = 物体在哪（世界坐标）
- UV = 贴图在哪（纹理坐标）

两个坐标系是独立的。一个物体可以在世界任何位置，但它的UV可以是任意值——这取决于建模师怎么展开UV。

## 实战：用UV做渐变

### 水平渐变

```glsl
shader_type spatial;

void fragment() {
    ALBEDO = vec3(UV.x, 0.0, 0.0);
}
```

左边是黑色（UV.x=0），右边是红色（UV.x=1）。

### 对角线渐变

```glsl
shader_type spatial;

void fragment() {
    float t = (UV.x + UV.y) * 0.5;  // 取平均
    ALBEDO = vec3(t, t, t);
}
```

左下角暗，右上角亮，形成对角线的灰度渐变。

### 圆形渐变

```glsl
shader_type spatial;

void fragment() {
    float d = distance(UV, vec2(0.5, 0.5));  // 到中心的距离
    ALBEDO = vec3(d);
}
```

中心是黑色（距离=0），边缘是白色（距离最大约0.707）。

但这有个问题：如果模型UV展开不是正方形区域，圆会变成椭圆。

为什么？因为UV是0-1的范围，不管模型实际形状如何。一个长条形的UV展开，UV.x走0到1对应长边，UV.y走0到1对应短边，比例就不一样了。

### 修正圆形渐变（考虑UV比例）

如果你的模型UV展开比例特殊，可以用UV的导数来修正：

```glsl
shader_type spatial;

void fragment() {
    vec2 centered_uv = UV - 0.5;
    float d = length(centered_uv);
    ALBEDO = vec3(d);
}
```

对于大多数标准模型（球体、立方体），UV展开比较均匀，圆形渐变直接用就行。如果效果变形，说明UV展开需要考虑模型具体情况调整。

这个问题更深入的解决方案涉及UV展开的知识，这里先不展开。

## 让渐变动起来

静态渐变看腻了，让它动。

### 波浪渐变

```glsl
shader_type spatial;

void fragment() {
    float wave = sin(UV.x * 10.0 + TIME * 3.0) * 0.5 + 0.5;
    ALBEDO = vec3(wave, wave, wave);
}
```

`UV.x * 10.0`：把0-1的范围重复10次，产生10个波峰。
`+ TIME * 3.0`：让波浪随时间移动。
`* 0.5 + 0.5`：把-1到1映射到0到1。

你会看到黑白相间的条纹在模型上流动。

### 呼吸灯效果

```glsl
shader_type spatial;

void fragment() {
    float d = distance(UV, vec2(0.5, 0.5));
    float pulse = sin(TIME * 3.0) * 0.5 + 0.5;  // 0到1波动
    float ring = smoothstep(0.3, 0.35, d) - smoothstep(0.35, 0.4, d);  // 环形
    ALBEDO = vec3(ring * pulse);
}
```

这是一个固定位置的环形，亮度随时间闪烁。

`smoothstep(a, b, x)` 的作用：
- x < a：返回0
- x > b：返回1
- a < x < b：平滑过渡

两个smoothstep相减，就得到一个环形区域。

## UV扭曲：让贴图"融化"

UV不只是用来读取贴图的，你还可以修改UV，实现扭曲效果。

```glsl
shader_type spatial;

void fragment() {
    vec2 distorted_uv = UV;
    distorted_uv.x += sin(UV.y * 10.0 + TIME) * 0.1;  // 水平扭曲
    distorted_uv.y += cos(UV.x * 10.0 + TIME) * 0.1;  // 垂直扭曲

    ALBEDO = texture(TEXTURE, distorted_uv).rgb;
}
```

`distorted_uv.x += sin(UV.y * 10.0 + TIME) * 0.1` 的意思：
- 根据UV.y的位置，给UV.x加一个偏移
- 不同的y位置，偏移量不同（sin函数）
- 偏移量随时间变化（+TIME）
- 偏移幅度是0.1（不会太夸张）

效果：贴图像水波纹一样扭曲流动。

<!-- [配图：UV扭曲效果，贴图呈现波浪扭曲] -->

## 常见陷阱

### 陷阱1：UV超出0-1范围

```glsl
vec2 bad_uv = UV * 2.0;  // 0到2
ALBEDO = texture(TEXTURE, bad_uv).rgb;
```

这会怎样？

默认情况下，UV超出0-1会重复采样（类似于平铺）。所以0.5会采样贴图中间，1.5也会采样贴图中间。

如果你不想要重复，可以在贴图导入设置里改成"Clamp"模式，超出范围就采样边缘颜色。

### 陷阱2：UV扭曲导致边缘撕裂

当你扭曲UV时，贴图边缘的像素可能会"越界"，采样到贴图外的区域。

解决方法：
1. 用Clamp模式
2. 或者手动限制UV范围：
   ```glsl
   distorted_uv = clamp(distorted_uv, 0.0, 1.0);
   ```

### 陷阱3：忘记UV是0-1

很多人直觉上以为UV是像素坐标（比如0-512），结果写出来效果不对。

记住：UV永远是0-1，和贴图实际分辨率无关。如果你需要像素坐标，可以用：
```glsl
vec2 pixel_coord = UV / TEXTURE_PIXEL_SIZE;
```

## 这篇你学到了什么

1. UV是贴图坐标，U=横，V=纵，范围0-1
2. `UV.x` 控制横向，`UV.y` 控制纵向
3. `distance(UV, center)` 可以计算到某点的距离
4. `smoothstep(a, b, x)` 可以创建平滑过渡
5. 修改UV可以实现扭曲效果
6. UV超出0-1会重复采样（默认行为）

## 下篇预告

现在你能用UV做出各种渐变和扭曲效果了。但有一个问题：这些效果都是基于模型自身的UV，如果你想让效果跟随世界位置变化怎么办？

下一章，我们来聊聊坐标系——Model、World、View、Screen，搞清楚shader里的空间变换。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲坐标系，这是理解3D shader的关键。
