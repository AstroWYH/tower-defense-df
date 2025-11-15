# GameFramework 快速学习指南

> 基于塔防Demo项目的GameFramework框架学习指南

## 📚 目录

1. [框架核心理念](#框架核心理念)
2. [核心组件系统](#核心组件系统)
3. [游戏启动流程（Procedure系统）](#游戏启动流程procedure系统)
4. [实体系统（Entity System）](#实体系统entity-system)
5. [数据层架构（三层设计）](#数据层架构三层设计)
6. [事件驱动通信](#事件驱动通信)
7. [学习路线图](#学习路线图)
8. [关键文件索引](#关键文件索引)
9. [实战练习](#实战练习)

---

## 框架核心理念

### 组件化架构

GameFramework 是一个**组件化驱动**的框架，所有功能都通过 `GameEntry` 单例访问：

```csharp
// 访问任何组件都是这样的模式
GameEntry.{Component}.Method()

// 例如：
GameEntry.Entity.ShowEntity(...);      // 显示实体
GameEntry.UI.OpenUIForm(...);          // 打开UI
GameEntry.Event.Fire(...);                // 触发事件
GameEntry.DataTable.GetDataTable<...>(); // 获取数据表
```

### 关键入口文件

- **`Assets/GameMain/Scripts/Base/GameEntry.cs`** - 游戏入口（MonoBehaviour）
- **`Assets/GameMain/Scripts/Base/GameEntry.Builtin.cs`** - 内置组件（框架提供）
- **`Assets/GameMain/Scripts/Base/GameEntry.Custom.cs`** - 自定义组件（项目扩展）

---

## 核心组件系统

### 内置组件（Builtin Components）

这些是框架提供的核心功能，通过 `GameEntry.Builtin.cs` 访问：

| 组件 | 作用 | 常用方法 |
|------|------|---------|
| **Procedure** | 流程管理（FSM状态机） | `StartProcedure<T>()` |
| **Entity** | 实体管理（塔、敌人、子弹） | `ShowEntity()`, `HideEntity()` |
| **DataTable** | 数据表（Excel导出的配置） | `GetDataTable<T>()` |
| **Event** | 事件系统（组件间通信） | `Subscribe()`, `Fire()`, `Unsubscribe()` |
| **Resource** | 资源管理（AssetBundle加载） | `LoadAsset()`, `UnloadAsset()` |
| **UI** | 界面管理 | `OpenUIForm()`, `CloseUIForm()` |
| **ObjectPool** | 对象池（性能优化） | `Spawn()`, `Unspawn()` |
| **ReferencePool** | 引用池（避免GC） | `Acquire<T>()`, `Release()` |
| **Localization** | 本地化（多语言） | `GetString()` |
| **Sound** | 声音管理 | `PlaySound()` |
| **Scene** | 场景管理 | `LoadScene()` |

### 自定义组件（Custom Components）

项目扩展的功能，通过 `GameEntry.Custom.cs` 访问：

- **BuiltinData** - 内置数据（构建信息、默认字典）
- **Item** - UI元素管理
- **Data** - 游戏数据管理（中心数据层）

---

## 游戏启动流程（Procedure系统）

### Procedure 是什么？

Procedure 是 GameFramework 的**流程管理系统**，基于**有限状态机（FSM）**实现，控制整个游戏的流程切换。

### 游戏启动流程链

```
Launch → Splash → CheckVersion → UpdateVersion → 
CheckResources → UpdateResources → Preload → Menu → Level
```

### Procedure 生命周期

每个 Procedure 都继承自 `ProcedureBase`，有以下生命周期方法：

```csharp
public class ProcedureLaunch : ProcedureBase
{
    // 1. 初始化（只执行一次）
    protected override void OnInit(ProcedureOwner procedureOwner) { }
    
    // 2. 进入流程
    protected override void OnEnter(ProcedureOwner procedureOwner) 
    {
        // 初始化语言、变体、默认字典等
    }
    
    // 3. 每帧更新
    protected override void OnUpdate(ProcedureOwner procedureOwner, 
        float elapseSeconds, float realElapseSeconds) 
    {
        // 检查条件，切换到下一个流程
        ChangeState<ProcedureSplash>(procedureOwner);
    }
    
    // 4. 离开流程
    protected override void OnLeave(ProcedureOwner procedureOwner, bool isShutdown) { }
    
    // 5. 销毁
    protected override void OnDestroy(ProcedureOwner procedureOwner) { }
}
```

### 流程切换方式

```csharp
// 切换到下一个流程
ChangeState<ProcedureSplash>(procedureOwner);
```

### 关键 Procedure 文件

- `ProcedureLaunch.cs` - 启动流程（初始化语言、变体）
- `ProcedureSplash.cs` - 闪屏流程
- `ProcedureMenu.cs` - 主菜单流程
- `ProcedureLevel.cs` - 关卡流程（游戏主逻辑）

---

## 实体系统（Entity System）

### 什么是实体系统？

实体系统是 GameFramework 的**对象池驱动**的游戏对象管理系统。所有游戏对象（塔、敌人、子弹）都通过实体系统管理，自动处理对象池的创建和回收。

### 实体三件套

每个实体由三部分组成：

1. **Prefab（预制体）** - `Assets/GameMain/Entity/` 目录下
2. **EntityData（实体数据）** - 初始化参数（位置、旋转、用户数据）
3. **EntityLogic（实体逻辑）** - 行为实现（继承自 `EntityLogic`）

### 实体生命周期

```csharp
public abstract class EntityTowerBase : EntityLogic
{
    // 1. 初始化（只执行一次，用于绑定组件）
    protected override void OnInit(object userData) 
    {
        // 获取组件引用
        CachedAnimation = GetComponent<Animation>();
    }
    
    // 2. 显示实体（每次从对象池取出时调用）
    protected override void OnShow(object userData)
    {
        // userData 就是 EntityData
        EntityDataTower entityData = userData as EntityDataTower;
        
        // 设置位置
        CachedTransform.localPosition = entityData.Position;
        
        // 订阅事件
        GameEntry.Event.Subscribe(UpgradeTowerEventArgs.EventId, OnUpgradeTower);
    }
    
    // 3. 每帧更新
    protected override void OnUpdate(float elapseSeconds, float realElapseSeconds) 
    {
        // 游戏逻辑
    }
    
    // 4. 隐藏实体（回到对象池时调用）
    protected override void OnHide(bool isShutdown, object userData)
    {
        // 取消订阅（防止内存泄漏）
        GameEntry.Event.Unsubscribe(UpgradeTowerEventArgs.EventId, OnUpgradeTower);
        
        // 清理引用，准备下次复用
        entityDataTower = null;
    }
}
```

### 如何显示一个实体

#### 方式1：使用 EntityExtension（推荐）

```csharp
// 显示一个敌人实体
GameEntry.Entity.ShowEntity<EntityEnemy>(
    serialId: 1,
    enumEntity: EnumEntity.BuggyEnemy,  // 实体类型枚举
    userData: EntityDataEnemy.Create(position)  // 实体数据
);
```

#### 方式2：使用 EntityLoader（管理多个实体）

```csharp
// 创建实体加载器
EntityLoader entityLoader = EntityLoader.Create(this);

// 显示实体（带回调）
entityLoader.ShowEntity<EntityEnemy>(
    EnumEntity.BuggyEnemy,
    (entity) => {
        Debug.Log("敌人加载成功！");
        EntityEnemy enemy = entity.Logic as EntityEnemy;
    },
    EntityDataEnemy.Create(position)
);

// 隐藏实体（会自动回对象池）
entityLoader.HideEntity(serialId);

// 清理所有实体
entityLoader.ReleaseAllEntity();
```

### 实体数据（EntityData）

```csharp
// 创建实体数据（使用引用池）
EntityDataTower entityData = EntityDataTower.Create(
    position: Vector3.zero,
    rotation: Quaternion.identity,
    userData: tower  // 自定义数据
);

// 使用完后会自动回池，无需手动释放
```

---

## 数据层架构（三层设计）

这是 GameFramework 中**最重要的数据管理模式**：

```
┌─────────────────────────────────────┐
│  第一层：DataTable（DRX 类）          │
│  - 自动生成，从 Excel 导出            │
│  - 只读配置数据                       │
│  - 例如：DRTower.cs                  │
└─────────────────────────────────────┘
              ↓ 初始化
┌─────────────────────────────────────┐
│  第二层：Runtime Data（XData 类）     │
│  - 运行时数据结构                     │
│  - 封装业务逻辑（多语言、计算属性）    │
│  - 例如：TowerData.cs                │
└─────────────────────────────────────┘
              ↓ 管理
┌─────────────────────────────────────┐
│  第三层：Data Manager（DataX 类）     │
│  - 中心化管理                         │
│  - 通过 GameEntry.Data 访问          │
│  - 例如：DataTower.cs                │
└─────────────────────────────────────┘
```

### 使用示例

```csharp
// 1. 第三层：通过 DataManager 获取
DataTower dataTower = GameEntry.Data.GetData<DataTower>();

// 2. 第二层：获取运行时数据
TowerData towerData = dataTower.GetTowerData(towerId);

// 3. 访问数据
string name = towerData.Name;        // 多语言名称
int entityId = towerData.EntityId;   // 实体ID
int maxHP = towerData.MaxHP;         // 最大生命值

// 4. 创建运行时对象
Tower tower = dataTower.CreateTower(towerId, level: 0);

// 5. 第一层：直接访问 DataTable（不推荐，应通过 DataManager）
IDataTable<DRTower> dtTower = GameEntry.DataTable.GetDataTable<DRTower>();
DRTower drTower = dtTower.GetDataRow(towerId);
```

### 为什么要三层？

- **DRTower（第一层）**：纯配置数据，自动生成，不要手动修改
- **TowerData（第二层）**：封装业务逻辑（如多语言、计算属性、等级数据）
- **DataTower（第三层）**：管理所有塔数据，提供查询接口、创建/销毁逻辑

### 数据流转过程

```
Excel表格 → 导出工具 → 二进制文件(.bytes) → DataTable加载 → 
Runtime Data初始化 → Data Manager管理
```

---

## 事件驱动通信

### 为什么使用事件系统？

GameFramework 使用事件系统实现组件间的**松耦合通信**，避免直接引用。

### 1. 创建事件

```csharp
public class UpgradeTowerEventArgs : GameEventArgs
{
    // 事件唯一ID（自动生成）
    public static readonly int EventId = typeof(UpgradeTowerEventArgs).GetHashCode();
    
    // 事件数据
    public Tower Tower { get; private set; }
    public int LastLevel { get; private set; }
    
    public override int Id => EventId;
    
    // 工厂方法（使用引用池）
    public static UpgradeTowerEventArgs Create(Tower tower, int lastLevel)
    {
        UpgradeTowerEventArgs e = ReferencePool.Acquire<UpgradeTowerEventArgs>();
        e.Tower = tower;
        e.LastLevel = lastLevel;
        return e;
    }
    
    // 清理方法（回池前调用）
    public override void Clear()
    {
        Tower = null;
        LastLevel = 0;
    }
}
```

### 2. 订阅事件

```csharp
protected override void OnShow(object userData)
{
    // 订阅升级塔事件
    GameEntry.Event.Subscribe(UpgradeTowerEventArgs.EventId, OnUpgradeTower);
}

private void OnUpgradeTower(object sender, GameEventArgs e)
{
    UpgradeTowerEventArgs args = (UpgradeTowerEventArgs)e;
    Debug.Log($"塔升级了！当前等级：{args.Tower.Level}");
}

protected override void OnHide(bool isShutdown, object userData)
{
    // 取消订阅（防止内存泄漏）
    GameEntry.Event.Unsubscribe(UpgradeTowerEventArgs.EventId, OnUpgradeTower);
}
```

### 3. 触发事件

```csharp
// 触发事件
GameEntry.Event.Fire(this, UpgradeTowerEventArgs.Create(tower, lastLevel));
```

### 事件系统的优势

- **解耦**：发送者和接收者不需要相互引用
- **性能**：使用 ReferencePool 避免 GC
- **灵活**：一个事件可以有多个监听者

---

## 学习路线图

### 第一周：理解框架核心

**目标**：理解 GameFramework 的基本架构和启动流程

1. **阅读核心文件**
   - `GameEntry.cs` - 理解组件系统
   - `ProcedureLaunch.cs` - 理解流程系统
   - `ProcedureMenu.cs` - 理解流程切换
   - `ProcedureLevel.cs` - 理解游戏主流程

2. **在 Unity 中实践**
   - 打开 `GameStart` 场景
   - 查看 `GameFramework` Prefab 上的组件配置
   - 运行游戏，按 `~` 键打开 Debugger
   - 观察各个组件的状态

3. **调试技巧**
   - 在 `ProcedureLaunch.OnEnter` 打断点，观察流程启动
   - 在 `ChangeState` 处打断点，观察流程切换

### 第二周：实体系统实践

**目标**：理解实体系统的创建、管理和回收

1. **阅读实体相关文件**
   - `EntityTowerBase.cs` - 塔的实体逻辑
   - `EntityEnemy.cs` - 敌人的实体逻辑
   - `EntityExtension.cs` - 实体扩展方法
   - `EntityLoader.cs` - 实体加载器

2. **查看预制体结构**
   - `Assets/GameMain/Entity/` 目录下的预制体
   - 理解 Prefab、EntityData、EntityLogic 的关系

3. **实战练习**
   - 在 `LevelControl.cs` 中找到实体生成逻辑
   - 尝试手动创建一个实体
   - 理解对象池的工作原理

### 第三周：数据流转实践

**目标**：理解数据从 Excel 到运行时的完整流程

1. **数据表系统**
   - 用 Excel 打开 `Assets/GameMain/DataTables/Tower.txt`
   - 修改数据，使用菜单生成 DataTable
   - 观察生成的 `DRTower.cs` 文件

2. **数据层架构**
   - 阅读 `DataTower.cs` - 数据管理器
   - 阅读 `TowerData.cs` - 运行时数据
   - 阅读 `DRTower.cs` - 数据表行
   - 理解三层架构的设计意图

3. **实战练习**
   - 添加一个新的塔类型
   - 修改塔的属性
   - 理解数据如何从配置到运行时的流转

### 第四周：热更新系统

**目标**：理解资源打包和热更新机制

1. **资源打包**
   - 阅读 `Assets/GameMain/Configs/ResourceBuilder.xml`
   - 使用 `GameFramework → Resource Tools` 构建 AssetBundle
   - 理解资源分组和依赖关系

2. **热更新流程**
   - 阅读 `ProcedureCheckVersion.cs`
   - 阅读 `ProcedureUpdateResources.cs`
   - 理解版本检查和资源更新流程

3. **本地测试**
   - 配置 HFS 本地服务器
   - 测试资源热更新
   - 理解分包下载机制

---

## 关键文件索引

| 文件路径 | 作用 | 重要性 |
|---------|------|--------|
| `Assets/GameMain/Scripts/Base/GameEntry.cs` | 框架入口 | ⭐⭐⭐⭐⭐ |
| `Assets/GameMain/Scripts/Procedure/ProcedureLaunch.cs` | 启动流程 | ⭐⭐⭐⭐⭐ |
| `Assets/GameMain/Scripts/Procedure/ProcedureLevel.cs` | 关卡流程 | ⭐⭐⭐⭐⭐ |
| `Assets/GameMain/Scripts/Level/LevelControl.cs` | 关卡控制 | ⭐⭐⭐⭐⭐ |
| `Assets/GameMain/Scripts/Entity/EntityLogic/EntityTowerBase.cs` | 实体示例 | ⭐⭐⭐⭐ |
| `Assets/GameMain/Scripts/Data/Tower/DataTower.cs` | 数据管理示例 | ⭐⭐⭐⭐ |
| `Assets/GameMain/Scripts/Extension/EntityLoader.cs` | 实体加载器 | ⭐⭐⭐⭐ |
| `Assets/GameMain/Scripts/Data/Tower/Tower.cs` | 运行时数据对象 | ⭐⭐⭐⭐ |

---

## 实战练习

### 练习1：添加一个新塔类型

1. 在 `Tower.txt` 中添加新塔的配置
2. 生成 DataTable
3. 创建新的 EntityLogic（继承 `EntityTowerBase`）
4. 创建对应的 Prefab
5. 在游戏中测试

### 练习2：实现一个新功能

1. 创建一个新的事件（如 `TowerDestroyedEventArgs`）
2. 在塔被摧毁时触发事件
3. 在 UI 中订阅事件，显示提示

### 练习3：优化性能

1. 使用 ReferencePool 管理临时对象
2. 使用 ObjectPool 管理频繁创建的对象
3. 使用事件系统替代直接引用

---

## 💡 快速调试技巧

1. **查看实体对象池**：按 `~` → 点击 "Object Pool" 标签
2. **查看事件订阅**：按 `~` → 点击 "Event" 标签
3. **查看引用池**：按 `~` → 点击 "Reference Pool" 标签
4. **查看数据表**：断点 `GameEntry.DataTable.GetDataTable<DRTower>()`
5. **查看流程状态**：断点 `ProcedureBase.ChangeState`

---

## 📖 推荐阅读顺序

1. **入门**：`GameEntry.cs` → `ProcedureLaunch.cs` → `ProcedureMenu.cs`
2. **实体系统**：`EntityExtension.cs` → `EntityTowerBase.cs` → `EntityLoader.cs`
3. **数据系统**：`DataTower.cs` → `TowerData.cs` → `DRTower.cs`
4. **游戏逻辑**：`LevelControl.cs` → `Tower.cs` → 各种 EntityLogic

---

## 🎯 常见问题

### Q: 如何访问框架组件？
A: 通过 `GameEntry.{Component}` 访问，例如 `GameEntry.Entity.ShowEntity(...)`

### Q: 如何切换流程？
A: 在 Procedure 的 `OnUpdate` 中调用 `ChangeState<NextProcedure>(procedureOwner)`

### Q: 如何创建实体？
A: 使用 `GameEntry.Entity.ShowEntity<T>(...)` 或 `EntityLoader.ShowEntity<T>(...)`

### Q: 如何访问数据？
A: 通过 `GameEntry.Data.GetData<DataX>()` 获取数据管理器，然后调用其方法

### Q: 如何触发事件？
A: 使用 `GameEntry.Event.Fire(this, EventArgs.Create(...))`

---

## 🔗 相关资源

- [GameFramework 官方文档](https://gameframework.cn/)
- [GameFramework GitHub](https://github.com/EllanJiang/GameFramework)
- [知乎专栏：GameFramework解析](https://zhuanlan.zhihu.com/p/426136370)

---

**祝你学习愉快！如有问题，随时提问！** 🚀

