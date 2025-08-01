# 數據結構設計

## 玩家數據結構

### 母巢核心數據
```typescript
interface HiveCore {
  level: number;
  experience: number;
  buildingSlots: number;
  unitCapacity: number;
  globalBonus: ResourceBonus;
  creepRadius: number;
}
```

### 資源數據
```typescript
interface Resources {
  biomass: ResourceData;
  soulCrystals: ResourceData;
  geneticCode: ResourceData;
  dominanceEssence: ResourceData;
}

interface ResourceData {
  current: number;
  capacity: number;
  productionRate: number;
  lastUpdate: timestamp;
}
```

### 單位數據
```typescript
interface Unit {
  id: string;
  type: UnitType;
  level: number;
  evolution: EvolutionPath;
  adaptations: Adaptation[];
  stats: UnitStats;
  location: NodeId;
}

interface DominatorUnit extends Unit {
  starLevel: number;
  skills: Skill[];
  assimilatedGenes: Gene[];
  mentalPower: number;
}
```

### 建築數據
```typescript
interface Building {
  id: string;
  type: BuildingType;
  level: number;
  position: Vector2;
  productionQueue: ProductionItem[];
  upgrades: BuildingUpgrade[];
}
```

## 世界數據結構

### 地圖節點
```typescript
interface MapNode {
  id: string;
  type: NodeType;
  position: Vector3;
  owner: PlayerId | null;
  resources: NodeResource[];
  enemies: Enemy[];
  creepCoverage: number;
  connections: NodeId[];
}
```

### 探索隊伍
```typescript
interface ExplorationTeam {
  id: string;
  units: Unit[];
  targetNode: NodeId;
  status: TeamStatus;
  departureTime: timestamp;
  estimatedReturn: timestamp;
  mission: MissionType;
}
```

## 配置數據結構

### 單位配置
```typescript
interface UnitConfig {
  baseStats: UnitStats;
  productionCost: Resources;
  productionTime: number;
  evolutionRequirements: EvolutionRequirement[];
  adaptationSlots: number;
}
```

### 建築配置
```typescript
interface BuildingConfig {
  buildCost: Resources;
  buildTime: number;
  maxLevel: number;
  levelBonuses: LevelBonus[];
  unlockRequirements: Requirement[];
}
```

## 計算引擎數據

### 離線計算結果
```typescript
interface OfflineResult {
  duration: number;
  resourcesGained: Resources;
  unitsProduced: Unit[];
  explorationsCompleted: ExplorationResult[];
  buildingsUpgraded: BuildingUpgrade[];
  eventsTriggered: GameEvent[];
}
```