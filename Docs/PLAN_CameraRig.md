# 方案 B：独立相机 Rig（Camera Rig）

> 目标：换人时镜头从旧角色平滑飞到新角色，而不是硬切；同时解决相机穿墙。
> 本文档只描述方案，**尚未对工程做任何修改**。

---

## 一、现状（实测数据，2026-11-23）

### 相机链路

```
角色 Aim 函数
  └ AddControllerYawInput / AddControllerPitchInput
      └ PlayerController.ControlRotation
          └ 角色 SpringArm (bUsePawnControlRotation = true)
              └ Camera
```

视角输入进入 **PlayerController 的 ControlRotation**，角色的 SpringArm 通过
`bUsePawnControlRotation` 读取它。这一点决定了 Rig 的可行性——Rig 只要读同一个
ControlRotation，视角就完全一致。

### 实测到的资产与参数

| 对象 | 现状 |
|---|---|
| `BP_ThirdPersonCharacter` | 父类 Character，组件：`CollisionCylinder`(根) → `SpringArmComponent_0` → `CameraComponent_0`，另有 `CharacterMesh0`、`CharMoveComp`、`Arrow` |
| SpringArm | `TargetArmLength = 400`，`SocketOffset = (0,0,0)`，`TargetOffset = (0,0,0)`，`bUsePawnControlRotation = true`，`bDoCollisionTest = true`，`ProbeSize = 12`，`bEnableCameraLag = false`，父组件 = CollisionCylinder |
| Camera | `FieldOfView = 90`，`bUsePawnControlRotation = false` |
| 角色旋转 | `bUseControllerRotationYaw = false`（靠 `bOrientRotationToMovement` 转身），`BaseEyeHeight = 64` |
| `warrior` / `witch` / `auxiliary` | 均继承 `BP_ThirdPersonCharacter`，无自有相机组件 |
| `BP_ThirdPersonPlayerController` | `bAutoManageActiveCameraTarget = true`（**关键**），`PlayerCameraManagerClass = None`，EventGraph 只有模板的 BeginPlay（DelayUntilNextTick → AddMappingContext ×2） |
| `controllerswitcher` | BeginPlay：抓 3 角色 → Delay 0.2 → EnableInput → Cast×3 存 AI → Set playercontrollerref → Possess[0] → SetViewTargetWithBlend[0]（BlendTime 0.0）<br>Tab：controlindex 自增（`>2` 归零）→ Possess[新] → SetViewTargetWithBlend[新]（BlendTime 0.0） |
| GameMode | `DefaultPawnClass = None`（不会额外 spawn 角色），`PlayerControllerClass = BP_ThirdPersonPlayerController_C` |
| 关卡 | `NewMap` 内正好 3 个角色：`warrior_C_3`、`witch_C_1`、`auxiliary_C_1` |

### 为什么现在是硬切

`SetViewTargetWithBlend` 的 `BlendTime` 传的是 **0.0**，等于没有混合。

更关键的是：`bAutoManageActiveCameraTarget = true` 时，`Possess` 会**立刻**把
ViewTarget 强制设成新 Pawn。所以就算把 BlendTime 改成 0.5，起点和终点是同一个
对象，等于没混——**这就是方案 A 必须先关掉那个开关的原因，也是必须走方案 B 的
根本原因：相机必须由我们自己的对象持有，才谈得上控制它怎么动。**

---

## 二、目标架构

```
PlayerController  ── ViewTarget 永远指向 ──►  BP_CameraRig（常驻 Actor）
                                                    │
                                              Root (SceneComponent)
                                                    │
                                              SpringArm (400, 碰撞开启)
                                                    │
                                              Camera (FOV 90)
                                                    │
                                    每帧读取 PC.ControlRotation 作为朝向
                                    每帧 VInterpTo 追当前角色的 CameraAnchor

三个角色（warrior / witch / auxiliary）
   └ CameraAnchor (SceneComponent, Tag="CameraAnchor", 位置 0,0,0)
        ↑ 只提供位置，不再持有相机
```

角色身上**不再有相机**，只留一个锚点。相机在 Rig 里，Rig 永远归 PC 所有。
换人时唯一变化的是 `Rig.TargetActor`，相机位置通过插值自然飞过去。

### 为什么这样能平滑

`VInterpTo` 是指数收敛：每帧朝目标移动固定比例。距离远时绝对速度大，距离近时
速度小，天然带 ease-out。所以 20 米的切换是"先快后慢划过去"，不是匀速平移。

---

## 三、改动清单（5 项）

### 1. 新建 `BP_CameraRig`（/Game/ThirdPerson/Blueprints/）

父类：`Actor`

组件层级：

| 组件 | 类型 | 关键设置 |
|---|---|---|
| `Root` | SceneComponent | 位置 (0,0,0) |
| `SpringArm` | SpringArmComponent | `TargetArmLength = 400`、`SocketOffset = (0,0,0)`、`TargetOffset = (0,0,0)`、`bDoCollisionTest = true`、`ProbeSize = 12`、**`bUsePawnControlRotation = false`**、`bEnableCameraLag = false` |
| `Camera` | CameraComponent | `FieldOfView = 90`，父 = SpringArm |

> `bUsePawnControlRotation` 必须为 **false**。Rig 是普通 Actor，没有 Controller，
> 打开这个开关它读不到 ControlRotation，相机会保持初始朝向、鼠标完全失效。
> 朝向由 Tick 里手动 `SetWorldRotation` 解决。

变量：

| 变量名 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `TargetActor` | Actor（对象引用） | None | 当前跟随的角色 |
| `TargetAnchor` | SceneComponent（对象引用） | None | 目标身上的锚点组件 |
| `FollowSpeed` | float | **6.0** | 位置插值速度，见参数表 |

函数 `SetTarget(NewTarget: Actor, bSnap: bool)`：

```
Set TargetActor = NewTarget
若 IsValid(NewTarget):
    Anchors = NewTarget -> Get Components by Tag(SceneComponent, "CameraAnchor")
    若 Anchors.Length > 0:
        Set TargetAnchor = Anchors[0]
    否则:
        Set TargetAnchor = 空
若 bSnap 且 IsValid(TargetAnchor):
    Set Actor Location(TargetAnchor -> Get World Location)     ← 直接吸附，不插值
```

Tick（`Event Tick`，接 `DeltaSeconds`）：

```
若 IsValid(TargetActor) 且 IsValid(TargetAnchor):
    PC = Get Player Controller(0)
    若 IsValid(PC):
        SpringArm -> Set World Rotation(PC -> Get Control Rotation)   ← 视角，不插值
    NewLoc = VInterpTo(Get Actor Location, TargetAnchor -> Get World Location,
                       DeltaSeconds, FollowSpeed)
    Set Actor Location(NewLoc, false, false)                          ← 位置，插值
```

两个要点：

- **旋转不做插值**。ControlRotation 是玩家实时鼠标输入，插值会产生拖影和延迟。
  只有位置插值。
- **先判空再操作**。开局第一帧 TargetActor 还是空，必须跳过，否则会报
  `Accessed None`。

### 2. 角色父类 `BP_ThirdPersonCharacter`

| 动作 | 内容 |
|---|---|
| 删除 | `SpringArmComponent_0`、`CameraComponent_0` |
| 新增 | `CameraAnchor`（SceneComponent），父 = `CollisionCylinder`，RelativeLocation `(0,0,0)`，Component Tags 加 `CameraAnchor` |

锚点 Z 用 0 是为了**完全复刻现在的观感**（原 SpringArm 就挂在胶囊中心）。
如果想抬高视角，改成 `(0,0,50)` 或 `(0,0,64)`（64 = BaseEyeHeight）即可，
三个角色会一起变，因为都在父类上。

> 三个子类不需要单独改，组件是继承的。

### 3. `BP_ThirdPersonPlayerController`

| 动作 | 内容 |
|---|---|
| 属性 | `bAutoManageActiveCameraTarget = false` |
| 变量 | `CameraRig`（BP_CameraRig 对象引用） |
| 函数 | `SetCameraTarget(NewTarget: Actor, bSnap: bool)` |
| BeginPlay | 末尾追加：`SetCameraTarget(空, false)` 或直接留空，由第一次调用懒创建 |

`SetCameraTarget` 实现：

```
若 IsValid(CameraRig) == false:
    Spawn Actor from Class(BP_CameraRig, 自身位置, 零旋转)
    Set CameraRig = 返回值
    Set View Target with Blend(CameraRig, BlendTime = 0.0)      ← 只在创建时设一次
若 IsValid(CameraRig):
    CameraRig -> Set Target(NewTarget, bSnap)
```

**懒创建是关键设计**：不管 switcher 和 PC 谁先 BeginPlay，第一次调
`SetCameraTarget` 时 Rig 一定会被创建出来，不存在时序竞争。

### 4. `controllerswitcher`（两处替换）

| 位置 | 原来 | 改为 |
|---|---|---|
| BeginPlay 末尾 | `SetViewTargetWithBlend(GetPlayerController(0), charactorarray[0], 0.0)` | `Cast to BP_ThirdPersonPlayerController(GetPlayerController(0))` → `SetCameraTarget(charactorarray[0], **true**)` |
| Tab 分支末尾 | `SetViewTargetWithBlend(GetPlayerController(0), charactorarray[controlindex], 0.0)` | 同上，但 `bSnap = **false**` |

**`SetViewTargetWithBlend` 必须删掉，不能保留。** 它会把 ViewTarget 设回角色，
Rig 立刻失效，等于白做。

`bSnap` 的区别：
- 开局传 `true` → Rig 直接出现在角色 0 身后，避免从原点飞过去
- 换人传 `false` → 走插值，产生飞行效果

### 5. 编译与保存

改完依次编译：`BP_CameraRig` → `BP_ThirdPersonCharacter`（连带三个子类）→
`BP_ThirdPersonPlayerController` → `controllerswitcher`，然后全局保存。

---

## 四、参数表（后续调手感就看这几个）

| 参数 | 位置 | 默认 | 调大的效果 | 建议范围 |
|---|---|---|---|---|
| `FollowSpeed` | BP_CameraRig | 6.0 | 相机跟得更紧、换人飞得更快 | 4 ~ 12 |
| `TargetArmLength` | BP_CameraRig.SpringArm | 400 | 相机离角色更远 | 300 ~ 500 |
| `SocketOffset` | BP_CameraRig.SpringArm | (0,0,0) | 相机左右/上下偏移，做"过肩" | Y 取 40~70 变过肩视角 |
| `CameraAnchor` 位置 | 角色父类 | (0,0,0) | 抬高/降低相机基点 | Z 取 0 ~ 64 |
| `FieldOfView` | BP_CameraRig.Camera | 90 | 视野更广 | 80 ~ 100 |

`FollowSpeed = 6` 时，跑动中相机滞后约 1 米（有跟随感），20 米切换约 0.7 秒到位。
觉得切换太慢就调到 8~10，觉得日常跟随太"黏"就降到 4~5。

---

## 五、已知风险与坑

**1. 删相机组件可能让蓝图编译失败**

如果角色蓝图或 AnimBP 里引用了 `FollowCamera` / `CameraComponent_0`，删组件后
会报 "节点引用了已删除的组件"。实测角色的 EventGraph 只有 Move / Aim / Jump /
StopJumping 四类调用，**没有相机引用**，风险低。但编译时仍要留意报错，出现就
把那个节点删掉。

**2. `bAutoManageActiveCameraTarget` 必须在 Possess 之前关掉**

否则每次 Possess 引擎都会把 ViewTarget 抢回去，表现是"Rig 建了但完全没效果"。
这个开关是 Class Defaults 级别的，改一次永久生效。

**3. 关掉之后开局会没有 ViewTarget**

如果 `SetCameraTarget` 一次都没被调用，PC 的 ViewTarget 是空的，画面会黑或者
停在 PC 原点。所以 switcher 的 BeginPlay **必须**保留那次调用（bSnap = true）。
懒创建机制保证了它一定会成功。

**4. SpringArm 的碰撞检测依赖 Rig 的实时位置**

Rig 位置是插值出来的，和角色有几百毫秒的偏差。角色贴墙瞬间，相机可能短暂
穿进墙里一点点再弹出来。这是插值方案的固有代价，`ProbeSize = 12` 能缓解。
如果明显，把 `FollowSpeed` 调高（滞后变小）。

**5. Tick 里必须判空**

`TargetActor` / `TargetAnchor` / PC 三个都要判。少判一个，开局第一帧就会刷
`Accessed None` 报错，虽然不影响运行，但会淹没真正的错误。

**6. 组件删除不可逆**

`remove_component` 删掉后没法"撤销"，只能手动加回来并重设参数。执行前会先
备份整个 `Content/ThirdPerson/Blueprints/` 到 `Saved/_backup_camera_rig/`。

---

## 六、执行顺序

顺序不能乱，否则中途编译会报引用错误：

1. **备份** `Content/ThirdPerson/Blueprints/` → `Saved/_backup_camera_rig/`
2. **建 `BP_CameraRig`**：组件 → 属性 → 变量 → `SetTarget` 函数 → Tick
3. **编译 `BP_CameraRig`**，确认无错
4. **改角色父类**：加 `CameraAnchor` → 删两个相机组件 → 编译（连带三个子类）
5. **改 PC**：关 AutoManage → 加 `CameraRig` 变量 → 写 `SetCameraTarget` → 编译
6. **改 switcher**：两处替换 → 编译
7. **全局保存**
8. **PIE 验证**

---

## 七、验证清单

PIE 后逐条确认：

- [ ] 开局视角在角色 0 身后，不是世界原点
- [ ] 鼠标移动，视角能上下左右转（说明 Rig 读到了 ControlRotation）
- [ ] WASD 移动正常，跳跃正常
- [ ] 按 Tab：镜头**平滑飞过去**，不是瞬间跳
- [ ] 切过去后能立刻移动（说明 Possess 正常）
- [ ] 连续按 Tab 循环三个角色，每次都有飞行感
- [ ] 走到墙边背对墙，相机不会穿进墙里（SpringArm 碰撞生效）
- [ ] Output Log 里没有 `Accessed None` 和 `Blueprint Runtime Error`

**如果视角不能转**：Rig 的 SpringArm 的 `bUsePawnControlRotation` 还是 true，
或者 Tick 里 `Set World Rotation` 没接上。

**如果切换还是硬切**：switcher 里的 `SetViewTargetWithBlend` 没删干净。

**如果开局黑屏**：`bAutoManageActiveCameraTarget` 关了但 `SetCameraTarget`
没被调用。

---

## 八、回滚方案

| 改动 | 回滚方式 |
|---|---|
| 新建 `BP_CameraRig` | 直接删除资产 |
| 角色父类删组件 | 从 `Saved/_backup_camera_rig/` 恢复 `BP_ThirdPersonCharacter.uasset` |
| PC 属性/变量/函数 | 同上，恢复 `BP_ThirdPersonPlayerController.uasset` |
| switcher 改图 | 同上，恢复 `controllerswitcher.uasset` |

备份是整个 Blueprints 目录，一条 `cp -r` 即可完整还原，风险可控。

---

## 九、工作量

| 步骤 | 预计 |
|---|---|
| 建 Rig + 组件 + 属性 | 10 min |
| Rig 图（SetTarget + Tick） | 30 min |
| 角色改造 | 10 min |
| PC 改造 | 20 min |
| switcher 改造 | 10 min |
| 编译 + PIE 联调 + 调参 | 30 min |
| **合计** | **约 1.5 ~ 2 小时** |

---

## 十、这个方案顺带解决的问题

- **相机穿墙**：SpringArm 的 `bDoCollisionTest` 在 Rig 里生效，这是方案 A
  做不到的（方案 A 的 Blend 是两点直线插值，中间会穿墙）。
- **后续运镜**：Boss 战要做的"入场推镜""受击震屏""处决特写"，全都在 Rig 上
  加，不用碰三个角色。
- **FOV 变化**：冲刺/受击时推拉 FOV，改 Rig 的 Camera 一处即可。

## 十一、这个方案不解决的问题

**换下的角色仍然会冻结。** 这是另一个根因：`Possess` 转移后旧角色没有
Controller，`CharacterMovement` 的 `bRunPhysicsWithNoController` 默认 false，
物理停摆，所以跳起来切人会悬在空中。

方案 B 只动相机，不碰这个。要修需要另外两步：把旧角色交给
`aicontrollerarray[oldindex]`，以及打开 `Run Physics with No Controller`。
这次不做，等相机稳定后再处理。
