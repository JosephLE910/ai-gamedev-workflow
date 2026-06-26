# 代码工坊 统一经验库

> 横跨 **SundayDrive（UE）**、**OutLaw（Unity）**、**OutLawArt（Blender）** 三个项目。
> 按工种分类索引。每条经验标注来源引擎 `[UE]` / `[Unity]` / `[Blender]` / `[跨引擎]`。
> 执行者读与当前 plan 类别匹配的章节；规划者写 plan 时按分类自动引用。

---

## 工种与 Plan 分类对应

| 工种 | Plan 标签 | 经验章节 | 条目数 |
|------|----------|---------|--------|
| 程序化生成 / 地编 | `PCG` | [§PCG](#pcg) | 8 |
| 玩法 / 业务系统 | `GAME` | [§GAME](#game) | 23 |
| 关卡搭建 | `LEVEL` | [§LEVEL](#level) | 4 |
| 美术 / 资产管线 | `ART` | [§ART](#art) | 18 |
| 音频系统 | `AUDIO` | [§AUDIO](#audio) | 5 |
| WebGL 构建 / 部署 | `WEB` | [§WEB](#web) | 11 |
| 移动端打包 | `MOBILE` | [§MOBILE](#mobile) | 16 |
| 流程 / 工具 / Skill | `META` | [§META](#meta) | 15 |
| 规划者专属 | `PLAN` | [§PLAN](#plan) | 3 |
| Bug 修复 | `FIX` | [§FIX](#fix) | 4 |
| 通用调试 | `DEBUG` | [§DEBUG](#debug) | 13 |

---

## §PCG — 程序化生成 / 地编

### PCG-1. 渲染与碰撞分离原则 [UE]

**来源**：Plan 09 — 围墙碰撞，5 轮迭代：HISM → ProceduralMesh 三角形 → BoxComponent 分离。

**原则**：渲染和碰撞用不同组件。不要试图用一种组件同时解决两个问题。

| 组件 | 职责 | 碰撞设置 |
|------|------|---------|
| UProceduralMeshComponent | 视觉渲染（顶点、法线、材质） | NoCollision |
| UBoxComponent / UStaticMeshComponent | 物理阻挡 | BlockAll |

**反模式**：
1. HISM 实例 → 无碰撞响应
2. ProceduralMesh 三角形碰撞 → 高速穿透卡墙
3. 同一组件既渲染又碰撞 → 难以各自调优

### PCG-2. 关卡已放置 Actor 不随 C++ 默认值更新 [UE]

**来源**：Plan 09 §10.1 — GridSize 从 5 改为 12 后，关卡中已摆的 Actor 仍显示 5。

**原因**：UE 序列化。Actor 实例属性保存在 .umap 文件中，编译后 C++ 默认值变了但序列化值不变。

**正确做法**：
1. 编译完成
2. `python3 -m soft_ue_cli set-property <ActorName> <PropertyName> <Value>`
3. 对每个改过的 UPROPERTY 重复
4. 编辑器内 Ctrl+S 保存关卡

### PCG-3. NavMesh 烘焙与代理半径 [UE]

**来源**：README §5 — 运行时动态生成不稳，车需要更大的代理半径。

- NavMesh 用 **静态烘焙**（RecastNavMesh `RuntimeGeneration=Static` + 编辑器 `RebuildNavigation`）
- 代理半径要 **手动调大**：车不是行人，~150cm
- PCG 大场景（300m+）NavMeshBounds 要同步扩

### PCG-4. HISM 原点不能离实例太远 [UE]

**来源**：README §5 — HISM 组件原点在关卡原点时，远距离实例有浮点精度问题。

**为什么**：Unreal 用 float（单精度）。离原点 10km+ 时精度降到 cm 级，HISM 实例会抖动。

**解法**：
- 场景 < 300m 无影响
- 分块/World Partition 时每个 HISM 放自己区块原点

### PCG-5. 坐标公式必须从代码推导 [UE]

**来源**：Plan 08 §8.4 — 人工推导路网坐标出错，导致 Actor 摆到楼里。

**正确做法**：写 plan 前 Grep `GetTotalSize()` 获取实际返回值，用代码里的公式算坐标。不凭直觉。

### PCG-6. PCGCityTest 的城市是 C++ `APCGCityGenerator` 生成，不是 UE PCG Graph [UE]

**来源**：Plan 27 Step 0-C — 规划假设楼宇在二进制 PCG 图里（读不到），实际是纯 C++。

PCGCityTest 的路网/楼宇由 `Source/SundayDrive/PCGCity/APCGCityGenerator` 代码生成（`GenerateBuildings()` 等）。**要调城市布局直接改 .cpp**，比操作二进制 PCG 图简单得多，规划者也能直接审。
- 加位置/尺寸扰动用**种子驱动的 `FRandomStream`**（`Seed*31337 + r*100 + c`），保持**确定性生成**（同种子同城）——别用真随机。
- ⚠️ 改楼布局会动到 baked NavMesh 的"楼脚挖洞"，楼移位/变大后要确认路网仍连通（警察寻路），必要时重烤。

### PCG-7. 程序化地面/网格的两个隐形坑：三角绕序 + HISM 不清旧实例 [UE]

**来源**：Plan 31（垃圾场地面起伏，六轮救火踩出来的）。

- **三角绕序**：自建 ProceduralMesh 地面，**碰撞能用但渲染不可见**（车能开、地面看不见）= 三角索引绕序反了，从上方看是背面、被背面剔除。碰撞不挑绕序、渲染挑。修：翻绕序（`TL,BL,TR`/`TR,BL,BR`）+ 法线同步反向朝上 + 显式给材质（空材质也会误判可见性）。
- **HISM 重散布前必须 `ClearInstances()`**：散布器（垃圾堆等）在 BeginPlay 再散布时，**.umap 里序列化的旧实例还在**，新实例叠上去 = 双份（"垃圾飘空中/两份"）。散布前先清。
- 配合 §PCG-2：改了生成参数默认值，已放置 Actor 不自动更新；运行时在 `BeginPlay` 生成前**钳制旧实例参数**到可用范围兜底。

### PCG-8. 程序化地形起伏会动到 NavMesh + 车辆生成（连锁风险） [UE]

**来源**：Plan 31 — 平地改噪声高度场后，连环出问题。

- 地面从平面改起伏 → **baked NavMesh 可能不再贴合**（AI 车在坡上寻路/接地异常）。改地形必须确认 navmesh 重烤/仍连通（同 §PCG-6 的楼宇风险）。
- **车辆生成到起伏地形**别用中心单点定 Z：用**车身覆盖范围多点射线**（四角+中心 5 点取最高地形点）+ ride-height（挂点+悬挂静止长+轮半径）落地，否则车架穿地或四轮射线起点埋进地形 → 接地检测失效、`bAnyWheelGrounded` 永假。
- 起伏幅度/频率是**可驾驶性**约束：高频大幅 = 断崖卡车。大尺度起伏 + 小幅中尺度坑洼叠加，入口区 `FlattenRadius` 衰减保持平坦。

---

## §GAME — 玩法 / 业务系统

### GAME-1. 悬挂必须放物理子步回调 [UE]

**来源**：README §5 — Tick 里施加力会随帧率变自激→爆炸弹飞。

- 用 `FBodyInstance::AddCustomPhysics` 回调
- `DefaultEngine.ini` 开 `bSubstepping=true`
- Tick 只用做非物理逻辑（输入读取、UI 更新）

### GAME-2. 碰撞判定用 NormalImpulse，不读撞后速度 [UE]

**来源**：README §5 — 碰撞事件回调里速度已被清零。

- 读 `FHitResult.NormalImpulse` / 质量 求撞击速度变化量
- 不要读 `GetVelocity()` ——撞后已重置为零

### GAME-3. 相机松跟随逻辑 [UE]

**来源**：README §5 + Plan 01 — 追尾相机已用书本方法重做。

- 倒车时不要锁死视角（松跟随）
- 撞击时加入震动衰减
- 以上已在 `SimpleCarPawn` 相机逻辑中实现

### GAME-4. 攻防伤害区分 [UE]

**来源**：Plan 02 — 伤害应该区分谁是 attacker、谁是 victim。

- `AttackerDamageMul`：冲撞方伤害倍率
- `VictimDamageMul`：被撞方伤害倍率
- 判定逻辑已在 `SimpleCarPawn` 中实现

### GAME-5. 任务目标坐标需暴露给其他系统 [UE]

**来源**：Plan 12 §9.1 — 小地图需要访问 PickupLocation/DropoffLocation，但字段是 protected。

**原则**：任何"任务目标坐标"类数据都应加 public getter，因为 HUD、小地图、AI 子系统都需要读。

```cpp
// DeliveryMission.h
FVector GetPickupLocation() const { return PickupLocation; }
FVector GetDropoffLocation() const { return DropoffLocation; }
```

### GAME-6. BrakeDecel 公式不要动 [UE]

**来源**：Plan 13 §8.2.1 — 规划中写 `(BrakePower / MassKg) / 100.f`，代码实测后纠正。

**正确公式**：`ActualDecel = BrakePower / MassKg`（单位：cm/s²）。
UE 中 `AddForce` 直接用力单位（kg·cm/s²），不需要 `/100` 或其他单位换算。

当前值：`2000000 / 1500 = 1333 cm/s²`，与原 BrakeDecel=1400 接近。

**教训**：如需调刹车手感，改 `BrakePower` UPROPERTY 值，不要改公式。

### GAME-7. 放置 Actor 用 EditorLevelLibrary，不要手动拖放 [UE]

**来源**：Plan 17 — 执行者把"在关卡放置 Actor"理解为手动操作，卡在 PIE 验证前交还给用户。

**原则**：只要能用 Python 桥接完成的操作，就不能停下来等人。
放 Actor、设属性、保存地图全部可以程序化完成：

```python
import unreal

# 放置 Actor（编辑器态，不在 PIE 里）
loc = unreal.Vector(-3400, -3000, 200)
rot = unreal.Rotator(0, 0, 0)
atd_class = unreal.load_class(None, "/Script/SundayDrive.AutonomousTestDriver")
actor = unreal.EditorLevelLibrary.spawn_actor_from_class(atd_class, loc, rot)

# 设置 UPROPERTY（注意：属性名用 snake_case）
actor.set_editor_property("test_goal", unreal.Vector(3400, 3000, 200))
actor.set_editor_property("b_auto_start", True)

# 保存关卡
unreal.EditorLevelLibrary.save_all_dirty_levels()
```

用 `python3 -m soft_ue_cli run-python-script --script /tmp/place_actor.py` 执行。
**先把脚本写到 `/tmp/`，再调桥接运行。**
**属性设置完后必须立即调 `save_all_dirty_levels()`——否则重启编辑器后 Actor 配置全部回滚。**

### GAME-8. 测试车要 `AutoPossessPlayer = Player0` [UE]

**来源**：Plan 17 — ATD 设为 `Disabled`，相机跟着玩家车，用户以为车没动。

```cpp
// AutonomousTestDriver.cpp constructor
AutoPossessPlayer = EAutoReceiveInput::Player0;
```

只有想让用户实时看到 AI 车移动时才设此项；纯后台批量测试（只看 JSON/CSV）则保持 `Disabled`。

### GAME-9. ATD 失败时先读 CSV，不要猜原因再改代码 [UE]

**来源**：Plan 17 Bug 定位过程 — 执行者看到 `status=timeout` 后直接改 `ArriveRadius`、移坐标，绕过了遥测数据。

**原则**：ATD 是 AI player，它的一切行为都已记录在 `/tmp/atd_debug.csv`（需开启 `bLogTelemetry=true` + 设 `TelemetryPath`）。
出现异常时的正确流程：

```
1. 读 CSV 最后 N 行：tail -50 /tmp/atd_debug.csv
2. 看 distTgt 趋势（是在接近还是反复震荡）
3. 看 pathPts（0 = NavMesh 无路；>0 = 有路但没走到）
4. 看碰撞列（NormalImpulse > 阈值 = 被挡住）
5. 根据数据定位原因，再写修复方案
```

不要在没看数据前就修改 `ArriveRadius`、`ReachRadius` 或移动目标坐标。这些改动会掩盖真实原因。

**规划者注意**：涉及 ATD 测试的修复计划，preamble 应要求执行者先附上 CSV 尾部数据再动手改代码。

### GAME-10. Heuristic Brain 无法完成密集障碍终点导航 [UE]

**来源**：Plan 17 — ATD 在取货点外 525 cm 处触发 wall-avoidance 死循环，永远无法进入 500 cm 触发圈。

**机制**：`DriverBrain_Heuristic` 的 wall-avoidance（侧向 raycast + 满偏方向盘）与 seek-steer 没有优先级调解。障碍间距 < 约 300 cm 的密集区域两者互相否定，车绕圈直到 timeout。

**Phase 1 能力边界**：goal 距最近障碍 ≥ 约 600 cm 时可靠到达；进入密集障碍区后约 525 cm 处失效。

**不要为让测试通过而清空目标周围障碍物**——这会掩盖真实的 AI 能力边界。记录上限，将"密集障碍终点导航"列为 Phase 2 GA Brain 的优化目标。

**更新（Plan 24，2026-06-16）**：有 NavMesh 的图里此问题大幅缓解——`DriverBrain_Heuristic` 本就先走 `FindPathToLocationSynchronously`，**只有 navmesh 无路才退化成 dead-reckon + wall-avoid 死循环**（即上文场景）。PCGCityTest 有 baked navmesh（挖掉楼脚），警察可全城高速沿街追逐。
残留的「贴墙死锁」也已修：① 卡死检测从瞬时速度(`<50cm/s`)改**参考点位移**——车顶墙磨蹭残抖 ~114cm/s 高于旧阈值、永远判不出卡，位移判定才抓得住；② 脱困从原地摇方向改**果断掉头子状态机**（侧向射线选开阔侧→倒车预旋→前进满舵掉头→位移不足换向重试）。验证：原永久死锁点 3s 内脱墙。

### GAME-11. 运行时建的 UActorComponent 必须 RegisterComponent() 才会 Tick [UE]

**来源**：Plan 21 — `UGunnerComponent` 用 `NewObject` 运行时创建，没 `RegisterComponent()` 不 Tick。

构造函数/蓝图里加的组件自动注册；**运行时 `NewObject<UComp>(Owner)` 创建的不会**，必须显式 `Comp->RegisterComponent()`，否则组件存在但不参与 Tick/物理。

### GAME-12. 依赖 PlayerController 的 widget 别在 BeginPlay 建 → PC 未生成会漏建/黑屏 [UE]

**来源**：Plan 24 — 主菜单在 GameMode `BeginPlay` 挂 → 本地 PC 还没生成 → 菜单漏建黑屏；HUD 同理。

GameMode `BeginPlay` 时本地 PlayerController 常**还没生成**，此刻 `CreateWidget`/`GetOwningPlayer` 拿不到 PC → widget 漏建。
**正确时机**：菜单走 `AGameModeBase::HandleStartingNewPlayer`（PC 已生成）；HUD/任务 UI 在 `Tick` 里**懒创建**（PC 就绪后第一帧建一次，加 `bCreated` 闸）。配合 Plan 23 教训：建完 `SetVisibility(Visible/HitTestInvisible)`。

### GAME-13. 撞击过滤：ImpactMinSpeed 太低→剐蹭都有伤害 [UE]

**来源**：Plan 26 — 玩家反馈"剐蹭都有伤害"，`ImpactMinSpeed=250cm/s`(9km/h)太低。

- `ImpactMinSpeed` 控制触发碰撞伤害的最小法向速度。250=9km/h，轻微蹭墙即触发。
- **修**：提到 600+(≈22km/h) 过滤剐蹭；同时降 `CarDamageScale`(0.05→0.035) 和 `WallDamageScale`(0.02→0.012) 降低伤害幅度。
- 两项**配合**用：提高门槛减少触发次数 + 降低倍率减少每次伤害量。

### GAME-14. 玩法系统需要「原因文字」反馈，光有数值条不够 [UE]

**来源**：Plan 26 — 玩家反馈"绕圈时怀疑度不知道为什么会涨/降"。

HUD 只有热度条+百分比，但玩家需要知道**当前行为导致了什么**。
**模式**：底层系统暴露 `GetXxxReason(FString& Text, FLinearColor& Color)` → HUD 每帧读+画。颜色编码行为（绿=安全/红=危险/橙=警告），文字说清楚速率和方向。

此模式可复用：通缉度原因、AI 状态诊断、任何"玩家不知道为什么 X 在变"的场景。

### GAME-15. 程序化合成音效：没音源也能有声（USynthComponent） [UE]

**来源**：Plan 27 — 没有警笛音源（只有 2 个枪声占位）。

不想等/找音频资产，可继承 `USynthComponent`，在音频渲染线程**程序化合成**波形——警笛 = 三角 LFO 扫频（600↔1200Hz）+ 二次谐波。**零音源依赖、无 asset、无 Android cook 丢音风险**，是"声音管线先跑通"的最快路径。
- `Build.cs` 要加 `AudioMixer` 依赖。
- `USynthComponent` 无默认构造函数 → 子类必须用 `FObjectInitializer` 构造。
- 适合警笛/引擎/UI 提示音这类合成音；真实感要求高的（撞击/语音）仍需 WAV。

### GAME-16. 多车场景里「会发声/会扫描」的共享组件必须门控 [UE]

**来源**：Plan 27 — 撞击声全图乱响、枪手隔墙开枪、警笛每帧卡顿。同一组件挂在 N 辆车上时，几类坑反复出现：

- **发声要限定来源**：`OnBodyHit` 挂每辆车 + 裸 `SoundWave` 无距离衰减 → 全图警车蹭墙都原音量响。修：只在 `IsPlayerControlled()` 的车上发声。
- **扫描/视线射线会命中自己人**：枪手 LOS 射线终点命中目标车自身碰撞体 → 若把"命中任何东西"都算遮挡，则**所有目标都被误判挡住、完全不开枪**。修：命中的是 `ASimpleCarPawn` 不算遮挡，只有非车的 `WorldStatic` 才算。
- **每帧调的开关要幂等**：每帧 `Start()` 循环音 → 反复重启卡顿。修：`SetEnabled` 内部判重，状态没变就 return。
- **占位音复用会串味**：撞击声复用枪声 → "被撞"听着像"中弹"。临时方案靠**降调/调音量**区分（撞击 pitch≈0.45 变沉闷）。

> 共享组件（GunnerComponent/SimpleCarPawn 这类）改动会同时影响所有用它的关卡（如垃圾场），改完要回归别的图。

### GAME-17. 引擎自带材质的参数名靠不住（尤其移动端）→ 自建 Unlit 材质 [UE]

**来源**：Plan 27 — 红蓝灯第一轮"不可见"，根因是 `SetVectorParameterValue("EmissiveColor")` 设的是引擎材质里**不存在的参数名** → 静默失效、显示默认暗色。

引擎内置材质的参数名/移动端支持都不可控。要稳：**自建一个 Unlit 自发光材质**（参数名 `EmissiveColor`/`Boost` 自己定），构造函数用 `ConstructorHelpers::FObjectFinder` 拿硬引用（随包 cook）。**验证**：PIE 里读回 DMI 确认参数真写进去了，别只看代码调了 `SetParameter` 就当成功（它失败是静默的）。

### GAME-18. 失去目标的「盲搜」：扩张环一招顶状态机 [UE]

**来源**：Plan 29（警察失明搜索，参考《Game AI Pro》target-lost→search）。

AI 丢失目标后该"在最后已知点附近找"，但别写复杂的 per-agent 状态机。一个公式就够：
- 记 `LastSeenLoc`（有 LOS 时更新）。无 LOS 时目标点 = `LastSeenLoc + 环上一点`，**环半径 `Ring = min(Max, ExpandRate × 失明秒数)`**。
- **半径从 0 随失明时间扩张** → 刚丢时 Ring≈0 全员收敛到最后已知点（= "先扑过去"），久没找到环变大（= "扩圈搜"）。两段行为一个公式出，不要额外状态机。
- 多 agent 散开：按 `槽位/N × 2π` **均分角** + 抖动 + 随时间慢转（游走感），别叠一团。
- 搜索点必须 **navmesh 可达**（过 `ResolveDrivablePoint` 之类，§GAME-10），换点只在到达/超时/无效时（非每帧，省 line trace）。
- 重获 LOS → 立刻恢复直追，清空搜索状态。

### GAME-19. 落地镜头手感：纯 Z 偏移容易被构图吞掉，组合"离地拉远 + 落地拉近/下沉" [UE]

**来源**：Plan 32 — 落地悬挂镜头震荡。

相机要跟悬挂"有避震感"，不要另造一个独立震动器；正确输入是车辆已有的悬挂压缩、接地状态、离地时间、下坠速度。但实测只把相机做 Z 下沉，即使日志里 `SocketOffset.Z≈-50cm`，第三人称追尾构图也可能体感很弱。

更可靠的体感组合：
- **离地阶段**：相机臂长平滑拉远，让玩家看到更多落点/坡面。
- **落地瞬间**：同一份悬挂压缩信号驱动相机下沉，同时臂长拉近，制造"砸下去"的冲击。
- **回弹阶段**：随悬挂高通/门控衰减，臂长和 Z 偏移一起回稳。

防晕关键：平路/起伏地形只给极弱 `GroundedScale`；明显落地才开门控。门控不要只看当前 `Vel.Z`，物理接触可能先吃掉下坠速度，建议在离地时记录 `RecentDownSpeed`，接地后用 `max(当前, recent)` 判定。小车/Rat 这种短悬挂车型，按压缩 cm 算体感会天然弱，需要补相机倍率；如果悬挂长期趴底，要先检查车型弹簧/阻尼参数，别只继续放大镜头。

### GAME-20. 「过去的你」回放：用状态快照，别用输入重放 [Unity]

**来源**：OutLaw Plan 01

固定 tick 录 `{位置, 朝向, 是否开火}` 快照，回放时**直接置位**——天然零漂移（实测 `echoMaxDrift=0.00000`）。**别**录输入做确定性重放：Unity 物理不保证确定性，会 desync。
- 唯一一处推进 tick（`RoundManager.FixedUpdate` 按固定顺序 `SimTick(tick)`），保证录制/回放/跨轮对齐，且与帧率解耦。
- 回响"在当前世界结算"：只存开火**方向**，开火时生成**新活体子弹**命中本轮场上的靶，不绑录制时的目标实体。验证手法：第 2 轮把旧靶挪走、看回响子弹改打当前靶（不同 instanceID）即证明。

### GAME-21. 灰盒场景整段代码化 [Unity]

**来源**：OutLaw Plan 01

场景里只放一个 `Bootstrap`，竞技场/角色/靶/对象池全在 `Awake()` 用代码搭，色块占位（`Sprite.Create` 生成 1px 白方块再 tint）。改布局/手感只动一个文件，可复现、好 diff、不用在编辑器里点。墙体检测用 marker 组件而非 Tag，省工程内建 Tag 配置。
> ⚠️ 例外见 DEBUG-11：已**序列化进场景**的 `public` 字段会被场景存档值覆盖代码默认，"只改 .cs"对它无效。

### GAME-22. 回响武器感知：录制结构带动作类型/武器，回放按"录制轮的武器" [Unity]

**来源**：OutLaw Plan 02

回响要做到"持刀的过去的你替你挡刀、持枪的远程输出"，录制快照必须从 `{Pos,Aim,Fired}` 扩成 `{Pos,Aim,Action,Weapon}`（Action∈None/FireGun/SwingKnife）。回放 `ReplayAction(action,weapon,...)` 按**记录的武器**执行、无冷却——回响=当时的你，用的是它那轮所持武器，不是玩家当前武器。验证：构造一轮持枪、中途切刀，看回响第 2 轮先播 FireGun 后播 SwingKnife（`ECHO_ACTION tick.. action=SwingKnife weapon=Knife`）。旧录制结构不兼容无所谓（每局重录）。

### GAME-23. 灰盒敌人用 Kinematic + 脚本驱动 + 钳位，别靠物理碰撞 [Unity]

**来源**：OutLaw Plan 02

敌人移动全脚本驱动时设 **Kinematic**（不参与物理去穿透，杜绝被挤出界/挤停）；接触伤害用**距离判定**不依赖碰撞体。代价：敌人之间/与玩家不再物理阻挡（可穿过），灰盒可接受。Spawn 要**先设好位置再 `SetActive(true)`**，别让模板在原点 (0,0) 就注册进物理。

---

## §LEVEL — 关卡搭建

### LEVEL-1. 新建关卡必须放灯光 [UE]

**来源**：Plan 10 §10.6 — PIE 全黑。TrashyardGenerator 只生成几何体不生成光源，
引擎配置 `r.AllowStaticLighting=False` + 关闭 Lumen/光追 = 渲染完全依赖手动灯光。
Plan 10 遗漏了 Plan 04 教训中的灯光步骤。

**铁律**：
- 关卡新建脚本中显式添加：`DirectionalLight` + `SkyAtmosphere` + `SkyLight`
- 或在 Generator 中加入灯光自检逻辑

### LEVEL-2. 新关卡用新 .umap [UE]

**来源**：README §5 — 不要覆盖 SimpleCarTest.umap。

- `SimpleCarTest.umap` 是回归测试关卡，永远不改
- 每个新场景新建 .umap
- GameMode Override 在 World Settings 里设置

### LEVEL-3. PCG 地形高度不可靠 [UE]

**来源**：Plan 12 §9.6 — 取货点坐标 (6400, 6400) 落在高台（Z=1101cm），车无法到达。

**根因**：人为设定的坐标不考虑 PCG 生成的实际地形高度。

**解决**：运行时做垂直射线检测 + 螺旋搜索找到可驾驶地面（Z ≤ 300cm）。
搜索用"螺旋展开"（先近后远 20m 步长 8 方向）比随机采样更优。

**教训**：任何关卡中的目标位置参数都应做运行时地面验证，不能信任硬编码坐标。

### LEVEL-4. PCG 地图 spawn 点要查"开阔度"，不能只看 overlap [UE]

**来源**：Plan 20 — 3 辆 AI 卡出生点。点在 NavMesh 上、无 actor 重叠，但四周是 PCG 建筑，车一动撞墙，`throttle=1 但 speed≈0` 耗到 timeout。

**根因**：**blocking 碰撞不触发 overlap 事件**，所以"无 overlap = 安全"是假象。PCG 建筑是 blocking，只查 overlap 会漏掉被围住的点。

**正确做法**：spawn 检测向多方向 raycast 查"开阔度"（如 8 向无墙），而非只避让其他 actor。硬编码点在 PCG 图换 seed 还会被围 → 稳妥是运行时 raycast 挑开阔点。

**数据驱动定位范式**：读 CSV(throttle=1/四轮着地/speed≈0)→ 排除 NavMesh(有路)→ 排除 overlap(假阴性)→ 设刚体速度(能动,排除物理)→ 改 spawn 空地(立即 86km/h)。一步步用数据砍假设。

---

## §ART — 美术 / 资产管线

### ART-1. AI 产线接入选型：直连腾讯云 ai3d API [Blender]

**来源**：OutLawArt Plan 01 Step0 调研，**待跑通验证**

三条路调研结论（Mac Apple Silicon、无 NVIDIA/CUDA）：
- **❌ 自托管 Blender addon**（官方 Hunyuan3D-2 addon / jfranmatheu Hunyuan3DBlenderBridge）：都驱动**本地 GPU 模型服务器**，Mac 无 CUDA → 不可行。
- **❌ blender-mcp 内置 Hunyuan**（ahujasid/blender-mcp PR #182）：升级到新 API 的 PR **未合并**（2026-06-04 被关，"等官方 MCP"）→ main 没有可用 Hunyuan 工具，不走它。
- **✅ 直连腾讯云 `ai3d` 云 API**（选定）：最可控、无需本地 GPU、可脚本化 headless 复现（合 ART-12）。云端生成 → 拉回网格 → 进 Blender。

**API 契约**（2025 版）：
- 服务 `ai3d`，版本 `2025-05-13`，域名 `ai3d.tencentcloudapi.com`（国内）/ `hunyuan.intl.tencentcloudapi.com`（国际站）。
- 异步作业：`SubmitHunyuanTo3DRapidJob`（图→3D，快、1 并发）或 `SubmitHunyuanTo3DProJob`（文+图、3 并发）→ 返回 `JobId`（24h 有效）→ 轮询 `QueryHunyuanTo3DRapidJob`/`...ProJob` → 拿 `ResultFile3Ds`（URL+Type，OBJ/GLB 等）下载。
- 入图：`ImageBase64` 或 `ImageUrl`（128–5000px，≤8MB）。Pro 可选 `GenerateType`(Normal/LowPoly/Geometry/Sketch)、`FaceCount`(3k–1.5M)、`EnablePBR`、`PolygonType`(Tri/Quad)。
- **鉴权**：TC3-HMAC-SHA256，需 **SecretId + SecretKey**（不是单个 key！），且控制台需**开通 ai3d 服务**（有配额/计费）。
- 限制：默认 3 并发（Rapid 1 并发），≤20 req/s。

⚠️ 以上为调研契约，**真正跑通后再补**：实际鉴权坑、Rapid vs Pro 质量差、配额/计费、返回格式、import 进 Blender 的朝向/缩放。

### ART-2. 实际走法：网页 Hunyuan3D 免费额度手动生成 [Blender]

**来源**：OutLawArt Plan 01，**实跑**

用户改用**网页混元免费额度手动生成模型**给我（绕开密钥/GPU/后端全部阻塞）——一次性 spike 最省。两条关键质量发现（决定成败）：
1. **务必要"带 PBR 贴图"的导出**。第一只模型导出**无贴图**、顶点色是**每个 part 一块乱涂的纯色**（身体紫、尾巴蓝/粉，和参考图无关）→ 完全没法用，我手凑顶点色也只是勉强。第二只勾了 PBR（4K base/metallic/normal/roughness、**单材质**、UV 全）→ 长相直接还原参考图、九尾橙渐变/金铃铛/黄眼全在。**教训：混元导出必须确认带 PBR 贴图。**
2. **AI 把参考图的"动作姿势"烤死进网格**。第二只按参考图的**行走姿势**生成——前爪抬在半空、bind pose 不是中性站姿 → 绑定/落地添大麻烦（见 ART-6）。**教训：喂"侧身、四腿直立、不动态"的中性站姿参考图**，后面省巨大功夫。
- 网格规格：FBX、**14 个 part、单材质、69342 tris**、有 UV、贴图打包在 fbx 内。
- 混元 Blender 插件路（调研 ART-1 提的）实测不必：插件只是壳、要本地/远程 GPU 服务器，而网页免费额度直接出网格最省事。

### ART-3. "有贴图节点"不等于"贴图已交付"：必须查 packed/has_data [Blender]

**来源**：OutLawArt Plan 03，wolf

wolf.fbx 的材质节点引用了 base/normal/roughness/metallic 四张 PBR 图，但实际 `has_data=False`、尺寸 `(0,0)`，路径写死到原作者机器 `/Users/bingnanliang/Downloads/output.fbm/...`，本机全局搜不到。结论：素材入库 Step 0 不能只看 material/节点名，要确认贴图**真正随文件打包或随包交付**；Blender 里至少查图片 `has_data=True` / 尺寸有效 / 路径可读。否则按灰模占位处理，并在交接里明确"UV 保留，贴图到位后热替换"。

### ART-4. 清理到 game-ready —— 轻，分钟级（headless 可复现） [Blender]

**来源**：OutLawArt Plan 01 实跑结论（九尾狐，带 PBR 的混元 FBX）

导入 FBX（贴图随导入打包）→ `join` 14 part → **Decimate Collapse 69k→10k**（COLLAPSE 会插值，**保 UV/顶点色/贴图**）→ 保留 PBR 材质 → 原点设脚底中心（按全局 bbox 居中 X/Y、Z 落 0）→ glTF GLB 导出（`export_yup=True`）。贴图 4K 内嵌太重（24MB）→ `image.scale(1024,1024)` 降到 1K → **2.9MB**。这步**不是瓶颈**。

### ART-5. 坑：FBX 导入给对象加 90° X 旋转 → 局部坐标 Y/Z 互换 [Blender]

**来源**：OutLawArt Plan 01

混元 FBX 导入后 `mesh.matrix_world` **非单位**（带 +90°X），`v.co`（局部）的 Y/Z 是反的。**任何按几何算骨位/脚位的代码必须用 `mesh.matrix_world @ v.co`（世界坐标）**，用局部会全错（脚算到 z=-0.45、尾巴区）。最终烤定时 `transform_apply` 把它归一。

### ART-6. 绑定才是真瓶颈（4 腿 + 9 尾的非标准四足；脚本搭架 + 自动权重） [Blender]

**来源**：OutLawArt Plan 01

- **骨架 32 骨**：hips-spine-neck-head + 4 腿×2 + 9 尾×2 + 双耳。脚位用"贴地顶点按象限聚类"找；**象限分界要用身体质心 `bodyC.x/y`，不是 0**（模型没居中在 0，用 0 会漏腿/只找到 3 条）。
- **自动权重（`parent_set ARMATURE_AUTO`）对挨太近的部件串权重**：耳朵被误分给会动的骨头、一动就飞 → **给耳朵单独加骨头**解决；9 尾基本独立（单尾甩 55° 邻居不太跟）。
- **bind pose 是动作姿势 → 落地最痛**：前腿骨太短够不到地（前爪本就在半空 z≈0.08）。**IK 拉伸会把腿撑粗（不可用）**；可行解 = **整体前倾 ~9° 让四脚落平 + 转颈/头摆正**（刚性旋转、零变形）。
- **重设 rest 工作流（保留权重，标准 Blender 做法）**：apply Armature 修改器（烤当前姿势进网格、保留顶点组）→ `pose.armature_apply()`（姿势设为新 rest）→ 重挂 Armature 修改器 → `parent_clear(KEEP_TRANSFORM)` + `transform_apply` 把对象级前倾烤进几何（对象归一、导出干净）。
- **动画**：idle / 走（对角步）/ 受击，3 个命名 action，glTF `export_animation_mode='ACTIONS'` 导成 3 个 clip。导出 glb 自检：1 skin/32 joints、3 animations、prim 带 `JOINTS_0/WEIGHTS_0`、3 贴图。
- ⚠️ **绑定大量靠在线交互修（blender-mcp GUI 桥：腿骨对齐/耳骨/落地都是边看渲染边调）**，不像清理能纯 headless 复现 —— **这是 AI 网格进游戏最大工时、最难自动化处，Phase 2 重点**。最省功夫的根治是从源头喂中性站姿参考图（见 ART-2）。

### ART-7. Blender 5.1 / 本机环境坑 [Blender]

**来源**：OutLawArt Plan 01

- `action.fcurves` 在 5.1 没了 → 用 `bpy.context.preferences.edit.keyframe_new_interpolation_type='LINEAR'` 设默认插值。
- 本机 Blender（brew cask）**没编译 ffmpeg**，渲不了 mp4 → 渲 PNG 序列 + 系统 Python Pillow 拼 GIF 预览（系统无 ffmpeg/imagemagick）。
- 渲染预览用 workbench `color_type='TEXTURE'` 显 PBR base 色，快且可靠。

### ART-8. 预绑定 Rigify 资产：复用蒙皮比重绑更划算，但骨数可能偏重 [Blender]

**来源**：OutLawArt Plan 03，wolf

wolf.fbx 自带 262 骨 Rigify metarig + 195 顶点组，且中性站姿、单 mesh、四脚基本落地。这类素材优先走"复用现成蒙皮"：Decimate COLLAPSE 可保 UV/权重（50k→12k 后形变干净），动作只需按现有骨名打 key，不必重跑自动权重。代价是导出 joints 可能偏多（wolf 262 joints，含脸/手指等冗余骨）；移动端若骨骼数/性能吃紧，再开单独任务精简骨架，别在首轮交付里冒险删骨丢权重。

### ART-9. Blender 5.1 FBX importer 可能被 LIGHT 节点击穿 [Blender]

**来源**：OutLawArt Plan 03，wolf

Blender 5.1.2 的 `io_scene_fbx` 导入某些含灯 FBX 时，会在 `blen_read_light()` 访问已不存在/不兼容的 `CyclesLightSettings.cast_shadow`，直接 `AttributeError` 让整个导入失败。wolf 用 `tools/blender/fbx_compat.py` monkeypatch：导入前包一层 light 读取，失败时退回普通 POINT 灯；资产流程随后删相机/灯。以后 headless 导入外部 FBX，先 `import fbx_compat`，不要把"灯节点坏了"误判成模型不可用。

### ART-10. rig 是 mesh 的父物体时，摆正/落地要防双旋和双平移 [Blender]

**来源**：OutLawArt Plan 03，wolf

wolf 第一次导出出现"骨骼和身体方向反了"：旧脚本先旋 rig，mesh 作为子物体已跟随一次，又单独旋 mesh，导致网格/骨骼相对方向错。修法：若 `mesh.parent == rig`，摆正前先 `parent_clear(KEEP_TRANSFORM)` 保留世界变换，再让 mesh/rig 各自只做一次旋转与落地。验证不要只看外形，要量骨骼关键点和绑定顶点方向是否一致（例如 nose 骨 head/tail 与 nose 权重点同向）。

### ART-11. Unity 端验证（gltFast 6.14.1） [Unity]

**来源**：OutLawArt Plan 01 实测

- **带 skin+动画的 glb 同样 gltFast 0 报错导入**：实例化即带 Animator（动画进来了）、SkinnedMesh 渲染贴图完整（非粉红）。沿用 ART-13，无需额外设置。
- 警告 `Shader 'Shader Graphs/glTF-pbrMetallicRoughness': fallback shader 'Hidden/Shader Graph/FallbackError' not found` **无害**，模型照常带贴图渲染。
- ⚠️ **Unity 首次启动可能 `No valid Unity Editor license found` 起不来**（licensing client 没初始化好）→ `pkill -f Unity.Licensing.Client` + `pkill Unity` 清进程后**重启一次即过**，不是真授权失效。
- 验证现场清理：MCP 截图存 `Assets/Screenshots/`，验完 `rm` 掉 png+meta+空目录、不存场景（不污染 OutLaw）。

### ART-12. blender-mcp 桥接（Blender 5.1.2 实测） [Blender→Unity]

**来源**：OutLaw Plan 03

ahujasid/blender-mcp 的 addon 标注最低 3.0，**实测 5.1.2 能跑**。架构：addon 在 Blender 进程内开 TCP **9876** socket，收发**裸 JSON**（无长度前缀），命令在主线程经 `bpy.app.timers` 执行 → **必须 GUI 模式**（`--background` 没事件循环 timer 不触发）。`uvx blender-mcp`（MCP server）只是把工具调用转发到这个 socket。
- **两条驱动路**：① 原生 MCP 工具（`.mcp.json` 加 `blender` server，但**要重启 Claude Code 才加载** `mcp__blender__*`）；② **直连 9876 socket**（`tools/blender/mcp_client.py`，走 addon 同协议，免重启，验桥/快迭代用这条）。
- **起桥坑**：GUI stdout **有缓冲**，启动 log 常不刷新 → **别等 log marker，`lsof -iTCP:9876` 看到 LISTEN 即桥起好**（硬证据）。免手点 N 面板：启动脚本里 `addon_utils.enable(...)` + `bpy.ops.blendermcp.start_server()`（见 `tools/blender/mcp_serve.py`）。

### ART-13. Blender→Unity glTF 管线 + Z-up/Y-up [Unity]

**来源**：OutLaw Plan 03

- **Unity 2022.3 内置不认 .glb/.gltf**（只原生 FBX/OBJ）→ 装 **`com.unity.cloud.gltfast`**（Unity 官方包，实测 6.14.1），.glb 即被 `GltfImporter` 当模型导入（mesh+材质子资产+prefab），0 报错。
- **Z-up/Y-up 自动搞定**：Blender 是 Z-up、Unity Y-up。Blender glTF exporter **默认 `+Y up`（`export_yup=True`）已做转换**，glTFast 导入后模型**直立、朝向/缩放正确**（实测 height 落在 Unity Y 轴，1 单位=1 米）。**无需在 Unity 端再转轴**。
- **原点/朝向约定**：建模时把脚底放 z=0、中心 x=y=0，再用 **3D cursor 设到 (0,0,0) + `origin_set(ORIGIN_CURSOR)`** → 原点在脚底中心。角色**正面（眼）朝 Blender -Y**，导出后在 Unity 里**正面朝 +Z**（= Unity forward 约定，`transform.forward` 指向眼看的方向）。viewer 想看正脸就把**相机放 +Z 侧朝 -Z**，别去转模型（转了破坏 forward 约定）。
- **可复现**：建模写成参数化 Blender Python 脚本提交（`tools/blender/build_pioneer.py`），既能经 socket 桥跑、也能 `Blender --background --python <脚本> -- <out.glb>` headless 出资产（**headless 更确定、导出用这条最稳**）。

### ART-14. 「回响态」ghost 在 Unity 侧做更灵活 [Unity]

**来源**：OutLaw Plan 03

"过去的你"用 **Built-in 管线 Fresnel rim 透明 shader**（`Assets/Art/Shaders/EchoGhost.shader`：两 Pass 先 Cull Front 再 Back，`Blend SrcAlpha OneMinusSrcAlpha` + `ZWrite Off`，边缘 `1-dot(N,V)` 描边发光）。**把同一 mesh 的所有材质槽替换成一个 Echo 材质**即得 ghost 变体——同轮廓认得出、半透+描边一眼区别于本体。比在 Blender 烤发光变体灵活（一个 shader 参数全调）。

### ART-15. Built-in 管线 emission 不自发光 [Unity]

**来源**：OutLaw Plan 03 待办

Blender 里给的 emission 材质（青色面罩/绿黏液点）导进 Unity **Built-in 管线、无 post-processing 时不会发亮**，只显基色。要真发光得**加 Bloom 后处理**（Built-in 用 Post-Processing Stack v2）**或换 URP**。当前 ArtViewer 没加 → 面罩/黏液点显白/暗，是已知项，留给美术调味阶段决定加 bloom 还是迁 URP。

### ART-16. game-ready glb 接进游戏角色：脚底原点/受击反馈/多渲染器三坑 [Unity]

**来源**：OutLaw Plan 08

把外来/AI 产线的 `.glb`（九尾狐）挂成敌人视觉，仿玩家 `AttachBody`（挂成逻辑根的 `"Body"` 子物体、碰撞体留逻辑根），踩到三个**任何模型接进来都会遇到**的坑：
- **脚底原点不可信**：ART-13 说"脚底原点"，但 AI/外部 glb 常**原点不在脚底**（九尾狐实测 `minY≈-0.152×scale`，直接挂陷地 0.24m）。**别写死偏移** → 挂模型时动态量渲染包围盒 `localPos.y = root.y - bounds.min.y` 把脚底抬到 y=0，对任意模型都稳（实测 `bodyMinY=0.000`）。
- **受击白闪在贴图模型上失效**：原 `EnemyBase` 受击设 `material.color=白`，URP/Lit 的 `_BaseColor` × 贴图 ≈ 无变化（看不见闪）。→ 贴图模型改**醒目色**（近战狐用红 `(1,0.25,0.2)`，红×贴图可见）。死亡变暗 `(0.25,…)` 仍可见。
- **单 `GetComponentInChildren<Renderer>` 漏 SkinnedMesh 多槽**：胶囊是单 MeshRenderer，模型是 SkinnedMesh（可能多槽）→ 改**收集全部渲染器**统一改色（`_renderers[]`/`_baseColors[]`）。
- 附带发现：`FlashAt`/`Health.OnDamaged` **此前是死代码**（敌人本无受击反馈）——换模型时才接上（仅近战怪，**射击怪零回归**：仍默认白闪、未接 OnDamaged）。
- **朝向纯视觉**：让模型 +Z 朝移动方向，只 `LookRotation` 转 `"Body"` 子物体，**不碰 `_rb.position`/`transform.position`（sim 决定论数据）** → 回放 `echoMaxDrift=0` 不受影响；胶囊碰撞径向对称，旋转不影响命中。

### ART-17. sim 驱动的敌人接 Animator：root motion off + 参数纯视觉不回写 sim [Unity]

**来源**：OutLaw Plan 09

给"位置由 sim 置位驱动"的敌人（MeleeEnemy）接动画状态机，**动画绝不能接管位移**（否则破回放零漂移）：
- **`Animator.applyRootMotion = false`**（模板构建期设）——否则 root motion 与 `SimTick` 的 sim 置位打架 → 漂移。验证：MELEE_TICK 日志全程 `rootMotion=False`、受击前后 `transform.position` 完全不变。
- 状态机参数（`isMoving` bool / `hit` trigger）**只读 sim 状态去 `SetBool`/`SetTrigger`，绝不回写 Snapshot/sim** → `echoMaxDrift=0` 不受影响。助手带 `_animator!=null && runtimeAnimatorController!=null` 双空保护 → 胶囊 fallback/缺资产安全跳过不崩。
- 控制器经 `UnityEditor.Animations.AnimatorController` API（execute_code）建，比 `manage_animation` 更精确控 transition 的 exitTime/trigger。clip `loopTime` 改 glb 导入设置麻烦 → 在控制器里用 `Hit→Idle hasExitTime=true exitTime=0.9` 让一次性动作播完即走，clip 内部是否循环无所谓。
- ⚠️ **取证教训**：别用 execute_code 在 Play 里**手动改 sim 实体**取证（手动 `TakeDamage` 撞上 GameOver/重开边界会冒伪 NRE，白白怀疑回归）；要日志证据就走 **verify-gated 打点**让它自然发生（见 META-12 / ART-16）。

### ART-18. 同一套敌人视觉挂载可服务近战/远程，但逻辑根尺寸要保持 [Unity]

**来源**：OutLaw Plan 10

把射击怪从胶囊换成 wolf 时，最稳做法不是让模型成为逻辑体，而是抽一个通用 `AttachEnemyBody(root,prefab,controller,...)`：逻辑根保留原碰撞/Health/Rigidbody/RangedEnemy，模型只作为 `"Body"` 子物体负责渲染、朝向和 Animator。这样近战狐/远程狼共用 bounds 脚底落地、`applyRootMotion=false`、多 Renderer 受击反馈等能力。

两条要点：
- **碰撞尺寸别被视觉模型带偏**：原射击胶囊是 root scale 下的默认 CapsuleCollider，换成空 root 后要显式补等价 CapsuleCollider（plan 10 用 `height=1.6/radius=0.35/center=0`）而不是拿 wolf bounds 当命中体；玩法/子弹命中/回放仍以逻辑根为准。
- **远程怪朝向目标只转 Body**：`RangedEnemy` 根据 `_lastAimDir` `LookRotation` 视觉体，移动/idle 由本 tick 是否实际位移驱动，受击触发 `FlashAt+TriggerHit`。这些全是纯视觉；不要改 `Snapshot`、`RoundManager`、`Sim3D` 或子弹逻辑。验证至少看 ranged `rootMotion=False`、`ENEMY_FIRE`、玩家中弹、近战狐回归和 `echoMaxDrift=0`。

---

## §AUDIO — 音频系统

### AUDIO-1. WebGL 自动播放必须有 AudioListener + 首次交互恢复 [Unity]

Unity WebGL 构建里 `AudioSource.Play()` 在浏览器交互前会被静默。实现：主相机挂 `AudioListener`，并在 `AudioManager` 里创建一次性输入监听对象，检测到任意键/鼠标/触摸后 `AudioListener.pause = false`，随后自毁。

### AUDIO-2. WebGL 下 AudioMixer + one-shot 不稳定 [Unity]

WebGL 对 `AudioMixerGroup` 的 one-shot 支持会导致"播放一次后无声"。解决：WebGL 运行时跳过 `outputAudioMixerGroup`，让 `AudioSource` 走默认输出；AudioMixer 资产仍保留给非 WebGL 平台或后续音乐/UI 分组。

### AUDIO-3. 用对象池 + PlayOneShot，避免 isPlaying 判断 [Unity]

WebGL 上 `AudioSource.isPlaying` 状态不可靠。用固定数量 `AudioSource` 轮询 + `PlayOneShot(clip)`，不判断 `isPlaying`。

### AUDIO-4. 短音效导入设置 [Unity]

`Force To Mono` + `Compression Format: Vorbis`（WebGL 推荐）+ `Preload AudioData=true`；单条 <200KB、总音频 <2MB，灰盒阶段够用。

### AUDIO-5. 占位音效可程序化生成 [Unity]

灰盒阶段用 `tools/audio/gen_sfx.py` 合成 6 条占位 WAV（noise/sweep/sine），总约 178KB；核心音效（玩家枪声）由用户提供，执行者只负责剪辑/重命名。

---

## §WEB — WebGL 构建 / 部署

> 人验一律走 WebGL 浏览器。出包：菜单 `Build/WebGL`（`GameBuilder.BuildWebGL`，输出 `Builds/WebGL`，Brotli + Decompression Fallback、Responsive 模板），再 `python3 -m http.server <port> --directory Builds/WebGL`（WebGL 不能 file:// 直开）。**构建经开着的编辑器触发**（batch build 抢 license 起不来）；BuildPlayer 同步阻塞主线程 → MCP 菜单调用会超时，优先用 `manage_build` 异步任务并轮询；若走菜单则起后台监控盯 editor.log 的 `[Build] result=` 判完成。

### WEB-1. 浏览器里看不到 console → 自建「遥测回传」让执行者直接读 [Unity]

WebGL 跑在用户浏览器，执行者看不到运行时状态/报错，靠人念数据极低效。**搭一条遥测**：构建里 `UnityWebRequest.Post` 把数据发到本机 `tools/web/telemetry_server.py`（带 CORS `*`，跨端口不被拦），执行者读 `/tmp/echo_input.log`。再 `Application.logMessageReceived += 转发 Exception/Error 到遥测` → **运行时异常直接拿到**（本任务就是靠这条一眼看出"BoxCollider doesn't exist"）。屏上叠加层（OnGUI 读原始 `Input.GetAxisRaw`/`GetKey`）也能让人不开 devtools 就读输入。脚手架默认关（`EchoInputDebug` 默认 0），调试时设 1。

### WEB-2. WebGL「黑屏」分三类，先分清是哪类 [Unity]

① **加载失败**：loader/wasm/data 非 200（用 `curl` 验 MIME，.wasm 要 `application/wasm`）。② **渲染黑**：Unity 播放器加载了（底部有 Unity/WebGL 条）、画布是相机背景色，但场景空 → 多半运行时异常（看遥测/WEB-1）或 shader 剥离（WEB-3）。③ **启动崩**：遥测一条没有 = 游戏逻辑没跑起来。先看「遥测有没有数据 + 服务器 curl」就能定位到类别。

### WEB-3. WebGL 会剥离「没被序列化资产引用」的 shader 和引擎模块 [Unity]

- **Shader 剥离**：运行时 `Shader.Find("Standard")` 取的材质，若没有任何序列化资产用到该 shader → build 里被剥 → `Find` 返回 null → 物件不可见（黑/品红）。修：把用到的 shader 加进 `ProjectSettings/GraphicsSettings.asset` 的 **Always Included Shaders**（Standard=fileID 46、Sprites/Default=10753、自定义 shader 用其 .mat.meta 里**材质自身**的 guid，别用 .mat 引用的 shader guid）。
- **引擎模块剥离（更隐蔽）**：本工程原是 2D，3D 物理组件全靠运行时 `AddComponent` 建、场景里**零序列化 3D 物理** → `Strip Engine Code` 把整个 **PhysicsModule 剥掉** → `AddComponent<BoxCollider>()` 返回 null → Bootstrap NRE → 黑屏。**增量构建会沿用旧模块集（蒙混过关），清缓存重建才暴露**。修：构建时 `PlayerSettings.stripEngineCode = false`（包从 12MB 涨到 31MB，灰盒可接受）；或在场景里放一个序列化的 3D 物理组件让模块被判定为"用到"。

### WEB-4. WebGL 输入/循环三连坑 [Unity]

- **键盘要焦点**：默认画布失焦键盘不进 Unity（"点一下动一下"）→ Bootstrap 里 `WebGLInput.captureAllKeyboardInput = true` 全局捕获。
- **失焦被节流**：`PlayerSettings.runInBackground` 构建默认 false → 画布失焦时浏览器把循环节流到慢动作（FixedUpdate 饿死、tick 龟速）→ 设 `true`。
- **`rb.MovePosition` 在 WebGL 几乎不挪 kinematic 体**（编辑器正常！）→ kinematic+脚本驱动的移动改**直接设 `rb.position`（+`transform.position`）**，和回响置位一致。拾取/触发同理：kinematic teleport 下 `OnTriggerEnter` 不可靠 → 改**距离判定**（GAME-23）。

### WEB-5. WebGL 缓存顽固：换端口或无痕，别只靠硬刷新 [Unity]

Unity WebGL 把构建缓存进浏览器 IndexedDB（按 origin），同 URL 反复换包时硬刷新(Cmd+Shift+R)清不掉、可能黑屏或加载旧包。**换端口**（`:8097` 与 `:8096` 是不同 origin → 全新缓存）或**无痕窗口**才彻底干净。验证迭代时每次换个新端口最省心。

### WEB-6. URP shader 别进 Always Included Shaders：会关掉变体剥离 → WebGL 构建卡数小时 [Unity]

迁 URP 后，沿用 WEB-3「把运行时 `Shader.Find` 的 shader 加进 Always Included Shaders」对 **URP shader 是灾难**：Always Included **绕过 URP 的 shader 变体剥离** → `Universal Render Pipeline/Lit` 的**整个变体矩阵（实测 ~294,912 个）**全量编译，WebGL 构建在 "N / 294912 variants ready" 卡到数小时（实测 10 分钟才 6.5%）。
- **正解**：URP shader 靠**序列化材质资产引用**进包（建一个 `Lit_Flat.mat` 用 URP/Lit，用序列化字段注入 `GreyboxBootstrap`，运行时 `new Material(它)` 而非 `Shader.Find`）。序列化引用让 shader 进包**且走正常剥离**（剥到 ~8192 变体，~1 分钟编完）。EchoGhost 同理靠 `Pioneer_Echo.mat`（场景引用）进包，**别**加 Always Included。
- **再砍变体**：URP 资产 `m_AdditionalLightsRenderingMode=Disabled`（只有 1 盏方向光时），去掉 `_ADDITIONAL_LIGHTS` 变体轴。
- 诊断：构建日志 `grep "variants ready"` 看数量级——上万是正常剥离，几十万就是 Always Included 漏进了 URP shader。

### WEB-7. 新 worktree 首次出 WebGL 包：先切平台，否则跨目标编译 `WebGLInput` 报 CS0103 [Unity]

新 worktree 的 active 平台默认 StandaloneOSX，编辑器编译时 `#if UNITY_WEBGL` 分支被排除（编辑器报 0 错、觉得没事）。直接 `Build/WebGL` 会跨目标编译该分支，`WebGLInput.captureAllKeyboardInput` 解析不到 → `error CS0103 ... 'WebGLInput' does not exist`，构建失败。
- **修**：出包前先 `manage_build action=platform target=webgl` 把 active 平台切到 WebGL（让编辑器以 WebGL 定义重编一遍），`read_console` 确认 0 CS 错，再 `Build/WebGL`。平台一切过就持久化在该 worktree，后续无需重切。

### WEB-8. 给别的手机/远程队友试玩：一键 localtunnel 出公网链接 [Unity]

要把本地 WebGL 包给**不在同一网络**的人玩（队友别的手机、招募对象），**不用局域网 IP、不用先上云**：本机 Node 起 localtunnel 即得公网 HTTPS 链接。
- **首选：双击 `tools/web/share-webgl.command`**（一键脚本，plan 06）——自动选空闲端口、起本地静态服务、取隧道密码、开隧道打印链接；Ctrl-C/关窗自动停服务**不留僵尸进程**。前提：先在 Unity 菜单 `Build/WebGL` 出包（脚本只分享 `Builds/WebGL/`，不构建）。
- 手动等价命令（脚本干的事）：
```bash
cd <Builds/WebGL>            # 当前要分享的构建目录
python3 -m http.server 8090 --bind 127.0.0.1 &   # 本地服务
npx -y localtunnel --port 8090                   # → your url is: https://xxxx.loca.lt
```
- 链接形如 `https://xxxx.loca.lt`（随机子串；脚本默认请求固定子域 `outlawplay`，被占用时自动随机）。
- ⚠️ **loca.lt 首访有拦截页**，要填 "Tunnel Password" = 本机**公网 IP**：`curl https://loca.lt/mytunnelpassword` 取（脚本已自动打印），填进去点 Continue。
- 前提：Mac 开着、http.server + 隧道进程在跑，**关了链接即失效**（临时分享用，不是持久托管）。
- ⚠️ **进程坟场坑**：手动跑 + 忘关会在端口 8080/8090… 堆一堆僵尸 http.server/lt（实际踩过 7+ 个）→ 一键脚本的 trap 清理就是治这个；手动跑务必收尾 `pkill -f "http.server"`/`pkill -f localtunnel`。
- 持久/稳定托管（赛事提交、长期分享）= **GitHub Pages `/outlaw/`（plan 06 通道③，已上线）**，用 `tools/web/deploy-github-pages.command` 更新；本脚本（通道②）是临时试玩快速路。代码协作仍走 **GitHub 私有库**（通道①，开发层，与试玩分享是两回事）。

### WEB-9. WebGL 包瘦身配方：strip + link.xml 保模块 + Brotli+Fallback [Unity]

小游戏 WebGL 包过大/加载慢（plan 07 前 49.4MB、wasm 33.7MB），三刀砍到 **11.5MB(-76.7%)、wasm 4.3MB**：
1. **`stripEngineCode=true` + Managed Stripping `High` + IL2CPP `OptimizeSize`**（Release）。**别再用 `stripEngineCode=false`** 绕模块剥离——那是 plan 04 的临时招，巨肥。
2. **运行时动态 `AddComponent` 的引擎模块**用 `Assets/link.xml` `<assembly fullname="UnityEngine.PhysicsModule" preserve="all"/>` 保住（本工程 3D 物理全靠运行时建，重开 strip 会触发 WEB-3 的 NRE）。**preserve 单个模块 ≪ 关全部 strip 的体积代价**。
3. **`compression=Brotli` + `webGLDecompressionFallback=true`**：fallback 让 loader **自带解压**，不依赖服务器 `Content-Encoding` → localtunnel/`http.server` 这种笨静态服务也能用压缩包（不破坏 WEB-8）。代价：loader.js 20KB→114KB（值）。
- ⚠️ **localtunnel 带宽有限**：包瘦 77% 但加载只降 30%（26→18s），**剩余瓶颈是隧道本身**，不是包——长期分享走 GitHub Pages（plan 06 通道③），临时试玩才用一键隧道脚本（WEB-8）。
- 构建运维坑：MCP 触发菜单构建会超时 + 桥重连**重放同一构建**（首包 totalErrors 计入桥断线日志，重试耗尽后 errors=0）；**Brotli 写盘期间文件暂显 0B，必须等 `brotli/bee_backend` 进程结束 + 尺寸稳定再量包**，别看旧 `[Build] result=` 就量。

### WEB-10. COS 新桶默认域名强制下载；GitHub Pages + Fallback 可直接托管 [Unity]

2024-01-01 后创建的腾讯云 COS 桶，即使开启静态网站且 `index.html` MIME 正确，默认 `cos-website` 域名仍可能返回 `Content-Disposition: attachment` + `x-cos-force-download: true`，浏览器只会下载；对象元数据无法覆盖这条平台策略，必须自定义域名。Plan 06 首传已用响应头坐实，因此按用户拍板改走现有 `JosephLE910.github.io`：游戏隔离在 `/outlaw/`，不覆盖博客。GitHub Pages 对 `.unityweb` 返回 `application/vnd.unity`、不设 `Content-Encoding`；配合 WEB-9 的 Decompression Fallback 可由 loader 自解压并正常渲染。更新后 Pages CDN 约有数分钟延迟，Unity IndexedDB 仍可能缓存旧包；自动验用 `?deploy=<commit>` 刷新 HTML，真人复验优先无痕窗口。

**以后统一效果验流程（2026-06-22 定）**：任务实现与自动验完成 → `Build/WebGL` → `tools/web/deploy-github-pages.command` → 执行者在线检查资源/MIME、场景渲染、浏览器 error/warning → 把固定 URL `https://josephle910.github.io/outlaw/` 给人实玩。反馈迭代继续覆盖同一 URL。local http server / localtunnel 只承担调试和临时排障，不作为"完成后看效果"的正式展示面。

### WEB-11. WebGL 出包：别用 delayCall 调度，且防 MCP 菜单重放 [Unity]

经 MCP 驱动出 WebGL 包的两个隐蔽坑：
- **`EditorApplication.delayCall += lambda` 调度构建会被静默丢弃**：lambda 活在 execute_code 的临时程序集里，编辑器一空闲卸了该程序集 → 回调指向已卸方法，**不执行也不报错**（白等十几分钟，log 在调度后静止、无 il2cpp/brotli 进程、Builds/WebGL 时间戳不变）。**正解：`execute_menu_item("Build/WebGL")`** 走持久程序集里的真实 MenuItem（同步阻塞会 MCP 超时，但构建在主线程真跑）。判"真在跑"硬证据：Builds/WebGL 被 `Directory.Delete` 删空 + 日志刷 `[x/y] Bee/Csc/shader variants` + il2cpp/brotli 进程出现。
- **菜单同步构建 MCP 超时 → 桥重连反复重放同一构建**（WEB-9 的量化）：实测重放 **7+ 次**（每次先删空 Builds/WebGL 再重建，停不下来），还会撞上部署时的删除竞态（`du: No such file`）。**治法**：① 看到一份**完整**产物（wasm/data 满尺寸 + brotli 进程清空）立刻 `cp -R` 快照到 /tmp，部署只认快照；② `kill` 掉该 worktree 编辑器斩断重放源（产物已快照，部署/在线验走 shell+curl 不需编辑器）。

---

## §MOBILE — 移动端打包

> 来源 Plan 22（Android APK 试玩）。真机=华为 Mate40Pro(Kirin 9000)。

### MOBILE-1. UE_LOG 在 Android logcat 上可靠 —— 是手机端的主数据通道 [UE]

桌面端 §DEBUG 说"UE_LOG 不可靠"，**那是桌面桥接 get-logs 的结论，安卓 logcat 不一样**。
`adb logcat` tag `UE` 每行带 UE 帧计数器 `[NN]`+时间戳，完全可靠。手机上 `/tmp` 不存在、文件写常被 scoped storage 静默吞，所以**遥测改走 UE_LOG**：每行 `UE_LOG(LogX, "[CARTEL:车名] <49列>")`，回收用 `adb logcat -d | grep CARTEL:车名 | sed 's/.*\] //'` 重建出与桌面同格式 CSV。按车名分流不混。帧率也直接从帧计数器/dt 算，不必手动 `stat fps`。

### MOBILE-2. 同目录同名 .umap + .uasset → 编辑器和 cook 选不同文件（静默不一致） [UE]

`Trashyard.umap`(在用) 和 `Trashyard.uasset`(历史 orphan) 同名共存：**编辑器加载 .uasset(旧)，打包 cook 用 .umap(新)**。改光照存进 .uasset → cook 没采用 → 手机上没变化，白查半天。
**排查法**：改完 reinstall+截图，没变 → cook 没用你改的文件，查重复 World。
**通用**：worktree/打包前先扫 `Content` 同名 .umap+.uasset，移走 orphan。

### MOBILE-3. 移动端运行时 LoadObject 的资产必须确保被 cook；自绘零依赖最稳 [UE]

单图 cook（`-map=单图`）只 cook 该图依赖，**漏掉 config 引用的资产**。引擎 `UTouchInterface` 虚拟摇杆贴图 `/Engine/MobileResources/HUD/*` 是 config 引用 → 没进包 → 运行时 `LoadObject` 返 null → 控件无图不画（代码无错却空白）。
确认：`find Saved/Cooked/Android_ASTC -iname '*VirtualJoystick*'` 为空。
**对策**：要么加进 always-cook，要么**纯 Slate 自绘**（`FSlateDrawElement::MakeBox/MakeLines`，零资产）。运行时 DMI 取硬引用材质(如 `BasicShapeMaterial`，被网格硬引用→必 cook)是安全的，区别于 config 引用贴图。

### MOBILE-4. 要收触摸的纯 C++ UserWidget 必须 SetVisibility(Visible) [UE]

纯 C++ `UserWidget`（无 WidgetTree）默认 **非 hit-test-visible**，触摸穿透，`NativeOnTouch*` 不触发。`NativeConstruct` 里 `SetVisibility(ESlateVisibility::Visible)` 才收得到。诊断靠 `[TOUCHUI]` logcat。

### MOBILE-5. 手机 SkyLight 别用 SLS_CAPTURED_SCENE（捕获 SkyAtmosphere → 死黑） [UE]

`SLS_CAPTURED_SCENE` 捕获 SkyAtmosphere 当环境光，但**手机上 SkyAtmosphere 不渲染**（需 IsSky 材质天空 mesh）→ 捕获全黑 → 全场只剩直射光，死黑看不清。
**修**：SkyLight 改 `SLS_SPECIFIED_CUBEMAP` + `/Engine/MapTemplates/Sky/DaylightAmbientCubemap`，恒定环境光，移动端正常。
⚠️ 仅修 SkyLight 不够——若同时有移动端自动曝光压暗（§MOBILE-12），画面仍黑。两者**共用**才恢复亮度。

### MOBILE-6. 单文件 APK 发朋友 + 桌面插件隔离 + Launcher 装版前置 [UE]

- **单文件分发**：`bPackageDataInsideApk=True` + `-clientconfig=Shipping` → 数据进 APK 不出分离 OBB，朋友点 APK 直装(开"未知来源")，无需数据线/adb/OBB。默认 Development 出**分离 OBB**，单装 APK 进得去但缺内容。
- **桌面专用插件别编进移动包**：`SundayDrive.uproject` 给 DeveloperTool 插件(如 SoftUEBridge)加 `PlatformAllowList=[Win64,Mac,Linux]`，比删源码干净。
- **Launcher 安装版 UE 缺 Android 二进制**：UBT 报 "Enable Android as an optional download component"，无 CLI 路径——**交用户**在 Epic Launcher 勾 Android 组件下载。Mac 上 UBT 从 `~/.zshrc` 的 `export ANDROID_HOME/NDKROOT/JAVA_HOME` 取 SDK；NDK 须对齐 UE 5.7 要求(实测 27.2.12479018)。

### MOBILE-7. Shipping 包剥离所有调试输出 → 给玩家看的反馈必须走 UMG/Slate [UE]

（Plan 23）`AddOnScreenDebugMessage`、`UE_LOG`、控制台、调试叠加在 **Shipping 包里全部不显示/编译掉**。
踩坑：胜负原来只 `AddOnScreenDebugMessage("YOU WIN/LOSE")`，朋友玩 Shipping 包"突然动不了、不知道死了"——根因就是反馈走了 debug 通道。
**铁律**：任何**面向玩家**的信息（胜负/死亡/受伤/提示）一律用真正的 widget。**面向自己**的自动验日志也别依赖 Shipping——验证用 Development 包，分发用 Shipping。

### MOBILE-8. Slate 内嵌 Roboto 字体单图 cook 必带，可放心画文字 [UE]

（Plan 23）`FCoreStyle::GetDefaultFontStyle("Bold", px)` 用的是 Slate 核心资源 Roboto，**单图 cook 也一定带**，真机渲染正常。
→ 自绘 UI 要画文字**不必引项目字体资产**（引了反而踩 §MOBILE-3 漏 cook）。文本居中用 `FSlateApplication::Get().GetRenderer()->GetFontMeasureService()->Measure()`。

### MOBILE-9. unlit 材质里烘进常量的发光色，C++ 改不动，得改材质资产本身 [UE]

（Plan 23）火花用 `M_Spark`（unlit），发光色是 `Constant3Vector` **常量节点**接 Emissive，**无暴露参数** → 运行时 `SetVectorParameterValue` 无效，C++ 改不了颜色。
**改法**：开 UE 编辑器走 SoftUEBridge `run-python-script`，`MaterialEditingLibrary.create_material_expression` 新建常量 → `connect_material_property(MP_EMISSIVE_COLOR)` → `recompile_material` → `save_asset`。
**坑**：该 UE 版 `Material` 没有 `get_expressions()`/`expression_collection`，**读不了现有节点**，只能新建+连线覆盖（旧节点留为未连线孤儿，不进 shader、无害）。要运行时可调就把材质参数化（`ScalarParameter`/`VectorParameter`）做成 DMI。

### MOBILE-10. 触屏滑过空白区别丢指针 → 否则要"抬手再点"才生效 [UE]

（Plan 23）多指触屏 widget：手指滑动经过非按钮空白区时，若 `ActiveTouches.Remove(ptr)` 丢掉这根指 → 滑到下一个键时该指已不在追踪表 → 键不响应，必须抬手重新点。
**改法**：滑过空白存 `-1`（保留追踪、当前不按任何键），滑到下一个键立刻重新生效。`IsPressed` 只认有效键号，`-1` 自然=不按。连续滑动操作（转向条/相邻键）尤其要这样。

### MOBILE-11. 中文 UI 文字要 cooked CJK 字体资产，Slate 默认 Roboto 无中文字形 [UE]

（Plan 24）§MOBILE-8 说 Slate 内嵌 Roboto 单图 cook 必带——但 **Roboto 没有 CJK 字形**，画中文出**方块**（桌面/手机都是）。
**对策**：要中文 UI 就引一个 cooked 的 CJK 字体资产（Plan 24 加了 `Content/Fonts/CJK_STHeiti.uasset`），用它的 FSlateFontInfo 画中文；该资产被 widget 硬引用 → 单图 cook 会带（区别于 config 引用，§MOBILE-3）。不想引资产就**菜单/HUD 用英文**（Plan 24 主菜单即英文）。Android 上更要确认字体进了包，否则方块。

### MOBILE-12. 移动端 Eye Adaptation 自动曝光压暗 → 修完 SkyLight 仍黑（双重根因） [UE]

（Plan 25）三个图都黑（含之前能看清的 Trashyard），修了 SkyLight `SpecifiedCubemap` 仍黑。第二个根因：**移动端 Eye Adaptation 自动曝光误判场景亮度，把画面压到极暗**。

**诊断**：`r.DefaultFeature.AutoExposure=0` 禁用自动曝光 → 画面恢复。可先 Mobile Preview 复现再确认（比反复打包快）。
**修**：`Config/Android/AndroidEngine.ini` `[ConsoleVariables]` 加：
```ini
r.DefaultFeature.AutoExposure=0
r.SkyLight.UpdateEveryFrame=1
```
**教训**：环境光丢失（只剩直射光+自发光可见）不一定是 SkyLight 的锅。排查顺序：先关 Auto Exposure 验证（改 Console Variable 秒级生效，比改 .umap 快），再查 SkyLight。

### MOBILE-13. `ConstructorHelpers::FObjectFinder` 只能在构造函数里用，运行时加载用 `LoadObject` [UE]

**来源**：Plan 27 — 在构造函数**之外**用 `FObjectFinder` 加载资产 → Android 真机 Fatal Error 闪退（桌面不崩，迷惑性强）。

`ConstructorHelpers::FObjectFinder/FClassFinder` 设计上**只在 `UObject` 构造函数里合法**。构造函数里用没问题（Cube mesh、材质硬引用都这么拿，随包 cook）；但**运行时/非构造函数**要加载资产，必须用 `LoadObject<T>(nullptr, TEXT("/Game/..."))`。
- 现象：桌面正常、Android 启动即 Fatal——抓 `adb logcat` 看 Fatal 栈。
- 两者都能让资产随包 cook（静态路径引用即可），区别只在"何时/何处加载"。

### MOBILE-14. 材质 Usage Flag 没勾 → cook 包里对应组件退回默认灰材质 [UE]

**来源**：Plan 30 — 子弹火花在 PC 上黄色发光、**手机上灰色无光**。查了半天 bloom/曝光都不是，真根因：`M_Spark` 没勾 **Used with Instanced Static Meshes**（`used_with_instanced_static_meshes`）。

`SparkBurst` 用 `UInstancedStaticMeshComponent`。材质要在某类组件上用，必须勾对应 **Usage Flag**（ISM/Skeletal/ParticleSprite/Niagara…）。没勾时：
- **编辑器/PIE**：UE 运行时**自动**补勾该 flag → 显示正常 → **PC 一直对**，问题被完全掩盖。
- **cook 包（手机）**：flag 没存进 .uasset → 该材质的对应 shader 变体没编 → 组件**退回默认灰材质**（WorldGridMaterial）→ 灰色、无 emissive、无光。
- **判别**：手机上材质显示成**灰色/默认** vs PC 正常 → 八成是 usage flag。颜色是"灰"而不是"暗的本色"是关键线索（本色变暗才是曝光/bloom）。
- **修**：Python `mat.set_editor_property("used_with_instanced_static_meshes", True)` → `recompile_material` → `save_asset`；或材质编辑器 Details→Usage 勾上保存。
- **教训**：移动端「材质显示不对（尤其变灰/默认）」**先查 usage flag / 材质 fallback**，别一上来归因 bloom/曝光/HDR。

### MOBILE-15. 移动端 scalability 把 bloom 关了（PostProcessQuality@0 → BloomQuality=0）→ HDR emissive 不发光 [UE]

**来源**：Plan 30 — HDR emissive（火花 7,6,0.25 / 发光标记）桌面 bloom 发光、手机无光晕。

真机 `sg.PostProcessQuality` 常落到 **0 档**，引擎 `Engine/Config/Android/AndroidScalability.ini [PostProcessQuality@0]` 把 `r.BloomQuality=0`（完全关 bloom）。桌面 `BaseScalability.ini` 同档是 4。
- **修**：`Config/Android/AndroidEngine.ini [ConsoleVariables] r.BloomQuality=4`。
- **优先级关键**：`[ConsoleVariables]`=SetByConsoleVariablesIni，**高于** SetByScalability，能压过运行时 scalability 重设；不必新建项目级 AndroidScalability.ini（那个**不一定 cook 进包**，别走这条）。也不用给 GameMode 加 PPV（PPV 只调 bloom 强度，BloomQuality cvar 才是 bloom pass 的总开关）。

### MOBILE-16. Mac↔安卓「天空/光照对齐」根治法：把不对称改对称，别单点补 cvar [UE]

**来源**：Plan 30 — 终结 Plan 22/25「手机偏暗/没天空」反复修的循环。根因是**两套管线 + 设置不对称**：
- **天空**：SkyAtmosphere **移动 forward 不渲染**（§MOBILE-5 同源）。靠它当天空 → 手机必然没天空。对称解：移除 SkyAtmosphere，加**移动端也能渲染**的天空（纯色/渐变 unlit 双面 sky sphere，或 cubemap），两端同款。sky sphere 半径要在移动端**远裁剪面内**（实测 1km OK）+ 大于玩法区 + 设 **NoCollision**（别挡车/射线）。
- **曝光**：别一端 auto 一端 fixed。两端都关 auto（`r.DefaultFeature.AutoExposure=0`）+ 玩法图无界 PPV `AEM_MANUAL` 固定曝光（manual **无 eye adaptation → 不会误判压黑**，解决 §MOBILE-12 的根）。实测移动 forward **吃 PPV manual 曝光**，不被 cvar 压掉。`ExtendDefaultLuminanceRange` 放 `[/Script/Engine.RendererSettings]` 是项目级两端共用。
- **对比基准**：理想是 Mac **Mobile Preview**（同管线）；但 Feature Level 切换没暴露给桥接（只能手点工具栏），且**真机才是 ground truth**——本次全程真机验证。曝光值/天空色桌面定起点、真机最终确认。
- **铁律**：凡影响光照/曝光/天空的设置，两端必须一致或平台无关；每加一个"安卓专属"补丁都在扩大不对称。改完三图（MainMenu/PCGCityTest/Trashyard）都要复验。

---

## §META — 流程 / 工具 / Skill

### META-1. 编辑器启动：直接可执行文件 > open -a [UE]

**来源**：Plan 09 §10.2 — macOS `open -a` 沙箱权限问题，启动 >5 分钟。

**正确命令**：
```bash
"/Users/Shared/Epic Games/UE_5.7/Engine/Binaries/Mac/UnrealEditor.app/Contents/MacOS/UnrealEditor" \
  "SundayDrive.uproject" &
```
12 秒内桥接就绪，无权限错误。

### META-2. PIE 截图方法 [UE]

**来源**：README §5 + Plan 12 实践。

- **不要**用 `HighResShot`（PIE 中常黑屏）
- **用桥接**：`soft_ue_cli capture screenshot --source pie-window --output-file PATH.png`
  直接截窗口像素，不等渲染管线，无黑屏问题
- 备选：`get-logs`（取 UE_LOG）与 CSV 遥测

### META-3. 编辑器桥接边界 [UE]

**来源**：Plan 08 §8.5 — 桥接无法模拟长按输入、无法在 PIE 中 set-property。

- `trigger-input` 是 press-and-release，无法长按油门测试加速
- "车不能动" 需人工 PIE 目视确认
- `save_current_level` 已 deprecated，用 `LevelEditorSubsystem` 替代（但 Python 未暴露）

### META-4. PIE Session 重启不重载代码 [UE]

**来源**：Plan 12 §9.5 — `pie-session stop` + `pie-session start` 后 session ID 不变，代码修改不生效。

**根因**：`pie-session stop` 只停止 PIE 世界，UE 编辑器进程中的缓存二进制仍在运行。

**解决**：改 C++ 后必须 `pkill -9 -f UnrealEditor` 再重启编辑器，不能仅靠 PIE stop/start。

**验证**：检查 `pie-session start` 返回的 `session_id` 是否变化。

### META-5. 规划 → 执行 → 经验 闭环 [跨引擎]

**来源**：流程优化（Plan 08/09 后引入）。

执行者必须：
1. 执行前验证代码默认值（Grep/Read 参数 → 对比 plan）
2. 执行后调用 `post-execution-report` Skill，写 **执行经验**
3. 如果改了 C++ 默认值，需额外 `set-property` + 保存关卡

### META-6. Skill 设计原则 [跨引擎]

**来源**：《如何写好 Skill》总结（AICoworker/）。

- 每个 Skill 含 `name` + `description`（YAML 头）+ 步骤检查点 + Few-Shot + 反模式
- 决策用 ASCII 分支图，不靠 LLM 推理
- 常见问题 ≠ SKILL.md FAQ（当且仅当决策流程无法通过结构消除歧义时才写 FAQ）

### META-7. UE4→UE5 迁移陷阱 [UE]

**来源**：Plan 12 §9.2 + §9.6，两者都踩了 UE5 API 变化。

| 陷阱 | UE4 | UE5 |
|------|-----|-----|
| Style 获取 | `FCoreStyle::Get()` | `FAppStyle::Get()` |
| Hit Actor | `Hit.Actor` | `Hit.GetActor()` |
| UE_LOG 特殊字符 | 无限制 | 中文箭头 `→` 等会导致编译错误（用英文 `to` 替代） |

**教训**：遇到 UE API 编译失败时，第一个怀疑是 UE4→UE5 命名变更。

### META-8. UMG Widget 可靠定位方式 [UE]

**来源**：Plan 12 §9.5 — `AddToViewport()` 在 UE5.7 中使用内部 SConstraintCanvas，
所有 UMG Slot API（`SetPositionInViewport` 等）静默失败。

**可靠方式**：作为 HUD CanvasPanel 的子控件加入：
```cpp
UCanvasPanel* RootCanvas = Cast<UCanvasPanel>(HudWidget->WidgetTree->RootWidget);
RootCanvas->AddChild(MinimapWidget);
UCanvasPanelSlot* Slot = Cast<UCanvasPanelSlot>(MinimapWidget->Slot);
Slot->SetAnchors(FAnchors(1.f, 0.f, 1.f, 0.f));  // 右上角
Slot->SetPosition(FVector2D(-216.f, 16.f));
Slot->SetSize(FVector2D(200.f, 200.f));
```

好处：随 HUD 生命周期管理（PIE 重启不泄漏）、标准 UMG API 可靠可用。

### META-9. Unity Build 会导致 file-static 函数名冲突 [UE]

**来源**：Plan 12 §9.2 — `TrashyardGenerator.cpp` 和 `PCGCityGenerator.cpp` 各有一个
file-static `AddBoxGeometry`。Unity build 或 adaptive non-unity 下两个同名函数会冲突。

**修复**：重命名 Trashyard 版本为 `AddBoxGeometryTrashyard`。
或者把所有 file-static 包进匿名 namespace `namespace {}`。但在 unity build 下匿名 namespace 仍可能冲突，
最可靠的是**函数名包含模块/类名前缀**。

### META-10. gitignore `Build/` 误伤源码 Build 目录；worktree 放大未追踪文件代价 [UE]

**来源**：Plan 20 — `-exec` worktree 首次编译炸，缺 `SoftUEBridge/.../Tools/Build/*.h`。

**根因**：`.gitignore` 的 `Build/`（排 UE 编译缓存）误伤了插件源码子目录 `Tools/Build/`，这些源码从未入库。单 worktree 时不暴露（磁盘有文件），但**新 worktree 不复制 git 未追踪文件** → 缺源码编译失败。

**修复**：`.gitignore` 加例外 `!**/Source/**/Build/` + `!**/Source/**/Build/**`。

**通用教训**：worktree 要求**所有源码入库**。任何"磁盘有但 git 没追踪"的源码/配置，新 worktree 都缺。搭 worktree 后第一件事：确认能在干净 checkout 上编译。

### META-11. 组件化重构「行为不变」的验证分级（纯输出 vs 反馈进仿真） [UE]

**来源**：Plan 28（REFACTOR：SimpleCarPawn 抽遥测+相机组件）。

重构要证"行为不变"，但证据强度按被抽职责的**性质分级**：
- **纯输出职责**（只读状态、往文件/视觉写，**绝不反馈进物理/控制**，如遥测、追尾相机视觉）：把组件做成**非 Tick 的"逻辑持有者"**、由原类**在原位逐行 verbatim 驱动**（不引入独立 `TickComponent`）→ 调用点/顺序/逻辑全没变，行为不变是**结构性保证**。此时 **构造性论证 + verbatim diff + 编译 + 物理核心 diff 未动** 即足够，不必跑帧级 CSV 基线。
- **会反馈进仿真的职责**（如血量/撞击 `OnBodyHit` 含 `AddImpulse`、AI 控制）：verbatim 论证不够强，**必须**用重构前后同输入的遥测 CSV 帧级对照实测。

**配套**：独立 `TickComponent` 会引入 Tick 时序漂移风险（§GAME 提过），重构求"行为不变"时优先"原位驱动"。先抽纯输出职责（最安全），反馈进仿真的留到能跑确定性基线后再动。

### META-12. 多 AI 接力：接手先 reset 到「最后已验证提交」，别叠未验证修复、别动构建中间产物 [跨引擎]

**来源**：Plan 30 — 执行中途换另一 AI 接手一段，它对同一 bug 误判方向、**连叠 4 个未验证提交**（还误判方向），并 `rm -rf Intermediate/Android` + 手动注 obb/gradle，**自以为打包链路坏了**（其实没坏）。接回后清理花的功夫比真修复还多。

- **接手第一步**：`git log` 找**最后一个真机/实测验证过的提交**，`git reset --hard` 到那里再单点续。未验证的"修复"是负资产，叠着只会放大混乱、难审难回滚。
- **一个 bug 一次只验一个假设**：改完**立刻真机验**再提交下一步。别在没验证的包上连续臆测（本次"被 scalability 覆盖""打包链路坏了"都是在打不进去的包上空想出的错判）。
- **别动构建中间产物排查问题**：`Intermediate/`、`Saved/Cooked/`、手动改 APK/obb/gradle 都是死路——它们是产物不是源。打包不对先回到 canonical 脚本（`scripts/package.sh`）干净重跑，而不是手撕产物。`bPackageDataInsideApk=True` 本就单包无 obb，别去"修"不存在的 obb 注入。
- **诊断归因顺序**（移动端"显示不对"）：先查**材质 usage flag / fallback**（§MOBILE-14）→ 再 scalability/cvar（§MOBILE-15）→ 最后才曝光/HDR。别一上来归因 bloom/曝光。

> 注：本条是**执行者侧**协作经验，写在经验库（LESSONS）。流程/规范的修订（WORKFLOW/RULES）是**规划者**的职责，执行者不擅自改。

### META-13. 2D→2.5D 维度迁移：模拟层留 2D，单点映射到 XZ [Unity]

**来源**：OutLaw Plan 04（REFACTOR）

行为不变的维度迁移，**别动模拟层**。建一个唯一映射边界 `Sim3D`（`ToWorld(v2)=(x,0,y)`、`ToSim(v3)=(x,z)`、`AimRotation(角)=Euler(0,90-角,0)` 让模型 forward(+Z) 指瞄准）。`Snapshot.Pos` 仍是 Vector2、录制/回放只认 sim 坐标 → **回放零漂移天然保住**（Step 0 + 全回归实测 echoMaxDrift=0.00000）。战斗脚本内部一律在 sim 平面算 AI/距离/命中（读 `ToSim(rb.position)`、写 `ToWorld(next)`），Weapon API 保持 Vector2 不变、调用点零改动。Physics2D→3D 全量迁，但全用 **Kinematic + 无重力**（沿用 GAME-23，零动力学）；玩家从 Dynamic 撞墙改 Kinematic + 钳位。**3D 命中盒赤道要落在 y=0**（子弹在平面 y=0 飞）：玩家/回响 CapsuleCollider 用 `center=Vector3.zero`，否则站立胶囊底部半径趋 0、侧向子弹打空。先做 Step 0 最小切片（仅玩家+1回响+相机）证漂移=0 再迁全套。

### META-14. 产线重资产量级适合 COS 单 AZ 归档 [跨引擎/Blender]

**来源**：OutLawArt Plan 02，2026-06-23 实跑

- 九尾狐 `gen/` 实测 `du -sh=219M`，460 个文件、实际 226,848,857 bytes；按 20–40 个资产、每个约 0.2–0.6GB，完整资产集约 4–24GB（中位约 9GB）。COS 桶容量无上限、对象数不限，这个规模没有容量风险。
- 腾讯云香港按量价（当天官方定价页切香港实查）：标准 ¥0.136/GB/月、低频 ¥0.09、归档 ¥0.033；外网下行 ¥0.75/GB，归档标准取回 ¥0.082/GB。4–24GB 归档仅约 ¥0.13–0.79/月；整库中位 9GB 偶尔标准恢复并下载一次约 ¥7.49。请求费在万次粒度，本项目可忽略。
- 个人新用户免费额度是 50GB 标准存储、180 天；只抵标准容量，不抵低频/归档、请求或流量。它是短期优惠，不应进入长期预算。Git LFS Free 虽含 10GiB 存储/下载额度，但 Blender/FBX 二进制每次改动会新增完整版本，容易越额；外接盘则不解决异地备份和分享。结论：长期不常读的 `gen/` 用 COS 归档划算。

### META-15. 归档桶必须和网站桶隔离，并做真实取回校验 [跨引擎/Blender]

**来源**：OutLawArt Plan 02，实跑

- 独立桶名强制 `outlaw-art-assets-*`、私有 ACL、不开静态网站；脚本使用独立变量 `COS_ARCHIVE_BUCKET` 并拒绝其他桶名，避免误碰 WebGL 桶。凭据只从 ignored `env/cos.env` 读取。
- `ARCHIVE` 最短计费 90 天、对象最小计费 64KB；读取前要恢复（快速 1–5 分钟、标准 3–5 小时、批量 5–12 小时）。不能只验上传列表：至少恢复并下载一个源文件，再和本地 SHA-256/字节做一致性比较。
- 重跑归档脚本不能无脑覆盖同名对象：覆盖等于删除旧归档版本，未满 90 天可能产生提前删除费。默认按对象键+大小跳过已有文件；只有确认同大小内容确实变化时才显式 `--force`。

---

## §PLAN — 规划者专属

> 本节给规划者（当前对话的 AI）在出 plan 之前读。不是给执行者用的。
> 📌 规划者**动作清单/规则**见 `PLANNER_RULES.md`；本节是带案例的细化经验。

### PLAN-1. 出 plan 前先搜代码库 [跨引擎]

**来源**：Plan 15 被撤销的教训 — 执行者提出"在三个文件加 SaveStringToFile"，规划者直接照写成 plan，
但项目已有 `SimpleCarPawn::OpenLogs()` + `WriteTelemetryLine()` 全套逐帧遥测基础设施。

**原则**：规划者收到需求后，在写 plan 之前，必须先用 Grep 搜相关代码，确认：
- 项目是否已有可复用的基础设施
- 执行者提的方案是"需求"还是"方案"（方案可能绕路）

**反例**：听到"需要日志"就开新日志文件；正解：听到"需要日志"→搜 `WriteTelemetry` →发现已有 →
plan 写"复用现有遥测，加 2 行即可"。

### PLAN-2. 区分需求与方案 [跨引擎]

**来源**：同上。执行者说的是"我需要在三个文件加 SaveStringToFile"，他是站在"我要自己实现"
的视角描述的。他的真实需求是"逐帧看到 AI 决策数据"，方案是"加日志"。

| 执行者说的（方案） | 真实需求 | 正确 plan |
|-------------------|---------|-----------|
| 在 RivalCarPawn/DeliveryMission/DriverBrain 加 SaveStringToFile | 逐帧知道 AI 走了哪个决策分支 | RivalCarPawn 复用 SimpleCarPawn 遥测，2 行 |

**原则**：规划者的职责不是批准执行者的实现方案，而是把执行者的需求翻译成更优方案。
如果执行者的方案与已有代码重复或绕路，plan 应该纠正方向，不是盖章通过。

### PLAN-3. 别凭估算下悲观结论否决方向，让一线实测 [跨引擎]

**来源**：Plan 19 — 规划者凭估算判定"RL 撑爆 16GB"，把内存设成 Step 0 硬门槛。
两个假设全错：
- torch Mac CPU 版仅 **0.27GB**（2GB 是 Win/Linux 的 `torch+cu124` CUDA 版）
- LA 训练器是**独立 Python 子进程**（`LearningExternalTrainer` + `train.py`），不在编辑器进程内

门槛轻松通过。**教训**：规划者在一线之外的估算只能作"风险提示"，不能作"否决"。
硬约束（内存/性能/可行性）写成"执行者先实测验证"的 gate，而非规划者拍板的结论。
这正是 plan 分层放权的依据——HOW 的可行性判断属于一线，规划者管 WHAT/验收。

---

## §FIX — Bug 修复

### FIX-1. GameMode 类型不匹配导致 PIE 崩溃 [UE]

**来源**：Plan 06 — `GlobalDefaultGameMode` 指向模板 BP，其 Controller CastChecked 崩溃。

**根因**：模板用的 `BP_VehicleAdvGameMode_C` 绑定了 `ASundayDrivePawn`，SimpleCarPawn 不是其子类。

**修复**：`GlobalDefaultGameMode` 改为 `/Script/SundayDrive.SimpleCarGameMode`。

**教训**：新建关卡 PIE 闪退时，先查 World Settings → GameMode，再查 ini 里的 GlobalDefaultGameMode。

### FIX-2. Mac 不支持热重载——改完 C++ 必须走完整重启流程 [UE]

**来源**：Plan 17 — 执行者改完 C++ 后试了 `Tools → Recompile C++ Source` 和 `build-and-relaunch`，都不支持，白绕一圈。

**Mac 上改 C++ 后的唯一合法流程**：
```bash
# 1. 关闭编辑器（用桥接或手动）
# 2. 编译
/Users/Shared/Epic\ Games/UE_5.7/Engine/Build/BatchFiles/Mac/Build.sh \
  SundayDriveEditor Mac Development \
  -Project="/Users/honghong/CodeWorkshop/SundayDrive/SundayDrive.uproject"
# 3. 重新打开编辑器
open "/Users/honghong/CodeWorkshop/SundayDrive/SundayDrive.uproject"
```

**每次改完 C++ 文件后，执行者必须先查本条再决定重编方式**——不要尝试任何"热重载"变体命令。

### FIX-3. 状态机切换时忘了隐藏上一阶段的标记物 [UE]

**来源**：Plan 26 — 玩家反馈警察模式接人时"黄色柱子和绿色柱子同时存在"。

`RefreshMarkers()` 中 Pickup 阶段 `BankMarker->SetVisibility(Circling || Pickup)`，但本意是 Circling 显示黄柱 → Pickup 黄柱消失、绿柱出现。少写了一个条件导致两个同时可见。

**教训**：改状态机可视标记物时，**逐阶段验证**每个 marker 的显隐，不要假设"上一阶段的会自动消失"。最好写个简单的 phase→visible markers 对照表作注释。

### FIX-4. `FHitResult::ImpactPoint` 是 `FVector_NetQuantize`，不是 `FVector` [UE]

**来源**：Plan 27 — `SimpleCarPawn::OnBodyHit` 里 `Hit.ImpactPoint.IsNearlyZero()` 编译报 `incompatible operand types`。

`ImpactPoint`/`ImpactNormal` 等是 `FVector_NetQuantize`（网络量化类型），不能直接当 `FVector` 调方法/做运算。**先 `FVector(Hit.ImpactPoint)` 显式转换**再用。

---

## §DEBUG — 通用调试

> 当 AI、碰撞、任务、渲染等任何系统出问题时，先用这一节的方法工具。

### DEBUG-1. UE 日志三通道对照 [UE]

| 通道 | API | 如何取 | 可靠性 |
|------|-----|--------|--------|
| Output Log | `UE_LOG(LogTemp, Warning, ...)` | 桥接 `get-logs` | **不可靠** — Plan 15 实测返回 0 条，可能被过滤 |
| 屏幕文字 | `GEngine->AddOnScreenDebugMessage` | 肉眼 / 截图 | **不可靠** — PIE 中可能不渲染或被遮挡 |
| 写文件 | `FFileHelper::SaveStringToFile(..., FILEWRITE_Append)` | `cat /tmp/xxx.log` | **可靠** — DeliveryMission 已有 `/tmp/dm_debug.log` 先例 |
| 逐帧 CSV | `SimpleCarPawn::OpenLogs()` + `WriteTelemetryLine()` | `head /tmp/car_debug.csv` | **可靠** — 49 列，`telemetry-review` Skill 可读 |

**原则**：排查运行时 bug，优先用文件通道，不依赖 UE_LOG 和屏幕文字。

### DEBUG-2. 调试前先查有无已有基础设施 [跨引擎]

**来源**：Plan 15 撤销 — 执行者想在三个文件新建日志系统排查 AI 卡死。

**原则**：调试某个系统前，先搜代码库：
- `Grep "WriteTelemetry|OpenLogs|SaveStringToFile|bLogTelemetry"` — 项目已有的日志/遥测
- 如果要调试的是 `SimpleCarPawn` 子类，先看父类是否已有遥测（`bLogTelemetry` / `OpenLogs` / `WriteTelemetryLine`）
- 复用已有 > 新建。2 行修复 > 三个文件加日志。

### DEBUG-3. 数据驱动调试流程 [跨引擎]

```
加日志 → PIE → 读日志 → Python 分析 → 结论
                                ↑
                     telemetry-review Skill 做这步
```

`data-driven-debug` Skill 已将此流程标准化（5 个 Phase）。需要 Python 分析 CSV 时别手写 grep，用 Skill。

### DEBUG-4. 先看数据再猜原因 [跨引擎]

**来源**：Plan 13 §8.5 — AI 车"卡在出生点"排查 4 轮。

| 轮次 | 假设 | 手段 | 结果 |
|------|------|------|------|
| 1 | 卡死检测冷静期不够 | 加 Grace Period | 无效 |
| 2 | 撞墙刹车锁死 | 冷静期跳过撞墙 | 无效 |
| 3 | Mission 没生成车 | bridge get-property | 假阴性 |
| 4 | **看 Telemetry CSV** | 两行补丁复用遥测 | **2 分钟定位** |

**根因**：车根本没卡在出生点。它正常开到了目标附近，但 NavMesh 路径终点与 PickupLocation 不匹配，原地打转。

**原则**：前 3 轮都是"猜→改→PIE→无效"循环。第 4 轮拿到 CSV，一眼看出时间线，根因完全不是猜测的方向。**加日志看数据，永远比猜原因快。**

### DEBUG-5. SaveStringToFile append 默认写 BOM，污染 CSV [UE]

**来源**：Plan 21 — `FFileHelper::SaveStringToFile` 默认 `ForceUTF8`，**append 模式每次调用都写 `EF BB BF`**，逐行 append 的遥测 CSV 每行被插 BOM，第一列脏。

**修**：逐行 append 用 `EEncodingOptions::ForceUTF8WithoutBOM`：
```cpp
FFileHelper::SaveStringToFile(Line, *Path,
    FFileHelper::EEncodingOptions::ForceUTF8WithoutBOM,
    &IFileManager::Get(), FILEWRITE_Append);
```

### DEBUG-6. UE5.7 Python line_trace 没命中返回 None；HitResult 用 to_tuple() [UE]

**来源**：Plan 20/21 — 桥接 Python 射线检测。
- `line_trace_single` **没命中返回 `None`**（不是空 HitResult），先判 None 再用。
- HitResult 字段**不走** `get_editor_property`，用 `to_tuple()`：`[0]=bBlockingHit`、`[5]=ImpactPoint`。

### DEBUG-7. bridge 自动跑「确定性 PIE 基线」目前不可用（三坑） [UE]

**来源**：Plan 28 — 想用 bridge 自动跑 PIE + 帧级遥测做重构回归基线，连撞三坑：
1. **`pie-session` 实时跑**：编辑器**失焦后台降频**，15s 墙钟只推进 ~1 帧（遥测仅 2 行）→ 跑不出有效轨迹。
2. **`pie-tick` 帧步进**：任务玩法 `start` 即 `SetGamePaused(true)`（GET READY 门）→ 报 `world time did not advance`。
3. 偶发**冻住 game thread**，bridge 无响应，只能 `kill` 编辑器重启。

**现状**：确定性 PIE 基线本机做不了。纯输出重构可绕过（见 §META-11）；要给"反馈进仿真"的重构做帧级基线，得**先单列 META plan 补 bridge 能力**（关后台降频 + 一个 start 不 pause 的测试入口），否则后续 REFACTOR 都卡这。

### DEBUG-8. 编辑器失焦 → Play 模式被节流，自动验证会"卡住" [Unity]

**来源**：OutLaw Plan 01 — 通过 MCP 进 Play 跑自动验证时，编辑器不在前台会被节流到近乎停（FixedUpdate 几乎不推进），日志卡在第一轮。**修复**：运行时设 `Application.runInBackground = true`（在 Bootstrap.Awake 里），后台即满速跑。

### DEBUG-9. 功能验证打结构化日志 [Unity]

**来源**：OutLaw Plan 01 — 统一前缀 `[EchoVerify] KEY k=v` 打 log，`read_console(filter_text="EchoVerify", format=plain)` 捞——省 token、好机器断言（位置/命中/instanceID/死亡截断都能直接读出来证明）。改脚本后先 `refresh_unity(wait_for_ready=true)` + `read_console(types=[error])` 确认 0 编译错误，再用新类型。

### DEBUG-10. `MovePosition` 与直接设 `rb.position` 同帧混用 = 互相抵消 [Unity]

**来源**：OutLaw Plan 02 — `rb.MovePosition(target)` 之后同帧又 `rb.position = x`（如钳位回当前位置）→ **直接赋值取消了 MovePosition**，物体原地不动（最隐蔽的"敌人不动"真因）。修：钳位作用在 **MovePosition 的目标**上（`next=Clamp(rb.position+move); rb.MovePosition(next)`），别赋 `rb.position`。二选一，别同帧混用。

### DEBUG-11. `AddComponent<T>()` 在 active 物体上立刻跑 `T.Awake()` [Unity]

**来源**：OutLaw Plan 02 — 在已 active 的物体上 `AddComponent<Health>()` 会**当场触发 `Awake()`**，用默认值初始化；之后再设字段（`health.max=20`）不会回头更新 Awake 里依赖该字段算出的状态（Current 仍是旧的 5）。→ 开局不满血类 bug。修：要么 AddComponent **前**设好、要么之后显式 `Reset()`。同理：`Awake` 里订阅的属性若在更晚的 `Spawn()` 才赋值，模板构建期会 NRE → 用 `GetComponent` + 空保护。

### DEBUG-12. `public` 序列化字段会被场景存档**覆盖**代码默认值 [Unity]

**来源**：OutLaw Plan 02，重要 — 改了 .cs 里 `public float arenaHalf` 的默认值，但场景里 `Bootstrap` 早把旧值序列化存了 → **运行时用的是场景里的旧值**，代码默认形同虚设（plan 02 的"扩大地图"一度根本没生效）。所以 GAME-21「改参数只动 .cs」**对已序列化进场景的字段无效**。这类全局尺寸：要么集中**由一处注入**（如 Bootstrap 注入 RoundManager 再下发），要么干脆别留 public 序列化给场景存。

### DEBUG-13. UPM 包别用本机 `file:` 绝对路径——队友 clone 必崩 [Unity]

**来源**：OutLaw 2026-06-23，队友实报 — `Packages/manifest.json` 里把 MCP 包写成 `"com.coplaydev.unity-mcp": "file:/Users/honghong/unity-mcp/MCPForUnity"`（本机绝对路径）提交进 git → **别的队友 clone 后机器上没这个路径** → Unity 开工程弹 `Package Manager Error: package.json cannot be found`。机器专属路径**绝不能进共享仓**。
- **修**：换成**钉死 commit 的 git URL**：`"https://github.com/CoplayDev/unity-mcp.git?path=/MCPForUnity#<commit>"`（从 `git -C ~/unity-mcp remote -v` 拿上游、`rev-parse HEAD` 拿 commit）。任何机器都能从 GitHub 解析、锁定同一版本不漂移。同步**删 `packages-lock.json` 里该包的旧 `source:"local"` 块**让 Unity 重解析（git 依赖的 lock 格式带 hash，别手写）。
- 适用任何本地包/工具：MCP 这类**只有部分人用的开发工具**也照此（git URL 让所有机器可解析，纯编辑器工具不影响 build；游戏代码不引用它）。