# 深淵巢穴單位系統技術規格書

## 文檔版本
- **版本**: 1.0
- **日期**: 2024年12月
- **目標**: 大規模單位渲染和管理系統的技術規格

## 設計目標

### 性能目標
- **PC端**: 1000+單位 @30FPS
- **移動端**: 200-300單位 @30FPS
- **內存效率**: 每單位<1KB內存佔用
- **網絡效率**: 批量同步，<100KB/s per 100 units

## 階段一：MVP大規模單位系統

### 1.1 單位數據架構

#### 分離式數據設計
```csharp
// 單位數據分離：熱數據 vs 冷數據
public struct UnitHotData
{
    // 每幀需要的數據（64字節對齊）
    public Vector3 position;
    public Quaternion rotation;
    public Vector3 velocity;
    public float health;
    public int targetId;
    public UnitState state;
    public float stateTimer;
}

public struct UnitColdData
{
    // 不常變化的數據
    public int unitId;
    public UnitType type;
    public int ownerId;
    public UnitStats baseStats;
    public Evolution[] evolutions;
    public Adaptation[] adaptations;
    public string prefabPath;
}

// 單位管理器 - 數據導向設計
public class UnitManager
{
    // 熱數據數組 - 緊密排列，緩存友好
    private UnitHotData[] hotData;
    private Dictionary<int, UnitColdData> coldData;
    
    // 空間分割 - 用於快速查詢
    private SpatialGrid spatialGrid;
    
    // 渲染批次 - 按類型分組
    private Dictionary<UnitType, RenderBatch> renderBatches;
}
```

### 1.2 大規模渲染系統

#### GPU實例化渲染
```csharp
// 渲染批次管理
public class UnitRenderBatch
{
    public UnitType unitType;
    public Mesh sharedMesh;
    public Material sharedMaterial;
    
    // GPU實例化數據
    private Matrix4x4[] transformMatrices;
    private Vector4[] instanceColors;
    private float[] healthRatios;
    
    // LOD系統
    private LODGroup[] lodGroups;
    private float[] lodDistances;
    
    public void Render(Camera camera)
    {
        // 視錐體剔除
        var visibleInstances = FrustumCull(camera);
        
        // LOD選擇
        var lodLevel = CalculateLOD(camera.transform.position);
        
        // GPU實例化渲染
        Graphics.DrawMeshInstanced(
            sharedMesh, 
            0, 
            sharedMaterial, 
            transformMatrices, 
            visibleInstances.Count,
            materialPropertyBlock
        );
    }
}
```

#### 多線程更新系統
```csharp
// 單位更新作業系統
public struct UnitUpdateJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<UnitColdData> coldData;
    public NativeArray<UnitHotData> hotData;
    [ReadOnly] public float deltaTime;
    
    public void Execute(int index)
    {
        var hot = hotData[index];
        var cold = coldData[index];
        
        // 移動更新
        UpdateMovement(ref hot, cold, deltaTime);
        
        // 戰鬥更新
        UpdateCombat(ref hot, cold, deltaTime);
        
        // 狀態更新
        UpdateState(ref hot, cold, deltaTime);
        
        hotData[index] = hot;
    }
}
```

### 1.3 AI系統架構

#### 分層AI系統
```csharp
// AI決策層次
public enum AILayer
{
    Strategic,  // 戰略AI（母巢級別）
    Tactical,   // 戰術AI（群體級別）
    Individual  // 個體AI（單位級別）
}

// 群體AI管理器
public class SwarmAIManager
{
    // 群體行為
    private Dictionary<int, SwarmGroup> swarmGroups;
    
    // 行為樹緩存
    private Dictionary<UnitType, BehaviorTree> behaviorTrees;
    
    // 決策頻率控制
    private float[] lastDecisionTime;
    private float[] decisionIntervals;
    
    public void UpdateSwarmAI(float deltaTime)
    {
        // 分幀更新，避免CPU峰值
        var unitsPerFrame = Mathf.CeilToInt(totalUnits / 60f);
        
        for (int i = 0; i < unitsPerFrame; i++)
        {
            var unitIndex = (currentFrameOffset + i) % totalUnits;
            UpdateUnitAI(unitIndex, deltaTime);
        }
        
        currentFrameOffset = (currentFrameOffset + unitsPerFrame) % totalUnits;
    }
}

// 群體行為定義
public struct SwarmGroup
{
    public int groupId;
    public List<int> unitIds;
    public Vector3 centerPosition;
    public Vector3 targetPosition;
    public SwarmBehavior behavior;
    public float cohesionRadius;
    public float separationRadius;
}
```

## 階段二：進化系統規格

### 2.1 進化數據結構

#### 進化樹系統
```csharp
// 進化節點
[System.Serializable]
public struct EvolutionNode
{
    public string evolutionId;
    public string displayName;
    public UnitType baseType;
    public UnitType evolvedType;
    
    // 進化需求
    public ResourceCost cost;
    public string[] requiredResearch;
    public int minUnitLevel;
    
    // 屬性變化
    public StatModifier[] statModifiers;
    public Ability[] newAbilities;
    public string[] removedAbilities;
    
    // 視覺變化
    public string evolvedPrefabPath;
    public AnimationClip evolutionEffect;
}

// 進化管理器
public class EvolutionManager
{
    // 進化樹數據
    private Dictionary<UnitType, EvolutionNode[]> evolutionTrees;
    
    // 進化隊列
    private Queue<EvolutionRequest> evolutionQueue;
    
    // 進化效果池
    private ObjectPool<GameObject> evolutionEffectPool;
    
    public bool CanEvolve(int unitId, string evolutionId)
    {
        // 檢查進化條件
        var unit = GetUnit(unitId);
        var evolution = GetEvolution(evolutionId);
        
        return CheckRequirements(unit, evolution);
    }
    
    public void StartEvolution(int unitId, string evolutionId)
    {
        var request = new EvolutionRequest
        {
            unitId = unitId,
            evolutionId = evolutionId,
            startTime = Time.time
        };
        
        evolutionQueue.Enqueue(request);
    }
}
```

### 2.2 適應性系統

#### 環境適應機制
```csharp
// 適應性特徵
[System.Serializable]
public struct Adaptation
{
    public string adaptationId;
    public string displayName;
    public AdaptationType type;
    public EnvironmentType targetEnvironment;
    
    // 適應效果
    public StatModifier[] environmentBonuses;
    public DamageResistance[] resistances;
    public MovementModifier movementChanges;
    
    // 獲取條件
    public int requiredExposureTime;
    public float adaptationChance;
}

// 環境系統
public class EnvironmentManager
{
    // 環境區域
    private Dictionary<Vector3, EnvironmentZone> environmentZones;
    
    // 單位環境暴露追蹤
    private Dictionary<int, EnvironmentExposure> unitExposures;
    
    public void UpdateEnvironmentalAdaptation(float deltaTime)
    {
        foreach (var unitId in activeUnits)
        {
            var position = GetUnitPosition(unitId);
            var environment = GetEnvironmentAt(position);
            
            UpdateExposure(unitId, environment, deltaTime);
            CheckAdaptationTrigger(unitId);
        }
    }
}
```

## 階段三：MMO擴展規格

### 3.1 網絡同步優化

#### 單位狀態同步
```csharp
// 網絡單位數據
public struct NetworkUnitData
{
    public int unitId;
    public Vector3 position;
    public byte facing; // 壓縮的旋轉數據
    public byte health; // 百分比健康值
    public UnitState state;
    public int targetId;
}

// 批量同步系統
public class UnitNetworkManager
{
    // 同步頻率控制
    private float[] lastSyncTime;
    private float[] syncIntervals;
    
    // 數據壓縮
    private BitPacker bitPacker;
    
    public byte[] PackUnitUpdates(List<int> unitIds)
    {
        bitPacker.Reset();
        
        foreach (var unitId in unitIds)
        {
            var unit = GetUnit(unitId);
            PackUnitData(unit, bitPacker);
        }
        
        return bitPacker.GetBytes();
    }
    
    // 預測性同步
    public void PredictUnitMovement(int unitId, float deltaTime)
    {
        var unit = GetUnit(unitId);
        
        // 客戶端預測
        var predictedPosition = unit.position + unit.velocity * deltaTime;
        
        // 平滑插值到服務器位置
        unit.position = Vector3.Lerp(
            predictedPosition, 
            unit.serverPosition, 
            Time.deltaTime * interpolationSpeed
        );
    }
}
```

### 3.2 分佈式單位管理

#### 區域單位管理
```csharp
// 區域單位管理器
public class RegionalUnitManager
{
    // 區域劃分
    private Dictionary<string, UnitRegion> regions;
    
    // 跨區域單位追蹤
    private Dictionary<int, string> unitRegionMap;
    
    // 區域間通信
    private IRegionCommunicator communicator;
    
    public void TransferUnit(int unitId, string fromRegion, string toRegion)
    {
        var unitData = ExtractUnitData(unitId);
        
        // 從源區域移除
        regions[fromRegion].RemoveUnit(unitId);
        
        // 發送到目標區域
        communicator.SendUnitTransfer(toRegion, unitData);
        
        // 更新映射
        unitRegionMap[unitId] = toRegion;
    }
}
```

## 性能優化規格

### 內存優化
```csharp
// 對象池管理
public class UnitObjectPool
{
    private Dictionary<UnitType, Queue<GameObject>> pools;
    private Dictionary<UnitType, int> maxPoolSizes;
    
    public GameObject GetUnit(UnitType type)
    {
        if (pools[type].Count > 0)
        {
            return pools[type].Dequeue();
        }
        
        return CreateNewUnit(type);
    }
    
    public void ReturnUnit(GameObject unit, UnitType type)
    {
        if (pools[type].Count < maxPoolSizes[type])
        {
            ResetUnit(unit);
            pools[type].Enqueue(unit);
        }
        else
        {
            DestroyImmediate(unit);
        }
    }
}
```

### CPU優化
```csharp
// 分幀處理系統
public class FrameDistributedProcessor
{
    private Queue<System.Action> taskQueue;
    private float maxProcessingTimePerFrame;
    
    public void Update()
    {
        var startTime = Time.realtimeSinceStartup;
        
        while (taskQueue.Count > 0 && 
               (Time.realtimeSinceStartup - startTime) < maxProcessingTimePerFrame)
        {
            var task = taskQueue.Dequeue();
            task.Invoke();
        }
    }
    
    public void ScheduleTask(System.Action task)
    {
        taskQueue.Enqueue(task);
    }
}
```

## 引擎改造指導

### 現有系統擴展

#### 1. 擴展現有Unit類
```csharp
// 擴展現有的Unit類
public partial class Unit
{
    // 新增大規模渲染支持
    public int renderBatchId;
    public int instanceId;
    
    // 新增進化系統
    public Evolution currentEvolution;
    public List<Adaptation> adaptations;
    
    // 保持現有接口不變
    public void SetTargetPosition(Vector3 targetPos)
    {
        // 現有實現保持不變
        // 新增批量更新邏輯
        UnitManager.Instance.MarkForBatchUpdate(this);
    }
}
```

#### 2. 擴展GridSearchHandler
```csharp
// 為大規模單位優化搜索
public partial class GridSearchHandler
{
    // 新增批量搜索
    public List<T> BatchSearch<T>(
        Vector3[] positions, 
        float[] radii,
        System.Func<T, bool> filter
    ) where T : Entity
    {
        // 批量處理多個搜索請求
        // 減少重複的空間查詢
    }
}
```

## 測試規格

### 性能測試
```csharp
// 性能測試套件
public class UnitPerformanceTests
{
    [Test]
    public void Test1000UnitsRendering()
    {
        // 創建1000個單位
        var units = CreateTestUnits(1000);
        
        // 測試渲染性能
        var frameTime = MeasureFrameTime();
        
        Assert.Less(frameTime, 33.33f); // 30FPS
    }
    
    [Test]
    public void TestMemoryUsage()
    {
        var initialMemory = GC.GetTotalMemory(true);
        
        var units = CreateTestUnits(1000);
        
        var finalMemory = GC.GetTotalMemory(false);
        var memoryPerUnit = (finalMemory - initialMemory) / 1000;
        
        Assert.Less(memoryPerUnit, 1024); // <1KB per unit
    }
}
```

## 實施時間表

### 第一階段（3週）
1. 實現基礎的大規模渲染系統
2. 創建單位對象池
3. 實現基礎的批量更新

### 第二階段（4週）
1. 實現進化系統
2. 添加適應性機制
3. 集成到現有戰鬥系統

### 第三階段（5週）
1. 實現網絡同步
2. 添加分佈式管理
3. 性能優化和測試

這個規格書為引擎團隊提供了清晰的改造方向，確保能夠支持從MVP到MMO的平滑擴展。