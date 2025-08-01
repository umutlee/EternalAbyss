# 深淵巢穴引擎整合指南

## 文檔版本
- **版本**: 1.0
- **日期**: 2024年12月
- **目標**: 為引擎團隊提供現有RTS引擎的整合改造指南

## 整合策略

### 核心原則
1. **最小侵入性**: 盡量保持現有代碼結構不變
2. **向後兼容**: 所有現有功能必須繼續工作
3. **漸進式擴展**: 分階段添加新功能
4. **接口穩定**: 新增接口設計為長期穩定

## 現有系統分析

### 已有優勢
- ✅ 完整的GridSearchHandler空間索引系統
- ✅ 成熟的TerrainManager地形管理
- ✅ 穩定的BuildingPlacement建築系統
- ✅ 完善的CustomEvents事件系統
- ✅ 良好的Entity基礎架構

### 需要擴展的部分
- 🔄 地形系統需要分塊化支持
- 🔄 單位系統需要大規模渲染優化
- 🔄 空間索引需要層次化擴展
- ➕ 需要添加菌毯系統
- ➕ 需要添加網絡同步框架

## 階段一：基礎擴展（MVP支持）

### 1.1 TerrainManager擴展

#### 文件位置
`Assets/RTS Engine/Terrain/Scripts/TerrainManager.cs`

#### 擴展方案
```csharp
// 在現有TerrainManager類中添加
public partial class TerrainManager : MonoBehaviour
{
    [Header("Chunk System - New")]
    [SerializeField] private int chunkSize = 256;
    [SerializeField] private bool enableChunkSystem = false; // MVP階段可關閉
    
    // 新增分塊管理
    private Dictionary<Vector2Int, TerrainChunk> loadedChunks;
    private ICreepManager creepManager;
    
    // 保持現有方法不變，內部添加分塊邏輯
    public float SampleHeight(Vector3 position, float radius, LayerMask navLayerMask)
    {
        // 現有邏輯保持不變
        float multiplier = 2.0f;
        while (multiplier <= maxHeightSampleRangeMultiplier)
        {
            if (NavMesh.SamplePosition(position, out NavMeshHit hit, radius * maxHeightSampleRangeMultiplier, navLayerMask))
            {
                // 新增：如果啟用分塊系統，進行分塊查詢
                if (enableChunkSystem)
                {
                    var chunk = GetChunkAt(position);
                    if (chunk != null)
                        return chunk.GetPreciseHeight(position);
                }
                
                return hit.position.y;
            }
            multiplier += 1.0f;
        }
        return position.y;
    }
    
    // 新增方法 - 不影響現有功能
    public TerrainChunk GetChunkAt(Vector3 worldPosition)
    {
        if (!enableChunkSystem) return null;
        
        var chunkCoord = WorldToChunkCoord(worldPosition);
        loadedChunks.TryGetValue(chunkCoord, out var chunk);
        return chunk;
    }
    
    // 新增菌毯支持
    public bool CanBuildOnCreep(Vector3 position)
    {
        return creepManager?.CanBuildAt(position) ?? true; // 默認允許建造
    }
    
    private Vector2Int WorldToChunkCoord(Vector3 worldPos)
    {
        return new Vector2Int(
            Mathf.FloorToInt(worldPos.x / chunkSize),
            Mathf.FloorToInt(worldPos.z / chunkSize)
        );
    }
}
```

### 1.2 GridSearchHandler擴展

#### 文件位置
`Assets/RTS Engine/Grid Search/Scripts/GridSearchHandler.cs`

#### 擴展方案
```csharp
// 在現有GridSearchHandler類中添加
public partial class GridSearchHandler : MonoBehaviour
{
    [Header("Large Scale Support - New")]
    [SerializeField] private bool enableBatchProcessing = false;
    [SerializeField] private int maxBatchSize = 100;
    
    // 新增批量搜索 - 為大規模單位優化
    public ErrorMessage BatchSearch<T>(
        SearchRequest[] requests,
        out SearchResult<T>[] results
    ) where T : Entity
    {
        results = new SearchResult<T>[requests.Length];
        
        if (!enableBatchProcessing)
        {
            // 回退到單個搜索
            for (int i = 0; i < requests.Length; i++)
            {
                Search(requests[i].position, requests[i].radius, requests[i].searchResources,
                       requests[i].validator, out T target);
                results[i] = new SearchResult<T> { target = target };
            }
            return ErrorMessage.none;
        }
        
        // 批量處理邏輯
        return ProcessBatchSearch(requests, results);
    }
    
    // 新增菌毯感知搜索
    public ErrorMessage SearchWithCreepFilter<T>(
        Vector3 sourcePosition, 
        float radius, 
        bool requireCreep,
        System.Func<T, ErrorMessage> IsTargetValid, 
        out T potentialTarget
    ) where T : Entity
    {
        // 包裝現有搜索，添加菌毯過濾
        return Search(sourcePosition, radius, false, (entity) =>
        {
            // 先檢查原有條件
            var originalResult = IsTargetValid(entity);
            if (originalResult != ErrorMessage.none)
                return originalResult;
            
            // 檢查菌毯條件
            if (requireCreep)
            {
                var terrainMgr = GameManager.Instance.TerrainMgr;
                if (!terrainMgr.CanBuildOnCreep(entity.transform.position))
                    return ErrorMessage.targetNotFound;
            }
            
            return ErrorMessage.none;
        }, out potentialTarget);
    }
}

// 新增數據結構
[System.Serializable]
public struct SearchRequest
{
    public Vector3 position;
    public float radius;
    public bool searchResources;
    public System.Func<Entity, ErrorMessage> validator;
}

[System.Serializable]
public struct SearchResult<T> where T : Entity
{
    public T target;
    public ErrorMessage result;
}
```

### 1.3 BuildingPlacement擴展

#### 文件位置
`Assets/RTS Engine/Buildings/Scripts/BuildingPlacement.cs`

#### 擴展方案
```csharp
// 在現有BuildingPlacement類中添加
public partial class BuildingPlacement : MonoBehaviour
{
    [Header("Creep System - New")]
    [SerializeField] private bool requireCreepForBuilding = false;
    [SerializeField] private string[] creepRequiredBuildings; // 需要菌毯的建築代碼
    
    // 擴展現有的CanPlaceBuilding方法
    public bool CanPlaceBuilding(Building building, bool showMessage)
    {
        // 保持現有邏輯
        if (gameMgr.GetFaction(GameManager.PlayerFactionID).FactionMgr.HasReachedLimit(building.GetCode(), building.GetCategory()))
        {
            if(showMessage)
                gameMgr.UIMgr.ShowPlayerMessage("Building " + building.GetName() + " has reached its placement limit", UIManager.MessageTypes.error);
            return false;
        }
        
        if(!RTSHelper.TestFactionEntityRequirements(building.FactionEntityRequirements, gameMgr.GetFaction(GameManager.PlayerFactionID).FactionMgr))
        {
            if(showMessage)
                gameMgr.UIMgr.ShowPlayerMessage("Faction entity requirements for " + building.GetName() + " are missing.", UIManager.MessageTypes.error);
            return false;
        }
        
        if (!gameMgr.ResourceMgr.HasRequiredResources(building.GetResources(), GameManager.PlayerFactionID))
        {
            if(showMessage)
                gameMgr.UIMgr.ShowPlayerMessage("Not enough resources to launch task!", UIManager.MessageTypes.error);
            return false;
        }
        
        // 新增：菌毯檢查
        if (requireCreepForBuilding || System.Array.Exists(creepRequiredBuildings, code => code == building.GetCode()))
        {
            if (currentBuilding != null && !gameMgr.TerrainMgr.CanBuildOnCreep(currentBuilding.transform.position))
            {
                if(showMessage)
                    gameMgr.UIMgr.ShowPlayerMessage("This building requires creep coverage!", UIManager.MessageTypes.error);
                return false;
            }
        }
        
        return true;
    }
}
```

## 階段二：新系統添加

### 2.1 菌毯系統實現

#### 新文件創建
`Assets/RTS Engine/Creep/Scripts/CreepManager.cs`

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace RTSEngine
{
    public class CreepManager : MonoBehaviour, ICreepManager
    {
        [Header("Creep Settings")]
        [SerializeField] private float expansionRate = 1.0f;
        [SerializeField] private int creepResolution = 128;
        [SerializeField] private Material creepMaterial;
        [SerializeField] private Texture2D creepTexture;
        
        // 菌毯數據
        private float[,] creepDensity;
        private List<GrowthCenter> growthCenters;
        private Bounds worldBounds;
        
        // 渲染相關
        private MeshRenderer creepRenderer;
        private MeshFilter creepMeshFilter;
        
        // 與現有系統集成
        private GameManager gameMgr;
        private TerrainManager terrainMgr;
        
        public void Init(GameManager gameMgr)
        {
            this.gameMgr = gameMgr;
            this.terrainMgr = gameMgr.TerrainMgr;
            
            InitializeCreepData();
            SetupRendering();
            
            // 註冊到現有事件系統
            CustomEvents.BuildingPlaced += OnBuildingPlaced;
        }
        
        private void OnBuildingPlaced(Building building)
        {
            // 當建築放置時，在其位置添加菌毯生長點
            if (building.GetCode().Contains("Hive") || building.GetCode().Contains("Spawner"))
            {
                AddGrowthCenter(building.transform.position, 5.0f);
            }
        }
        
        public bool CanBuildAt(Vector3 position)
        {
            var density = GetCreepDensity(position);
            return density > 0.1f; // 需要至少10%的菌毯密度
        }
        
        public float GetCreepDensity(Vector3 position)
        {
            // 將世界坐標轉換為菌毯網格坐標
            var localPos = transform.InverseTransformPoint(position);
            var x = Mathf.Clamp(Mathf.FloorToInt((localPos.x / worldBounds.size.x + 0.5f) * creepResolution), 0, creepResolution - 1);
            var z = Mathf.Clamp(Mathf.FloorToInt((localPos.z / worldBounds.size.z + 0.5f) * creepResolution), 0, creepResolution - 1);
            
            return creepDensity[x, z];
        }
        
        public void AddGrowthCenter(Vector3 position, float strength)
        {
            growthCenters.Add(new GrowthCenter
            {
                position = new Vector2(position.x, position.z),
                strength = strength,
                radius = strength * 2.0f
            });
        }
        
        private void Update()
        {
            UpdateCreepExpansion(Time.deltaTime);
            UpdateRendering();
        }
        
        private void UpdateCreepExpansion(float deltaTime)
        {
            // 簡化的菌毯擴張算法
            for (int x = 0; x < creepResolution; x++)
            {
                for (int z = 0; z < creepResolution; z++)
                {
                    var worldPos = GridToWorldPosition(x, z);
                    var growth = CalculateGrowthAt(worldPos, deltaTime);
                    
                    creepDensity[x, z] = Mathf.Clamp01(creepDensity[x, z] + growth);
                }
            }
        }
        
        private float CalculateGrowthAt(Vector2 position, float deltaTime)
        {
            float totalGrowth = 0f;
            
            foreach (var center in growthCenters)
            {
                var distance = Vector2.Distance(position, center.position);
                if (distance < center.radius)
                {
                    var influence = (1.0f - distance / center.radius) * center.strength;
                    totalGrowth += influence * expansionRate * deltaTime;
                }
            }
            
            return totalGrowth;
        }
        
        // 其他必要方法...
    }
}
```

### 2.2 大規模單位渲染系統

#### 新文件創建
`Assets/RTS Engine/Units/Scripts/UnitRenderManager.cs`

```csharp
using System.Collections.Generic;
using UnityEngine;
using Unity.Collections;
using Unity.Jobs;

namespace RTSEngine
{
    public class UnitRenderManager : MonoBehaviour
    {
        [Header("Rendering Settings")]
        [SerializeField] private bool enableGPUInstancing = true;
        [SerializeField] private int maxInstancesPerBatch = 1000;
        [SerializeField] private float[] lodDistances = { 50f, 100f, 200f };
        
        // 渲染批次
        private Dictionary<string, UnitRenderBatch> renderBatches;
        
        // 與現有系統集成
        private GameManager gameMgr;
        
        public void Init(GameManager gameMgr)
        {
            this.gameMgr = gameMgr;
            renderBatches = new Dictionary<string, UnitRenderBatch>();
            
            // 註冊到現有事件系統
            CustomEvents.UnitCreated += OnUnitCreated;
            CustomEvents.FactionEntityDead += OnUnitDestroyed;
        }
        
        private void OnUnitCreated(Unit unit)
        {
            RegisterUnitForBatchRendering(unit);
        }
        
        private void OnUnitDestroyed(FactionEntity entity)
        {
            if (entity is Unit unit)
            {
                UnregisterUnitFromBatchRendering(unit);
            }
        }
        
        private void RegisterUnitForBatchRendering(Unit unit)
        {
            var batchKey = unit.GetCode();
            
            if (!renderBatches.ContainsKey(batchKey))
            {
                CreateRenderBatch(batchKey, unit);
            }
            
            renderBatches[batchKey].AddUnit(unit);
            
            // 隱藏原始渲染器以避免重複渲染
            if (enableGPUInstancing)
            {
                var renderers = unit.GetComponentsInChildren<MeshRenderer>();
                foreach (var renderer in renderers)
                {
                    renderer.enabled = false;
                }
            }
        }
        
        private void CreateRenderBatch(string batchKey, Unit templateUnit)
        {
            var batch = new UnitRenderBatch();
            batch.Initialize(templateUnit, maxInstancesPerBatch);
            renderBatches[batchKey] = batch;
        }
        
        private void Update()
        {
            if (!enableGPUInstancing) return;
            
            // 更新所有渲染批次
            foreach (var batch in renderBatches.Values)
            {
                batch.UpdateAndRender(Camera.main);
            }
        }
    }
    
    public class UnitRenderBatch
    {
        private List<Unit> units;
        private Matrix4x4[] matrices;
        private MaterialPropertyBlock propertyBlock;
        private Mesh sharedMesh;
        private Material sharedMaterial;
        
        public void Initialize(Unit templateUnit, int maxInstances)
        {
            units = new List<Unit>();
            matrices = new Matrix4x4[maxInstances];
            propertyBlock = new MaterialPropertyBlock();
            
            // 從模板單位獲取網格和材質
            var meshFilter = templateUnit.GetComponentInChildren<MeshFilter>();
            var meshRenderer = templateUnit.GetComponentInChildren<MeshRenderer>();
            
            if (meshFilter != null) sharedMesh = meshFilter.sharedMesh;
            if (meshRenderer != null) sharedMaterial = meshRenderer.sharedMaterial;
        }
        
        public void AddUnit(Unit unit)
        {
            if (!units.Contains(unit))
            {
                units.Add(unit);
            }
        }
        
        public void RemoveUnit(Unit unit)
        {
            units.Remove(unit);
        }
        
        public void UpdateAndRender(Camera camera)
        {
            if (units.Count == 0 || sharedMesh == null || sharedMaterial == null)
                return;
            
            // 更新變換矩陣
            for (int i = 0; i < units.Count && i < matrices.Length; i++)
            {
                if (units[i] != null)
                {
                    matrices[i] = units[i].transform.localToWorldMatrix;
                }
            }
            
            // GPU實例化渲染
            Graphics.DrawMeshInstanced(
                sharedMesh,
                0,
                sharedMaterial,
                matrices,
                Mathf.Min(units.Count, matrices.Length),
                propertyBlock
            );
        }
    }
}
```

## 階段三：網絡同步框架

### 3.1 網絡管理器基礎

#### 新文件創建
`Assets/RTS Engine/Networking/Scripts/NetworkManager.cs`

```csharp
using System.Collections.Generic;
using UnityEngine;

namespace RTSEngine.Networking
{
    public class NetworkManager : MonoBehaviour
    {
        [Header("Network Settings")]
        [SerializeField] private bool enableNetworking = false; // MVP階段可關閉
        [SerializeField] private float syncInterval = 0.1f;
        [SerializeField] private int maxPlayersPerRegion = 100;
        
        // 網絡組件
        private Dictionary<int, NetworkPlayer> connectedPlayers;
        private NetworkSyncManager syncManager;
        
        // 與現有系統集成
        private GameManager gameMgr;
        
        public void Init(GameManager gameMgr)
        {
            this.gameMgr = gameMgr;
            
            if (!enableNetworking)
            {
                Debug.Log("Network system disabled for single player mode");
                return;
            }
            
            InitializeNetworking();
        }
        
        private void InitializeNetworking()
        {
            connectedPlayers = new Dictionary<int, NetworkPlayer>();
            syncManager = new NetworkSyncManager();
            
            // 註冊到現有事件系統
            CustomEvents.UnitCreated += OnUnitCreated;
            CustomEvents.BuildingPlaced += OnBuildingPlaced;
        }
        
        private void OnUnitCreated(Unit unit)
        {
            if (enableNetworking)
            {
                syncManager.QueueUnitSync(unit);
            }
        }
        
        private void OnBuildingPlaced(Building building)
        {
            if (enableNetworking)
            {
                syncManager.QueueBuildingSync(building);
            }
        }
        
        // 為未來MMO擴展預留的接口
        public void RegisterPlayer(int playerId, Vector3 spawnPosition)
        {
            // 實現玩家註冊邏輯
        }
        
        public void TransferPlayerToRegion(int playerId, string regionId)
        {
            // 實現玩家區域轉移邏輯
        }
    }
}
```

## 整合測試策略

### 測試階段劃分

#### 階段一測試（2週）
1. **兼容性測試**: 確保所有現有功能正常工作
2. **基礎擴展測試**: 測試新增的分塊和菌毯功能
3. **性能基準測試**: 建立性能基準線

#### 階段二測試（3週）
1. **大規模單位測試**: 測試1000單位渲染性能
2. **菌毯系統測試**: 測試菌毯擴張和建築限制
3. **集成測試**: 測試新舊系統的協同工作

#### 階段三測試（4週）
1. **網絡同步測試**: 測試多人遊戲同步
2. **壓力測試**: 測試系統在高負載下的表現
3. **端到端測試**: 完整的遊戲流程測試

### 測試工具

#### 自動化測試腳本
```csharp
// 測試腳本示例
public class IntegrationTests : MonoBehaviour
{
    [Test]
    public void TestBackwardCompatibility()
    {
        // 測試現有功能是否正常
        var gameMgr = FindObjectOfType<GameManager>();
        Assert.IsNotNull(gameMgr);
        
        var terrainMgr = gameMgr.TerrainMgr;
        Assert.IsNotNull(terrainMgr);
        
        // 測試現有方法
        var height = terrainMgr.SampleHeight(Vector3.zero, 1.0f, terrainMgr.GetGroundTerrainMask());
        Assert.IsTrue(height >= 0);
    }
    
    [Test]
    public void TestCreepSystem()
    {
        var creepMgr = FindObjectOfType<CreepManager>();
        if (creepMgr != null)
        {
            creepMgr.AddGrowthCenter(Vector3.zero, 5.0f);
            
            // 等待菌毯擴張
            yield return new WaitForSeconds(1.0f);
            
            var density = creepMgr.GetCreepDensity(Vector3.zero);
            Assert.IsTrue(density > 0);
        }
    }
}
```

## 部署指南

### 版本控制策略
1. **主分支**: 保持現有穩定版本
2. **開發分支**: 進行新功能開發
3. **功能分支**: 每個新系統獨立分支
4. **測試分支**: 集成測試專用分支

### 發布流程
1. **功能完成**: 在功能分支完成開發
2. **單元測試**: 通過所有單元測試
3. **集成測試**: 合併到開發分支進行集成測試
4. **性能測試**: 確保性能指標達標
5. **發布準備**: 合併到主分支，準備發布

這個整合指南為引擎團隊提供了詳細的改造路線圖，確保能夠在保持現有功能穩定的前提下，逐步添加新的功能和優化。