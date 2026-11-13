# 유닛 충돌 회피와 클라이언트 동기화 검토

현재 서버의 유닛 충돌 처리에 어떤 문제가 있는지, LoL은 이 문제를 어떻게 푸는지, 대안(RVO2/ORCA, A* 재경로)을 시뮬레이터로 비교한 결과와 클라이언트 동기화 관점의 적용 방안을 정리합니다.

> **요약**
> - 현재 서버는 겹친 뒤에 위치를 밀어냅니다(순간이동). 이 과정에서 **감지 반경과 밀어내기 반경이 달라 유닛이 서로 끌려가는 버그**가 있고, 위치를 보정할 때마다 이동 패킷이 나갑니다.
> - LoL은 **서버 권위** 게임입니다. 유닛 경로는 **웨이포인트로 전달**되고, 회피는 **경로 탐색 단계에서 다른 유닛을 고려**하는 방식입니다(Riot 공식 글). RVO2/ORCA를 쓴다는 공식 근거는 없습니다.
> - 시뮬레이터 비교 결과 **유닛을 고려한 A* 재경로 + 부드러운 밀어내기**가 모든 시나리오에서 도착했고, 웨이포인트 클라이언트 모델과도 맞습니다. ORCA는 움직임은 가장 매끄럽지만 좁은 길과 정지 무리 앞에서 막히고, 매 틱 속도가 바뀌어 웨이포인트 동기화와 맞지 않습니다.
> - 남은 위험은 **밀어내기 보정이 만드는 패킷**입니다. 클라이언트가 그리고 있을 위치와 실제 위치의 차이가 기준 이상일 때만 재전송하는 방식(dead-band)이 필요하고, 실제 4.20 클라이언트로 검증해야 합니다.

관련 문서: [pathfinding.md](pathfinding.md) (지형·A*·이동 구조), [ai.md](ai.md) (AI 유닛)

---

## 1. 현재 서버의 충돌 처리

처리 순서(60Hz 틱, [Game.cs:39](GameServerLib/Game.cs#L39)):

1. `Map.Update` → `CollisionHandler.Update()` ([MapScriptHandler.cs:98](GameServerLib/Handlers/MapScriptHandler.cs#L98))
   - 오브젝트마다: 중심점이 벽 셀이면 지형 탈출, QuadTree로 후보를 찾아 겹치면 `OnCollision` 호출
   - 끝나면 QuadTree를 통째로 다시 지음 (`UpdateQuadTree`)
2. `ObjectManager.Update` → 각 유닛 `Move()` 로 웨이포인트를 따라 직진 ([AttackableUnit.cs:926](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L926))

따라서 다음 틱의 QuadTree 조회는 **한 틱 전 위치**(이동 전) 기준입니다.

### 1.1 QuadTree — [QuadTree.cs](QuadTree/QuadTree.cs)

- 노드 용량으로 분할하는 방식이 아닙니다. 객체를 **자기 원을 통째로 담는 가장 깊은 사분면**에 넣습니다. 분할선에 걸친 객체는 위쪽 노드에 남아 더 많은 조회에서 검사됩니다.
- 지형은 들어가지 않습니다. NavigationGrid의 크기만 빌려 씁니다. 지형 충돌은 그리드를 직접 조회합니다.
- 바운드는 `CollisionRadius` 원입니다 ([CollisionHandler.cs:57-60](GameServerLib/Handlers/CollisionHandler.cs#L57-L60)).

### 1.2 충돌 반응 — [AttackableUnit.OnCollision](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L278-L319)

```csharp
// We should not teleport here because Pathfinding should handle it.
// TODO: Implement a PathfindingHandler, and remove currently implemented manual pathfinding.
Vector2 exit = Extensions.GetCircleEscapePoint(Position, PathfindingRadius + 1, collider.Position, collider.PathfindingRadius);
...
SetPosition(exit, false);
```

- 코드 주석 스스로 "원래는 경로 탐색이 처리해야 한다"고 적고 있습니다.
- `SetPosition` 은 `_movementUpdated = true` 를 켜므로([AttackableUnit.cs:195-198](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L195-L198)) **보정할 때마다 그 틱에 이동 패킷**이 나갑니다.
- 탈출점이 벽이면 `GetClosestTerrainExit` 로 50유닛 셀 중심까지 점프할 수 있습니다.

### 1.3 반경 불일치로 인한 끌어당김 버그

| 단계 | 사용하는 반경 | 미니언 값 |
|---|---|---|
| 겹침 감지 `IsCollidingWith` ([GameObject.cs:226-229](GameServerLib/GameObjects/GameObject.cs#L226-L229)) | `CollisionRadius` 합 | 48 + 48 = **96** |
| 밀어낼 목표 거리 `GetCircleEscapePoint` ([Extensions.cs:353-359](GameServerCore/Extensions.cs#L353-L359)) | `PathfindingRadius + 1` + 상대 `PathfindingRadius` | 36.7 + 35.7 = **72.5** |

`GetCircleEscapePoint` 의 결과는 `p1 + 단위벡터(p1→p2) × (거리 − 목표거리)` 입니다. 두 미니언 거리가 **72.5 ~ 96** 이면 "충돌"로 판정되는데, 이때 (거리 − 목표거리) > 0 이라 **상대 쪽으로 당겨집니다**. 미니언 무리는 72.5 간격으로 빨려 들어가고, 그 상태에서도 여전히 충돌 판정이라 매 틱 처리 비용이 듭니다.

값 출처: `Content/LeagueSandbox-Default/Stats/Blue_Minion_Basic/Blue_Minion_Basic.json` (`GameplayCollisionRadius` 48, `PathfindingCollisionRadius` 35.7437), Annie는 55 / 30.

### 1.4 그 밖의 특성

- 지형 충돌은 **중심점이 벽 셀에 들어갔을 때만** 처리합니다(반경 0, [CollisionHandler.cs:141](GameServerLib/Handlers/CollisionHandler.cs#L141)).
- 두 유닛 중심이 정확히 같으면 방향 벡터가 0이 됩니다(정규화 결과는 확인하지 않았음).
- [ObjAIBase.cs:555-587](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L555-L587) 에 유닛에 막혔을 때 탈출 웨이포인트를 직접 주는 임시 로직이 따로 있습니다.

---

## 2. LoL은 어떻게 하는가 (공식 자료)

| 항목 | 내용 | 출처 |
|---|---|---|
| 아키텍처 | "League of Legends is a server-authoritative multiplayer game, hosted on dedicated servers" | [The Tech Behind Swarm](https://www.riotgames.com/en/news/the-tech-behind-swarm) |
| 이동 동기화 | "In both classic League and Swarm, each unit receives its path in the form of waypoints." / "Both client and server smoothly interpolate the unit's position along the path in a deterministic way." | 같은 글 |
| 유닛 회피 | "Along each path explored, collisions are checked with other minions." 기존 방식을 "agent-based pathfinding and collision avoidance" 라고 부름. 알고리즘 이름은 밝히지 않음 | 같은 글 |
| 속도 흔들림과 desync | 셀마다 속도가 조금씩 바뀌는 것만으로 "an obvious desync between client and server" 가 생겨 이동 처리를 단순화함 | 같은 글 |
| Swarm 모드 | 미니언 수백 마리 때문에 흐름장(flowfield) 전역 경로 + 유닛 크기만큼 "collision stamp" 로 전환 | 같은 글 |
| 결정론 | 서버를 재생·상태 복원 가능하게 만든 작업. 100% 결정론은 불가능해서 통제 불가 입력은 녹화로 보완 | [Determinism in League of Legends](https://technology.riotgames.com/node/67) 시리즈 |

정리하면 LoL은 **회피를 경로 변경으로 표현**하고, 클라이언트는 받은 경로를 서버와 같은 규칙으로 따라갑니다. 매 틱 속도를 바꾸는 방식(ORCA 등)은 이 모델과 맞지 않는다는 것을 Swarm의 desync 사례가 보여 줍니다.

확인하지 못한 것: RVO2/ORCA 사용 여부, 클라이언트 예측·지연 보상·틱레이트의 공식 설명. 커뮤니티 자료에 따르면 유닛 충돌은 pathing radius 기준이고 유체화(Ghost)로 무시되며, 첫 웨이브 미니언은 일정 지점까지 유체화 상태로 이동합니다([LoL Wiki: Unit collision](https://wiki.leagueoflegends.com/en-us/Unit_collision)).

---

## 3. 대안 검토

### 3.1 RVO / ORCA / RVO2

| 이름 | 정체 | 평가 |
|---|---|---|
| RVO (2008) | 최초 알고리즘. 후보 속도 샘플링 | 느리고 결과가 샘플에 좌우됨. 쓸 이유 없음 |
| ORCA (2011) | 이웃마다 허용 속도 반평면을 만들고 2D 선형계획으로 원하는 속도에 가장 가까운 안전 속도를 고름 | 빠르고 결정적. 국소 회피라 좁은 길 교착은 못 풂 |
| RVO2 | ORCA 구현 라이브러리(UNC, Apache-2.0) | C# 포트 [snape/RVO2-CS](https://github.com/snape/RVO2-CS) 가 있음 |

RVO2-CS 메모 (로컬 사본 `D:\projects\LeagueSandbox\RVO2-CS`):
- NuGet 패키지 없음. 소스 복사 방식. 현재 저장소는 `net10.0` 대상이지만 net10 전용 문법은 보이지 않아 .NET 6 빌드 가능성이 큼(미검증).
- 서버에 넣는다면 `Simulator`/`KdTree` 는 쓰지 않고 `Agent.ComputeNewVelocity` + `LinearProgram1/2/3` + `RVOMath` + `Line` 만 가져와, 이웃은 기존 QuadTree에서 넘기는 편이 깔끔합니다.
- 웹용 포트(`rvo2.js`, npm `rvo2`)는 방치되었거나 Node 네이티브 바인딩이라 시뮬레이터에는 직접 포팅했습니다.

### 3.2 동기화 방식 자체: 서버 권위 vs 락스텝

`D:\projects\LockstepFramework` (Unity용 결정론적 락스텝) 와 비교한 결과, LoL 장르에는 서버 권위가 맞습니다.

| 항목 | 서버 권위 (이 서버, LoL) | 락스텝 (LockstepFramework) |
|---|---|---|
| 안개/맵핵 | 시야 내 정보만 전송(`BroadcastPacketVision`) | 모든 클라이언트가 전체 상태를 가져 구조적으로 맵핵 가능 |
| 입력 반응 | 클라이언트 선표시 가능 | 입력이 왕복 + 입력 프레임(32Hz ÷ 2 = 62.5ms) 대기 |
| 치팅 | 서버가 판정 | 결과 위조는 어렵지만 정보 노출은 막지 못함 |
| 재접속 | 현재 상태 전송 | 전체 입력 재시뮬레이션 필요(LSF에는 기능 없음) |
| 대역폭·서버 비용 | 상대적으로 큼 | 작음(입력만) |
| 프레임워크 상태 | 구현 품질 문제는 있으나 구조는 맞음 | README에 개발 중단 명시(Unity 2019.1), 시뮬레이션 코드 24개 파일이 `UnityEngine` 참조, 롤백·재접속 없음 |

---

## 4. 시뮬레이터 — [tools/collision-lab.html](tools/collision-lab.html)

브라우저로 바로 여는 단일 HTML 파일입니다. 같은 시나리오를 두 맵에서 서로 다른 방식으로 동시에 돌립니다.

### 4.1 재현한 것

- 60Hz 틱, 처리 순서(충돌 → QuadTree 재구성 → 이동), QuadTree 삽입 규칙, `IsCollidingWith`, `GetCircleEscapePoint`, 지형 중심점 검사와 셀 중심 탈출
- 실제 반경 값(미니언 48 / 35.74, 애니 55 / 30), 이동 속도(325 / 335)
- ORCA: RVO2-CS `Agent.cs` 의 `ComputeNewVelocity`, `LinearProgram1/2/3`, `Simulator.AddObstacle`, KdTree 장애물 조회 조건을 줄 단위로 JS로 옮김. 벽은 다각형 장애물

### 4.2 방식

| 방식 | 내용 |
|---|---|
| 서버 원본 | 현재 코드 그대로 |
| 반경 통일 | 감지도 `PathfindingRadius` 로. 끌어당김 제거, 순간이동은 유지 |
| 부드러운 밀어내기 | 겹친 깊이의 15%만 매 틱 밀어냄 |
| ORCA | 속도 선택으로 미리 회피. 이웃 반경 400, 최대 10, 시간 지평 조절 가능. "정지 유닛은 비키지 않음" 옵션(고정 장애물, 상대가 100% 회피) |
| A* 재경로 | LoL식. 25유닛 격자 A*에 주변 유닛을 장애물로 넣고, 경로를 단순화해 웨이포인트로 따라감. 막히거나 0.5초 정체 시 재계산(최소 0.2초 간격). 목표가 막혀 있으면 가장 가까운 빈 칸. **우선순위 규칙**: 정지 유닛(600 이내)은 항상 장애물, 이동 유닛(250 이내)은 나보다 목표에 더 가까운 유닛만 장애물. 남는 겹침은 부드러운 밀어내기 |

### 4.3 계측

틱 처리 시간, 틱당 거리 검사 수, 유닛당 순간이동 거리, 끌어당김 횟수, 몸체 겹침, 속도 효율, 정체 유닛, **초당 이동 패킷**(위치 보정 + 새 경로. ORCA는 방향 10°/속력 20% 변화 시 100ms 간격으로 보낸다고 가정한 추정치), ORCA의 LP3 발생 수, A* 호출 수와 평균 확장 노드.

---

## 5. 비교 결과

Node에서 렌더링을 빼고 돌린 결과입니다(ORCA 시간 지평 0.5초). 표기: 도착 수 · 최대 겹침 깊이 · 초당 이동 패킷.

| 시나리오 | 서버 원본 | 부드러운 밀어내기 | ORCA | A* 재경로 |
|---|---|---|---|---|
| 웨이브 정면 충돌 12마리, 10초 | 0/12 · 32.8 · 593 | 12/12 · 27.3 · 68 | **12/12 · 0 · 8** | 12/12 · 34.3 · 40 |
| 좁은 통로(폭 100) 14마리, 30초 | 3/14 · 68.5 · 641 | 14/14 · 50.6 · 285 | 4/14 · 0.7 · 37 | **14/14 · 44.1 · 131** |
| 정지 미니언 20마리 사이로 챔피언, 10초 | 0/1 · 67.7 · 658 | 1/1 · 14.5 · 86 | 0/1 · 0 · 12 | **1/1 · 0 · 0** |
| 26마리 한 점 집결, 30초 | 7/26 · 71.4 · 1414 | 26/26 · 55.4 · 572 | 1/26 · 1.0 · 47 | **26/26 · 39.7 · 239** |

해석:

- **서버 원본**: 모든 지표에서 가장 나쁨. 패킷이 다른 방식의 2~10배.
- **ORCA**: 겹침·순간이동이 거의 0으로 움직임 품질은 최고. 그러나 좁은 통로에서 앞의 두 마리가 양쪽 모서리 벽에 붙어 입구를 막는 교착(국소 회피의 알려진 한계), 정지 무리 앞에서 정지. 집결의 도착 1/26은 "목표 근처에서 멈춤" 규칙이 없어서이며 실제로는 겹치지 않고 모여 있음. 패킷 수는 추정치.
- **A* 재경로**: 모든 시나리오 도착. 정지 무리는 처음부터 우회 경로를 잡아 재전송 0회. 약점은 우선순위가 높은 유닛이 뒤 유닛을 무시하며 생기는 일시적 겹침(최대 약 40)과 좁은 통로에서의 재계산 비용(틱당 약 0.4ms, JS).
- 우선순위 규칙 이전에는 A* 재경로도 좁은 통로에서 이동 유닛끼리 서로를 기다리는 교착으로 30초에 6/14만 통과했습니다.
- 성능: 26마리 기준 모든 방식이 JS에서 틱당 1ms 미만.

---

## 6. 클라이언트 동기화 분석

### 6.1 이동 패킷 구조

`WaypointGroup` → `MovementDataNormal { SyncID = Environment.TickCount, TeleportID?, Waypoints }` ([PacketExtensions.cs:134-146](GameServerLib/Packets/PacketExtensions.cs#L134-L146)). 웨이포인트는 `[현재 위치, 남은 웨이포인트…]` 입니다([PacketExtensions.cs:97-108](GameServerLib/Packets/PacketExtensions.cs#L97-L108)). 유닛의 `_movementUpdated` 가 켜진 틱에 시야가 있는 클라이언트에게 전송됩니다([AttackableUnit.cs:333-336](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L333-L336)).

### 6.2 LoL식(A* 재경로)에서 문제없는 부분

경로가 바뀔 때 새 경로를 한 번 보내고, 다음 재전송까지는 서버와 클라이언트가 같은 경로를 같은 속도로 따라갑니다. 클라이언트 수정이 필요 없습니다.

### 6.3 위험 요소

| 위험 | 내용 | 대응 |
|---|---|---|
| 밀어내기 보정 패킷 | `SetPosition` 마다 패킷. 부드러운 밀어내기는 겹친 동안 유닛당 매 틱(최대 60/s). 시뮬레이터의 A* 재경로 패킷 대부분이 이것 | **dead-band**: 서버에서만 보정하고, 마지막으로 보낸 경로 기준 클라이언트 추정 위치와 실제 위치 차이가 기준(예: 25~50) 이상일 때만 재전송 |
| 재경로 빈도 | 경로가 자주 바뀌면 패킷 증가, 회전이 잦아 보임 | 유닛별 재전송 최소 간격(예: 200ms), 새 경로가 이전과 거의 같으면 생략 |
| 밀어내기 자체 | 우선순위 높은 유닛이 뒤 유닛을 밀며 겹침 발생 | 바로 앞 유닛은 우선순위와 무관하게 장애물로 보는 등 경로 단계 회피 강화(교착 재발 주의) |
| 네트워크 지연 | 패킷 도착 시점에 서버 유닛은 RTT/2 만큼 앞서 있음. 재전송이 잦을수록 차이가 자주 드러남 | 현재 서버에도 있는 문제. `SyncID` 를 서버 게임 시간 기준으로 바꾸는 것은 클라이언트의 사용 방식을 확인한 뒤 별도 과제로 |
| 4.20 클라이언트 동작 | 작은 위치 차이를 부드럽게 보정하는지, 끊기게 표시하는지 코드로 알 수 없음 | 실제 클라이언트로 검증 필요 |

### 6.4 ORCA를 쓴다면

매 틱 속도를 그대로 보낼 수 없으므로 속도를 짧은 웨이포인트로 바꿔 일정 조건마다 재전송해야 합니다. Swarm 사례처럼 미세한 속도 변화도 desync를 만들 수 있어 조정 비용이 큽니다.

---

## 7. 적용 계획 (제안)

| 단계 | 작업 | 위치 |
|---|---|---|
| 0 | 반경 불일치 수정: 감지를 `PathfindingRadius` 로 통일(끌어당김 제거) | [GameObject.cs:226-229](GameServerLib/GameObjects/GameObject.cs#L226-L229) 또는 `OnCollision` 에서 재검사 |
| 1 | 순간이동을 부드러운 밀어내기로 교체 + dead-band 재전송 | [AttackableUnit.cs:299-318](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L299-L318) |
| 2 | A*에 유닛 장애물 + 우선순위 규칙 추가. `PathingHandler.IsWalkable(checkObjects)` 활용 검토 | [NavigationGrid.GetPath](GameServerLib/Content/Navigation/NavigationGrid.cs#L205-L288), [PathingHandler.cs:102](GameServerLib/Handlers/PathingHandler.cs#L102) |
| 3 | 막힘·정체 감지 시 재경로(최소 간격). 죽은 코드인 `PathingHandler` 주기 재검증 정리 | [PathingHandler.cs:27](GameServerLib/Handlers/PathingHandler.cs#L27) |
| 4 | `RecalculateAttackPosition` 의 임시 탈출 로직 정리 | [ObjAIBase.cs:555-587](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L555-L587) |
| 5 | 실제 4.20 클라이언트로 미니언 웨이브·좁은 길·지연 환경 검증 | — |

각 단계는 설정 플래그로 켜고 끌 수 있게 넣어 기존 동작과 비교할 수 있게 하는 것을 권장합니다. 0~1단계와 dead-band 효과는 시뮬레이터에 옵션을 추가해 먼저 수치로 확인할 수 있습니다.

---

## 8. 열린 질문

- 4.20 클라이언트가 `MovementDataNormal` 의 첫 웨이포인트(현재 위치)가 화면 위치와 다를 때 어떻게 보정하는가
- `SyncID` 를 클라이언트가 지연 보정에 쓰는가
- 실제 LoL 미니언의 회피 우선순위 규칙(공식 자료 없음)

## 9. 참고 자료

- [The Tech Behind Swarm — Riot Games](https://www.riotgames.com/en/news/the-tech-behind-swarm)
- [Determinism in League of Legends: Introduction](https://technology.riotgames.com/node/67) · [Implementation](https://technology.riotgames.com/node/68) · [Unified Clock](https://technology.riotgames.com/node/73) · [Fixing Divergences](https://technology.riotgames.com/node/84)
- [RVO2 Library (UNC)](https://gamma.cs.unc.edu/RVO2/) · [snape/RVO2-CS](https://github.com/snape/RVO2-CS) · [RVO2 2.0 What's New (RVO → ORCA)](https://gamma.cs.unc.edu/RVO2/documentation/2.0/whatsnew.html)
- [LoL Wiki: Unit collision](https://wiki.leagueoflegends.com/en-us/Unit_collision) (커뮤니티)
