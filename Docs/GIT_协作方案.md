# UE 项目 Git 协作方案

> 项目：`E:/Epic/UE_5.8/project/threepersonswitch`
> 团队：3 人
> 本文档只给方案和命令，**未对项目做任何操作**。

---

## 一、先看清体积构成（这决定了策略）

实测数据：

| 目录 | 体积 | 是否提交 | 说明 |
|---|---|---|---|
| `Content/` | **130 MB** | ✅ 提交 | 全部资产 |
| `Config/` | 19 KB | ✅ 提交 | GameMode、输入、渲染配置 |
| `Docs/` | 40 KB | ✅ 提交 | 设计文档 |
| `threepersonswitch.uproject` | 小 | ✅ 提交 | 项目入口 |
| `Intermediate/` | **123 MB** | ❌ 排除 | 编译中间产物，可重建 |
| `Saved/` | 11 MB | ❌ 排除 | 日志、自动保存、本地设置 |
| `DerivedDataCache/` | 1.9 MB | ❌ 排除 | 着色器缓存，可重建 |

**真正要推的是 130 MB，不是 265 MB。** 排除三个目录后体积直接减半，而且这三个目录
每天都在变，提交进去只会制造无意义的冲突。

### 130 MB 里是什么

| 内容 | 体积 | 占比 |
|---|---|---|
| `Content/Characters/Mannequins/`（引擎模板：Manny/Quinn 骨骼、网格、动画、贴图） | **126 MB** | 97% |
| `Content/ThirdPerson/`（你们的蓝图） | 664 KB | 0.5% |
| `Content/LevelPrototyping/` | 2.8 MB | 2% |
| `Content/Input/` | 340 KB | 0.3% |
| `Content/map/`（关卡） | 64 KB | — |
| `Content/__ExternalActors__/`（WorldPartition 外部 Actor） | 504 KB | — |

244 个 `.uasset` / `.umap`，最大单文件 20 MB（`T_Manny_02_BN.uasset`，一张贴图）。

**结论**：Mannequins 必须提交——同伴克隆下来没有它，三个角色全是空壳。
但它占了 97% 的体积，这也是后面 LFS 配额吃紧的根本原因。

---

## 二、托管平台怎么选

三个人的小项目，主要看两点：**国内访问速度** 和 **LFS 免费额度**。

| 平台 | 免费 LFS 额度 | 国内速度 | 适合度 |
|---|---|---|---|
| **Gitee（码云）** | 约 1 GB（需实名认证） | 快 | ⭐⭐⭐⭐ 推荐 |
| **GitHub** | 1 GB 存储 + 1 GB/月流量 | 慢（clone 130MB 可能十几分钟甚至失败） | ⭐⭐ 能用但痛苦 |
| **自建 Gitea / GitLab**（学校服务器） | 取决于硬盘 | 看网络 | ⭐⭐⭐ 有服务器就最优 |
| **GitHub 学生包** | LFS 额度扩容 | 慢 | ⭐⭐⭐ 有学生认证可以考虑 |

### 为什么 GitHub 免费版对这个项目偏紧

1 GB/月的 **流量** 额度，3 个人首次 clone 就是 130 MB × 3 = **390 MB**。
同伴重装一次环境、换台电脑再 clone 一次，当月额度就见底了。

Gitee 的 LFS 流量限制宽松一些，且国内 clone 130 MB 通常一两分钟完成，
GitHub 在国内可能反复超时重试。

> **如果三个人都在校园网内**，还有个更省事的方案：一个人做"主仓库"放在局域网
> 共享盘，另外两人直接 push/pull 到那台机器，完全不消耗云端额度。
> 代价是没有异地访问能力，且主仓库那台机器必须常开。

### 必须做的一件事：关掉 GitHub/Gitee 的"文件大小警告"

20 MB 的单文件在 LFS 里完全正常，但如果误用普通提交（没走 LFS），
超过 50 MB 会被警告、100 MB 会被拒绝。所以 **LFS 配置必须先于首次提交**（见第四节）。

---

## 三、仓库结构决策

### 要提交

```
threepersonswitch/
├── .gitignore          ← 新建
├── .gitattributes      ← 新建（LFS 规则，最关键）
├── threepersonswitch.uproject
├── Config/             ← 全部
├── Content/            ← 全部（含 Mannequins）
└── Docs/               ← 全部
```

### 不提交（写进 .gitignore）

```
Intermediate/          编译中间产物，每人本地生成
Saved/                 日志、自动保存、本地编辑器设置
DerivedDataCache/      着色器缓存
Binaries/              纯蓝图项目目前没有，但以后加 C++ 会有
.vs/                   Visual Studio 缓存
```

> **为什么 `Saved/` 一定要排除**：里面有 `Saved/Config/WindowsEditor/`，
> 存的是每台机器的窗口布局、最近打开的文件、自动保存的临时蓝图。
> 提交进去后，三个人每次开编辑器都会产生冲突，而且这些冲突毫无意义。

---

## 四、`.gitignore` 和 `.gitattributes`

### 4.1 `.gitignore` 完整内容

放在项目根目录：

```gitignore
# ===== UE5 生成目录 =====
Binaries/
DerivedDataCache/
Intermediate/
Saved/
Build/

# ===== IDE / 编译器 =====
.vs/
.vscode/
.idea/
*.sln
*.suo
*.VC.db
*.opensdf
*.sdf
*.opendb
*.pdb
*.obj
*.ipch

# ===== 系统文件 =====
.DS_Store
Thumbs.db
desktop.ini

# ===== UE 临时文件 =====
*.autosave
*.backup
*.tmp

# ===== 个人实验目录（多人协作时通常各自独立）=====
Content/Developers/
```

> `Content/Developers/` 目前是空的，但以后如果有人想做"试验性资产"，
> 放在这里就不会污染主干。这是 UE 的官方约定。

### 4.2 `.gitattributes` 完整内容

**这个文件是整个方案的核心**，它决定哪些文件走 LFS、哪些可以被锁定。

```gitattributes
# ===== UE 二进制资产：走 LFS，且可锁定 =====
*.uasset filter=lfs diff=lfs merge=lfs -text lockable
*.umap   filter=lfs diff=lfs merge=lfs -text lockable
*.ubulk  filter=lfs diff=lfs merge=lfs -text
*.uexp   filter=lfs diff=lfs merge=lfs -text
*.upk    filter=lfs diff=lfs merge=lfs -text

# ===== 图片 =====
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.tga filter=lfs diff=lfs merge=lfs -text
*.exr filter=lfs diff=lfs merge=lfs -text
*.hdr filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text

# ===== 音频 / 视频 =====
*.wav filter=lfs diff=lfs merge=lfs -text
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mov filter=lfs diff=lfs merge=lfs -text

# ===== 3D 模型 =====
*.fbx filter=lfs diff=lfs merge=lfs -text
*.obj filter=lfs diff=lfs merge=lfs -text
*.abc filter=lfs diff=lfs merge=lfs -text

# ===== 字体 =====
*.ttf filter=lfs diff=lfs merge=lfs -text
*.otf filter=lfs diff=lfs merge=lfs -text
```

### 关键点解释

**`lockable` 只加在 `.uasset` 和 `.umap` 上**，其他类型不需要。
它让这两个扩展名的文件支持 `git lfs lock`，也就是"占用"机制——
这是解决"两人同改一个蓝图必丢一人工作"的唯一有效手段。

**`-text` 不能省**。不加的话 Git 会把二进制当文本处理，可能做换行符转换，
导致资产损坏（打开报"文件已损坏"）。这是 UE 项目最常见的灾难性错误。

**顺序不能错**：`.gitattributes` 必须在**首次 `git add` 资产之前**就位。
如果先把 `.uasset` 普通提交了，再配 LFS，那些文件不会自动转过去，
需要用 `git lfs migrate import` 重写历史（见第十节）。

---

## 五、初始化仓库（第一次，由你执行）

```bash
cd /e/Epic/UE_5.8/project/threepersonswitch

# 1. 初始化
git init
git lfs install --local          # 只对本仓库生效

# 2. 配置身份（同伴也要各自配）
git config user.name  "你的名字"
git config user.email "你的邮箱"

# 3. 中文文件名正常显示（项目里有 Docs/STEPS_CameraRig_手工步骤.md）
git config core.quotepath false

# 4. 创建 .gitignore 和 .gitattributes（内容见第四节）

# 5. 先加规则文件，再加内容 —— 顺序非常重要
git add .gitattributes
git add .gitignore
git add .

# 6. 确认没有把 Intermediate / Saved 加进去
git status --short | grep -E "Intermediate|Saved|DerivedDataCache"
# 应该没有任何输出

# 7. 确认 LFS 接管成功（应该列出 244 个 uasset/umap）
git lfs status
git lfs ls-files | wc -l

# 8. 提交
git commit -m "init: UE5 character-switch project (3 chars, camera rig)"

# 9. 关联远程仓库（地址换成你在 Gitee/GitHub 建好的空仓库）
git remote add origin https://gitee.com/你的用户名/threepersonswitch.git
git branch -M main
git push -u origin main
```

### 第 7 步怎么判断成功

`git lfs ls-files` 的输出行数应该接近 **244**（`.uasset` + `.umap` 总数）。
如果只有个位数或者 0，说明 `.gitattributes` 没生效，**停下来别推**，
检查文件名是不是写成了 `.gitattribute`（少个 s，这是经典手误）。

### 第 8 步推送可能较慢

130 MB 首次上传，Gitee 一般 2–5 分钟，GitHub 可能 10 分钟以上。
不要中途 Ctrl+C，中断会留下不完整的 LFS 对象，需要 `git lfs push --all` 补传。

---

## 六、邀请同伴

### Gitee

仓库页 → 管理 → 仓库成员管理 → 添加成员 → 搜索用户名 → 权限选 **开发者**（可推送）。

### GitHub

仓库页 → Settings → Collaborators → Add people → 输入用户名或邮箱 → 对方接受邮件邀请。

> **不要把仓库设成公开**。UE 项目里的资产可能包含你们买的或下载的第三方素材，
> 公开会有版权风险。

---

## 七、同伴克隆（发给他们的步骤）

```bash
# 1. 装 git 和 git-lfs（Windows 版 git 安装包已自带 lfs，装完执行一次全局启用）
git lfs install

# 2. 克隆
git clone https://gitee.com/你的用户名/threepersonswitch.git
cd threepersonswitch

# 3. 确认 LFS 文件真的下载了（这一步最容易被忽略）
git lfs pull
git lfs ls-files | wc -l        # 应该接近 244
```

### 怎么判断克隆是否完整

打开 `Content/ThirdPerson/Blueprints/warrior.uasset`：

- **正常**：文件大小 37 KB 左右，是二进制
- **异常**：文件只有 130 字节左右，内容是几行文本（`version https://git-lfs...`）

如果是第二种，说明 LFS 没拉下来，执行 `git lfs pull`。
**这种情况下直接打开 UE 会报"资产损坏"**，不要以为是项目坏了。

### 克隆后第一次打开

1. 双击 `.uproject`（会提示"模块缺失，是否重新编译"→ 纯蓝图项目选"是"即可）
2. UE 会生成 `Intermediate/`、`Saved/`、`DerivedDataCache/`，这些都在 `.gitignore` 里，不会污染仓库
3. 首次打开会编译着色器，慢是正常的

---

## 八、在 UE 里配置 Git LFS（强烈建议）

命令行能提交，但在编辑器里操作更顺手，而且**只有配置了 Source Control，
UE 才会在打开被锁定的文件时提示你**。

### 配置步骤

1. `Project Settings` → 左侧搜 `Source Control`
2. **Provider** 选 `Git LFS 2`
3. 点 **Accept Settings**
4. 回到主界面，左下角 `Source Control` 面板 → **Connect**

### 配置后的能力

| 操作 | 位置 | 作用 |
|---|---|---|
| Check Out | 右键资产 → Source Control → Check Out | **锁定文件**，别人改不了 |
| Check In | 右键资产 → Source Control → Check In | 提交并自动解锁 |
| 查看锁定 | Source Control 面板 → Locks 标签 | 看谁占了什么 |

> 如果 Provider 下拉里没有 `Git LFS 2`，去 `Edit → Plugins` 搜 `Git LFS`，
> 启用后重启编辑器。UE5 默认已内置。

---

## 九、文件锁定工作流（这是防冲突的核心）

### 为什么必须锁定

`.uasset` 是二进制。两个人同时改同一个蓝图，Git **无法合并**，
最后 push 的那个人要么覆盖对方的工作，要么被拒绝然后手工选一边。
没有第三条路。

### 锁定的正确用法

**改之前先锁**：

```bash
git lfs lock Content/ThirdPerson/Blueprints/warrior.uasset
```

或在 UE 里右键 → Check Out。

**改完提交并解锁**：

```bash
git add Content/ThirdPerson/Blueprints/warrior.uasset
git commit -m "feat(warrior): 平A三段连招"
git push
git lfs unlock Content/ThirdPerson/Blueprints/warrior.uasset
```

**看谁锁了什么**：

```bash
git lfs locks
```

**别人锁着，但确实需要紧急改**：

```bash
git lfs unlock <文件> --force
```

> `--force` 是危险操作，对方未提交的改动会丢失。用之前先在群里问一句。

### UE 里的锁定体验

配置好 Source Control 后，如果 `warrior.uasset` 被同伴锁定，
你双击打开时会弹提示"该文件已被 XXX 锁定，是否继续"。
这时候**不要强行改**，去群里协调。

---

## 十、三人分工建议（决定冲突频率）

按现在的文件结构，冲突热点很清楚：

| 文件 | 冲突风险 | 原因 |
|---|---|---|
| `BP_ThirdPersonCharacter`（父类） | 🔴 极高 | 三个角色都继承它，任何改动影响全员 |
| `NewMap.umap`（关卡） | 🔴 极高 | 所有人都在这里测，`.umap` 完全无法合并 |
| `controllerswitcher` | 🟠 高 | 换人逻辑，谁都可能想调 |
| `BP_ThirdPersonPlayerController` | 🟠 高 | 输入、相机都挂在这 |
| `Input/IMC_Default`、`IA_*` | 🟠 高 | 加按键时都会碰 |
| `warrior` / `witch` / `auxiliary` | 🟢 低 | 各自独立 |
| `BP_CameraRig` | 🟢 低 | 相机独立后只归一人 |

### 建议的分工

| 成员 | 负责范围 | 说明 |
|---|---|---|
| **你** | 父类 `BP_ThirdPersonCharacter`、`BP_ThirdPersonPlayerController`、`controllerswitcher`、`BP_CameraRig`、`NewMap.umap`、`Input/*` | 全队共享骨架。**这些文件只有你能改**，别人要改必须提需求给你 |
| **同伴 A** | `warrior`、`witch` + 对应的动画/特效资产 | 角色实现，互相独立 |
| **同伴 B** | `auxiliary`、Boss/敌人、关卡布景、UI | 内容制作 |

### 一条铁律

> **父类和关卡，永远只能一个人改。**

这两类文件被两个人碰，必然有一方的工作消失。如果同伴 B 想在关卡里加个 Boss，
让他告诉你位置，由你在 `NewMap` 里摆——听起来麻烦，但比丢一天工作便宜得多。

### 如果关卡必须多人编辑

UE5 的 World Partition 把 Actor 拆成 `__ExternalActors__` 下的独立文件，
理论上不同区域的 Actor 可以分开提交。但你们现在的 `__ExternalActors__` 只有 504 KB，
说明关卡还是小规模，**不值得为它做分区域协作**，直接单人负责更省事。

---

## 十一、分支策略：不要用 Git Flow

这是 UE 项目和其他项目最大的不同。

**常规项目**：每人一个 feature 分支，做完合并到 main。
**UE 项目**：分支对 `.uasset` 毫无意义——合并时 Git 只能让你选一边，等于白做。

### 推荐：所有人直接在 `main` 上工作

```
main  ← 三个人都直接提交
```

配合**文件锁定**来串行化对同一文件的修改。这是 3 人小团队最省心的方式。

### 什么时候才需要分支

| 场景 | 做法 |
|---|---|
| 想试验一个可能推翻的方案 | 开分支，验证完要么合并要么丢弃。丢弃分支时**不要合并**，直接删 |
| 要打一个可回退的版本（比如答辩前） | 打 tag：`git tag v1.0-demo && git push --tags` |
| 大改架构 | 开分支，改完让所有人停下、切过来验证 |

日常开发**不要开分支**，开了反而增加合并地狱。

---

## 十二、每天的固定动作

### 开工（第一件事，别先打开 UE）

```bash
git pull
git lfs pull
git lfs locks          # 看谁锁了什么，避免撞车
```

### 收工

```bash
git status             # 确认改了哪些
git add .
git commit -m "feat(witch): 火球术命中特效"
git push
git lfs unlock --all   # 把今天锁的都释放（如果你确定改完了）
```

> `git lfs unlock --all` 会解锁你持有的所有文件。如果还有没改完的，
> 用 `git lfs unlock <具体文件>` 只解锁改完的。

### 提交信息建议格式

```
feat(warrior): 平A第三段加位移
fix(camera): 换人时镜头穿墙
refactor(switcher): 索引改用取模
```

括号里写模块名，方便以后 `git log` 过滤。

---

## 十三、冲突发生了怎么办

### 情况 1：push 被拒绝（远程有你没有的提交）

```bash
git pull --rebase
git push
```

`--rebase` 会把你的提交接到对方后面，历史更干净。如果有冲突会提示。

### 情况 2：`.uasset` 冲突（两人都改了同一文件）

Git 会标记冲突，但二进制文件**没法手工解决**。只能二选一：

```bash
# 保留自己的版本
git checkout --ours Content/ThirdPerson/Blueprints/warrior.uasset

# 或保留对方的版本
git checkout --theirs Content/ThirdPerson/Blueprints/warrior.uasset

git add Content/ThirdPerson/Blueprints/warrior.uasset
```

**选之前一定要在群里确认谁的工作更重要**，被丢掉的那份只能重做。

### 情况 3：文件被 LFS 锁定，push 被拒

```bash
git lfs locks                     # 确认是谁锁的
git lfs unlock <文件> --force     # 强制解锁（先征得同意）
```

### 情况 4：资产打开报"损坏"

九成是 LFS 没拉全：

```bash
git lfs pull
git lfs fsck
```

`git lfs fsck` 会检查本地对象完整性。

### 情况 5：想回到某个能跑的版本

```bash
git log --oneline                 # 找到目标提交
git checkout <commit> -- Content/  # 只恢复 Content
```

或者整体回退（会丢未提交改动，慎用）：

```bash
git reset --hard <commit>
```

---

## 十四、十个必须知道的坑

**1. `.gitattributes` 写错文件名**

`.gitattribute`（少 s）是最常见的手误。写错后 LFS 完全不生效，
130 MB 二进制会以普通方式提交，仓库瞬间膨胀且无法逆转。

**2. 先提交资产，后配 LFS**

LFS 只对配置之后 add 的文件生效。已经提交的二进制需要用
`git lfs migrate import --include="*.uasset,*.umap" --everything` 重写历史。
这个命令会改写所有 commit hash，**只能在还没和别人共享仓库时做**。

**3. 忘了 `git lfs install`**

同伴克隆下来全是 130 字节的指针文件，UE 打开报损坏。
装完 git 后必须执行一次（全局的，一台机器一次）。

**4. 提交了 `Saved/`**

`Saved/Config/WindowsEditor/` 里有每台机器的窗口布局、自动保存的临时蓝图。
提交后三人每天冲突，而且冲突内容全是垃圾。

**5. 在编辑器里改完没保存就 commit**

UE 的蓝图改动在内存里，Ctrl+S 才落盘。`git status` 看不到未保存的改动。
**提交前先在 UE 里 Ctrl+Shift+S（全部保存）**，再 `git add`。

**6. 两人同时改 `.umap`**

关卡是最容易冲突的文件。`.umap` 无法合并，谁后 push 谁覆盖。
**关卡只让一个人改。**

**7. 用 Git 客户端（如 GitHub Desktop）时没开 LFS**

部分客户端默认不启用 LFS。用之前确认设置里勾了 "Use Git LFS"。

**8. 中文路径显示成 `\344\270\255\346\226\207`**

```bash
git config core.quotepath false
```

只是显示问题，不影响文件本身。

**9. LFS 配额超限，push 被拒**

Gitee/GitHub 都会在超限时拒绝推送，报 `batch response: This repository is over its data quota`。
处理：删旧版本（`git lfs prune`）、或买流量包、或换仓库。

**10. 直接把整个 `Saved/` 或 `Intermediate/` 拖进 Git 客户端**

图形客户端有时会忽略 `.gitignore`（配置错误时）。
**第一次提交前一定要 `git status` 检查一遍文件清单**。

---

## 十五、体积增长监控

二进制项目无法避免膨胀，但要心里有数。

```bash
# 当前 LFS 占用
git lfs ls-files -s | awk '{s+=$3} END {printf "%.1f MB\n", s/1048576}'

# 仓库总大小（含历史）
du -sh .git
```

### 经验值

| 阶段 | 预期体积 | 说明 |
|---|---|---|
| 首次提交 | 130 MB | 主要是 Mannequins |
| 一学期正常开发 | 200–400 MB | 每次改蓝图新增完整副本 |
| 超过 1 GB | 需要处理 | 接近免费额度上限 |

### 超过 1 GB 时的处理

```bash
# 1. 清理本地无用对象（不影响远程）
git lfs prune

# 2. 如果是历史里有大文件误提交，用 migrate 重写
git lfs migrate import --include="*.uasset" --everything
```

> 项目周期短（学期作业）的话，130 MB 起步 + 适度增长，一学期基本不会超。
> 真超了就新建一个仓库，把当前状态作为新的起点提交——历史丢失，但项目能继续。

---

## 十六、执行顺序总结

按这个顺序做，一步都别跳：

1. **决定平台**（推荐 Gitee，看第二节对比）
2. **平台上建空仓库**（不要勾选"初始化 README"，否则首次 push 会冲突）
3. **本地写 `.gitignore` 和 `.gitattributes`**（内容照抄第四节）
4. **`git init` + `git lfs install --local`**
5. **先 `git add .gitattributes`，再 `git add .`**（顺序决定 LFS 是否生效）
6. **`git lfs ls-files | wc -l` 确认接近 244**
7. **commit + 关联远程 + push**
8. **邀请两个同伴**
9. **同伴按第七节克隆，确认 `git lfs pull` 后文件不是指针**
10. **三人各自在 UE 里配置 Source Control（第八节）**
11. **按第十节分工，明确"父类和关卡归谁"**
12. **每天开工 `git pull`，收工 `git push`**

---

## 附：这个方案没解决的事

**Mannequins 占了 97% 体积，但你们只用它当占位。**

如果后期换成自己的角色模型，可以考虑把 Mannequins 从仓库里剔除
（`git rm -r --cached Content/Characters/Mannequins`），让同伴各自从
UE 模板迁移一次。这样仓库能从 130 MB 降到 5 MB 以内，LFS 配额压力彻底消失。

但**现在不要做**——三个角色的蓝图都引用着 Mannequins 的骨骼和网格，
剔除后所有人打开都是空的。等美术资源到位、引用切换完成后再考虑。
