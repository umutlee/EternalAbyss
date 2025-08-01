# 深淵巢穴開發對話記憶

## 項目背景
- **項目名稱**: 深淵巢穴 (Deep Abyss Hive)
- **類型**: Idle RTS 遊戲
- **核心概念**: 玩家扮演深淵母巢集合意志，指導蟲群無限擴張與進化
- **技術棧**: Unity 2022.3 LTS + Node.js + MongoDB + Redis

## 當前開發狀態

### 已完成的規格文檔
1. **地形系統技術規格書** (`TERRAIN_SYSTEM_SPECIFICATION.md`)
   - 漸進式擴展架構：MVP → 多人 → MMO
   - 地形分塊系統設計
   - 菌毯系統完整規格
   - 空間索引系統架構
   - 網絡同步協議設計
   - 性能指標和實施計劃

### 核心設計決策
1. **漸進式架構**: 從單機MVP逐步擴展到MMO，保持向後兼容
2. **菌毯系統**: 128x128密度場，支持實時擴張和多人衝突解決
3. **分塊管理**: 256x256米分塊，支持動態加載/卸載
4. **接口設計**: 預留擴展接口，確保未來升級不破壞現有功能

### 技術架構重點
- **ITerrainChunk接口**: 統一的地形分塊管理
- **ICreepManager接口**: 菌毯系統核心管理器
- **ISpatialIndex接口**: 可擴展的空間索引系統
- **網絡同步協議**: 支持增量更新和衝突解決

## 開發規劃狀態

### 當前階段: 地形系統規格完成
- ✅ 完成地形系統技術規格書
- ✅ 定義核心接口和數據結構
- ✅ 制定三階段實施計劃
- ⏳ 待開始：引擎改造實施

### 下一步計劃
1. **引擎改造實施** (預計6-8週)
   - 第一階段：基礎分塊系統 (2週)
   - 第二階段：菌毯系統集成 (3週)
   - 第三階段：網絡同步框架 (3週)

2. **其他系統規格制定**
   - 單位系統規格
   - 建築系統規格
   - 資源系統規格
   - UI/UX系統規格

## 關鍵技術細節

### 菌毯系統核心
```csharp
// 菌毯管理器接口
public interface ICreepManager
{
    bool CanBuildAt(Vector3 position);
    float GetCreepDensity(Vector3 position);
    void AddGrowthCenter(Vector3 position, float strength);
    void UpdateCreepExpansion(float deltaTime);
    CreepUpdateData GetDirtyData();
    void ApplyUpdate(CreepUpdateData data);
}
```

### 地形分塊接口
```csharp
public interface ITerrainChunk 
{
    string ChunkId { get; }
    Vector3 WorldPosition { get; }
    TerrainData GetTerrainData();
    CreepData GetCreepData();
    void LoadAsync();
    void UnloadAsync();
}
```

## 設計原則
1. **向後兼容**: 所有新功能不破壞現有系統
2. **漸進擴展**: 支持從單機到MMO的平滑升級
3. **性能優先**: 明確的性能指標和優化策略
4. **模塊化設計**: 清晰的接口分離和職責劃分

## 文檔結構
- 主要規格文檔位於 `.vibedev/specs/idle-rts-full-development-plan/`
- 當前活躍文檔: `TERRAIN_SYSTEM_SPECIFICATION.md`
- 項目概述: `README.md`

## 開發團隊角色
- **遊戲設計**: 基於星際蟲族 + 輪迴樂園卡拉風格
- **技術架構**: 現代化全棧架構設計
- **商業模式**: F2P + 便利性付費策略