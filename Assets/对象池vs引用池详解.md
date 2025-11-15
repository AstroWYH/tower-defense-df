# 对象池（ObjectPool）vs 引用池（ReferencePool）详解

> GameFramework 中两个池系统的区别和应用场景

## 📊 核心区别对比

| 特性 | 引用池（ReferencePool） | 对象池（ObjectPool） |
|------|----------------------|---------------------|
| **管理对象** | C# 纯对象（实现了 `IReference` 接口） | Unity GameObject（继承自 `ObjectBase`） |
| **涉及Unity** | ❌ 不涉及 Unity 引擎 | ✅ 涉及 Unity 引擎 |
| **主要目的** | 减少 GC（垃圾回收）压力 | 减少 Instantiate/Destroy 开销 |
| **对象类型** | 数据类、事件参数、临时对象 | 游戏对象、实体、子弹等 |
| **创建方式** | `new T()` 或 `ReferencePool.Acquire<T>()` | `Instantiate()` 或 `ObjectPool.Spawn()` |
| **销毁方式** | `ReferencePool.Release()` | `ObjectPool.Unspawn()` 或 `Destroy()` |
| **生命周期** | 纯 C# 对象生命周期 | Unity GameObject 生命周期 |
| **内存管理** | 托管内存（C# 堆） | Unity 内存 + 托管内存 |

---

## 🔄 引用池（ReferencePool）

### 什么是引用池？

引用池用于管理 **C# 纯对象**（不涉及 Unity GameObject），目的是**减少 GC 压力**，避免频繁创建和销毁临时对象。

### 使用条件

对象必须实现 `IReference` 接口：

```csharp
public interface IReference
{
    void Clear();  // 清理方法，回池前调用
}
```

### 典型应用场景

#### 1. 事件参数（EventArgs）

```csharp
public class UpgradeTowerEventArgs : GameEventArgs
{
    public static readonly int EventId = typeof(UpgradeTowerEventArgs).GetHashCode();
    
    public Tower Tower { get; private set; }
    public int LastLevel { get; private set; }
    
    public override int Id => EventId;
    
    // 工厂方法：从引用池获取
    public static UpgradeTowerEventArgs Create(Tower tower, int lastLevel)
    {
        UpgradeTowerEventArgs e = ReferencePool.Acquire<UpgradeTowerEventArgs>();
        e.Tower = tower;
        e.LastLevel = lastLevel;
        return e;
    }
    
    // 清理方法：回池前调用
    public override void Clear()
    {
        Tower = null;
        LastLevel = 0;
    }
}

// 使用
GameEntry.Event.Fire(this, UpgradeTowerEventArgs.Create(tower, lastLevel));
// 事件系统内部会自动调用 Release，无需手动释放
```

#### 2. 实体数据（EntityData）

```csharp
public class EntityData : IReference
{
    public Vector3 Position { get; set; }
    public Quaternion Rotation { get; set; }
    public object UserData { get; set; }
    
    // 工厂方法：从引用池获取
    public static EntityData Create(Vector3 position, object userData = null)
    {
        EntityData entityData = ReferencePool.Acquire<EntityData>();
        entityData.Position = position;
        entityData.Rotation = Quaternion.identity;
        entityData.UserData = userData;
        return entityData;
    }
    
    // 清理方法：回池前调用
    public void Clear()
    {
        Position = Vector3.zero;
        Rotation = Quaternion.identity;
        UserData = null;
    }
}

// 使用
EntityData data = EntityData.Create(position);
GameEntry.Entity.ShowEntity<EntityEnemy>(serialId, entityId, data);
// Entity 系统内部会自动调用 Release，无需手动释放
```

#### 3. 临时数据对象

```csharp
public class AttackerData : IReference
{
    public float Range { get; private set; }
    public float FireRate { get; private set; }
    public bool IsMultiAttack { get; private set; }
    
    public static AttackerData Create(float range, float fireRate, bool isMultiAttack)
    {
        AttackerData data = ReferencePool.Acquire<AttackerData>();
        data.Range = range;
        data.FireRate = fireRate;
        data.IsMultiAttack = isMultiAttack;
        return data;
    }
    
    public void Clear()
    {
        Range = 0;
        FireRate = 0;
        IsMultiAttack = false;
    }
}

// 使用
AttackerData data = AttackerData.Create(10f, 1f, true);
// ... 使用数据
ReferencePool.Release(data);  // 使用完后释放
```

### 引用池的特点

✅ **优点**：
- 减少 GC 压力（避免频繁分配/回收）
- 性能开销小（纯 C# 对象）
- 自动管理（框架内部自动释放）

⚠️ **注意事项**：
- 必须实现 `IReference` 接口
- 必须实现 `Clear()` 方法清理数据
- 使用完后记得调用 `Release()`（某些场景框架会自动释放）

---

## 🎮 对象池（ObjectPool）

### 什么是对象池？

对象池用于管理 **Unity GameObject**，目的是**减少 Instantiate/Destroy 的开销**，避免频繁创建和销毁游戏对象。

### 使用条件

对象必须继承自 `ObjectBase`（通常通过 Entity 系统间接使用）。

### 典型应用场景

#### 1. 实体系统（Entity System）

GameFramework 的实体系统**内部使用对象池**管理 GameObject：

```csharp
// 显示实体（内部使用对象池）
GameEntry.Entity.ShowEntity<EntityEnemy>(
    serialId: 1,
    entityId: (int)EnumEntity.BuggyEnemy,
    userData: EntityDataEnemy.Create(position)
);

// 隐藏实体（自动回对象池）
GameEntry.Entity.HideEntity(serialId);
```

**内部流程**：
1. `ShowEntity` → 从对象池获取 GameObject（如果没有则 Instantiate）
2. `HideEntity` → GameObject 回对象池（不 Destroy）

#### 2. 直接使用对象池（不常用）

```csharp
// 创建对象池
GameEntry.ObjectPool.RegisterObjectPool<BulletObject>(
    "Bullet",
    capacity: 10,
    expireTime: 30f,
    priority: 0
);

// 生成对象（从对象池获取）
GameObject bullet = GameEntry.ObjectPool.Spawn("Bullet").Target as GameObject;

// 回收对象（回对象池）
GameEntry.ObjectPool.Unspawn(bullet);
```

### 对象池的特点

✅ **优点**：
- 减少 Instantiate/Destroy 开销（Unity 最昂贵的操作之一）
- 减少内存分配（复用 GameObject）
- 提高性能（特别是频繁创建/销毁的对象）

⚠️ **注意事项**：
- 需要管理 GameObject 的生命周期
- 需要处理对象的状态重置
- 通常通过 Entity 系统间接使用

---

## 🔍 实际项目中的使用

### 引用池使用示例

在塔防项目中，引用池用于：

1. **事件参数**：`UpgradeTowerEventArgs`、`HideTowerEventArgs` 等
2. **实体数据**：`EntityData`、`EntityDataTower`、`EntityDataEnemy` 等
3. **临时数据**：`AttackerData`、`UIGameOverFormOpenParam` 等
4. **运行时对象**：`Tower`、`Enemy` 等（实现了 `IReference`）

```csharp
// 示例：Tower 类使用引用池
public class Tower : IReference
{
    // ...
    
    public static Tower Create(TowerData towerData, int serialId, int level = 0)
    {
        Tower tower = ReferencePool.Acquire<Tower>();  // 从引用池获取
        tower.towerData = towerData;
        tower.Level = level;
        tower.SerialId = serialId;
        return tower;
    }
    
    public void Clear()  // 回池前清理
    {
        towerData = null;
        Level = 0;
        SerialId = 0;
    }
}

// 使用
Tower tower = Tower.Create(towerData, serialId, level);
// ... 使用 tower
ReferencePool.Release(tower);  // 释放回池
```

### 对象池使用示例

在塔防项目中，对象池用于：

1. **实体 GameObject**：塔、敌人、子弹等（通过 Entity 系统）
2. **UI 元素**：某些动态 UI（通过 UI 系统）

```csharp
// Entity 系统内部使用对象池
// 你只需要调用 ShowEntity/HideEntity，框架会自动管理对象池

// 显示敌人（内部从对象池获取 GameObject）
GameEntry.Entity.ShowEntity<EntityEnemy>(
    serialId: 1,
    entityId: (int)EnumEntity.BuggyEnemy,
    userData: EntityDataEnemy.Create(position)
);

// 隐藏敌人（GameObject 回对象池，不 Destroy）
GameEntry.Entity.HideEntity(serialId);
```

---

## 📝 如何选择使用哪个？

### 使用引用池（ReferencePool）的场景

✅ **使用引用池，如果：**
- 对象是纯 C# 类（不涉及 Unity GameObject）
- 对象实现了 `IReference` 接口
- 对象频繁创建和销毁（减少 GC）
- 对象是临时数据（事件参数、数据对象等）

**典型例子**：
- `EntityData`、`EventArgs`、`Tower`、`Enemy` 等数据类

### 使用对象池（ObjectPool）的场景

✅ **使用对象池，如果：**
- 对象是 Unity GameObject
- 对象需要显示在场景中
- 对象频繁创建和销毁（减少 Instantiate/Destroy）
- 对象有视觉表现（模型、UI 等）

**典型例子**：
- 塔的 GameObject、敌人的 GameObject、子弹的 GameObject

### 两者结合使用

在实际项目中，**两者经常结合使用**：

```csharp
// 1. 使用引用池创建数据对象
EntityDataEnemy data = EntityDataEnemy.Create(position);

// 2. 使用对象池创建 GameObject（通过 Entity 系统）
GameEntry.Entity.ShowEntity<EntityEnemy>(serialId, entityId, data);
// ↑ 内部使用对象池管理 GameObject

// 3. 隐藏实体（GameObject 回对象池）
GameEntry.Entity.HideEntity(serialId);
// EntityData 会自动释放回引用池
```

---

## 🎯 性能对比

### 不使用池的情况

```csharp
// ❌ 不好的做法：频繁创建/销毁
for (int i = 0; i < 1000; i++)
{
    EntityData data = new EntityData();  // 每次 new，产生 GC
    // ... 使用
    // data 被 GC 回收
}

// ❌ 不好的做法：频繁 Instantiate/Destroy
for (int i = 0; i < 1000; i++)
{
    GameObject bullet = Instantiate(bulletPrefab);  // 昂贵的操作
    // ... 使用
    Destroy(bullet);  // 昂贵的操作
}
```

### 使用池的情况

```csharp
// ✅ 好的做法：使用引用池
for (int i = 0; i < 1000; i++)
{
    EntityData data = ReferencePool.Acquire<EntityData>();  // 从池获取
    // ... 使用
    ReferencePool.Release(data);  // 回池，不产生 GC
}

// ✅ 好的做法：使用对象池（通过 Entity 系统）
for (int i = 0; i < 1000; i++)
{
    int serialId = GameEntry.Entity.ShowEntity<EntityBullet>(...);  // 从池获取
    // ... 使用
    GameEntry.Entity.HideEntity(serialId);  // 回池，不 Destroy
}
```

**性能提升**：
- 引用池：减少 90%+ 的 GC 分配
- 对象池：减少 95%+ 的 Instantiate/Destroy 开销

---

## 💡 最佳实践

### 1. 所有临时数据对象都使用引用池

```csharp
// ✅ 实现 IReference
public class MyData : IReference
{
    public int Value { get; set; }
    
    public static MyData Create(int value)
    {
        MyData data = ReferencePool.Acquire<MyData>();
        data.Value = value;
        return data;
    }
    
    public void Clear()
    {
        Value = 0;
    }
}
```

### 2. 所有游戏对象都通过 Entity 系统（内部使用对象池）

```csharp
// ✅ 使用 Entity 系统
GameEntry.Entity.ShowEntity<EntityEnemy>(...);
GameEntry.Entity.HideEntity(serialId);
```

### 3. 事件参数必须使用引用池

```csharp
// ✅ 事件参数使用引用池
public class MyEventArgs : GameEventArgs
{
    public static MyEventArgs Create(...)
    {
        return ReferencePool.Acquire<MyEventArgs>();
    }
    
    public override void Clear() { ... }
}
```

### 4. 记得释放引用

```csharp
// ✅ 使用完后释放
MyData data = MyData.Create(100);
// ... 使用
ReferencePool.Release(data);  // 记得释放
```

---

## 🔑 总结

| 问题 | 引用池 | 对象池 |
|------|--------|--------|
| **管理什么？** | C# 纯对象 | Unity GameObject |
| **解决什么问题？** | GC 压力 | Instantiate/Destroy 开销 |
| **什么时候用？** | 临时数据、事件参数 | 游戏对象、实体 |
| **如何实现？** | 实现 `IReference` | 继承 `ObjectBase`（通常通过 Entity） |
| **如何获取？** | `ReferencePool.Acquire<T>()` | `ObjectPool.Spawn()` 或 `Entity.ShowEntity()` |
| **如何释放？** | `ReferencePool.Release()` | `ObjectPool.Unspawn()` 或 `Entity.HideEntity()` |

**记住**：
- **引用池** = 数据对象 = 减少 GC
- **对象池** = 游戏对象 = 减少 Instantiate/Destroy

两者配合使用，可以大幅提升游戏性能！🚀

