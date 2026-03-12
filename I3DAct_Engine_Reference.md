# i3dAct 引擎参考文档

> 本文档描述 i3dAct 引擎与 Godot 4.4 的命名对应关系，供 AI 助手快速理解引擎特性。

---

## 概述

i3dAct 引擎基于 Godot 4.4 架构，对节点名称、函数名称及文件扩展名进行了调整以提高语义清晰度。主要修改原则：

1. **Node → Item**: 所有 `Node` 相关命名改为 `Item`
2. **Bulk**: 物理相关节点使用 `Bulk` 后缀替代 `Body`
3. **Render**: 可视化实例使用 `Render` 后缀替代 `Instance`
4. **下划线移除**: 函数名移除下划线前缀，采用 PascalCase
5. **信号重命名**: `ready` 信号改为 `start`

### 文件扩展名映射

| 文件类型 | Godot 4.4 | i3dAct | 说明 |
|----------|-----------|--------|------|
| 脚本文件 | `.gd` | `.s3` | GDScript 脚本 |
| 场景文件 | `.tscn` | `.iscn` | 文本场景文件 |

**示例：**
```
# Godot 4.4 项目结构
player/
├── player.gd          # 脚本文件
└── player.tscn        # 场景文件

# i3dAct 项目结构
player/
├── player.s3          # 脚本文件
└── player.iscn        # 场景文件
```

---

## 一、节点名称映射表

### 1.1 基础节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| Node | Item | 所有节点的基类 |
| Node3D | Item3D | 所有 3D 对象的基类 |
| Node3DGizmo | Item3DGizmo | 3D 节点可视化辅助 |
| GraphNode | GraphItem | 图节点 |

### 1.2 物理碰撞节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| CollisionObject3D | ColliderObjectBase | 可碰撞物体基类，用于检测物理交互 |
| CollisionShape3D | ColliderShape | 向父级提供碰撞形状 |
| CollisionPolygon3D | ColliderPolygon | 向父级提供加厚多边形碰撞形状 |
| PhysicsBody3D | PhysicsBulkBase | 物理实体抽象基类（静态/动态/角色） |
| StaticBody3D | StaticBulk | 静态碰撞体，用于固定不可移动的物体 |
| RigidBody3D | RigidBulk | 刚体物理模拟，支持重力、碰撞和物理力交互 |
| CharacterBody3D | CharacterBulk | 角色控制器，专用于角色移动和碰撞响应 |
| AnimatableBody3D | AnimatableBulk | 动画驱动的物理体，不受外力影响 |
| SoftBody3D | SoftBulk | 软体物理模拟（布料、橡胶等） |
| PhysicalBone3D | PhysicalBone | 物理骨骼，使骨骼对物理做出反应 |
| VehicleBody3D | VehicleBulk | 车辆物理模拟（悬挂、轮胎、转向） |
| VehicleWheel3D | WheelCollider | 车轮物理模拟 |
| Area3D | AreaTrigger | 3D 空间区域，检测物体进入/退出 |

### 1.3 可视化渲染节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| VisualInstance3D | VisualRender | 可视化 3D 节点基类 |
| GeometryInstance3D | GeometryRender | 基于几何图形的视觉实例基类 |
| MeshInstance3D | MeshRender | 3D 模型实例化节点 |
| MultiMeshInstance3D | MultiMeshRender | 多实例渲染（植被等大量重复物体） |
| ImporterMeshInstance3D | ImporterMeshRender | 优化导入网格的实例化节点 |
| SpriteBase3D | SpriteBase | 3D 环境中的 2D 精灵基类 |
| Sprite3D | Sprite | 3D 环境中显示 2D 纹理 |
| AnimatedSprite3D | AniSprite | 3D 世界中的 2D 精灵动画 |
| Label3D | TextRender | 3D 空间文本显示 |
| Decal | DecalActor | 贴花投影 |

### 1.4 粒子系统节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| CPUParticles3D | ParticlesCPU | CPU 驱动轻量级粒子系统 |
| GPUParticles3D | ParticlesGPU | GPU 加速粒子系统（火焰、烟雾等） |
| GPUParticlesAttractor3D | ParticleAspiratorGPUBase | 粒子吸引力场基类 |
| GPUParticlesAttractorBox3D | ParticleAspiratorBoxGPU | 盒子形状粒子吸引器 |
| GPUParticlesAttractorSphere3D | ParticleAspiratorSphereGPU | 球形状粒子吸引器 |
| GPUParticlesAttractorVectorField3D | ParticleAspiratorVectorFieldGPU | 向量场粒子运动控制 |
| GPUParticlesCollision3D | ParticleColliderGPUBase | 粒子碰撞检测基类 |
| GPUParticlesCollisionBox3D | ParticleColliderBoxGPU | 盒子形状粒子碰撞 |
| GPUParticlesCollisionSphere3D | ParticleColliderSphereGPU | 球形状粒子碰撞 |
| GPUParticlesCollisionSDF3D | ParticleColliderSDFGPU | 带符号距离场粒子碰撞 |
| GPUParticlesCollisionHeightField3D | ParticleColliderHeightFieldGPU | 高度图粒子碰撞 |

### 1.5 光照节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| Light3D | Light | 光源基类 |
| DirectionalLight3D | DirectionalLight | 平行光（太阳光） |
| OmniLight3D | PointLight | 点光源 |
| SpotLight3D | SpotLight | 聚光灯 |
| FogVolume | FogVolume | 体积雾 |

### 1.6 全局光照与环境节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| LightmapGI | Lightmass | 全局光照烘焙系统 |
| LightmapProbe | LightProbe | 动态物体照明探针 |
| VoxelGI | VoxelGI | 体素全局光照 |
| ReflectionProbe | ReflectionProbe | 环境反射贴图捕获 |
| OccluderInstance3D | OcclusionRender | 遮挡剔除实例 |

### 1.7 CSG 构造实体几何节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| CSGShape3D | ICSGShapeBase | CSG 形状基类 |
| CSGPrimitive3D | ICSGPrimitiveBase | CSG 图元基类 |
| CSGBox3D | ICSGBox | CSG 盒子 |
| CSGCylinder3D | ICSGCylinder | CSG 圆柱 |
| CSGMesh3D | ICSGMesh | CSG 网格 |
| CSGPolygon3D | ICSGPolygon | CSG 多边形 |
| CSGSphere3D | ICSGSphere | CSG 球体 |
| CSGTorus3D | ICSGTorus | CSG 圆环 |
| CSGCombiner3D | ICSGCombiner | CSG 组合器 |

### 1.8 摄像机节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| Camera3D | Camera | 3D 摄像机（透视/正交投影） |
| SpringArm3D | SpringArm | 弹簧臂摄像机（第三人称跟随） |
| XRCamera3D | XRCamera | VR/AR 摄像机 |

### 1.9 音频节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| AudioListener3D | AudioListener | 3D 音频监听器 |
| AudioStreamPlayer3D | AudioPlayer | 3D 空间音频播放器 |

### 1.10 骨骼与动画节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| Skeleton3D | SkeletonMesh | 骨骼层级管理 |
| SkeletonModifier3D | SkeletonRevamp | 骨骼修改器基类 |
| LookAtModifier3D | LookAtRevamp | 骨骼看向目标修改器 |
| RetargetModifier3D | RetargetRevamp | 骨骼重定向修改器 |
| SkeletonIK3D | (隐藏) | 骨骼 IK（已隐藏） |
| PhysicalBoneSimulator3D | PhysicalBoneSimu | 物理骨骼模拟器 |
| SpringBoneSimulator3D | SpringBoneSimu | 弹簧骨骼模拟器（头发、配饰） |
| SpringBoneCollision3D | SpringBoneCollider | 弹簧骨骼碰撞体基类 |
| SpringBoneCollisionCapsule3D | SpringBoneColliderCapsule | 弹簧骨骼胶囊体碰撞 |
| SpringBoneCollisionPlane3D | SpringBoneColliderPlane | 弹簧骨骼平面碰撞 |
| SpringBoneCollisionSphere3D | SpringBoneColliderSphere | 弹簧骨骼球体碰撞 |
| BoneAttachment3D | BoneSlot | 骨骼绑定节点（武器挂载） |
| RootMotionView | RootMotionView | 根运动视图（编辑器） |

### 1.11 物理关节节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| Joint3D | JointBase | 物理关节基类 |
| PinJoint3D | PinJoint | 单点连接关节 |
| HingeJoint3D | HingeJoint | 铰链关节（旋转限制） |
| SliderJoint3D | SliderJoint | 滑动关节（轴向移动限制） |
| ConeTwistJoint3D | ConeTwistJoint | 球窝关节 |
| Generic6DOFJoint3D | Generic6DOFJoint | 六自由度关节 |

### 1.12 导航与路径节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| NavigationRegion3D | NavigationRegion | 导航网格区域 |
| NavigationLink3D | NavigationLink | 导航链接（跨越障碍） |
| NavigationObstacle3D | NavigationObstacle | 动态导航障碍物 |
| Path3D | Route | 路径定义 |
| PathFollow3D | RouteMove | 路径跟随 |
| Marker3D | SpaceMarker | 空间标记点（路径点） |
| RayCast3D | RayCast | 射线检测 |
| ShapeCast3D | ShapeCast | 形状投射检测 |
| GridMap | GridMap | 网格地图编辑器 |

### 1.13 可见性检测节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| VisibleOnScreenNotifier3D | OnScreenVisibleSignal | 可见性检测信号 |
| VisibleOnScreenEnabler3D | OnScreenVisibleEnabler | 可见性自动启用/禁用 |

### 1.14 XR (VR/AR) 节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| XROrigin3D | XROrigin | VR/AR 场景原点 |
| XRNode3D | XRNode | XR 位置跟踪节点 |
| XRController3D | XRController | XR 手柄控制器 |
| XRAnchor3D | XRAnchor | AR 空间锚点 |
| XRBodyModifier3D | XRBulkRevamp | XR 身体追踪（隐藏） |
| XRHandModifier3D | XRHandRevamp | XR 手部追踪（隐藏） |
| XRFaceModifier3D | XRFaceRevamp | XR 面部追踪（隐藏） |
| OpenXRHand | OpenXRHand | OpenXR 手部跟踪 |
| OpenXRVisibilityMask | OpenXRVisibilityMask | OpenXR 可见性遮罩 |
| OpenXRCompositionLayer | OpenXRCompositionLayer | OpenXR 合成层基类 |
| OpenXRCompositionLayerQuad | OpenXRCompositionLayerQuad | OpenXR 四边形合成层 |
| OpenXRCompositionLayerCylinder | OpenXRCompositionLayerCylinder | OpenXR 圆柱合成层 |
| OpenXRCompositionLayerEquirect | OpenXRCompositionLayerEquirect | OpenXR 球形合成层 |

### 1.15 远程控制节点

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| RemoteTransform3D | RemoteTransform | 远程变换同步控制 |

---

## 二、脚本函数映射表

### 2.1 生命周期函数

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `_ready()` | `iStart()` | 节点首次加入场景树时调用，用于初始化 |
| `_process(delta: float)` | `Update()` | 每帧调用，用于非物理逻辑（UI、输入响应） |
| `_physics_process(delta: float)` | `FixedUpdate()` | 固定时间间隔调用（默认60Hz），用于物理计算 |
| `_input(event: InputEvent)` | `OnInput()` | 接收全局输入事件 |
| `_enter_tree()` | `OnEnable()` | 节点加入场景树时调用（早于 iStart） |
| `_exit_tree()` | `OnDisable()` | 节点从场景树移除时调用，用于资源释放 |

### 2.2 进程控制函数

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `set_process()` | `set_update()` | 启用/禁用 Update 调用 |
| `set_process_input()` | `set_on_input()` | 启用/禁用 OnInput 调用 |
| `set_physics_process()` | `set_fixedupdate()` | 启用/禁用 FixedUpdate 调用 |

### 2.3 Timer 函数

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `timer.start()` | `timer.begin()` | 启动计时器 |

### 2.4 SceneTree 方法

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `get_first_node_in_group()` | `get_first_item_in_group()` | 获取组中第一个节点 |
| `get_nodes_in_group()` | `get_items_in_group()` | 获取组中所有节点 |
| `get_node_count_in_group()` | `get_item_count_in_group()` | 获取组中节点数量 |
| `get_node_name()` | `get_item_name()` | 获取节点名称 |
| `get_node_count()` | `get_item_count()` | 获取节点数量 |

### 2.5 AnimationNodeBlendTree 方法

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `get_node_name()` | `get_item_name()` | 获取节点名称 |

### 2.6 SkeletonIK3D 方法

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `start(one_time: bool = false)` | `begin(one_time: bool = false)` | 启动 IK |

### 2.7 Variant 常量

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `NODE_TYPE` | `ITEM_PATH` | 节点类型常量 |

### 2.8 信号重命名

| Godot 4.4 | i3dAct | 描述 |
|-----------|--------|------|
| `ready` | `start` | 节点准备就绪信号 |

---

## 三、命名规则总结

### 3.1 节点命名规则

1. **Node → Item**: 所有包含 `Node` 的命名替换为 `Item`
   - `Node3D` → `Item3D`
   - `get_node_name()` → `get_item_name()`

2. **Body → Bulk**: 物理相关节点使用 `Bulk` 后缀
   - `RigidBody3D` → `RigidBulk`
   - `StaticBody3D` → `StaticBulk`
   - `CharacterBody3D` → `CharacterBulk`

3. **Instance → Render**: 可视化实例使用 `Render` 后缀
   - `MeshInstance3D` → `MeshRender`
   - `VisualInstance3D` → `VisualRender`

4. **CSG 前缀**: CSG 节点添加 `I` 前缀
   - `CSGShape3D` → `ICSGShapeBase`
   - `CSGBox3D` → `ICSGBox`

5. **粒子吸引器**: `Attractor` → `Aspirator`
   - `GPUParticlesAttractor3D` → `ParticleAspiratorGPUBase`

6. **修改器**: `Modifier` → `Revamp`
   - `SkeletonModifier3D` → `SkeletonRevamp`
   - `LookAtModifier3D` → `LookAtRevamp`

7. **骨骼模拟**: `Simulator` → `Simu`
   - `PhysicalBoneSimulator3D` → `PhysicalBoneSimu`
   - `SpringBoneSimulator3D` → `SpringBoneSimu`

### 3.2 函数命名规则

1. **移除下划线前缀**: 生命周期函数采用 PascalCase
   - `_ready()` → `iStart()` (特别注意：首字母小写 i)
   - `_process()` → `Update()`
   - `_physics_process()` → `FixedUpdate()`

2. **node → item**: 方法名中的 `node` 替换为 `item`
   - `get_node_name()` → `get_item_name()`

3. **start → begin**: Timer 相关
   - `timer.start()` → `timer.begin()`

### 3.3 特殊情况

| 原名称 | 新名称 | 说明 |
|--------|--------|------|
| `SkeletonIK3D` | 隐藏 | 该节点在 i3dAct 中隐藏 |
| `XRBodyModifier3D` | 隐藏 | 该节点在 i3dAct 中隐藏 |
| `XRHandModifier3D` | 隐藏 | 该节点在 i3dAct 中隐藏 |
| `XRFaceModifier3D` | 隐藏 | 该节点在 i3dAct 中隐藏 |

---

## 四、与其他引擎对比参考

| 功能 | i3dAct | Godot 4.4 | Unity | Unreal |
|------|--------|-----------|-------|--------|
| 3D 基类 | Item3D | Node3D | GameObject | Actor |
| 刚体 | RigidBulk | RigidBody3D | Rigidbody | RigidBody |
| 角色控制 | CharacterBulk | CharacterBody3D | CharacterController | CharacterMovementComponent |
| 静态碰撞 | StaticBulk | StaticBody3D | Static Rigidbody | StaticMeshComponent |
| 触发区域 | AreaTrigger | Area3D | Trigger Collider | Trigger Volume |
| 摄像机 | Camera | Camera3D | Camera | CameraComponent |
| 网格实例 | MeshRender | MeshInstance3D | MeshRenderer | StaticMeshComponent |
| 骨骼网格 | SkeletonMesh | Skeleton3D | SkinnedMeshRenderer | SkeletalMeshComponent |
| 平行光 | DirectionalLight | DirectionalLight3D | Directional Light | DirectionalLight |
| 点光源 | PointLight | OmniLight3D | Point Light | PointLight |
| 聚光灯 | SpotLight | SpotLight3D | Spot Light | SpotLight |
| 导航区域 | NavigationRegion | NavigationRegion3D | NavMesh | NavMeshBoundsVolume |
| 粒子(GPU) | ParticlesGPU | GPUParticles3D | VFX Graph | Niagara |
| 粒子(CPU) | ParticlesCPU | CPUParticles3D | Particle System | Cascade |

---

## 五、使用示例

### 5.1 生命周期函数示例

```gdscript
# Godot 4.4
func _ready():
    print("节点初始化")

func _process(delta):
    print("每帧更新")

func _physics_process(delta):
    print("物理更新")

# i3dAct
func iStart():
    print("节点初始化")

func Update():
    print("每帧更新")

func FixedUpdate():
    print("物理更新")
```

### 5.2 节点类型示例

```gdscript
# Godot 4.4
@export var body: RigidBody3D
@export var camera: Camera3D
@export var mesh: MeshInstance3D

# i3dAct
@export var body: RigidBulk
@export var camera: Camera
@export var mesh: MeshRender
```

### 5.3 场景树操作示例

```gdscript
# Godot 4.4
var nodes = get_tree().get_nodes_in_group("enemies")
var first = get_tree().get_first_node_in_group("enemies")

# i3dAct
var items = get_tree().get_items_in_group("enemies")
var first = get_tree().get_first_item_in_group("enemies")
```

---

## 六、版本信息

- **基准版本**: Godot 4.4
- **目标引擎**: i3dAct
- **文档版本**: 1.0
- **生成日期**: 2026-03-12

---

> 提示：本参考文档基于引擎节点和函数修改对照表生成，如有更新请同步修改此文档。
