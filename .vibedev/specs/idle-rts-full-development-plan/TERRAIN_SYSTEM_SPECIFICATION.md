# 深淵巢穴地形系統技術規格書

## 文檔版本
- **版本**: 1.0
- **日期**: 2024年12月
- **目標**: 為引擎團隊提供地形系統改造的詳細技術規格

## 設計原則

### 漸進式擴展架構
1. **MVP階段**: 簡化實現，支持單人遊戲
2. **多人階段**: 擴展為小規模多人支持
3. **MMO階段**: 完整的大世界無縫架構
4. **核心不變**: 接口和數據結構保持向後兼容

## 階段一：MVP地形系統規格

### 1.1 基礎地形架構

#### 地形分塊系統
```csharp
// 基礎分塊接口 - 為未來擴展預留
public interface ITerrainChunk 
{
    string ChunkId { get; }
    Vector3 WorldPosition { get; }
    int ChunkSize { get; }
    bool IsLoaded { get; }
    
    // MVP階段簡化實現
    TerrainData GetTerrainData();
    CreepData GetCreepData();
    
    // 為未來擴展預留的接口
    void LoadAsync();
    void UnloadAsync();
    ITerrainChunk[] GetNeighbors();
}

// MVP階段的簡化實現
public class SinglePlayerTerrainChunk : ITerrainChunk
{
    // 單機版本的簡化實現
    // 所有數據在內存中，無需複雜的加載/卸載邏輯
}
```

#### 地形數據結構
```csharp
// 地形基礎數據 - 設計為可序列化和網絡傳輸
[System.Serializable]
public struct TerrainChunkData
{
    // MVP階段：256x256高度圖
    public float[] heightmap;
    public byte[] biomeMap;
    public ResourceNodeData[] resourceNodes;
    
    // 為MMO預留的邊界數據
    public BorderHeightData borderData;
    
    // 版本控制，用於未來數據遷移
    public int dataVersion;
}

// 邊界數據結構 - 為無縫連接預留
[System.Serializable]
public struct BorderHeightData
{
    public float[] northBorder;
    public float[] southBorder;
    public float[] eastBorder;
    public float[] westBorder;
}
```

### 1.2 菌毯系統規格

#### 菌毯數據結構
```csharp
// 菌毯分塊數據 - 支持實時更新和網絡同步
[System.Serializable]
public struct CreepChunkData
{
    // MVP階段：128x128密度場
    public float[] densityField;
    public GrowthCenter[] growthCenters;
    public Vector2[] expansionFront;
    
    // 時間戳用於同步
    public long lastUpdateTime;
    public bool isDirty;
    
    // 為MMO預留的跨分塊連接數據
    public Dictionary<string, BorderConnection> borderConnections;
}

// 生長中心點 - 菌毯擴張的起始點
[System.Serializable]
public struct GrowthCenter
{
    public Vector2 position;
    public float strength;
    public float radius;
    public CreepType type;
    public int ownerId; // 為多人遊戲預留
}
```

#### 菌毯管理器接口
```csharp
public interface ICreepManager
{
    // 基礎功能
    bool CanBuildAt(Vector3 position);
    float GetCreepDensity(Vector3 position);
    void AddGrowthCenter(Vector3 position, float strength);
    void RemoveGrowthCenter(Vector3 position);
    
    // 擴張控制
    void SetExpansionRate(float rate);
    void UpdateCreepExpansion(float deltaTime);
    
    // 為網絡同步預留
    CreepUpdateData GetDirtyData();
    void ApplyUpdate(CreepUpdateData data);
    
    // 事件系統
    event System.Action<Vector3, float> OnCreepExpanded;
    event System.Action<Vector3> OnCreepReceded;
}
```

### 1.3 空間索引系統

#### 漸進式空間索引
```csharp
// 空間索引基礎接口
public interface ISpatialIndex<T>
{
    void Insert(T item, Bounds bounds);
    void Remove(T item);
    void Update(T item, Bounds newBounds);
    List<T> Query(Bounds area);
    List<T> QueryRadius(Vector3 center, float radius);
}

// MVP階段：簡化的四叉樹實現
public class SimpleQuadTree<T> : ISpatialIndex<T>
{
    // 簡化實現，為未來的八叉樹擴展預留接口
}

// 為MMO預留的複雜空間索引
public class HierarchicalSpatialIndex<T> : ISpatialIndex<T>
{
    // 多層次索引：世界級 -> 分塊級 -> 局部級
    // MVP階段可以是空實現，保持接口兼容
}
```

## 階段二：多人擴展規格

### 2.1 網絡同步系統

#### 地形同步協議
```csharp
// 地形更新數據包
[System.Serializable]
public struct TerrainUpdatePacket
{
    public string chunkId;
    public TerrainUpdateType updateType;
    public byte[] deltaData; // 增量數據
    public long timestamp;
    public int playerId;
}

// 菌毯同步數據包
[System.Serializable]
public struct CreepUpdatePacket
{
    public string chunkId;
    public Vector2[] changedCells;
    public float[] newDensities;
    public long timestamp;
    public int ownerId;
}
```

### 2.2 衝突解決機制

#### 菌毯衝突處理
```csharp
public interface ICreepConflictResolver
{
    // 處理多個玩家菌毯的重疊
    CreepResolutionResult ResolveConflict(
        CreepClaim[] claims, 
        Vector2 position
    );
    
    // 邊界混合算法
    float[] BlendBorders(
        float[] border1, 
        float[] border2, 
        float blendRatio
    );
}
```

## 階段三：MMO完整規格

### 3.1 分佈式地形系統

#### 區域服務器架構
```csharp
// 區域服務器接口
public interface IRegionServer
{
    // 分塊管理
    Task<ITerrainChunk> LoadChunkAsync(string chunkId);
    Task UnloadChunkAsync(string chunkId);
    
    // 玩家管理
    void RegisterPlayer(int playerId, Vector3 position);
    void UnregisterPlayer(int playerId);
    void UpdatePlayerPosition(int playerId, Vector3 newPosition);
    
    // 跨服務器通信
    Task<bool> TransferPlayer(int playerId, IRegionServer targetServer);
    void SyncBorderData(string chunkId, BorderSyncData data);
}
```

### 3.2 大世界無縫連接

#### 分塊邊界處理
```csharp
public class SeamlessTerrainManager
{
    // 邊界高度插值
    public float[] InterpolateBorderHeights(
        float[] border1, 
        float[] border2, 
        int resolution
    );
    
    // 菌毯邊界混合
    public float[] BlendCreepBorders(
        CreepBorderData[] neighborBorders,
        Vector2 blendCenter
    );
    
    // 實時邊界同步
    public void SyncBorderWithNeighbors(
        string chunkId, 
        BorderUpdateData data
    );
}
```

## 性能規格要求

### MVP階段性能目標
- **地形分塊大小**: 256x256米
- **菌毯精度**: 128x128密度場
- **更新頻率**: 10Hz
- **內存使用**: <500MB
- **CPU使用**: <30%

### MMO階段性能目標
- **地形分塊大小**: 512x512米
- **同時加載分塊**: 9個（3x3網格）
- **玩家容量**: 每分塊100人
- **網絡帶寬**: <1MB/s per player
- **延遲要求**: <100ms

## 數據持久化規格

### 數據庫架構
```sql
-- 地形分塊表
CREATE TABLE terrain_chunks (
    chunk_id VARCHAR(32) PRIMARY KEY,
    world_x INT NOT NULL,
    world_z INT NOT NULL,
    heightmap_data BLOB,
    biome_data BLOB,
    resource_nodes JSON,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- 菌毯數據表
CREATE TABLE creep_data (
    chunk_id VARCHAR(32) PRIMARY KEY,
    density_field BLOB,
    growth_centers JSON,
    expansion_front JSON,
    owner_id INT,
    last_update TIMESTAMP,
    FOREIGN KEY (chunk_id) REFERENCES terrain_chunks(chunk_id)
);
```

## 引擎改造指導

### 現有RTS引擎需要的修改

#### 1. TerrainManager擴展
```csharp
// 擴展現有的TerrainManager
public partial class TerrainManager
{
    // 新增分塊管理功能
    private Dictionary<string, ITerrainChunk> loadedChunks;
    private ICreepManager creepManager;
    
    // 保持現有接口不變
    public float SampleHeight(Vector3 position, float radius, LayerMask navLayerMask)
    {
        // 現有實現保持不變
        // 新增分塊查詢邏輯
    }
    
    // 新增菌毯相關方法
    public bool CanBuildOnCreep(Vector3 position)
    {
        return creepManager.CanBuildAt(position);
    }
}
```

#### 2. GridSearchHandler擴展
```csharp
// 擴展現有的GridSearchHandler
public partial class GridSearchHandler
{
    // 新增菌毯感知的搜索
    public ErrorMessage SearchWithCreepFilter<T>(
        Vector3 sourcePosition, 
        float radius, 
        bool requireCreep,
        System.Func<T, ErrorMessage> IsTargetValid, 
        out T potentialTarget
    ) where T : Entity
    {
        // 在現有搜索基礎上添加菌毯過濾
    }
}
```

## 實施建議

### 階段性實施計劃

#### 第一階段（2週）
1. 實現基礎的ITerrainChunk接口
2. 創建SinglePlayerTerrainChunk實現
3. 擴展TerrainManager支持分塊查詢

#### 第二階段（3週）
1. 實現基礎菌毯系統
2. 集成到現有建築放置系統
3. 添加菌毯視覺效果

#### 第三階段（4週）
1. 實現網絡同步框架
2. 添加多人菌毯衝突解決
3. 性能優化和測試

### 風險控制
1. **向後兼容**: 所有新接口都不破壞現有功能
2. **漸進測試**: 每個階段都有獨立的測試方案
3. **回滾機制**: 每個階段都可以回滾到上一個穩定版本

## 總結

這個規格書提供了一個完整的、可擴展的地形系統架構，能夠從簡單的MVP逐步擴展到複雜的MMO系統，同時保持代碼的向後兼容性和系統的穩定性。引擎團隊可以基於這些規格進行有針對性的改造，確保每一步都是朝著最終目標前進的。