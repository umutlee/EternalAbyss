# API 規格說明

## 認證系統
```
POST /auth/login
POST /auth/refresh
POST /auth/logout
```

## 玩家數據
```
GET /player/profile
PUT /player/profile
GET /player/resources
GET /player/hive
PUT /player/hive/upgrade
```

## 單位管理
```
GET /units
POST /units/produce
PUT /units/evolve
POST /units/deploy
GET /units/dominators
PUT /units/dominators/upgrade
```

## 建築系統
```
GET /buildings
POST /buildings/construct
PUT /buildings/upgrade
DELETE /buildings/demolish
```

## 探索系統
```
GET /map/nodes
POST /exploration/dispatch
GET /exploration/teams
PUT /exploration/recall
GET /exploration/results
```

## 研究系統
```
GET /research/tree
POST /research/start
GET /research/progress
POST /research/complete
```

## 實時更新
```
WebSocket /ws/game-updates
- resource_update
- production_complete
- exploration_return
- combat_result
- research_complete
```

## 離線計算
```
POST /offline/calculate
GET /offline/results
POST /offline/claim
```