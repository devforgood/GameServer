# 길찾기 · 이동 · 충돌 처리 구조

LeagueSandbox 서버가 지형을 어떻게 읽고, 경로를 어떻게 찾고, 유닛을 어떻게 움직이며, 유닛끼리 겹칠 때 어떻게 처리하는지 코드 기준으로 정리한 문서입니다.

> 요약
> - **네비 메시를 서버에서 굽지 않습니다.** 4.20 클라이언트에 들어 있는 미리 구워진 격자 파일(`AIPath.aimesh_ngrid`)을 그대로 읽어 씁니다. 셀 하나는 50×50 유닛입니다.
> - 길찾기는 이 격자 위에서 도는 **8방향 A\*** 입니다. 유닛 반지름만큼 원을 쓸어 지나가는 검사(`CastCircle`)로 통행을 판정하고, 끝나면 시야 기반 경로 단순화(string pulling)를 합니다.
> - 챔피언은 **클라이언트가 먼저 경로를 계산해 웨이포인트를 보내고**, 서버는 검증만 하다가 막힌 구간이 있을 때만 A\*를 다시 돌립니다.
> - 유닛은 웨이포인트를 **일정 속도로 직선 보간**하며 따라갑니다(60Hz 틱).
> - **RVO/ORCA 같은 속도 기반 회피는 없습니다.** 매 틱 원-원 겹침을 검사해 겹쳤으면 **위치를 밀어내는(position correction)** 방식만 씁니다. 경로 계산은 다른 유닛을 전혀 고려하지 않습니다.

---

## 0. 전체 흐름

```
[서버 시작]
  Package.LoadPackage()
    └─ Content/LeagueSandbox-Default/AIMesh/Map{N}/*.aimesh_ngrid 를 전부 NavigationGrid 로 파싱
  MapScriptHandler.Init()
    └─ NavigationGrid = ContentManager.GetNavigationGrid()   ("AIPath" 또는 NavGridOverride)
    └─ CollisionHandler / PathingHandler 생성

[매 틱, 60Hz]  Game.Update(diff)
  1) Map.Update
       ├─ CollisionHandler.Update()   ← 지형/유닛 겹침 검사 → 위치 밀어내기
       └─ PathingHandler.Update()     ← (3초 주기 경로 재검증, 실제로는 동작 안 함 — 7장 참고)
  2) ObjectManager.Update
       ├─ 각 오브젝트 Update()        ← AI 판단(RefreshWaypoints 등) + Move()/DashMove()
       └─ 바뀐 이동 정보 → WaypointGroup 패킷 전송
```

---

## 1. 지형 데이터: 클라이언트의 Navigation Grid를 그대로 사용

### 1.1 어디서 오는가

| 항목 | 내용 |
|---|---|
| 파일 | `Content/LeagueSandbox-Default/AIMesh/Map{N}/AIPath.aimesh_ngrid` |
| 출처 | 리그 오브 레전드 클라이언트의 레벨 데이터(`LEVELS/Map{N}/AIPath.aimesh_ngrid`)를 뽑아 둔 것. Riot 툴이 미리 구운 결과물이라 **서버에는 굽는(bake) 코드가 없습니다.** |
| 로드 | [Package.cs:365-373](GameServerLib/Content/Package.cs#L365-L373): 맵 폴더의 `*.aimesh_ngrid` 를 모두 읽어 파일명(확장자 제외)을 키로 저장 |
| 선택 | [Package.cs:247-256](GameServerLib/Content/Package.cs#L247-L256): 기본은 `"AIPath"`, 맵 스크립트의 `NavGridOverride` 로 교체 가능 (예: Map8 `AIPathASCENSION`) |
| 같이 있는 `AIPath.aimesh` | 삼각형 메시 원본입니다. **서버는 읽지 않습니다.** Map4는 `.aimesh` 만 있고 `.aimesh_ngrid` 가 없어서 네비 그리드를 로드할 수 없습니다. |

현재 포함된 그리드(헤더를 직접 읽어 확인한 값):

| 맵 | 버전 | 셀 크기 | 셀 수 (X×Y) |
|---|---|---|---|
| Map1 (구 소환사의 협곡) | 3.1 | 50 | 293 × 294 |
| Map8 | 3.257 / ASCENSION 5.259 | 50 | 280 × 231 |
| Map10 | 3.1 | 50 | 306 × 129 |
| Map11 | 5.1 | 50 | 295 × 296 |
| Map12 | 3.1 | 50 | 248 × 240 |

### 1.2 파일 구조와 파싱 — [NavigationGrid.cs:98-172](GameServerLib/Content/Navigation/NavigationGrid.cs#L98-L172)

```
byte   major                  (2, 3, 5, 7 만 지원)
ushort minor                  (major != 2 일 때만)
Vector3 MinGridPosition, MaxGridPosition   // 월드 좌표 바운딩 박스
float  CellSize               // 50
uint   CellCountX, CellCountY
Cell[CellCountX*CellCountY]   // v2/3/5: ReadVersion5, v7: ReadVersion7 + 이후 flags 배열 별도
(v5) ushort RegionTags[] / (v7) uint RegionTags[]
(v5+) RegionTagTable          // 레인/정글 구역 태그 (v5: 4그룹, v7: 8그룹)
uint SampledHeightsCountX, Y; Vector2 SampledHeightsDistance; float SampledHeights[]
HintGrid                      // 읽기만 하고 미사용
```

셀 하나([NavigationGridCell.cs:49-118](GameServerLib/Content/Navigation/NavigationGridCell.cs#L49-L118))에는 클라이언트 A\* 의 런타임 상태가 그대로 덤프되어 있습니다(`ArrivalCost`, `Heuristic`, `IsOpen`, `SessionId`, `AdditionalCost`, `RefHintNode` 등). 서버가 실제로 의미 있게 쓰는 값은 아래 셋입니다.

- **`Flags`** — 통행/시야 판정의 핵심 ([NavigationGridCellFlags.cs](GameServerCore/Enums/NavigationGridCellFlags.cs))
  - `NOT_PASSABLE (0x2)`: 벽
  - `SEE_THROUGH (0x40)`: 시야는 통과하지만 걸을 수는 없음 (건물·포탑 등이 이 방식으로 그리드에 구워져 있음)
  - `HAS_GRASS (0x1)`: 부시
  - `HAS_GLOBAL_VISION (0x100)`
- **`Locator (X, Y)`** — 셀의 격자 좌표
- **`ArrivalCost`, `AdditionalCost`, `Heuristic`** — A\* 비용에 그대로 더해짐 (2장)

### 1.3 좌표계

| 공간 | 원점 | 변환 |
|---|---|---|
| 서버 월드 좌표 | 맵 좌하단(블루 우물 뒤) | — |
| 격자 좌표 | `MinGridPosition` | `(world - Min) / CellSize` — [TranslateToNavGrid](GameServerLib/Content/Navigation/NavigationGrid.cs#L345-L352) |
| 패킷(클라이언트) 좌표 | 맵 중앙, 1/2 스케일 | `(world - MiddleOfMap) / 2` — [MovementVector.cs](GameServerLib/MovementVector.cs), 역변환은 [HandleMove.TranslateFromCenteredCoordinates](GameServerLib/Packets/PacketHandlers/HandleMove.cs#L111-L116) |

높이는 `SampledHeights` 를 쌍선형 보간해서 구합니다([GetHeightAtLocation](GameServerLib/Content/Navigation/NavigationGrid.cs#L571-L642)). 패킷에 Y 값을 채울 때만 쓰이고, 길찾기는 순수 2D입니다.

### 1.4 통행 판정 기본 연산

| 함수 | 동작 |
|---|---|
| `IsWalkable(cell)` | `!NOT_PASSABLE && !SEE_THROUGH` |
| `IsWalkable(pos, radius)` | radius가 0이면 셀 하나만, 아니면 원 안에 들어오는 모든 셀이 통행 가능해야 true |
| `CastRay` | 격자 레이 트레이싱(playtechs 알고리즘). 통행/시야/부시 판정 |
| **`CastCircle(a, b, r)`** | a→b 로 반지름 r 원을 쓸어 지나가는 캡슐 영역의 모든 셀 검사. 시작/끝 원 + 양옆 평행선 두 개로 외곽을 잡고, 행마다 X 범위 안쪽을 채워 검사. **막히면 true** — [NavigationGrid.cs:787-840](GameServerLib/Content/Navigation/NavigationGrid.cs#L787-L840) |
| `GetClosestTerrainExit(pos, r)` | pos가 걸을 수 없으면 45°씩 돌며 반지름을 1씩 키우는 나선형으로 이동해, 처음 만나는 통행 가능 지점 반환 |

---

## 2. 길찾기: 격자 A\* + 경로 단순화

진입점: [NavigationGrid.GetPath(from, to, distanceThreshold)](GameServerLib/Content/Navigation/NavigationGrid.cs#L181-L310). 래퍼는 [PathingHandler.GetPath](GameServerLib/Handlers/PathingHandler.cs#L117-L137)이고, `distanceThreshold` 에는 보통 유닛의 `PathfindingRadius`(CharData의 `PathfindingCollisionRadius`, 없으면 40)를 넘깁니다.

### 2.1 알고리즘

1. **목표 보정**: `to = GetClosestTerrainExit(to, radius)` — 벽 안을 찍어도 가장 가까운 통행 가능 지점으로 옮김.
2. 시작/목표 셀이 같으면 `[from, to]` 를 바로 반환.
3. **우선순위 큐 A\***
   - 큐 원소: *지금까지의 셀 리스트 전체*와 누적 비용 g
   - 이웃: 8방향(자기 자신 포함 3×3, 자기 자신은 closed라 건너뜀)
   - 간선 유효성: 현재 셀 중심 → 이웃 셀 중심을 `CastCircle(..., radius)` 로 검사. 막히면 그 이웃을 closed 처리. 시작 셀은 셀 중심 대신 실제 `from` 좌표를, 목표 셀은 검사 없이 통과시킴.
   - 비용: `g' = g + 1 + cell.ArrivalCost + cell.AdditionalCost` (대각선도 1)
   - 우선순위: `g' + cell.Heuristic + |이웃 중심 - 목표|` (격자 단위 유클리드 거리)
   - **큐에 넣는 순간 closed 처리** (일반적인 "꺼낼 때 closed" 방식과 다름)
4. **SmoothPath** ([NavigationGrid.cs:316-337](GameServerLib/Content/Navigation/NavigationGrid.cs#L316-L337)): 마지막으로 남긴 점 j에서 i까지 `CastCircle` 이 통과하면 중간 점을 버리는 그리디 string pulling.
5. 반환: `[from, (중간 셀 중심들, 월드 좌표), to]`

### 2.2 특성 / 주의점

- 다른 유닛은 전혀 고려하지 않습니다. 셀의 `IsOpen`/`ActorList` 는 읽어 오기만 하고 어디서도 갱신하거나 참조하지 않습니다. 동적 장애물(유닛, 소환물)은 경로 계산에 없습니다.
- 넣을 때 closed 처리하고 대각선도 비용 1이라 **최단 경로가 보장되지 않습니다.** SmoothPath가 결과를 상당 부분 보정해 줍니다.
- 확장할 때마다 경로 리스트를 통째로 복사하므로 길이가 긴 경로에서는 메모리/GC 부담이 큽니다.
- 경로를 찾지 못하면 `null` 을 반환합니다. 호출 측에서 대체 처리를 해야 합니다.

---

## 3. 이동 명령 → 경로 결정

### 3.1 챔피언 (플레이어 입력) — [HandleMove.cs](GameServerLib/Packets/PacketHandlers/HandleMove.cs)

4.20 클라이언트는 **자기 쪽에서 같은 navgrid로 경로를 계산해서** `MovementRequest` 에 웨이포인트 리스트로 담아 보냅니다. 서버는 이 경로를 검증합니다.

```
MoveTo / AttackTo / AttackMove / Use:
  waypoints = req.Waypoints → 월드 좌표 변환
  waypoints[0] = 서버가 알고 있는 champion.Position   // 시작점은 서버 기준으로 강제
  for 각 구간 (i → i+1):
      if CastCircle(waypoints[i], waypoints[i+1], PathfindingRadius) 가 막힘:
          path = GetPath(waypoints[i], 마지막 웨이포인트, PathfindingRadius)   // 서버 A*
          i 이후를 path 로 교체 (실패하면 waypoints[i] 에서 멈춤)
          break
  champion.SetWaypoints(waypoints)

PetHard*: 펫은 클라이언트 경로 없이 서버가 GetPath(pet.Position, req.Position) 로 계산
```

막힌 구간이 없으면 서버는 **A\*를 돌리지 않고 클라이언트 경로를 그대로 씁니다.**

### 3.2 서버가 직접 경로를 만드는 경우

| 호출 위치 | 언제 |
|---|---|
| [ObjAIBase.RefreshWaypoints](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L592-L668) | 타겟을 쫓을 때(`AttackTo`). 사거리 밖이면 **매 틱** `GetPath(Position, target.Position)` 호출 |
| [ObjAIBase.SetSpellToCast](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L885-L910) | 스킬 시전 위치까지 걸어갈 때 |
| [AttackableUnit.SetPathTrueEnd](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L964-L986) | 스크립트 등에서 목적지를 지정할 때 |
| [AttackableUnit.SetPosition(repath:true)](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L195-L235) | 위치가 강제로 바뀐 뒤, 원래 목적지까지 경로를 다시 계산할 때 |
| [Champion.cs:307](GameServerLib/GameObjects/AttackableUnits/AI/Champion.cs#L307) | 리스폰 위치로 이동 |

### 3.3 라인 미니언 — [LaneMinionAI.cs](Content/LeagueSandbox-Scripts/AIScripts/LaneMinionAI.cs)

- 레인 경로는 **맵 스크립트에 하드코딩된 좌표 리스트**입니다(`LevelScript.MinionPaths`, 예: [Map1/LevelScript.cs:41](Content/LeagueSandbox-Scripts/Maps/Map1/LevelScript.cs#L41)). 퍼플 팀은 리스트를 뒤집어 씁니다.
- 평소에는 A\* 없이 `[현재 위치, 다음 레인 웨이포인트]` 직선 경로만 설정합니다.
- 타겟을 쫓다가 레인으로 복귀할 때만 `GetPath` 로 A\* 를 한 번 돌립니다.
- 웨이포인트 도달 판정(`WaypointReached`)은 서로 겹쳐 있는 미니언들을 하나의 큰 원으로 합쳐서 판정합니다. 뭉친 미니언이 같은 웨이포인트를 두고 다투지 않게 하려는 처리입니다.

---

## 4. 유닛 이동 시 경로 참조

### 4.1 경로 표현

[AttackableUnit](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L85-L95):

- `Waypoints : List<Vector2>` — 항상 `[0]` 이 현재(설정 시점) 위치
- `CurrentWaypointKey` — 다음에 향할 인덱스 (초기값 1)
- `SetWaypoints(list)` 는 `list[0] == Position` 이고 `CanChangeWaypoints()` 일 때만 받아들임. 받아들이면 `_movementUpdated = true` (네트워크 동기화 플래그)

### 4.2 일반 이동 — [AttackableUnit.Move](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L926-L957)

```
maxDist = MoveSpeed * diff(ms) / 1000
loop:
    dir = CurrentWaypoint - Position
    if maxDist < |dir|:  Position += normalize(dir) * maxDist; 끝
    else:                Position = CurrentWaypoint; maxDist -= |dir|; CurrentWaypointKey++
                         (마지막 웨이포인트면 끝)
```

- 일정 속도 직선 보간이고 가속/회전 속도 개념은 없습니다. 한 틱 안에서 남은 이동 거리를 다음 구간으로 넘기기 때문에 웨이포인트를 건너뛰어도 속도가 유지됩니다.
- 이동 중에는 지형을 다시 검사하지 않습니다. 지형 침범은 다음 틱 CollisionHandler가 사후에 보정합니다(5장).
- [ObjAIBase.Move](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L448-L459): `MoveOrder` 가 `CastSpell / OrderNone / Stop / Taunt` 면 이동하지 않습니다. 이동 가능 여부(`CanMove`)는 CC 상태(기절·속박·제압 등)와 시전 상태를 반영합니다.

### 4.3 대시/강제 이동 — [AttackableUnit.DashMove](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L862-L919)

- `MovementParameters` 가 있으면 일반 이동 대신 대시 처리를 합니다.
- 위치 대시(`DashToLocation`): 목적지만 `GetClosestTerrainExit` 로 보정하고, 직선으로 이동합니다(중간 벽 무시).
- 대상 추적 대시(`DashToTarget`): 매 틱 대상 방향으로 이동, 두 유닛 반지름 합만큼 앞에서 멈춤.
- 대시 중에는 유닛 충돌과 지형 밀어내기를 모두 건너뜁니다.

### 4.4 클라이언트로 경로 전송

- 이동 정보가 바뀌면(`SetWaypoints`, `SetPosition`, `TeleportTo`, `StopMovement`) `_movementUpdated` 가 켜집니다.
- 매 틱 끝 `ObjectManager` 가 `HoldMovementDataUntilWaypointGroupNotification` → [`NotifyWaypointGroup`](GameServerLib/Packets/PacketNotifier.cs#L4128) 로 `WaypointGroup` 패킷을 보냅니다.
- 보내는 내용은 **현재 위치 + 아직 남은 웨이포인트**입니다(중앙 원점, 1/2 스케일로 압축 — [PacketExtensions.GetCenteredWaypoints](GameServerLib/Packets/PacketExtensions.cs#L97-L110)).
- 클라이언트는 이 웨이포인트를 받아 스스로 보간 재생합니다. 서버가 매 틱 위치를 보내는 방식이 아닙니다. 경로가 바뀔 때만 전송합니다.

---

## 5. 충돌 처리 (RVO 여부)

### 5.1 결론: RVO/ORCA/스티어링 회피 없음

코드 전체(`GameServerLib`, `GameServerCore`, `GameMaths`, 스크립트)에서 RVO·ORCA·velocity obstacle·separation·steering 관련 구현은 없습니다. 대신 아래의 **겹침 검출 → 위치 밀어내기** 방식만 사용합니다.

### 5.2 CollisionHandler — [CollisionHandler.cs](GameServerLib/Handlers/CollisionHandler.cs)

매 틱 `Map.Update` → `CollisionHandler.Update()`:

1. 등록된 오브젝트마다 `UpdateCollision(obj)`
   - **지형**: `IsWalkable(obj.Position)` — **반지름 없이 중심점 셀 하나만** 검사. 벽 셀이면 `OnCollision(null, isTerrain: true)`.
   - **오브젝트**: QuadTree에서 주변 후보를 뽑고, `|p1 - p2| < r1 + r2` (`CollisionRadius` 기준 원-원) 이면 `OnCollision(other)`.
2. QuadTree를 **매 틱 전부 비우고 다시 삽입** (이동한 오브젝트 섹터 갱신용).

대상 분류:

| | 충돌체 (QuadTree에 들어감) | 충돌 영향 받음 (OnCollision 호출됨) |
|---|---|---|
| 챔피언·미니언·몬스터 | O | O |
| 포탑 / 건물 | O | X |
| 미사일 | X | O |
| LevelProp, Particle | X | X |

### 5.3 충돌 반응 — [AttackableUnit.OnCollision](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L278-L319)

- 미사일·섹터·건물·자기 Region과의 충돌은 무시합니다. 건물은 navgrid에 이미 구워져 있다는 전제입니다.
- **지형과 겹침**: `GetClosestTerrainExit(Position, PathfindingRadius + 1)` 로 **순간이동**합니다(`SetPosition(exit, repath:false)`).
- **유닛과 겹침**: 대시 중이거나 `Ghosted` 상태면 무시. 아니면
  `exit = GetCircleEscapePoint(내 위치, 내 반지름+1, 상대 위치, 상대 반지름)` — 두 원이 딱 닿는 위치까지 **상대 반대 방향으로 밀어냅니다.** 그 점이 벽이면 다시 `GetClosestTerrainExit` 로 보정합니다.
- 쌍방이 각자 `OnCollision` 을 받아 각각 밀려납니다. 질량·우선순위 개념은 없습니다.
- `SetPosition(..., repath:false)` 는 `Waypoints[0]` 만 바꾸고 `_movementUpdated` 를 켭니다. 따라서 **밀려날 때마다 새 WaypointGroup 패킷이 나갑니다.**

### 5.4 그 밖의 겹침 회피

- [ObjAIBase.RecalculateAttackPosition](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L553-L587): 타겟이 사거리 밖일 때 주변 유닛과 `PathfindingRadius` 로 겹치면 탈출 지점으로 짧은 경로를 줍니다. [MinionAI.cs:80](Content/LeagueSandbox-Scripts/AIScripts/MinionAI.cs#L80)(일반 미니언/펫)에서만 쓰이고, LaneMinionAI에서는 주석 처리되어 있습니다.
- 라인 미니언의 `WaypointReached` 클러스터 판정(3.3).

### 5.5 이 방식의 결과

- 유닛은 서로를 피해 돌아가지 않습니다. 일단 경로대로 걸어가서 겹친 다음, 다음 틱에 밀려나는 일을 반복합니다. 뭉친 미니언이나 좁은 길목에서 떨림/미끄러짐이 생기는 원인입니다.
- 매 틱 갱신되는 위치는 A\*가 고려하지 않은 다른 유닛을 기준으로 정해집니다. 그래서 경로와 실제 위치가 계속 어긋나고 패킷도 자주 나갑니다.

---

## 6. 반지름 정리

| 속성 | 출처 | 쓰는 곳 |
|---|---|---|
| `PathfindingRadius` | `CharData.PathfindingCollisionRadius`, 없으면 40 | A\* 의 `CastCircle` 반지름, 지형 탈출, 대시 정지 거리, 유닛 밀어내기 거리 |
| `CollisionRadius` | 생성자 인자 → `CharData.GameplayCollisionRadius` → 40 | 오브젝트 간 겹침 판정, QuadTree 바운드, 평타 사거리 보정 |

겹침은 `CollisionRadius` 로 검출하고, 밀어내는 거리는 `PathfindingRadius` 로 계산합니다. 두 값이 다르면 밀어낸 뒤에도 여전히 겹친 상태로 판정되거나 필요 이상으로 밀려날 수 있습니다.

---

## 7. 알려진 한계 / 확인된 문제

| # | 내용 | 위치 |
|---|---|---|
| 1 | **`PathingHandler.Update` 의 3초 주기 경로 재검증이 사실상 죽은 코드.** `AddPathfinder` 를 호출하는 곳이 없어 `_pathfinders` 가 항상 비어 있음 | [PathingHandler.cs:27](GameServerLib/Handlers/PathingHandler.cs#L27) |
| 2 | 경로 계산이 동적 장애물(유닛)을 무시. `PathingHandler.IsWalkable(checkObjects)` 옵션이 있지만 기본값 false이고 호출 측에서 켜지 않음 | [PathingHandler.cs:102](GameServerLib/Handlers/PathingHandler.cs#L102) |
| 3 | 속도 기반 충돌 회피(RVO 등) 없음. 겹침 후 위치 보정만 함 | 5장 |
| 4 | 지형 충돌 검사가 중심점 하나만 봄 → 몸체 일부가 벽에 들어가도 감지 못함 | [CollisionHandler.cs:141](GameServerLib/Handlers/CollisionHandler.cs#L141) |
| 5 | 빠른 오브젝트가 한 틱에 상대를 통과하는 터널링 미처리 (코드 TODO) | [CollisionHandler.cs:149](GameServerLib/Handlers/CollisionHandler.cs#L149) |
| 6 | 추적 중 `RefreshWaypoints` 가 매 틱 A\* 실행 → 쫓는 유닛이 많으면 비용 증가 | [ObjAIBase.cs:661](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L661) |
| 7 | A\*: 넣을 때 closed, 대각선 비용 1 → 최적 경로 비보장. 확장할 때마다 경로 리스트 전체 복사 | [NavigationGrid.cs:205-288](GameServerLib/Content/Navigation/NavigationGrid.cs#L205-L288) |
| 8 | `GetCell` 의 경계 검사가 `x > CellCountX` (≥ 가 맞음) → x == CellCountX 이면 다음 행 첫 셀로 잘못 매핑될 수 있음 | [NavigationGrid.cs:397](GameServerLib/Content/Navigation/NavigationGrid.cs#L397) |
| 9 | `GetClosestTerrainExit` 나선 탐색에 상한이 없음. 격자 밖으로 나가면 radius 0일 땐 영원히 실패, radius > 0일 땐 셀이 없어서 "통행 가능"으로 판정됨 | [NavigationGrid.cs:879-895](GameServerLib/Content/Navigation/NavigationGrid.cs#L879-L895) |
| 10 | 지형↔유닛 충돌 루프: 안전 경로로 재경로 → 다시 벽 충돌 → … 날카로운 모서리에서 빠져나오지 못하는 경우가 있음 (코드 TODO) | [AttackableUnit.cs:218-220](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L218-L220) |
| 11 | 위치 대시는 중간 지형을 검사하지 않음 (목적지만 보정) | [AttackableUnit.cs:1524](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L1524) |
| 12 | `HintGrid`, 셀의 `IsOpen`/`ActorList`, `.aimesh` 삼각형 메시는 읽거나 존재만 하고 미사용 | 1장 |

---

## 8. 관련 파일

| 파일 | 역할 |
|---|---|
| [Content/Navigation/NavigationGrid.cs](GameServerLib/Content/Navigation/NavigationGrid.cs) | ngrid 파싱, A\*, 스무딩, Ray/Circle cast, 높이 |
| [Content/Navigation/NavigationGridCell.cs](GameServerLib/Content/Navigation/NavigationGridCell.cs) | 셀 구조, 버전별 읽기 |
| [GameServerCore/Enums/NavigationGridCellFlags.cs](GameServerCore/Enums/NavigationGridCellFlags.cs) | 셀 플래그 |
| [Content/Package.cs](GameServerLib/Content/Package.cs) | navgrid 파일 로드/선택 |
| [Handlers/MapScriptHandler.cs](GameServerLib/Handlers/MapScriptHandler.cs) | 그리드·핸들러 생성, 틱마다 Collision/Pathing 갱신 |
| [Handlers/PathingHandler.cs](GameServerLib/Handlers/PathingHandler.cs) | GetPath/IsWalkable 래퍼 |
| [Handlers/CollisionHandler.cs](GameServerLib/Handlers/CollisionHandler.cs) | QuadTree 기반 겹침 검출 |
| [GameObjects/AttackableUnits/AttackableUnit.cs](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs) | 웨이포인트, Move/DashMove, OnCollision |
| [GameObjects/AttackableUnits/AI/ObjAIBase.cs](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs) | AI 이동 조건, 타겟 추적 재경로 |
| [Packets/PacketHandlers/HandleMove.cs](GameServerLib/Packets/PacketHandlers/HandleMove.cs) | 클라이언트 이동 요청 검증 |
| [Packets/PacketExtensions.cs](GameServerLib/Packets/PacketExtensions.cs) | 웨이포인트 패킷 변환 |
| [Content/LeagueSandbox-Scripts/AIScripts/LaneMinionAI.cs](Content/LeagueSandbox-Scripts/AIScripts/LaneMinionAI.cs) | 라인 미니언 레인 추종 |
| [QuadTree/QuadTree.cs](QuadTree/QuadTree.cs) | 공간 분할 |
