# 技術架構文檔

## 系統架構概覽

### 客戶端架構
- **引擎**: Unity 2022.3 LTS
- **平台**: iOS/Android
- **渲染**: 2D/2.5D混合
- **UI框架**: UGUI + DOTween

### 服務端架構
- **語言**: Node.js/TypeScript
- **數據庫**: MongoDB + Redis
- **消息隊列**: RabbitMQ
- **部署**: Docker + Kubernetes

## 核心模塊設計

### 1. 決策系統 (Decision System)
```
DecisionManager
├── EvolutionDecision (進化決策)
├── ResourceAllocation (資源分配)
├── ExplorationPriority (探索優先級)
└── UnitDevelopment (單位培養)
```

### 2. 自動化引擎 (Automation Engine)
```
AutomationCore
├── ProductionManager (生產管理)
├── CombatSimulator (戰鬥模擬)
├── ResourceProcessor (資源處理)
└── ExpansionController (擴張控制)
```

### 3. 進度計算系統 (Progress Calculator)
```
ProgressEngine
├── OfflineCalculator (離線計算)
├── RealtimeUpdater (實時更新)
├── MilestoneTracker (里程碑追蹤)
└── BalanceValidator (平衡驗證)
```

## 數據流設計

### 決策 → 執行流程
1. 玩家做出決策
2. 客戶端驗證合法性
3. 服務端接收並處理
4. 自動化引擎開始執行
5. 定期同步結果到客戶端

### 離線處理機制
- 服務端持續計算玩家進度
- 登錄時一次性同步所有變化
- 防止時間作弊和數據篡改