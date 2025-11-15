# GameFramework 快速参考卡片

> 常用API和代码片段速查表

## 🚀 核心访问方式

```csharp
// 所有功能都通过 GameEntry 访问
GameEntry.{Component}.Method()
```

---

## 📦 实体系统（Entity）

### 显示实体

```csharp
// 方式1：直接显示
GameEntry.Entity.ShowEntity<EntityEnemy>(
    serialId: 1,
    entityId: (int)EnumEntity.BuggyEnemy,
    userData: EntityDataEnemy.Create(position)
);

// 方式2：使用扩展方法
GameEntry.Entity.ShowEntity<EntityEnemy>(
    serialId: 1,
    enumEntity: EnumEntity.BuggyEnemy,
    userData: EntityDataEnemy.Create(position)
);

// 方式3：使用 EntityLoader（推荐，便于管理）
EntityLoader loader = EntityLoader.Create(this);
loader.ShowEntity<EntityEnemy>(
    EnumEntity.BuggyEnemy,
    (entity) => { /* 成功回调 */ },
    EntityDataEnemy.Create(position)
);
```

### 隐藏实体

```csharp
// 隐藏实体（自动回对象池）
GameEntry.Entity.HideEntity(serialId);

// 或使用 EntityLoader
loader.HideEntity(serialId);
loader.ReleaseAllEntity(); // 释放所有实体
```

### 创建实体数据

```csharp
// 使用引用池创建（自动管理）
EntityDataTower data = EntityDataTower.Create(
    position: Vector3.zero,
    rotation: Quaternion.identity,
    userData: tower
);
```

---

## 📊 数据系统（Data）

### 获取数据管理器

```csharp
// 获取数据管理器
DataTower dataTower = GameEntry.Data.GetData<DataTower>();
DataPlayer dataPlayer = GameEntry.Data.GetData<DataPlayer>();
DataLevel dataLevel = GameEntry.Data.GetData<DataLevel>();
```

### 访问数据表

```csharp
// 获取数据表
IDataTable<DRTower> dtTower = GameEntry.DataTable.GetDataTable<DRTower>();

// 获取单行数据
DRTower drTower = dtTower.GetDataRow(towerId);

// 获取所有数据
DRTower[] allTowers = dtTower.GetAllDataRows();
```

### 创建运行时对象

```csharp
// 创建塔
Tower tower = dataTower.CreateTower(towerId, level: 0);

// 获取塔数据
TowerData towerData = dataTower.GetTowerData(towerId);

// 升级塔
dataTower.UpgradeTower(tower.SerialId);

// 销毁塔
dataTower.DestroyTower(tower.SerialId);
```

---

## 🎯 事件系统（Event）

### 创建事件

```csharp
public class MyEventArgs : GameEventArgs
{
    public static readonly int EventId = typeof(MyEventArgs).GetHashCode();
    public override int Id => EventId;
    
    public int Value { get; private set; }
    
    public static MyEventArgs Create(int value)
    {
        MyEventArgs e = ReferencePool.Acquire<MyEventArgs>();
        e.Value = value;
        return e;
    }
    
    public override void Clear()
    {
        Value = 0;
    }
}
```

### 订阅/取消订阅事件

```csharp
// 订阅
GameEntry.Event.Subscribe(MyEventArgs.EventId, OnMyEvent);

// 取消订阅
GameEntry.Event.Unsubscribe(MyEventArgs.EventId, OnMyEvent);

// 事件处理
private void OnMyEvent(object sender, GameEventArgs e)
{
    MyEventArgs args = (MyEventArgs)e;
    Debug.Log($"事件值：{args.Value}");
}
```

### 触发事件

```csharp
GameEntry.Event.Fire(this, MyEventArgs.Create(100));
```

---

## 🎮 流程系统（Procedure）

### 切换流程

```csharp
// 在 Procedure 的 OnUpdate 中
ChangeState<ProcedureMenu>(procedureOwner);
```

### 获取当前流程

```csharp
ProcedureBase current = GameEntry.Procedure.CurrentProcedure;
float time = GameEntry.Procedure.CurrentProcedureTime;
```

---

## 🖼️ UI系统

### 打开/关闭UI

```csharp
// 打开UI
GameEntry.UI.OpenUIForm(
    uiFormAssetName: "UIFormMainMenu",
    uiGroupName: "Default",
    userData: null
);

// 关闭UI
GameEntry.UI.CloseUIForm(uiForm.SerialId);
GameEntry.UI.CloseUIForm(uiForm);
```

### 获取UI

```csharp
// 获取UI表单
UIForm uiForm = GameEntry.UI.GetUIForm("UIFormMainMenu");
UIForm uiForm = GameEntry.UI.GetUIForm(uiFormSerialId);
```

---

## 🔊 声音系统（Sound）

### 播放声音

```csharp
// 播放音效
int soundId = GameEntry.Sound.PlaySound(
    soundAssetName: "Sound_ButtonClick",
    soundGroupName: "UI",
    userData: null
);

// 播放音乐
int musicId = GameEntry.Sound.PlaySound(
    soundAssetName: "Music_Menu",
    soundGroupName: "Music",
    userData: null
);

// 停止声音
GameEntry.Sound.StopSound(soundId);
```

---

## 📦 对象池（ObjectPool）

### 生成/回收对象

```csharp
// 生成对象
GameObject obj = GameEntry.ObjectPool.Spawn(
    assetName: "Bullet",
    userData: null
);

// 回收对象
GameEntry.ObjectPool.Unspawn(obj);
```

---

## 🔄 引用池（ReferencePool）

### 获取/释放引用

```csharp
// 获取引用（从池中取出）
Tower tower = ReferencePool.Acquire<Tower>();
MyEventArgs args = ReferencePool.Acquire<MyEventArgs>();

// 释放引用（回池）
ReferencePool.Release(tower);
ReferencePool.Release(args);
```

---

## 🌐 本地化（Localization）

### 获取本地化字符串

```csharp
// 获取本地化字符串
string text = GameEntry.Localization.GetString("Key.MainMenu.Title");

// 设置语言
GameEntry.Localization.Language = Language.ChineseSimplified;
```

---

## 📁 资源系统（Resource）

### 加载资源

```csharp
// 加载资源
GameEntry.Resource.LoadAsset(
    assetName: "Tower_Cannon",
    loadAssetCallbacks: new LoadAssetCallbacks(
        (assetName, asset, duration, userData) => {
            // 加载成功
            GameObject prefab = asset as GameObject;
        },
        (assetName, status, errorMessage, userData) => {
            // 加载失败
            Log.Error("加载失败：{0}", errorMessage);
        }
    )
);

// 卸载资源
GameEntry.Resource.UnloadAsset(asset);
```

---

## 🎬 场景系统（Scene）

### 加载场景

```csharp
// 加载场景
GameEntry.Scene.LoadScene(
    sceneAssetName: "Level_01",
    priority: 0,
    loadSceneCallbacks: new LoadSceneCallbacks(
        (sceneAssetName, duration, userData) => {
            // 加载成功
        },
        (sceneAssetName, status, errorMessage, userData) => {
            // 加载失败
        }
    )
);
```

---

## 🛠️ 常用工具类

### 日志

```csharp
// 不同级别的日志
GameFrameworkLog.Info("信息：{0}", message);
GameFrameworkLog.Warning("警告：{0}", message);
GameFrameworkLog.Error("错误：{0}", message);
GameFrameworkLog.Fatal("致命错误：{0}", message);

// 或使用简写
Log.Info("信息");
Log.Warning("警告");
Log.Error("错误");
```

### 文本格式化

```csharp
// 格式化文本
string text = Utility.Text.Format("玩家 {0} 得分：{1}", playerName, score);
```

---

## 📝 实体逻辑模板

```csharp
public class MyEntity : EntityLogic
{
    protected override void OnInit(object userData)
    {
        base.OnInit(userData);
        // 获取组件引用
    }
    
    protected override void OnShow(object userData)
    {
        base.OnShow(userData);
        // 初始化逻辑
        // 订阅事件
        GameEntry.Event.Subscribe(MyEventArgs.EventId, OnMyEvent);
    }
    
    protected override void OnUpdate(float elapseSeconds, float realElapseSeconds)
    {
        base.OnUpdate(elapseSeconds, realElapseSeconds);
        // 每帧更新
    }
    
    protected override void OnHide(bool isShutdown, object userData)
    {
        base.OnHide(isShutdown, userData);
        // 取消订阅
        GameEntry.Event.Unsubscribe(MyEventArgs.EventId, OnMyEvent);
        // 清理引用
    }
}
```

---

## 📝 Procedure 模板

```csharp
public class MyProcedure : ProcedureBase
{
    protected override void OnInit(ProcedureOwner procedureOwner)
    {
        base.OnInit(procedureOwner);
    }
    
    protected override void OnEnter(ProcedureOwner procedureOwner)
    {
        base.OnEnter(procedureOwner);
        // 进入流程时的初始化
    }
    
    protected override void OnUpdate(ProcedureOwner procedureOwner, 
        float elapseSeconds, float realElapseSeconds)
    {
        base.OnUpdate(procedureOwner, elapseSeconds, realElapseSeconds);
        
        // 检查条件，切换流程
        if (condition)
        {
            ChangeState<NextProcedure>(procedureOwner);
        }
    }
    
    protected override void OnLeave(ProcedureOwner procedureOwner, bool isShutdown)
    {
        base.OnLeave(procedureOwner, isShutdown);
        // 离开流程时的清理
    }
    
    protected override void OnDestroy(ProcedureOwner procedureOwner)
    {
        base.OnDestroy(procedureOwner);
    }
}
```

---

## 🎯 常用模式

### 模式1：使用 EntityLoader 管理多个实体

```csharp
public class LevelControl : MonoBehaviour
{
    private EntityLoader m_EntityLoader;
    
    private void Start()
    {
        m_EntityLoader = EntityLoader.Create(this);
    }
    
    private void SpawnEnemy()
    {
        m_EntityLoader.ShowEntity<EntityEnemy>(
            EnumEntity.BuggyEnemy,
            (entity) => { /* 成功 */ },
            EntityDataEnemy.Create(position)
        );
    }
    
    private void OnDestroy()
    {
        m_EntityLoader.ReleaseAllEntity();
    }
}
```

### 模式2：事件驱动通信

```csharp
// 发送方
GameEntry.Event.Fire(this, UpgradeTowerEventArgs.Create(tower, lastLevel));

// 接收方
protected override void OnShow(object userData)
{
    GameEntry.Event.Subscribe(UpgradeTowerEventArgs.EventId, OnUpgradeTower);
}

private void OnUpgradeTower(object sender, GameEventArgs e)
{
    UpgradeTowerEventArgs args = (UpgradeTowerEventArgs)e;
    // 处理事件
}

protected override void OnHide(bool isShutdown, object userData)
{
    GameEntry.Event.Unsubscribe(UpgradeTowerEventArgs.EventId, OnUpgradeTower);
}
```

### 模式3：数据访问

```csharp
// 获取数据管理器
DataTower dataTower = GameEntry.Data.GetData<DataTower>();

// 获取配置数据
TowerData towerData = dataTower.GetTowerData(towerId);

// 创建运行时对象
Tower tower = dataTower.CreateTower(towerId, level: 0);

// 使用数据
int damage = tower.Damage;
float range = tower.Range;
```

---

## ⚠️ 注意事项

1. **实体数据使用引用池**：创建 EntityData 时使用 `Create` 方法，会自动管理
2. **事件必须取消订阅**：在 `OnHide` 中取消订阅，防止内存泄漏
3. **引用池对象使用后释放**：使用 `ReferencePool.Release()` 释放
4. **流程切换在 OnUpdate**：不要在 `OnEnter` 中立即切换流程
5. **数据访问通过 DataManager**：不要直接访问 DataTable，应通过 DataManager

---

**快速查找，随时参考！** 📚

