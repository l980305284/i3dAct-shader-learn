# 让剖切动起来：五种切割动画，三行代码一种

第8篇到第11篇，我们从平面切割一路做到 SDF 布尔运算。切面能变色了，切边能发光了，形状也能组合了——但全是静止的。

这篇让它们动起来。核心思路一句话：**把原本固定的参数，换成随时间变化的量**。框架代码完全不变，变的只是 `clip` 那一行。

## 准备工作

以下所有效果都基于第11篇的框架。为了节省篇幅，这里只贴变化的部分。完整框架回顾一下关键结构：

```glsl
void fragment() {
    float clip = /* 这里每篇换 */;

    vec3 base = texture(TEXTURE, UV).rgb;
    bool is_back = dot(normalize(NORMAL), normalize(VIEW)) > 0.0;

    if (clip < 0.0) {
        if (is_back) { ALBEDO = section_color; }
        else { discard; }
    } else {
        ALBEDO = base;
        float edge = smoothstep(edge_width, 0.0, clip);
        EMISSION = edge_color * edge * 2.0;
    }
}
```

后面每个效果，你只需要替换 `clip` 那一行。

## 效果一：上下扫描

最直接的动画——让切面沿 Y 轴上下移动。`sin(TIME)` 在 -1 到 1 之间振荡，乘上幅度就是往复扫描：

```glsl
// 静态版（第8篇）：
// float clip = dot(world_pos, normalize(plane_normal)) - plane_distance;

// 动态版：
float scan_dist = sin(TIME * 0.5) * 3.0;  // 3秒一个周期，幅度±3
float clip = dot(world_pos, normalize(plane_normal)) - scan_dist;
```

`TIME * 0.5` 控制速度，数字越小越慢。`* 3.0` 控制扫描范围，单位是世界空间距离。

想让扫描只走一趟不回头？用脚本驱动更合适，本篇最后一节会讲。

## 效果二：旋转切割

平面方程不变，但让法线方向绕 Y 轴转起来：

```glsl
float angle = TIME * 0.3;
vec3 rot_normal = vec3(cos(angle), 0.0, sin(angle));
float clip = dot(world_pos, rot_normal) - 0.0;
```

`cos(angle)` 给 X 分量，`sin(angle)` 给 Z 分量，Y 保持 0——法线在 XZ 平面转圈。效果：切面像雷达一样绕着模型扫。

换旋转轴也很简单：
- 绕 X 轴：`vec3(0.0, cos(angle), sin(angle))`
- 倾斜轴：`vec3(cos(angle), 0.5, sin(angle))` — Y 分量非零就是倾斜的

## 效果三：球形脉冲

SDF 球体半径用 `sin` 做呼吸式缩放，像心脏跳动：

```glsl
float pulse = 1.5 + sin(TIME * 2.0) * 0.8;  // 半径在 0.7 ~ 2.3 之间
float clip = length(world_pos - sphere_center) - pulse;
```

`sin(TIME * 2.0)` 频率比扫描高一倍，节奏更快。`* 0.8` 是振幅——半径变化范围。

想让脉冲有"蓄力-爆发"的感觉？把 `sin` 换成 `abs(sin(...))`，或者用 `pow(sin(...), 4.0)` 让峰值更尖：

```glsl
float pulse = 1.5 + pow(abs(sin(TIME * 1.5)), 4.0) * 1.5;
```

效果：球体快速膨胀后缓慢收缩，像冲击波。

## 效果四：沿路径移动

让 SDF 球心沿圆形轨道绕圈，切割区域跟着球跑：

```glsl
vec3 orbit_center = vec3(cos(TIME * 0.7) * 1.5, sin(TIME * 0.5) * 1.0, sin(TIME * 0.7) * 1.5);
float clip = length(world_pos - orbit_center) - 0.8;
```

XY 不同频率 → 轨道不是正圆，而是利萨如曲线，运动轨迹更自然。

换成直线路径更简单——球从左滑到右：

```glsl
float t = fract(TIME * 0.2);  // 0→1 循环
vec3 sliding_center = mix(vec3(-2.0, 0.0, 0.0), vec3(2.0, 0.0, 0.0), t);
float clip = length(world_pos - sliding_center) - 0.8;
```

`fract` 取小数部分，让 `t` 在 0 到 1 之间循环。`mix` 在起点和终点之间线性插值。效果：球从左到右，到右边后瞬间跳回左边重来。

## 效果五：多形状联合

把效果三和效果四组合，两个球各自运动，取并集——两个钻头同时挖：

```glsl
// 球1：脉冲
float r1 = 1.2 + sin(TIME * 2.0) * 0.5;
float d1 = length(world_pos - vec3(0.0, 1.0, 0.0)) - r1;

// 球2：绕圈
vec3 orbit = vec3(cos(TIME * 0.7) * 1.5, -1.0, sin(TIME * 0.7) * 1.5);
float d2 = length(world_pos - orbit) - 0.6;

float clip = min(d1, d2);  // 并集：两个球一起挖
```

换成差集，球2在球1的切割区域里再挖洞：

```glsl
float clip = max(-d2, d1);  // 球1挖出的区域，球2再掏空
```

这就是 SDF 布尔运算 + 动画的威力——**每个形状独立运动，组合方式随时换**。

## Shader 内动画 vs 脚本驱动

上面所有效果都用 `TIME` 驱动，适合**自动循环的视觉效果**——UI 扫描线、场景装饰、技能特效预览。

需要**响应游戏逻辑**时，用脚本传参。比如角色血量变化：

```s3
// 脚本里
func Update():
    var hp_ratio = current_hp / max_hp  # 0.0 ~ 1.0
    clip_material.set_shader_parameter("clip_progress", hp_ratio)
```

shader 里：

```glsl
uniform float clip_progress : hint_range(0.0, 1.0) = 0.5;
// 用 clip_progress 控制切割位置
float clip = dot(world_pos, normalize(plane_normal)) - (clip_progress * 4.0 - 2.0);
```

或者更丝滑——用 tween 驱动：

```s3
func play_scan():
    var tween = create_tween().set_loops()
    tween.tween_method(
        func(v): clip_material.set_shader_parameter("clip_progress", v),
        0.0, 1.0, 3.0
    )
    tween.tween_method(
        func(v): clip_material.set_shader_parameter("clip_progress", v),
        1.0, 0.0, 3.0
    )
```

3 秒切过去，3 秒切回来，循环。

**选哪种？**

| 场景 | 用 TIME | 用脚本 |
|------|---------|--------|
| 自动循环动画 | ✅ | 也能，但没必要 |
| 响应玩家操作 | ❌ | ✅ |
| 需要精确控制节奏 | ❌ | ✅ (tween) |
| 多个材质同步 | 看情况 | ✅ (同一参数) |

## 这篇你学到了

- 五种切割动画：扫描、旋转、脉冲、路径、多形状联合
- 核心模式：**把参数换成 `TIME` 的表达式，框架不动**
- 多形状联合 = 每个形状独立运动 + 布尔运算
- `TIME` 适合自动循环，脚本传参适合响应游戏逻辑

## 下篇预告

系列二到这里就结束了。从第6篇 Noise 到第12篇动画，你手里的 SDF 切割框架已经能应付大部分项目需求。

下一篇我们进入系列三——**全屏 Shader**。不再切模型，而是对整个屏幕做后处理：颜色滤镜、屏幕扭曲、暗角特效……换一个维度玩 shader。

---

如果这篇文章对你有帮助，点个赞吧！你的支持是我持续更新下去最大的动力。下一章我们讲全屏 Shader 入门，东西完全不一样了。