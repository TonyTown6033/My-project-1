# My-project-1 — Unity 2D 动作游戏

## 项目概述

这是一个 Unity 2D 横版动作游戏，玩家控制角色（当前模型为 "Devil"）在场景中移动、跳跃、冲刺并攻击敌人。包含完整的敌人 AI、子弹系统、HUD、对象池和存档系统。

## 技术栈

| 层级 | 技术 | 版本 |
|------|------|------|
| 引擎 | Unity | 6000.3.10f1 (Unity 6) |
| 语言 | C# | — |
| 渲染管线 | URP (Universal Render Pipeline) | 17.3.0 |
| 输入系统 | Input System | 1.18.0 |
| 2D 动画 | 2D Animation / PSD Importer | 13.0.4 / 12.0.1 |
| UI | UGUI + TextMesh Pro | — |
| 虚拟摇杆 | Joystick Pack | — |
| 补间动画 | iTween | — |
| 云构建 | Unity Cloud Build | 2.0.7 |

## 场景结构

| 场景 | 用途 |
|------|------|
| `StartScene` | 主菜单（新游戏 / 继续 / 退出） |
| `Scene01` | 主游戏场景 |

场景切换：`SceneManager.LoadScene(1)` 从主菜单跳转到 Scene01。

## 目录说明

```
Assets/
├── Scripts/
│   ├── Common/          # 全局工具：ApplicationController（单例）、Tags 枚举、FindChildInAnyLayer
│   ├── StartSystem/     # 主菜单逻辑：StartSystem.cs
│   ├── Player/          # 玩家全部逻辑（见下方详解）
│   ├── Enemy/           # 敌人 AI、对象池、生成器
│   ├── Bullet/          # 子弹对象池与碰撞检测
│   ├── HUD/             # 血量条、HUD 对象池
│   ├── Effect/          # 特效播放与对象池
│   └── DataSystem/      # 存档读写（明文文本文件）
├── Scenes/              # Unity 场景文件
├── Resources/
│   └── Prefab/          # 通过 Resources.Load 动态加载的 Prefab（子弹、敌人等）
├── Settings/            # URP 渲染器设置
└── [第三方资产目录]      # Cainos、Anogame、Space_Exploration_GUI_Kit 等
```

## 核心架构

### 单例模式（项目全局统一用法）

所有跨场景管理器均使用同一套 `Awake` + `DontDestroyOnLoad` 单例模式：

```csharp
static Foo instance;
void Awake() {
    if (instance == null) { instance = this; DontDestroyOnLoad(this); }
    else if (instance != this) Destroy(gameObject);
}
public static Foo Instance { get { /* lazy init */ } }
```

主要单例：`ApplicationController`、`PlayerProperty`、`BulletPool`、`EnemyPool`、`UserSaveData`、`EnemySpecialEffect`、`SpecialEffectPlay`

### 对象池模式（统一用法）

子弹、敌人、HUD 均使用 `Dictionary<TEnum, Queue<GameObject>>` 结构：
- `GetXxx(type, ...)` — 从池取出或 `Resources.Load` 实例化
- `AddXxx(type, go)` — 归还对象池（`SetActive(false)`）

### 玩家系统

| 脚本 | 职责 |
|------|------|
| `PlayerProperty` | 单例数据中心：血量、速度、状态、动画器引用 |
| `PlayerMovement` | 移动、跳跃、冲刺（Coroutine），读取 VariableJoystick |
| `PlayerAction` | 普通攻击、受击处理、无敌帧 |
| `PlayerJumpProcess` | 落地检测，更新 `isGrounded` |
| `PlayerBuff` | Buff 效果（待扩展） |

玩家状态机：
- `PlayerState` 枚举：`Front`、`Side`、`Back`（对应不同骨骼动画节点）
- `PlayerStatus` 枚举：`idle`、`walk`、`suprise`、`jump`、`talk`、`sleep`、`damaged`、`sprint`

### 敌人系统

| 脚本 | 职责 |
|------|------|
| `EnemyDecision` | AI 状态机：巡逻 → 追击 → 攻击 → 受击 → 睡眠 |
| `EnemyProperty` | 敌人数据（速度、攻击力、巡逻点等） |
| `EnemyPool` | 对象池管理 |
| `EnemyGenerate` | 定时生成敌人 |
| `EnemyAttackCheck` / `EnemyColliderCheck` | 触发器检测攻击/碰撞范围 |

`EnemyType`：`CloseCombatLittleEnemy`、`RangedCombatLittleEnemy`
`EnemyStatus`：`Patrolling`、`Pursite`、`Attack`、`Sleep`、`Idle`、`Damaged`

### 子弹系统

`BulletPool.GetBullet(attackPower, firePoint, moveType, bulletType, startPos, direction)` — 统一发射入口。

子弹 Prefab 存放于 `Resources/Prefab/{BulletsType}.prefab`，按需动态加载。

### 存档系统

明文文本文件，路径：`Application.persistentDataPath + "/playerInfo.txt"`

格式：
```
UserName:xxx
UserID:0
Score:0
```

读写由 `UserSave`（写）和 `StartSystem.Event_B_Continue`（读）负责。

## 命名规范

| 类别 | 规范 | 示例 |
|------|------|------|
| 类名 | PascalCase | `PlayerProperty`, `EnemyDecision` |
| 枚举 | PascalCase | `PlayerState`, `BulletsType` |
| 私有字段 | camelCase | `enemyRB`, `bulletPool` |
| 公有字段 | camelCase | `hp`, `attackPower` |
| 方法 | PascalCase | `GetBullet`, `ChangeStatus` |
| UI 事件方法 | `Event_B_XXX` | `Event_B_Start`, `Event_B_Suprise` |
| 动画事件方法 | `AnimEvent_XXX` | `AnimEvent_PlayerNormalAttackBullet` |

## 开发注意事项

- **Resources.Load 路径**：子弹和敌人 Prefab 必须放在 `Assets/Resources/Prefab/` 下，文件名与枚举值完全一致。
- **Tags**：所有 GameObject Tag 通过 `Tags` 枚举定义，用 `Tags.Player.ToString()` 方式引用，避免硬编码字符串。
- **动画参数**：玩家动画参数在启动时转为 Hash 缓存（`sideParamterHash`、`frontParamterHash`、`floatParamterHash`），新增参数需同步更新对应枚举。
- **输入**：同时支持虚拟摇杆（移动端）和键盘（Space=攻击、K=跳跃、Z=冲刺）。
- **帧率**：`ApplicationController` 在 `Awake` 中锁定 60 FPS，关闭 VSync。
- **iTween**：UI 淡出效果使用 iTween，见 `UserSave.FadeOut()`。

## 常用操作

- **打开项目**：用 Unity Hub 选择 Unity 6000.3.10f1 打开本目录
- **运行游戏**：从 `StartScene` 开始运行
- **添加新敌人类型**：在 `EnemyType` 枚举添加条目 → 在 `Resources/Prefab/` 放对应 Prefab
- **添加新子弹类型**：在 `BulletsType` 枚举添加条目 → 在 `Resources/Prefab/` 放对应 Prefab
