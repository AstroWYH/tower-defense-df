# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a tower defense game built with Unity 2022.3.17f1c1 and GameFramework 2020.12.31. It's a re-implementation of Unity's Tower Defense Template 1.4 using the GameFramework architecture, demonstrating production-ready patterns including hot update support, data-driven design, and comprehensive resource management.

## Development Commands

### Editor Mode (Default Development)
- Open project in Unity Editor 2022.3.17f1c1 or compatible
- Ensure "Editor Resource Mode" is enabled on the Base component in the scene
- Main scene: `Assets/GameMain/Scenes/GameStart`
- Press Play to run directly from project resources (no AssetBundles needed)

### Data Table Management
```
Tools → Generate DataTables          # Converts .txt tables to binary .bytes
Tools → Generate DataTable Code      # Auto-generates C# DataReader classes
```
- Source tables: `Assets/GameMain/DataTables/*.txt` (Excel-exported format)
- Always regenerate both after modifying table schemas
- Code generation uses template: `Assets/GameMain/Configs/DataTableCodeTemplate.txt`

### Building AssetBundles (for Package/Release Mode)
```
GameFramework → Resource Tools
```
- Configuration: `Assets/GameMain/Configs/ResourceBuilder.xml`
- Output directory configured in XML (currently: `C:/Users/未闻花名/Desktop/AB`)
- Project is configured for zero redundancy and zero circular dependencies
- Supports per-level resource splitting and file system packaging

### Testing Hot Update
1. Disable "Editor Resource Mode" on Base component
2. Set Resource Component to "Updatable" mode
3. Build AssetBundles via Resource Tools
4. Deploy bundles to web server (use HFS for local testing)
5. Configure server URLs in BuildInfo
6. Run game - it will check version and download updates

### Debugging
- Press `~` key in-game to open built-in Debugger
- Shows FPS, memory usage, object pools, entity info
- Console window with filtering and reference pool statistics

## High-Level Architecture

### GameFramework Component-Based System

The entire game is built on GameFramework's component architecture. All systems are accessed via `GameEntry.{Component}`:

**Core Framework Components:**
- **Base**: Framework configuration and lifecycle
- **Procedure**: FSM-driven game flow (Launch → Splash → Menu → Level)
- **Entity**: Pooled entity management (towers, enemies, projectiles)
- **DataTable**: Binary table reader with auto-generated code
- **Event**: Centralized event bus for inter-system communication
- **Resource**: AssetBundle loading with hot update support
- **UI**: UI form management with groups and depth control
- **Localization**: Multi-language support (English, Chinese Simplified/Traditional)
- **FSM**: Finite State Machine manager
- **Sound**: Audio system with sound groups
- **ObjectPool/ReferencePool**: Memory management for GC optimization

**Custom Game Components:**
- **BuiltinData**: Build info and default dictionaries
- **Data**: Central data access layer
- **Item**: UI element management

### Game Flow (Procedure System)

The game uses an FSM-based procedure system for state management:
```
Launch → Splash → CheckVersion → UpdateVersion →
CheckResources → UpdateResources → Preload →
Menu ↔ LoadingScene ↔ Level
```

Each Procedure is an FSM state with `OnEnter`/`OnUpdate`/`OnLeave` lifecycle methods. Data passes between procedures via `VarInt32`/`VarString`.

### Entity System Architecture

Entities are pooled game objects with a three-part structure:
1. **Entity Prefab**: GameObject in `Assets/GameMain/Entity/`
2. **EntityData**: Initialization parameters (inherits `EntityData`)
3. **EntityLogic**: Behavior implementation (inherits `EntityLogic`)

**EntityLoader Pattern**: Wrapper that tracks entity lifetime and provides callbacks for show/hide/destroy events.

**Entity Types:**
- **Towers**: 6 types (Assault Cannon, Rocket Platform, Plasma Lance, Energy Pylon, EMP Generator, Missile Array)
- **Enemies**: 8 types (Buggy, Copter, Tank, Boss + Super variants)
- **Projectiles**: Multiple types with different behaviors
- **Effects**: Visual and audio effects

### Data Layer Architecture (Three-Tier)

1. **DataTable Layer** (`DRX` classes):
   - Binary table row readers (auto-generated)
   - Read-only access to Excel-exported data
   - Examples: `DRTower`, `DREnemy`, `DRLevel`

2. **Runtime Data Layer** (`XData` classes):
   - Runtime data structures with game state
   - Examples: `TowerData`, `EnemyData`, `LevelData`
   - Initialized from DataTable rows

3. **Manager Layer** (`DataX` classes):
   - Centralized data management
   - Examples: `DataTower`, `DataEnemy`, `DataLevel`
   - Accessed via `GameEntry.Data`

**Preload Pattern**: Data loads during `ProcedurePreload`, check `IsPreloadReady` before proceeding.

### Event-Driven Communication

All inter-system communication uses `GameEntry.Event`:
- Custom events inherit `GameEventArgs`
- Events use `ReferencePool` to avoid GC
- Subscribe in `OnEnter`, unsubscribe in `OnLeave`
- Examples: `ChangeSceneEventArgs`, `BuildTowerEventArgs`, `SpawnEnemyEventArgs`

**EventSubscriber Pattern**: Helper class for managing event subscriptions lifecycle.

## Key Gameplay Systems

### Tower System
- **Tower Components**:
  - `EntityTowerBase`: Base tower logic with health and upgrade system
  - `Targetter`: Enemy targeting logic with range detection
  - `Attacker`: Attack execution and damage calculation
  - `Launcher`: Projectile spawning (Common/Super/Ballistic types)

- **Tower Placement**:
  - `IPlacementArea` interface for placement validation
  - `TowerPlacementGrid`: Grid-based placement with `IntVector2`
  - `SingleTowerPlacementArea`: Pre-defined placement spots
  - Preview system shows range and validates placement

- **Tower Upgrades**: 3 levels (except Missile Array), enhances damage/range/fire rate

### Enemy System
- **AI States** (FSM-based):
  - `EnemyMoveState`: Follows level path
  - `EnemyAttackTowerState`: Attacks blocking towers
  - `EnemyAttackHomeBaseState`: Attacks player base
  - `FlyingEnemyPushingThroughState`: Flying enemies bypass towers

- **Pathfinding**: `LevelPath` with weighted random selection
- **Combat**: Health system with death callbacks and energy rewards

### Level Control System
`LevelControl` manages entire level lifecycle:
- Wave spawning and progression
- Tower building and upgrade coordination
- Enemy tracking and defeat detection
- Victory/defeat conditions and star rating
- Pause/resume support via `IPause` interface

### Wave System
- Multiple waves per level defined in DataTables
- Wave elements specify enemy type, count, spawn interval
- Sequential progression with player-triggered start
- Success: Defeat all waves while keeping base alive

### Resource Management
- **Energy**: Build/upgrade towers, gained from kills and Energy Pylon towers
- **Base Health**: Game over when reaches 0

## Important Implementation Patterns

### Reference Pooling (Critical for Performance)
- Extensive use of `ReferencePool.Acquire<T>()` and `ReferencePool.Release(obj)`
- All temporary objects should implement `IReference` with `Clear()` method
- Used for: Events, Data classes, EntityData, FSM states
- Always release references in cleanup methods

### Asset Management
- `AssetUtility` provides centralized asset path generation
- Resources organized by type and variant (for localization)
- Binary format for data tables (smaller, faster than text)
- Asset paths follow convention: `Assets/GameMain/{Category}/{AssetName}`

### Localization System
- Three languages: English, ChineseSimplified, ChineseTraditional
- Asset variants: `en-us`, `zh-cn`, `zh-tw`
- `LocalizeText` component for dynamic UI translation
- Dictionary fallback to default language

### Hot Update Architecture
1. **Version Checking**: Compare local vs server version
2. **Resource Groups**: Base resources + per-level packages
3. **Download on Demand**: Levels download when first accessed
4. **File System**: AssetBundles packed into virtual file systems
5. **Update Flow**: CheckVersion → UpdateVersion → CheckResources → UpdateResources

## Common Development Workflows

### Adding a New Entity
1. Create prefab in `Assets/GameMain/Entity/{Type}/`
2. Add row to `Entity.txt` with prefab path
3. Create `EntityLogic` class inheriting `EntityLogic`
4. Create `EntityData` class inheriting `EntityData`
5. Use `EntityLoader` to show/hide entity with callbacks

### Adding a New UI Form
1. Create prefab in `Assets/GameMain/UI/UIForms/`
2. Add to `UIForm.txt` with UIGroup assignment
3. Create class inheriting `UGuiForm`
4. Open via `GameEntry.UI.OpenUIForm(UIFormId.XXX, userData)`

### Adding a New Procedure
1. Create class inheriting `ProcedureBase`
2. Override `OnEnter`/`OnUpdate`/`OnLeave`
3. Register in Base component's Available Procedures list
4. Transition via `ChangeState<ProcedureType>()`

### Modifying Data Tables
1. Edit Excel, export to `.txt` in `Assets/GameMain/DataTables/`
2. Run "Generate DataTables" to create `.bytes`
3. Run "Generate DataTable Code" if schema changed
4. Access via `GameEntry.DataTable.GetDataTable<DRX>()`

### Adding New Events
1. Create class inheriting `GameEventArgs`
2. Override `Id` property with unique event ID
3. Implement `Clear()` for reference pooling
4. Fire via `GameEntry.Event.Fire(this, eventArgs)`
5. Subscribe via `GameEntry.Event.Subscribe(eventId, handler)`

## Directory Structure Guide

```
Assets/GameMain/
├── Scripts/
│   ├── Base/              # GameEntry and component registration
│   ├── Procedure/         # Game flow FSM states
│   ├── Entity/            # Entity logic (towers, enemies, projectiles)
│   ├── Tower/             # Tower-specific systems (targeting, attacking)
│   ├── Level/             # Level management and wave control
│   ├── UI/                # UI forms and components
│   ├── Data/              # Data management layer
│   ├── DataTable/         # Auto-generated DataTable readers (DO NOT EDIT)
│   ├── Event/             # Custom game events
│   ├── FSM/               # Enemy AI states
│   ├── Extension/         # Helper classes (EntityLoader, ItemLoader)
│   └── Utility/           # Static utility classes
├── DataTables/            # Excel-exported data (.txt + .bytes)
├── Configs/               # Build & resource configurations (XML)
├── Entity/                # Prefabs for game entities
├── UI/                    # UI prefabs
├── Localization/          # Multi-language dictionaries
└── Scenes/                # Unity scenes (GameStart, Menu, Level1-5)
```

## Critical Notes

- **DO NOT** modify files in `Assets/GameMain/Scripts/DataTable/` - they are auto-generated
- **ALWAYS** use ReferencePool for temporary objects to avoid GC
- **NEVER** hardcode data - use DataTables for all configuration
- **REMEMBER** to subscribe/unsubscribe events properly to avoid memory leaks
- **USE** Editor Resource Mode during development for faster iteration
- **TEST** hot update mode before release builds

## Dependencies

- Unity 2022.3.17f1c1
- GameFramework 2020.12.31
- Unity AI Navigation 1.1.5
- TextMeshPro 3.0.6
- ICSharpCode.SharpZipLib (compression)

## Reference Links

- GameFramework GitHub: https://github.com/EllanJiang/GameFramework
- Framework Analysis: https://zhuanlan.zhihu.com/p/426136370
- Original Template: Tower Defense Template 1.4 (Unity Asset Store)
