# Noise纹理：给Shader加点"噪"

第5篇末尾我们写了一个溶解效果，里面用了一张 `dissolve_noise` 纹理。这张纹理就是 Noise——看起来随机、但又有一定连续性的灰度图。

没有它，溶解边缘会像刀切一样整齐，非常假。有了 Noise，边缘才有那种自然侵蚀的感觉。

但 Noise 的用途远不止溶解。水面波纹、云层、火焰、地形高度——这些"看起来随机但又不是完全随机"的效果，都需要 Noise。

## 纯数学太"完美"

先用纯数学做个水面效果：

```glsl
shader_type spatial;

void fragment() {
    float wave = sin(UV.x * 20.0 + TIME * 2.0) * 0.5 + 0.5;
    ALBEDO = vec3(wave * 0.3, wave * 0.5, 0.8);
}
```

效果：规则的蓝白条纹来回摆动，像理发店门口的旋转灯。

真实的水面不是这样的。真实水面有无数大小不一的波纹，方向不同、速度不同、互相干涉。用 sin 叠加可以做到，但代码会很复杂：

```glsl
// 想做出"真实"水面，要叠加十几层 sin...
float wave = sin(UV.x * 10.0 + TIME) * 0.5
           + sin(UV.y * 15.0 + TIME * 1.3) * 0.3
           + sin((UV.x + UV.y) * 8.0 + TIME * 0.7) * 0.4
           + ... // 还要再加很多层
```

手写这种"伪随机"的叠加既累又容易重复。Noise 纹理就是来解决这个问题的。

## NoiseTexture2D：不用手写算法

好消息：i3dAct 内置了 `NoiseTexture2D`，你不需要自己实现 Perlin Noise 或者 Simplex Noise。

### 创建 Noise 纹理

在 i3dAct 编辑器里：
1. 在 FileSystem 面板右键 → 新建资源
2. 选择 `NoiseTexture2D`
3. 在 Inspector 里配置参数

常用参数：
- **Width/Height**：纹理尺寸，默认 512x512。越大细节越多，但内存占用也越大。
- **Noise**：选择噪声类型，一般用 `FastNoiseLite`
- **Seamless**：是否无缝平铺。做循环效果（比如滚动的云）要打开
- **As Normal Map**：是否生成法线贴图。做凹凸表面时有用

### 传入 Shader

和第4篇的纹理一样，用 `uniform sampler2D` 传入：

```glsl
shader_type spatial;

uniform sampler2D noise_tex;

void fragment() {
    float n = texture(noise_tex, UV).r;
    ALBEDO = vec3(n);
}
```

把 `NoiseTexture2D` 拖到材质的 `noise_tex` 属性上，模型就会变成一张灰度噪声图。

### 让 Noise 动起来

静态的 noise 没什么用。让 UV 随时间偏移，noise 就流动起来了：

```glsl
shader_type spatial;

uniform sampler2D noise_tex;
uniform float flow_speed = 0.1;

void fragment() {
    vec2 moving_uv = UV + vec2(TIME * flow_speed, 0.0);
    float n = texture(noise_tex, moving_uv).r;
    ALBEDO = vec3(n);
}
```

效果：噪声纹理像云一样缓缓向一侧飘动。调大 `flow_speed`，飘得更快。

## 单层 Noise 太平淡

上面的代码只采样了一层 noise。效果看起来...还是有点像云，但太均匀了，缺少细节。

真实自然界的东西都有"层次"：大波浪上叠着小波浪，小波浪上叠着更小的涟漪。这个规律叫**分形**（Fractal），在 shader 里用**FBM**（Fractal Brownian Motion，分形布朗运动）来模拟。

### FBM 原理

叠加多层 noise，每层频率翻倍、强度减半：

```
第1层：频率 1，强度 1.0
第2层：频率 2，强度 0.5
第3层：频率 4，强度 0.25
第4层：频率 8，强度 0.125
...
```

数学上：

```glsl
float fbm(vec2 uv) {
    float value = 0.0;
    float amplitude = 0.5;
    float frequency = 1.0;

    for (int i = 0; i < 5; i++) {
        value += amplitude * noise(uv * frequency);
        amplitude *= 0.5;
        frequency *= 2.0;
    }

    return value;
}
```

但在 i3dAct 里，你不需要自己写这个循环。`NoiseTexture2D` 的 `FastNoiseLite` 本身就支持分形参数：

- **Fractal Type**：选 `FBm`
- **Fractal Octaves**：层数，建议 4-6
- **Fractal Lacunarity**：频率倍增比，默认 2.0（每层频率翻倍）
- **Fractal Gain**：强度衰减比，默认 0.5（每层强度减半）

调这些参数，NoiseTexture2D 生成出来的纹理就已经是 FBM 叠加后的结果了。你直接采样就行，shader 代码不用变。

### 什么时候自己写 FBM，什么时候用预设

| 情况 | 做法 |
|------|------|
| 静态或简单流动效果 | 用 NoiseTexture2D 的 FBM 预设 |
| 需要动态调整层数/频率 | 在 shader 里写 FBM 循环，采样同一纹理的不同缩放 |
| 需要 3D noise（随时间变化体积） | 自己写，或者多张 2D noise 叠加模拟 |

## 实战：水面效果

用 noise 做一片流动的、有层次感的水面：

```glsl
shader_type spatial;

uniform sampler2D noise_tex;
uniform vec3 water_deep : source_color = vec3(0.0, 0.1, 0.3);
uniform vec3 water_shallow : source_color = vec3(0.0, 0.4, 0.6);
uniform float speed = 0.05;

void fragment() {
    // 两层 noise，方向不同、速度不同
    vec2 uv1 = UV + vec2(TIME * speed, TIME * speed * 0.3);
    vec2 uv2 = UV * 2.0 + vec2(-TIME * speed * 0.7, TIME * speed * 0.5);

    float n1 = texture(noise_tex, uv1).r;
    float n2 = texture(noise_tex, uv2).r;

    // 叠加两层，产生更复杂的图案
    float combined = n1 * 0.6 + n2 * 0.4;

    // 用 smoothstep 控制深浅水色的分布
    float t = smoothstep(0.3, 0.7, combined);
    vec3 color = mix(water_deep, water_shallow, t);

    // 加一点高光感
    float highlight = smoothstep(0.6, 0.8, combined) * 0.5;

    ALBEDO = color + highlight;
    ROUGHNESS = 0.1;  // 水面比较光滑
}
```

这里的关键：
1. 两层 noise，不同缩放（UV vs UV * 2.0）、不同方向、不同速度
2. `smoothstep` + `mix` 把噪声值映射成颜色（第5篇学的三剑客）
3. 高光区域让水面有波光粼粼的感觉

调参方向：
- `speed` 控制流动快慢
- `uv2` 的 `* 2.0` 控制细节密度，想更细碎就调更大
- `water_deep` / `water_shallow` 改水色，脏水、绿水、蓝水都能调

## 实战：云层效果

云层和水面的区别在于：云有"形状"——有些地方浓密，有些地方稀薄，甚至完全没有。

```glsl
shader_type spatial;

uniform sampler2D cloud_noise;
uniform float cloud_density : hint_range(0.0, 1.0) = 0.5;
uniform float cloud_speed : hint_range(0.0, 0.2) = 0.02;
uniform vec3 sky_color : source_color = vec3(0.3, 0.5, 0.8);
uniform vec3 cloud_color : source_color = vec3(1.0, 1.0, 1.0);

void fragment() {
    // 三层 noise 叠加，模拟不同尺度的云团
    vec2 uv_base = UV + vec2(TIME * cloud_speed, 0.0);

    float n1 = texture(cloud_noise, uv_base).r;           // 大云团
    float n2 = texture(cloud_noise, uv_base * 2.5 + 0.3).r; // 中云团
    float n3 = texture(cloud_noise, uv_base * 6.0 + 0.7).r; // 边缘细节

    // FBM 风格的叠加
    float cloud_shape = n1 * 0.5 + n2 * 0.3 + n3 * 0.2;

    // 用密度阈值控制云的"厚实"程度
    // density 越低，需要的 cloud_shape 越高才能显示云
    float threshold = 1.0 - cloud_density;
    float cloud_mask = smoothstep(threshold, threshold + 0.3, cloud_shape);

    // 云内部也要有明暗变化
    float cloud_shade = smoothstep(threshold, threshold + 0.5, cloud_shape);
    vec3 final_cloud = mix(cloud_color * 0.7, cloud_color, cloud_shade);

    vec3 final_color = mix(sky_color, final_cloud, cloud_mask);

    ALBEDO = final_color;
}
```

这个效果用在天空盒或者 2D 背景上很合适。如果用在 3D 平面上，把 `ALPHA` 也控制一下，让云薄的地方透明：

```glsl
ALPHA = cloud_mask * 0.9;  // 云薄的地方半透明
```

然后把材质的 Transparent 模式打开。

## 实战：火焰/热浪扭曲

用 noise 做热浪扭曲效果，比纯 sin 自然得多：

```glsl
shader_type spatial;

uniform sampler2D main_tex;
uniform sampler2D noise_tex;
uniform float distortion_strength : hint_range(0.0, 0.1) = 0.02;
uniform float rise_speed = 0.3;

void fragment() {
    // noise 向上流动，模拟热空气上升
    vec2 noise_uv = UV * 2.0 + vec2(0.0, -TIME * rise_speed);
    float n = texture(noise_tex, noise_uv).r;

    // 把 noise 值映射成 UV 偏移量
    vec2 distortion = vec2(
        (n - 0.5) * distortion_strength,
        (texture(noise_tex, noise_uv + vec2(0.5)).r - 0.5) * distortion_strength * 0.5
    );

    // 越往上扭曲越强（热空气上升）
    distortion *= UV.y;

    vec2 final_uv = UV + distortion;
    ALBEDO = texture(main_tex, final_uv).rgb;
}
```

这个 shader 挂到一个火焰或者熔岩的材质上，后面的纹理就会像透过热空气看东西一样扭曲晃动。

`distortion *= UV.y` 这行让底部（UV.y 接近 0）扭曲弱，顶部（UV.y 接近 1）扭曲强，符合火焰上热下冷的直觉。

## 用 noise 做随机分布

noise 还有一个妙用：**伪随机分布**。比如你想在模型表面随机撒一些斑点：

```glsl
shader_type spatial;

uniform sampler2D noise_tex;
uniform float spot_threshold : hint_range(0.0, 1.0) = 0.7;
uniform vec3 spot_color : source_color = vec3(1.0, 0.2, 0.2);

void fragment() {
    vec3 base = texture(TEXTURE, UV).rgb;
    float n = texture(noise_tex, UV * 3.0).r;

    // noise 值大于阈值的区域显示斑点
    float spot = step(spot_threshold, n);

    ALBEDO = mix(base, spot_color, spot * 0.8);
}
```

效果：模型表面随机分布着红色斑点，像豹纹、锈迹、或者病变组织。调 `spot_threshold` 可以控制斑点的稀疏程度——阈值越高，斑点越少。

这里用了 `step` 而不是 `smoothstep`，因为斑点需要锐利边缘。如果你想要斑点边缘柔和，把 `step` 换成 `smoothstep(spot_threshold, spot_threshold + 0.1, n)`。

## 常见陷阱

### 陷阱1：Noise 纹理分辨率不够

如果你的模型很大，512x512 的 noise 纹理拉伸后会看到明显的像素块。解决方案：
- 提高纹理尺寸到 1024 或 2048
- 在 shader 里对 UV 取 `fract()`，让 noise 平铺重复
- 使用 Seamless 模式避免平铺接缝

```glsl
// 让 noise 无缝平铺
vec2 tiled_uv = fract(UV * 4.0);
float n = texture(noise_tex, tiled_uv).r;
```

### 陷阱2： too many texture samples

前面云层例子里采样了 3 次 noise。如果你再叠两层，每次 fragment 要采样 5 次。在低端设备上可能成为瓶颈。

优化方案：
- 预先用 NoiseTexture2D 的 FBM 参数生成多层叠加的纹理，shader 只采样一次
- 或者把多层 noise 烘焙到一张图的 R、G、B 通道里，一次采样拿到三层

```glsl
// 一张纹理，三个通道存三层 noise
vec3 noise_layers = texture(noise_packed, UV).rgb;
float n1 = noise_layers.r;
float n2 = noise_layers.g;
float n3 = noise_layers.b;
```

### 陷阱3：noise 流动方向太单一

如果只往一个方向偏移 UV，长时间看会很单调。让 noise 有"盘旋"感：

```glsl
// 单纯的平移
vec2 uv = UV + vec2(TIME * speed, 0.0);

// 更有机的流动：平移 + 轻微旋转
float angle = TIME * 0.05;
vec2 centered = UV - vec2(0.5);
vec2 rotated = vec2(
    centered.x * cos(angle) - centered.y * sin(angle),
    centered.x * sin(angle) + centered.y * cos(angle)
);
vec2 uv = rotated + vec2(0.5) + vec2(TIME * speed, 0.0);
```

### 陷阱4：忘记 normalize

noise 纹理的返回值范围理论上是 0-1，但经过 FBM 叠加后可能超出这个范围。用之前最好 clamp 一下：

```glsl
float n = texture(noise_tex, UV).r;
n = clamp(n, 0.0, 1.0);  // 确保在有效范围内
```

## 这篇你学到了什么

1. **为什么需要 Noise**：纯数学太规律，noise 让效果有自然的随机感
2. **NoiseTexture2D**：i3dAct 内置，Inspector 里配置，不用手写算法
3. **让 noise 流动**：UV + TIME * speed 实现动画
4. **FBM 叠加**：多层 noise（不同频率/速度/方向）叠加出丰富细节
5. **实战应用**：水面、云层、热浪扭曲、随机斑点
6. **优化**：预计算 FBM、通道打包、控制采样次数

## Noise 用法速查

| 需求 | 做法 |
|------|------|
| 让贴图流动更自然 | 用 noise 偏移 UV，代替固定的 sin/cos |
| 做云/雾/烟 | 多层 noise + 阈值控制密度 + smoothstep 柔化 |
| 做水面 | 2-3层 noise，不同方向速度，mix 深浅色 |
| 做热浪扭曲 | noise 驱动 UV 偏移，强度随高度增加 |
| 随机分布斑点 | noise + step/smoothstep 做遮罩 |
| 溶解/侵蚀边缘 | noise 作为阈值输入给 smoothstep |
| 节省采样次数 | 预计算 FBM 或用 RGB 通道打包多层 noise |

## 下篇预告

你现在掌握了 UV、坐标系、纹理采样、数学函数、Noise 纹理。单个知识都学会了，但怎么把它们串起来做一个完整的效果？

下一篇是**综合实战**：用前面学过的所有东西，做一个完整的**溶解特效**（Dissolve）。不是第5篇那种简易版，而是带边缘发光、颜色变化、方向控制、可以做成技能特效的完整版本。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们把前面学的全部串起来，做一个完整的溶解特效，内容更实战。
