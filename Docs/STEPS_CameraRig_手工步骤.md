# 独立相机 Rig — 手工操作步骤

> 本文档对应 `PLAN_CameraRig.md` 方案 B 的落地执行。  
> 生成时间：2026-11-23 08:35

---

## 零、已经完成的部分（不用重做）

| 项目                | 状态                                                                                                                      |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| 蓝图目录备份            | ✅ `Saved/_backup_camera_rig/`（8 个资产完整）                                                                                  |
| `BP_CameraRig` 蓝图 | ✅ 已创建，路径 `/Game/ThirdPerson/Blueprints/BP_CameraRig`                                                                    |
| `SpringArm` 组件    | ✅ `TargetArmLength=400`、`bUsePawnControlRotation=false`、`bDoCollisionTest=true`、`ProbeSize=12`、`bEnableCameraLag=false` |
| `Camera` 组件       | ✅ 挂在 SpringArm 下，`FieldOfView=90`                                                                                       |
| 变量 `TargetActor`  | ✅ 类型 **Pawn**（对象引用）                                                                                                     |
| 变量 `TargetAnchor` | ✅ 类型 **SceneComponent**（对象引用）                                                                                           |
| 变量 `FollowSpeed`  | ✅ 类型 float，默认值 **6.0**                                                                                                  |
| 函数 `SetTarget`    | ✅ 已创建，参数 `NewTarget`(Pawn) 和 `bSnap`(bool) 已建好，**函数体是空的**                                                               |

组件层级现在是：`DefaultSceneRoot → SpringArm → Camera`

> 说明：`TargetActor` 特意做成 Pawn 而不是 Actor，因为 Tick 里要调 `Get Control Rotation`，  
> 那是 Pawn 的函数。做成 Pawn 后可以直接连，省一个 Cast。

---

## 一、完成 `SetTarget` 函数体

打开 `BP_CameraRig` → 左侧 Functions 里双击 `SetTarget`。

现在图里只有一个 `SetTarget` 入口节点，带两个输出引脚：`NewTarget`(Pawn) 和 `bSnap`(bool)。

### 1.1 先存目标

```
FunctionEntry.exec ──► Set TargetActor ──► (下一步)
FunctionEntry.NewTarget ──► Set TargetActor 的 TargetActor 输入
```

操作：

1. 从入口节点的白色执行引脚拖出，松开，搜 `Set TargetActor`，选 **Set TargetActor**
2. 把入口的 `NewTarget` 蓝色引脚拖到 setter 的 `TargetActor` 引脚上

### 1.2 判空 NewTarget

```
Set TargetActor.exec ──► Branch
Is Valid (NewTarget) ──► Branch 的 Condition
```

操作：

1. 从 `Set TargetActor` 的执行输出拖出，搜 `Branch`，选 **Branch**
2. 搜 `Is Valid`，选**纯函数版**（蓝色，没有执行引脚的那个，图标是问号）
3. 把 `NewTarget` 拖到 `Is Valid` 的 `Object` 输入
4. `Is Valid` 的返回值拖到 `Branch` 的 `Condition`

> 关键：一定要选**纯函数版**的 `Is Valid`（只有输入输出、没有白色执行引脚）。  
> 带执行引脚的那个会让后面的连线结构变复杂。

### 1.3 找锚点组件

从 `Branch` 的 **True** 执行引脚拖出：

```
Branch.True ──► Get Components by Tag
                  self           = NewTarget
                  Component Class = Scene Component
                  Tag            = CameraAnchor
```

操作：

1. 搜 `Get Components by Tag`
2. `self` 引脚接 `NewTarget`
3. `Component Class` 下拉选 **Scene Component**
4. `Tag` 填 `CameraAnchor`（注意大小写要完全一致）

返回值是一个**数组**（粉色/青色引脚，标着 "Actor Component 对象引用数组"）。

### 1.4 数组非空才取第 0 个

```
Get Components by Tag.ReturnValue ──► Length ──► > 0 ──► Branch
Branch.True ──► Get (a copy) 索引 0 ──► Set TargetAnchor
```

操作：

1. 从数组引脚拖出，搜 `Length`，选数组版的 **Length**（返回 int）
2. 从 Length 结果拖出，搜 `>`，B 引脚填 `0`
3. 从比较结果拖出，搜 `Branch`
4. 从上一步 `Branch`（判空 NewTarget 的那个）的 **True** 拖到新 Branch 的执行输入  
   —— 注意是执行线串联：`Branch(判空).True → Branch(判长度).execute`
5. 从数组引脚拖出，搜 `Get`，选 **Get (a copy)**，索引填 `0`
6. 从 `Branch(判长度).True` 拖出，搜 `Set TargetAnchor`
7. 把 `Get` 的输出接到 `Set TargetAnchor` 的 `TargetAnchor` 引脚

### 1.5 需要吸附时立刻定位

```
Set TargetAnchor.exec ──► Branch
bSnap ──► Branch.Condition
Branch.True ──► Branch
                Is Valid(TargetAnchor) ──► Condition
                True ──► Set Actor Location
                           New Location = TargetAnchor → Get World Location
```

操作：

1. 从 `Set TargetAnchor` 执行输出拖出，搜 `Branch`
2. 把入口节点的 `bSnap` 引脚拖到 `Condition`
3. `Branch.True` 拖出，再搜一个 `Branch`
4. 搜 `Is Valid`（纯函数），`Object` 接 `Get TargetAnchor`
5. 返回值接 `Condition`
6. `True` 拖出，搜 `Set Actor Location`
7. **另起一个** `Get World Location` 节点，把 `Get Target Anchor` 接到**这个节点自己的输入引脚**（中文界面显示为「目标」，英文为「Self」）
8. `Get World Location` 的**输出**（紫色向量）接到 `Set Actor Location` 的 `New Location`

`Set Actor Location` 的 `Sweep` 和 `Teleport` 保持默认 `false` 即可。

> ⚠️ **这一步最常见的错法**：把 `Get Target Anchor` 直接拖到 `Set Actor Location`
> 的 `Target` 引脚上。连不上是**正常的**——那个引脚要的是 **Actor**（"要对哪个
> Actor 设置位置"），而 `TargetAnchor` 是 **SceneComponent**，类型不兼容。
>
> 正确的数据流是**两级**，必须经过 `Get World Location` 做类型转换：
>
> ```
> Get Target Anchor (SceneComponent 蓝色)
>     └──► Get World Location 的「目标」引脚
>              └──► Return Value (Vector 紫色) ──► Set Actor Location 的 New Location
> ```
>
> `Set Actor Location` 的 `Target` 引脚**保持默认（self）**，不要连任何东西。
> 这个函数的意思是"把 Rig 自己移动到某个坐标"，Target 指的是 Rig 自身，
> 所以不接就是 self。

> **如果还是连不上**：说明建的不是 SceneComponent 版的位置节点。
> 备选路径：`Get Target Anchor` → 搜 `Get World Transform` → 输出接 `Break Transform`
> → 取 `Location` 接 `New Location`。效果等价，只是多一个节点。

### 1.6 编译

点 **Compile**，应该零错误零警告。

> 此时函数图里有 4 个 Branch。别嫌乱，位置拖开一点就好读：  
> 判空 NewTarget → 判数组长度 → 判 bSnap → 判 TargetAnchor 有效。



---

## 二、写 Event Tick

打开 `BP_CameraRig` → `EventGraph`（图里已有 BeginPlay / ActorBeginOverlap / Tick 三个事件，后两个没用，可以留着）。

```
Event Tick.exec ──► Branch (Is Valid(TargetActor))
                     True ──► Branch (Is Valid(TargetAnchor))
                               True ──► Sequence
                                          Then 0: SpringArm → Set World Rotation
                                          Then 1: Set Actor Location
```

### 2.1 两层判空

1. 从 `Event Tick` 的执行输出拖出，搜 `Branch`
2. 搜 `Is Valid`（纯函数），`Object` 接 `Get TargetActor`
3. 返回值接 `Condition`
4. `True` 拖出，再搜一个 `Branch`
5. 再搜 `Is Valid`，`Object` 接 `Get TargetAnchor`
6. 返回值接第二个 `Condition`

### 2.2 加 Sequence 分两路

1. 从第二个 `Branch` 的 `True` 拖出，搜 `Sequence`（中文界面叫 **序列**）
2. 把 `Then 0` 和 `Then 1` 都用上

> 为什么用 Sequence 而不是直接串：旋转和位置是两个互不依赖的操作，  
> 分开接读起来清楚，以后要在中间插逻辑也方便。

### 2.3 Then 0：让相机朝向跟随鼠标

```
Sequence.Then 0 ──► Set World Rotation
                      Target        = SpringArm（组件）
                      New Rotation  = TargetActor → Get Control Rotation
```

操作：

1. 从 `Then 0` 拖出，搜 `Set World Rotation`
2. `Target` 引脚拖到组件面板里的 `SpringArm` 上（会生成 Get SpringArm）
3. 搜 `Get Control Rotation`，`Target` 接 `Get TargetActor`
4. 输出接到 `New Rotation`

> **这一步是整个方案的核心**。`Get Control Rotation` 读的是 Pawn 的 Controller  
> 的 ControlRotation，也就是玩家的鼠标输入。角色的 `Aim` 函数往这里写，  
> Rig 从这里读，所以手感和你现在完全一致。
>
> 如果搜不到 `Get Control Rotation`，检查 `TargetActor` 变量类型是不是 **Pawn**  
> （不是 Actor）。Actor 上没有这个函数。

### 2.4 Then 1：让相机位置追过去

```
Sequence.Then 1 ──► Set Actor Location
                      Target        = self（默认）
                      New Location  = VInterpTo 的返回值

VInterpTo:
  Current      = Get Actor Location（self）
  Target       = TargetAnchor → Get World Location
  Delta Time   = Event Tick 的 DeltaSeconds
  Interp Speed = Get FollowSpeed
```

操作：

1. 从 `Then 1` 拖出，搜 `Set Actor Location`
2. 搜 `VInterpTo`，选 **Vector 版本**（引脚是黄色/紫色向量）
3. `Current` 接 `Get Actor Location`（Target 留空 = self）
4. `Target` 接 `Get World Location`，其 `Target` 接 `Get TargetAnchor`
5. `Delta Time` 接 `Event Tick` 的 `DeltaSeconds`
6. `Interp Speed` 接 `Get FollowSpeed`
7. `VInterpTo` 的输出接 `Set Actor Location` 的 `New Location`

### 2.5 编译 + 保存

Compile → 零错误 → Ctrl+S。

---

## 三、改角色父类 `BP_ThirdPersonCharacter`

> 三个子类（warrior / witch / auxiliary）不用动，组件是继承的。

### 3.1 删两个相机组件

1. 打开 `BP_ThirdPersonCharacter`
2. Components 面板里选中 **`SpringArmComponent_0`** → 右键 → **Delete**（或按 Delete 键）
3. 再选中 **`CameraComponent_0`** → Delete

> 删除顺序无所谓，但两个都要删。删完 Components 面板应该只剩：  
> `CollisionCylinder`（根）、`CharacterMesh0`、`CharMoveComp`、`Arrow`

### 3.2 加相机锚点

1. Components 面板 → **Add** → 搜 `Scene` → 选 **Scene Component**
2. 重命名为 **`CameraAnchor`**
3. 把它拖到 **`CollisionCylinder`** 上（成为胶囊的子组件，和其他组件同级）
4. Details 面板 → Transform → **Location 设为 `(0, 0, 0)`**
5. Details 面板 → **Tags** → 找到 **Component Tags** → 点 `+` → 填 **`CameraAnchor`**

> **注意区分两个 Tags**：Details 面板里有两个叫 Tags 的地方，  
> 一个在 Actor 级别（Attributes 区块），一个在 Component 级别。  
> 要填的是 **Component Tags**（在组件 Details 的 Tags 分类下，有个数组）。  
> 如果填错到 Actor Tags，Rig 的 `Get Components by Tag` 会找不到。

> Location 用 `(0,0,0)` 是为了完全复刻现在的观感——原来 SpringArm 就挂在  
> 胶囊中心。想抬高视角改成 `(0,0,50)` 或 `(0,0,64)` 都行，改一次三个角色一起变。

### 3.3 编译

点 Compile。

**可能出现的情况**：如果报 "节点引用了已删除的组件"，说明蓝图里有节点连着旧相机。  
实测角色的 EventGraph 只有 Move / Aim / Jump / StopJumping，没有相机引用，  
**大概率不会有这个错**。真出现了就把那个报错节点删掉。

编译通过后，三个子类应该自动继承新组件。可以打开 `warrior` 看一眼，  
Components 面板里应该有 `CameraAnchor`、没有 SpringArm 和 Camera。

---

## 四、改 `BP_ThirdPersonPlayerController`

### 4.1 关掉自动相机管理（关键）

1. 打开 `BP_ThirdPersonPlayerController`
2. 点工具栏 **Class Defaults**
3. Details 面板搜索框输入 `Auto Manage`
4. 把 **Auto Manage Active Camera Target** 的勾**取消**

> 这个不关，`Possess` 每次都会把 ViewTarget 抢回角色，Rig 立刻失效。  
> 表现是"Rig 建了但完全没效果"，很容易误判成 Rig 写错了。

### 4.2 加变量

1. 新建变量 **`CameraRig`**
2. 类型选 **BP_CameraRig**（对象引用，不是类引用）

### 4.3 加函数 `SetCameraTarget`

1. My Blueprint → Functions → `+` → 命名 **`SetCameraTarget`**
2. 加两个输入参数：
   - `NewTarget`，类型 **Pawn**
   - `bSnap`，类型 **Boolean**

> 参数用 Pawn 不用 Actor，是为了能直接传给 Rig 的 `SetTarget`（它要 Pawn），  
> 省一个 Cast。switcher 传进来的 `charactorarray[i]` 是 BP_ThirdPersonCharacter，  
> 本身就是 Pawn 子类，类型兼容。

1. 函数体：

```
FunctionEntry.exec ──► Branch (Is Valid(CameraRig))
                        False ──► Spawn Actor from Class
                                    Class = BP_CameraRig
                                    Spawn Transform = Get Actor Transform（self）
                                  ──► Set CameraRig
                                  ──► Set View Target with Blend
                                        Target = self
                                        New View Target = CameraRig 的返回值
                                        Blend Time = 0.0
                                  ──► 汇合到下面
                        True  ──► 汇合到下面

汇合点 ──► CameraRig → Set Target
              NewTarget = 入口的 NewTarget
              bSnap     = 入口的 bSnap
```

操作要点：

1. 从入口执行引脚拖出，搜 `Branch`
2. 搜 `Is Valid`（纯函数），`Object` 接 `Get CameraRig`
3. `False` 拖出，搜 `Spawn Actor from Class`，Class 选 `BP_CameraRig`
   - `Spawn Transform` 接 `Get Actor Transform`（self）
4. `Spawn` 的执行输出 → 搜 `Set CameraRig`，值接 Spawn 的返回值
5. `Set CameraRig` 执行输出 → 搜 `Set View Target with Blend`
   - `Target` 接 self（或留空）
   - `New View Target` 接 `Get CameraRig`
   - `Blend Time` 填 `0.0`
6. **汇合点那个 `Set Target` 是函数调用，不是变量赋值。** 它调用的就是你在
   `BP_CameraRig` 里刚写好的 `SetTarget` 函数（把新目标转发给 Rig 的那一步）。
   创建方式：从 `Get CameraRig` 的**蓝色输出引脚**拖出来，搜索框输入 `SetTarget`，
   在 **BP_CameraRig** 分类下选 **Set Target**。
7. 把 `Set View Target` 的执行输出、以及第 2 步 `Branch` 的 `True`，**两个都连到**
   它的执行输入（一个执行输入引脚允许接多条线，谁先到谁触发）
8. `Set Target` 的三个引脚：
   - `Target` —— 自动接上 `CameraRig`（从哪个对象拖出来就接哪个，不用手动连）
   - `NewTarget` —— 接入口节点的 `NewTarget`
   - `bSnap` —— 接入口节点的 `bSnap`

> **为什么要多这一步**：`SetCameraTarget` 的职责是"确保 Rig 存在，然后把目标交给它"。
> 前半段（Spawn / Set View Target）只在第一次执行，后半段（`Set Target`）每次都要执行。
> 所以两条执行流最后都汇到同一个 `Set Target` 上。
>
> **搜索时注意**：不要用全局搜索框搜 "Set Target"，会搜到一堆同名的引擎节点
> （比如各种组件的 Set Target）。一定要**从 `Get CameraRig` 的引脚拖出**再搜，
> 这样列表里只会出现 `BP_CameraRig` 自己的成员，`SetTarget` 就在里面。

> **懒创建是这里的关键设计**：不管 switcher 和 PC 谁先 BeginPlay，  
> 第一次调 `SetCameraTarget` 时 Rig 一定会被创建出来，不存在时序竞争。  
> 所以 PC 自己的 BeginPlay 里**不需要**加任何东西。

### 4.4 编译 + 保存

---

## 五、改 `controllerswitcher`

### 5.1 BeginPlay 里的那一处

找到 BeginPlay 链尾的 **`Set View Target with Blend`** 节点（`BlendTime = 0.0`，`NewViewTarget = GetArrayItem_4`，也就是 `charactorarray[0]`）。

1. **右键 → Delete** 删掉这个节点
2. 从它前面那个 `Possess` 节点的执行输出拖出，搜 `Cast to BP_ThirdPersonPlayerController`
   - `Object` 接 `Get Player Controller(0)`
3. `Cast` 的 `As BP Third Person Player Controller` 输出 → 搜 `Set Camera Target`
   - `Target` 接 Cast 的输出
   - `NewTarget` 接 `GetArrayItem_4`（charactorarray[0]）
   - `bSnap` 勾选 **true**

> `bSnap = true` 表示开局直接吸附到角色身后，不从原点飞过去。

### 5.2 Tab 分支里的那一处

找到 Tab 逻辑末尾的 **`Set View Target with Blend`**（`NewViewTarget = GetArrayItem_5`，即 `charactorarray[controlindex]`）。

同样处理：

1. 删掉 `Set View Target with Blend`
2. 从 `Possess` 的执行输出拖出 → `Cast to BP_ThirdPersonPlayerController`
   - `Object` 接 `Get Player Controller(0)`
3. `Set Camera Target`：`NewTarget` 接 `GetArrayItem_5`，`bSnap` **不勾**（false）

> `bSnap = false` 才会走插值，产生"镜头飞过去"的效果。

### 5.3 检查

**两个 `Set View Target with Blend` 必须都删掉。** 留一个，那次切换就会硬切回角色相机，  
表现为"有时候平滑有时候跳"。

### 5.4 编译 + 保存

---

## 六、验证

按这个顺序测，出问题能快速定位到哪一步：


| # | 操作 | 预期 | 不对的话查 |
|---|---|---|---|
| 1 | PIE 启动 | 视角在角色 0 身后，不在原点 | switcher 的 `bSnap=true` 是否接上；Rig 的 SetTarget 是否被调用 |
| 2 | 移动鼠标 | 视角能上下左右转 | Tick 的 `Set World Rotation` 是否接 `Get Control Rotation`；SpringArm 的 `bUsePawnControlRotation` 是否为 false |
| 3 | WASD / 空格 | 移动跳跃正常 | 与相机无关，是输入链路 |
| 4 | 按 Tab | 镜头**平滑飞过去** | switcher 里 `Set View Target with Blend` 是否删干净；`bSnap` 是否误勾 |
| 5 | 切完立刻按 WASD | 能移动 | Possess 是否正常（相机无关） |
| 6 | 连续 Tab 循环 | 每次都有飞行感 | 同上 |
| 7 | 背靠墙 | 相机不穿墙 | SpringArm 的 `bDoCollisionTest` |
| 8 | 看 Output Log | 没有 `Accessed None` | Tick 里两层 `Is Valid` 是否都接了 |

### 常见故障对照

| 现象 | 原因 |
|---|---|
| 开局黑屏 / 视角在原点 | `bAutoManageActiveCameraTarget` 关了但 `SetCameraTarget` 没被调用 |
| 视角完全不能转 | SpringArm 的 `bUsePawnControlRotation` 还是 true（Rig 是 Actor，读不到），或 Tick 的旋转没接 |
| 切换还是硬切 | switcher 里还有 `Set View Target with Blend` 没删 |
| 相机不动，钉在一个位置 | Tick 没接执行线，或 `TargetAnchor` 一直是空（Component Tags 填错位置） |
| 相机抖动 | Tick 里重复调用了 `Get Actor Location`（应该只用一次，用 bind/变量存） |
| 镜头有延迟、发飘 | 误把旋转也做了插值（旋转必须直接 Set，不插值） |

---

## 七、调参

全部在 `BP_CameraRig` 上：

| 参数 | 位置 | 当前值 | 调整方向 |
|---|---|---|---|
| `FollowSpeed` | 变量默认值 | 6.0 | 调大 → 跟得更紧、切换更快（建议 4~12） |
| `TargetArmLength` | SpringArm | 400 | 调大 → 相机更远（300~500） |
| `SocketOffset` | SpringArm | (0,0,0) | Y 填 40~70 变成过肩视角 |
| `CameraAnchor` Location | 角色父类 | (0,0,0) | Z 填 50~64 抬高视角 |
| `FieldOfView` | Camera | 90 | 80~100 |

调完记得 Ctrl+S。

---

## 八、出问题怎么回滚

备份在 `Saved/_backup_camera_rig/`，里面有改动前的 8 个资产。

```bash
# 全部还原（会覆盖你所有的改动）
cp -r "E:/Epic/UE_5.8/project/threepersonswitch/Saved/_backup_camera_rig/." \
      "E:/Epic/UE_5.8/project/threepersonswitch/Content/ThirdPerson/Blueprints/"
```

只还原某一个：

```bash
cp "E:/Epic/UE_5.8/project/threepersonswitch/Saved/_backup_camera_rig/BP_ThirdPersonCharacter.uasset" \
   "E:/Epic/UE_5.8/project/threepersonswitch/Content/ThirdPerson/Blueprints/"
```

`BP_CameraRig` 是新建的，直接删资产即可。

> 还原前**先关编辑器**，否则内存里的版本会在关闭时把磁盘覆盖回去。

---

## 九、这次不解决的问题

**换下的角色仍然会冻结、跳起来切人会悬空。**

原因是 `Possess` 转移后旧角色没有 Controller，`CharacterMovement` 的
`bRunPhysicsWithNoController` 默认 false，物理停摆。跟相机无关。

修法（等相机稳定后再做）：
1. 把旧角色交给 `aicontrollerarray[oldindex]`
2. 三个角色（或父类）的 CharacterMovement 勾上 **Run Physics with No Controller**
