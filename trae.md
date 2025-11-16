# file: c:\nightgamer\unity\projects\tower-defense-df\trae.md
# Entity 系统解析与使用模板

## 原理概览
- 分层结构
  - 接口层：`GameFramework.Entity` 提供 `IEntity`, `IEntityGroup`, `IEntityManager`（统一生命周期与管理）
  - 运行时桥接：`UnityGameFramework.Runtime.EntityComponent` 管理实体分组、显示/隐藏、父子挂载、事件派发
  - 项目扩展：`Flower` 层提供数据驱动与便利调用（`EntityLogicWithData`, `EntityLogicEx`, `EntityLoader`, `EntityExtension`, `DataEntity`）
- 显示流程
  - 入口：`EntityComponent.ShowEntity(...)` → 传入 `Type` 与 `userData`
  - 数据驱动：`DataEntity` 将 `entityId` 映射到 `AssetPath` 与 `EntityGroupData`
  - 分组创建：`EntityExtension` 检查分组，不存在则按 `PoolParamData` 自动创建
  - 事件派发：成功/失败/依赖资源加载事件通过 `EventComponent` 通知

## 关键文件
- `Assets/GameFramework/Scripts/Runtime/Entity/EntityLogic.cs` 生命周期（`OnInit/OnShow/OnUpdate/OnHide`）
- `Assets/GameFramework/Scripts/Runtime/Entity/EntityComponent.cs` 分组与显示/隐藏、附加/解除、事件转发
- `Assets/GameMain/Scripts/Entity/Entity.cs` 项目层数据统一应用（位置、旋转、释放 `EntityData`）
- `Assets/GameMain/Scripts/Entity/EntityLogicEx.cs` 封装事件订阅与 `EntityLoader`/`ItemLoader`
- `Assets/GameMain/Scripts/Extension/EntityLoader.cs` 以 `serialId` 管理显示、隐藏与成功回调
- `Assets/GameMain/Scripts/Entity/EntityExtension.cs` 便捷显示（枚举或 id），确保分组存在
- `Assets/GameMain/Scripts/Data/Entity/DataEntity.cs` 装配 `EntityData` 与 `EntityGroupData`
- `Assets/GameMain/Scripts/Enum/EnumEntity.cs` 枚举到数据表的桥接

## 使用模板

### 实体逻辑（推荐继承 `EntityLogicEx`）
```csharp
using UnityGameFramework.Runtime;
using UnityEngine;

namespace Flower
{
    public class EntityFoo : EntityLogicEx
    {
        protected override void OnShow(object userData)
        {
            base.OnShow(userData);
        }

        protected override void OnUpdate(float elapse, float realElapse)
        {
        }

        protected override void OnHide(bool isShutdown, object userData)
        {
            base.OnHide(isShutdown, userData);
        }
    }
}
```

### 通过扩展显示实体（数据驱动）
```csharp
var entityComp = GameEntry.Entity;
int serialId = entityComp.GenerateSerialId();
entityComp.ShowEntity<EntityFoo>(serialId, (int)EnumEntity.AssaultCannon, Flower.EntityData.Create(pos, rot));
```

### 在实体内部显示子实体（并挂载）
```csharp
int serial = ShowEntity<EntityFooChild>(EnumEntity.RadiusVisualiser, entity =>
{
    GameEntry.Entity.AttachEntity(entity, this.Entity);
}, Flower.EntityData.Create(transform.position, transform.rotation));
```

### 使用 Loader 管理一组实体
```csharp
var loader = Flower.EntityLoader.Create(this);
int serial = loader.ShowEntity<EntityFoo>((int)EnumEntity.RocketPlatform, e => { });
loader.HideEntity(serial);
loader.HideAllEntity();
ReferencePool.Release(loader);
```

### 父子实体附加与解除
```csharp
GameEntry.Entity.AttachEntity(childEntity, parentEntity);
GameEntry.Entity.AttachEntity(childEntity, parentEntity, "Pivot/Socket");
GameEntry.Entity.DetachEntity(childEntity);
GameEntry.Entity.DetachChildEntities(parentEntity);
```

## 建议
- 统一继承 `EntityLogicWithData` 或 `EntityLogicEx`，自动应用位姿并清理数据
- 子实体优先用 `EntityLogicEx` 的 `ShowEntity<T>`，在回调中 `AttachEntity`
- 使用 `EntityExtension` 保证分组存在与对象池参数一致
- 事件订阅用 `EntityLogicEx.Subscribe/UnSubscribeAll`，避免泄漏