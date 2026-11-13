# AI 유닛 구조

챔피언·미니언·포탑·정글 몬스터·펫 같은 AI 유닛이 서버에서 어떻게 구성되고, 매 틱 어떤 순서로 판단하고 행동하는지 코드 기준으로 정리한 문서입니다. 이동·길찾기·충돌은 [pathfinding.md](pathfinding.md)에서 다룹니다.

> 요약
> - AI는 **두 층**으로 나뉩니다.
>   - **엔진 층** `ObjAIBase` (C#, `GameServerLib`): 모든 AI 유닛이 공유합니다. 현재 타겟과 이동 명령(`MoveOrder`)을 보고 추적·평타·스킬 시전을 수행합니다.
>   - **스크립트 층** `IAIScript` (`Content/LeagueSandbox-Scripts/AIScripts`): 유닛 종류별로 다릅니다. **누구를 타겟으로 할지, 어떤 명령을 내릴지**만 정하고 실행은 엔진에 맡깁니다.
> - 행동트리나 범용 상태머신은 없습니다. 엔진은 `MoveOrder`(OrderType) 하나를 상태처럼 씁니다. `AIState` 열거형은 펫 스크립트에서만 사용합니다.
> - 실제로 동작하는 AI 스크립트는 **LaneMinionAI, TurretAI, BasicJungleMonsterAI, Pet** 네 개입니다. 챔피언(플레이어·봇)에는 AI 스크립트가 없습니다. **봇은 플래그만 있고 AI가 없어서 가만히 서 있습니다.**

---

## 1. 클래스 계층

```
GameObject
 └─ AttackableUnit            체력·버프·스탯, 웨이포인트 이동, 충돌 반응
     └─ ObjAIBase             ★ AI 공통: 스펠 슬롯, 평타, 타겟, MoveOrder, AIScript/CharScript
         ├─ Champion          플레이어(패킷으로 조작) / 봇(IsBot)
         ├─ Minion            소환물(AddMinion), 와드
         │   ├─ LaneMinion    라인 미니언 (레인 웨이포인트 보유)
         │   ├─ Monster       정글 몬스터 (MonsterCamp 소속)
         │   └─ Pet           펫·분신 (소유자 챔피언)
         └─ BaseTurret        포탑 (이동하지 않음)
             ├─ LaneTurret
             └─ AzirTurret
```

파일: [GameServerLib/GameObjects/AttackableUnits/AI/](GameServerLib/GameObjects/AttackableUnits/AI/)

---

## 2. ObjAIBase: 엔진 쪽 AI

### 2.1 생성할 때 하는 일 — [ObjAIBase.cs:91-235](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L91-L235)

1. **반지름·시야**: `CollisionRadius`, `PathfindingRadius`, `VisionRadius` 를 CharData(캐릭터 inibin/json)에서 읽습니다. 값이 없으면 40 / 40 / 1100을 씁니다.
2. **스펠 슬롯** 구성 (`Spells : Dictionary<short, Spell>`)
   | 슬롯 | 내용 |
   |---|---|
   | 0-3 | Q/W/E/R (`CharData.SpellNames`) |
   | 패시브 | `CharData.PassiveData.PassiveLuaName` |
   | 소환사 주문 2칸, 아이템 6-12, 귀환, 룬 15-44 | 기본은 `BaseSpell` |
   | 45-60 Extra | `CharData.ExtraSpells` (자동 레벨 1) |
   | 64-72 / 73-81 | 일반 평타 / 치명타 평타 (`CharData.BasicAttacks`) |
3. **CharScript** 로드: `CharScripts.CharScript{Model}`. 캐릭터 고유 로직(패시브 등)을 담고, 없으면 `CharScriptEmpty`.
4. **AIScript** 로드: `AIScripts.{aiScript}` 클래스를 이름으로 찾아 인스턴스를 만들고 `OnActivate(this)` 를 호출합니다. 못 찾으면 [`EmptyAIScript`](GameServerLib/Scripting/CSharp/EmptyAIScript.cs)를 씁니다. 로더는 [CSharpScriptEngine.CreateObject](GameServerLib/Scripting/CSharp/CSharpScriptEngine.cs#L188-L207)이고, `"네임스페이스.클래스명"` 으로 정확히 일치하는 클래스를 찾습니다.

### 2.2 매 틱 업데이트 순서 — [ObjAIBase.Update](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L1062-L1108)

`ObjectManager.Update` 에서 오브젝트마다 60Hz로 호출됩니다.

```
ObjAIBase.Update(diff)
 1. base.Update            (AttackableUnit) 버프 → 스탯(0.5초마다) → Replication → ★이동(Move/DashMove)
 2. CharScript.OnUpdate
 3. AIScript.OnUpdate      ← _aiPaused 가 false 일 때만. 예외는 로그만 남기고 무시
 4. 모든 Spell.Update      쿨다운·시전 진행
 5. Inventory.OnUpdate
 6. UpdateTarget()         ★ 엔진 전투 로직 (2.4)
 7. 평타 쿨다운 감소
 8. 죽은 펫 참조 정리

ObjectManager 가 모든 Update 후
 LateUpdate               타겟이 Targetable 을 잃으면 Untarget
```

> 이동(1)이 스크립트 판단(3)보다 먼저 실행됩니다. 그래서 스크립트가 이번 틱에 정한 경로나 명령은 **다음 틱부터** 위치에 반영됩니다.

### 2.3 핵심 상태 변수

| 변수 | 의미 |
|---|---|
| `MoveOrder : OrderType` | 현재 명령 (`MoveTo`, `AttackTo`, `AttackMove`, `Stop`, `Hold`, `CastSpell`, `Taunt`, `Pet*` …). 엔진 판단의 핵심 상태 |
| `TargetUnit` | 평타/스킬 대상 |
| `SpellToCast` | 사거리에 들어가면 쓸 예약 스킬 (평타보다 우선) |
| `IsAttacking`, `HasMadeInitialAttack`, `_autoAttackCurrentCooldown` | 평타 진행 상태 |
| `_aiState : AIState` | 스크립트용 세부 상태. 엔진은 읽지 않음 |
| `_aiPaused` | true면 AIScript.OnUpdate 를 건너뜀 (`PauseAI`) |

`UpdateMoveOrder(order)` ([ObjAIBase.cs:1336-1363](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L1336-L1363))
- `ApiEventManager.OnUnitUpdateMoveOrder` 를 먼저 발행합니다. 리스너가 false를 돌려주면 **명령이 거부됩니다**(펫 스크립트가 이걸로 명령을 걸러냄).
- `Stop`/`OrderNone`/`PetHardStop` 이면 이동을 멈추고 타겟도 해제합니다. `Hold`/`Taunt` 는 이동만 멈춥니다.

`SetTargetUnit(target, networked)` ([ObjAIBase.cs:971-989](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L971-L989))
- 타겟이 사라지면 `OnTargetLost` 이벤트를 발행합니다.
- `networked` 면 `AI_TargetS2C` / `AI_TargetHeroS2C` 패킷을 보냅니다.

### 2.4 UpdateTarget: 엔진의 전투 판단 — [ObjAIBase.cs:1161-1330](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L1161-L1330)

위에서부터 처음 맞는 조건 하나만 실행됩니다.

```
죽었음                                  → 평타 취소, 타겟 해제
타겟 없음                               → 진행 중 평타 취소
타겟이 죽음 / 안 보임 / 타겟 불가        → 평타 취소, 타겟 해제
평타 중                                 → 타겟이 (사거리 + 타겟 반지름) 밖이고 취소 가능하면 취소
SpellToCast 있음 & 유효 대상            → 시전 사거리 안이면 Cast, 아니면 RefreshWaypoints(추적)
적 타겟 있음 (MoveOrder != CastSpell)
   사거리 = Range + 타겟.CollisionRadius
   ├ 사거리 안 & 평타 준비됨 & CanAttack
   │     → 치명타 판정(CriticalChance) → 일반/치명 평타 스펠 선택
   │     → AutoAttackSpell.Cast(타겟) ; 쿨다운 = 1 / 공격속도
   └ 사거리 밖                           → RefreshWaypoints → 매 틱 A* 로 추적
타겟 없음 & MoveOrder == AttackMove     → AcquisitionRange 안에서 가장 가까운 적을 타겟으로
```

- 평타 데미지는 [`AutoAttackHit`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L265-L296)에서 계산합니다(투사체 적중 시). AD × 치명타 배율로 물리 데미지를 주고, 실명(BLIND) 상태면 MISS 처리합니다.
- 추적 경로(`RefreshWaypoints`)는 [pathfinding.md 3.2](pathfinding.md#32-서버가-직접-경로를-만드는-경우)를 참고하세요. 포탑은 이 함수를 빈 함수로 오버라이드해서 움직이지 않습니다.

### 2.5 이동/행동 가능 조건

- [`CanMove`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L298-L311): 대시 중이거나, `CanMove`·`CanMoveEver` 플래그가 있고 시전/채널링(이동 불가 스킬)이 아니며 기절·속박·제압·수면·그물 상태가 아닐 때.
- [`CanChangeWaypoints`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L313-L319): 살아 있고, 시전 중이 아니고, 취소 불가 채널링 중이 아닐 때.
- [`ObjAIBase.Move`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L448-L459): `MoveOrder` 가 `CastSpell / OrderNone / Stop / Taunt` 면 웨이포인트가 있어도 움직이지 않습니다.

### 2.6 도움 요청 (Call for Help) — [ObjAIBase.TakeDamage](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L1123-L1155)

유닛이 피해를 받으면 **모든 오브젝트를 순회**하면서 아래 조건을 모두 만족하는 아군의 `AIScript.OnCallForHelp(attacker, victim)` 를 호출합니다.
- 같은 팀이고 살아 있음
- 그 아군의 스크립트가 `AIScriptMetaData.HandlesCallsForHelp == true` (현재는 LaneMinionAI만)
- 피해자와 공격자가 둘 다 **피해자의** `AcquisitionRange` 안에 있음

### 2.7 타겟 우선순위 — [ObjAIBase.ClassifyTarget](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L385) / [ClassifyUnit](GameServerCore/Enums/ClassifyUnit.cs)

숫자가 작을수록 우선입니다. `victim` 을 함께 넘기면(도움 요청 상황) 1~5 구간으로 판정합니다.

| 값 | 분류 | | 값 | 분류 |
|---|---|---|---|---|
| 1 | 아군 챔피언을 공격하는 챔피언 | | 8 | 원거리(캐스터) 미니언 |
| 2 | 아군 챔피언을 공격하는 미니언 | | 9 | 근접 미니언 |
| 3 | 아군 미니언을 공격하는 미니언 | | 10 | 포탑 |
| 4 | 아군 미니언을 공격하는 포탑 | | 11 | 챔피언 |
| 5 | 아군 미니언을 공격하는 챔피언 | | 12 / 13 | 억제기 / 넥서스 |
| 6 | 미니언(기타) | | 14 | 기본값 |
| 7 | 슈퍼/대포 미니언 | | | |

---

## 3. AI 스크립트 인터페이스

[IAIScript.cs](GameServerLib/Scripting/CSharp/Interfaces/IAIScript.cs)

```csharp
public interface IAIScript
{
    AIScriptMetaData AIScriptMetaData { get; set; }   // BehaviorTree, MinionRoamState, HandlesCallsForHelp
    void OnActivate(ObjAIBase owner) { }              // 생성자에서 1회
    void OnUpdate(float diff) { }                     // 매 틱 (AI 일시정지면 건너뜀)
    void OnCallForHelp(AttackableUnit attacker, AttackableUnit victim) { }
}
```

- 스크립트는 `AIScripts` 네임스페이스의 클래스이고, 서버 시작 시 `LeagueSandbox-Scripts` 를 컴파일해 로드합니다.
- 스크립트는 주로 `ApiFunctionManager` 정적 함수(`GetUnitsInRange`, `EnumerateUnitsInRange`, `TeleportTo` …)와 `ApiEventManager` 이벤트 구독(`OnTakeDamage`, `OnUnitUpdateMoveOrder`, `OnTargetLost` …)으로 엔진과 주고받습니다.
- `AIScriptMetaData.BehaviorTree`, `MinionRoamState` 는 정의만 되어 있고 쓰는 곳이 없습니다.

### 유닛별 스크립트 지정

| 유닛 | 스크립트 | 지정 위치 |
|---|---|---|
| 라인 미니언 | `LaneMinionAI` | 맵 스크립트 `LaneMinionAI` 속성 → [CreateLaneMinion](GameServerLib/API/APIMapFunctionManager.cs#L135-L142) |
| 라인 포탑·우물 포탑 | `TurretAI` | [Map1/LevelScriptObjects.cs:26](Content/LeagueSandbox-Scripts/Maps/Map1/LevelScriptObjects.cs#L26) → `CreateLaneTurret` |
| 정글 몬스터 | `BasicJungleMonsterAI` | [Map1/NeutralMinionSpawn.cs](Content/LeagueSandbox-Scripts/Maps/Map1/NeutralMinionSpawn.cs) `CreateJungleMonster(..., aiScript:)` |
| 펫 | `Pet` | [Pet](GameServerLib/GameObjects/AttackableUnits/AI/Pet.cs) 생성자 기본값 |
| 챔피언 | 없음 (`EmptyAIScript`) | — |
| 스킬 소환물 (`AddMinion`) | 없음, 기본 `aiPaused = true` | [ApiFunctionManager.AddMinion](GameServerLib/API/ApiFunctionManager.cs#L440) |

---

## 4. 유닛별 구현

### 4.1 챔피언 — [Champion.cs](GameServerLib/GameObjects/AttackableUnits/AI/Champion.cs)

- **AI 스크립트가 없습니다.** 플레이어가 보낸 `MovementRequest`([HandleMove](GameServerLib/Packets/PacketHandlers/HandleMove.cs))로 `MoveOrder`/`TargetUnit`/웨이포인트가 정해지고, 나머지는 엔진의 `UpdateTarget` 이 처리합니다.
- 챔피언 고유 처리: 리스폰 타이머(`MapData.DeathTimes[레벨]`), 스킬 포인트·레벨업, 골드, 15초 막타 판정 타이머(`_championHitFlagTimer`), 상점·특성·룬.
- **봇**: `clientInfo.PlayerId == -1` 이면 `IsBot = true` 가 됩니다. 이 플래그는 패킷에 실리기만 하고, 봇을 움직이는 AI 코드는 없습니다.

### 4.2 라인 미니언 — [LaneMinionAI.cs](Content/LeagueSandbox-Scripts/AIScripts/LaneMinionAI.cs)

Riot이 공개한 미니언 AI 설명을 `//Quote:` 주석으로 옮겨 놓고 그 순서대로 구현한 스크립트입니다. 이 프로젝트에서 가장 완성도가 높은 AI입니다.

**재평가 시점** (`OnUpdate`): 대시 중이 아닐 때, 다음 중 하나가 일어나면 `ReevaluateBehavior()` 를 실행하고 결과를 `UpdateMoveOrder` 로 적용합니다.
- 250ms 주기 (`minionActionTimer`)
- 현재 타겟이 방금 죽거나 무효가 됨 (`TargetJustDied`)
- 도움 요청 목록에 새 후보가 생김 (`FoundNewTarget(true)`)

**ReevaluateBehavior 우선순위**
1. 현재 타겟이 유효하면 계속 공격합니다(`AttackTo`). 단 **4초 동안 공격하지 못했으면** 그 타겟을 500ms 무시 목록에 넣고 넘어갑니다.
2. `AcquisitionRange` 안에서 새 타겟을 찾습니다(`FoundNewTarget`). 정렬 기준은 우선순위(도움 요청이면 그 상황의 우선순위 2.7) → 거리입니다. 찾으면 `AttackTo`.
3. 타겟이 없으면 레인 웨이포인트를 따라갑니다. 도달 판정은 [pathfinding.md 3.3](pathfinding.md#33-라인-미니언--laneminionaics)을 참고하세요. 레인으로 복귀할 때만 A\*를 씁니다. `MoveTo`.
4. 웨이포인트를 다 지나면 `Stop`.

- 유효 타겟 조건: 적 팀이고 살아 있으며, 획득 사거리 안에 있고, 팀 시야에 보이고, Targetable이며, 보호 상태가 아님.
- `OnCallForHelp`: 공격자별로 가장 높은 우선순위를 `unitsAttackingAllies` 에 기록합니다. 이 목록은 다음 틱에 비웁니다.
- **미완성 부분**: "첫 웨이브 행동"(같은 타겟을 치는 아군 수로 분산하는 로직)은 공격자 수를 항상 0으로 두는 상태입니다. 겹침 재평가(`RecalculateAttackPosition`)는 주석 처리되어 있습니다.

### 4.3 포탑 — [TurretAI.cs](Content/LeagueSandbox-Scripts/AIScripts/TurretAI.cs), [BaseTurret.cs](GameServerLib/GameObjects/AttackableUnits/AI/BaseTurret.cs)

- 평타 중이 아닐 때만 `CheckForTargets()` 를 실행합니다.
  - 타겟이 없으면: 사거리 안 적 중 `ClassifyTarget` 우선순위가 가장 높은 유닛을 고릅니다. 우선순위가 같으면 **먼저 순회된 유닛**이 선택됩니다(거리 비교 없음).
  - 타겟이 챔피언이 아니면: 사거리 안에서 **아군 챔피언을 공격 중인 적 챔피언**이 있으면 그쪽으로 바꿉니다(포탑 어그로).
- 타겟이 사거리 밖으로 나가면 타겟을 해제합니다.
- 엔진 쪽 특수 처리: `RefreshWaypoints`·`OnCollision` 을 빈 함수로 오버라이드해서 움직이거나 밀려나지 않습니다. 전장의 안개 영향을 받지 않습니다(`IsAffectedByFoW = false`). 생성될 때 시야 800, 은신 감지 Region을 만듭니다.

### 4.4 정글 몬스터 — [BasicJungleMonsterAI.cs](Content/LeagueSandbox-Scripts/AIScripts/BasicJungleMonsterAI.cs)

스크립트 주석에도 "EXTREMELY basic"이라고 적혀 있는 최소 구현입니다.
- `OnTakeDamage` 이벤트를 구독합니다. 맞으면 **같은 캠프의 몬스터 전부**가 공격자를 타겟으로 삼습니다. 추적과 공격은 엔진의 `UpdateTarget` 이 합니다.
- 전투 중 캠프 중심에서 **800 이상** 멀어지면 `ResetCamp`: 타겟 해제 → 원래 위치로 걸어감(`SetPathTrueEnd`) → 체력 완전 회복.
- 평상시에는 경로가 끝나면 처음 바라보던 방향으로 돌아섭니다.
- 리쉬 타이머, 귀환 중 무적, 몬스터별 스킬은 없습니다.

### 4.5 펫 — [AIScripts/Pet.cs](Content/LeagueSandbox-Scripts/AIScripts/Pet.cs), [AI/Pet.cs](GameServerLib/GameObjects/AttackableUnits/AI/Pet.cs)

`AIState` 를 실제로 쓰는 유일한 스크립트입니다.
- **명령 필터**: `OnUnitUpdateMoveOrder` 를 구독합니다. Hard 계열 상태(`AI_PET_HARDATTACK/HARDMOVE/...`)에서는 일반 `MoveTo/AttackTo/AttackMove/Stop` 명령을 거부합니다. 명령마다 `AI_PET_MOVE`, `AI_PET_ATTACK`, `AI_PET_HOLDPOSITION` 등으로 상태를 바꿉니다.
- **OnTargetLost**: 타겟을 잃으면 상태를 되돌립니다(예: `RETURN_ATTACKING` → `RETURN`).
- **TimerScan** (0.15초 의도)
  - 소유자가 없으면 죽습니다.
  - 소유자와 **2000 이상** 떨어지면 소유자 옆으로 순간이동합니다.
  - 적이 없고 복귀 반경(`DefaultPetReturnRadius`)을 벗어나면 `AI_PET_RETURN` 으로 바뀝니다. 다만 실제로 소유자에게 걸어가는 코드는 TODO로 남아 있습니다.
- **TimerFindEnemies** (0.15초 의도): IDLE/RETURN/ATTACKMOVE/HOLD 상태에서 475 이내 적을 우선순위로 찾아 공격 상태로 전환합니다.
- 분신(Clone)이면 원본의 아이템을 복사하고, 수명 버프(`CloneBuff`)가 끝나면 사라집니다.
- 공포(Fear) 타이머, 상태 알림(`NotifySetState`) 등은 비어 있거나 주석 처리되어 있습니다.

### 4.6 스킬 소환물 / 와드 — [Minion.cs](GameServerLib/GameObjects/AttackableUnits/AI/Minion.cs)

- `AddMinion` 으로 만들고, AI 스크립트가 없으며 기본적으로 AI가 일시정지 상태입니다. 동작은 스킬·버프 스크립트가 직접 제어합니다.
- `ignoreCollision` 이면 `Ghosted` 상태가 되어 충돌 밀어내기를 받지 않습니다. `targetable=false` 면 타겟 불가가 됩니다.

---

## 5. 확인된 문제 / 미구현

| # | 내용 | 위치 |
|---|---|---|
| 1 | **봇 AI 없음.** `IsBot` 플래그만 있고 판단 코드가 없어서 봇 챔피언은 가만히 서 있음 | [Champion.cs:81-84](GameServerLib/GameObjects/AttackableUnits/AI/Champion.cs#L81-L84) |
| 2 | **펫 스캔 타이머가 리셋되지 않음.** `_timerScan`/`_timerFindEnemies` 를 누적만 하고 0으로 되돌리지 않아서, 0.15초가 지난 뒤로는 **매 틱** 실행됨 | [AIScripts/Pet.cs:267-277](Content/LeagueSandbox-Scripts/AIScripts/Pet.cs#L267-L277) |
| 3 | 펫 소유자가 없을 때 `Die` 를 호출하고도 return 하지 않아서, 바로 다음 줄 `owner.Position` 에서 NullReferenceException 발생 (ObjAIBase가 잡아 로그만 남김) | [AIScripts/Pet.cs:299-306](Content/LeagueSandbox-Scripts/AIScripts/Pet.cs#L299-L306) |
| 4 | `MinionAI.cs` 의 클래스 이름이 오타(`MinonAI`)이고 이 스크립트를 지정하는 곳도 없음 → 사용되지 않는 코드 | [MinionAI.cs:12](Content/LeagueSandbox-Scripts/AIScripts/MinionAI.cs#L12) |
| 5 | `RefreshWaypoints` 의 조건 `AttackMove \|\| AttackTerrainOnce \|\| AttackTerrainSustained && !IsPathEnded()` 에서 `&&` 가 마지막 항에만 걸림 (괄호 누락으로 보임) | [ObjAIBase.cs:619-622](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L619-L622) |
| 6 | 도움 요청이 피해 한 번마다 **전체 오브젝트를 순회** (O(N)). 범위 판정도 피해자의 획득 사거리 기준 | [ObjAIBase.cs:1128-1154](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L1128-L1154) |
| 7 | `AttackMove` 타겟 탐색도 전체 오브젝트 순회 (QuadTree 미사용) | [ObjAIBase.cs:1291-1315](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L1291-L1315) |
| 8 | 사거리 밖 타겟을 추적하는 동안 매 틱 A\* 실행 | [pathfinding.md 7장 #6](pathfinding.md#7-알려진-한계--확인된-문제) |
| 9 | 이동이 AI 판단보다 먼저 실행되어 명령 반영이 1틱 늦음 | 2.2 |
| 10 | 포탑 타겟 선택에서 우선순위가 같으면 순회 순서로 결정됨(거리 무시). 코드 주석에도 명시 | [TurretAI.cs:52-53](Content/LeagueSandbox-Scripts/AIScripts/TurretAI.cs#L52-L53) |
| 11 | 스크립트 이름이 `""` 인 유닛(챔피언 등)도 스크립트를 찾으려다 실패해서, 생성될 때마다 "Could not find script: AIScripts." 경고가 로그에 남음 | [CSharpScriptEngine.cs:205](GameServerLib/Scripting/CSharp/CSharpScriptEngine.cs#L205) |
| 12 | `AIScriptMetaData.BehaviorTree`/`MinionRoamState` 미사용, `AIState` 는 펫 외에 미사용 | 3장 |
| 13 | 정글 몬스터: 리쉬 타이머·복귀 중 무적·몬스터 스킬 없음. 라인 미니언: 첫 웨이브 분산 로직 미완성 | 4.2, 4.4 |

---

## 6. 관련 파일

| 파일 | 역할 |
|---|---|
| [AI/ObjAIBase.cs](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs) | AI 공통 엔진: 스펠, 평타, 타겟, 명령, 도움 요청 |
| [AI/Champion.cs](GameServerLib/GameObjects/AttackableUnits/AI/Champion.cs) | 챔피언 (리스폰, 레벨, 골드) |
| [AI/Minion.cs](GameServerLib/GameObjects/AttackableUnits/AI/Minion.cs), [LaneMinion.cs](GameServerLib/GameObjects/AttackableUnits/AI/LaneMinion.cs), [Monster.cs](GameServerLib/GameObjects/AttackableUnits/AI/Monster.cs), [Pet.cs](GameServerLib/GameObjects/AttackableUnits/AI/Pet.cs) | 미니언 계열 |
| [AI/BaseTurret.cs](GameServerLib/GameObjects/AttackableUnits/AI/BaseTurret.cs), [LaneTurret.cs](GameServerLib/GameObjects/AttackableUnits/AI/LaneTurret.cs) | 포탑 |
| [Scripting/CSharp/Interfaces/IAIScript.cs](GameServerLib/Scripting/CSharp/Interfaces/IAIScript.cs), [AIScriptMetadata.cs](GameServerLib/Scripting/CSharp/AIScriptMetadata.cs), [EmptyAIScript.cs](GameServerLib/Scripting/CSharp/EmptyAIScript.cs) | 스크립트 인터페이스 |
| [GameServerCore/Enums/ClassifyUnit.cs](GameServerCore/Enums/ClassifyUnit.cs), [AIState.cs](GameServerCore/Enums/AIState.cs) | 우선순위, AI 상태 열거형 |
| [Content/LeagueSandbox-Scripts/AIScripts/](Content/LeagueSandbox-Scripts/AIScripts/) | LaneMinionAI, TurretAI, BasicJungleMonsterAI, Pet, MinionAI(미사용) |
| [API/APIMapFunctionManager.cs](GameServerLib/API/APIMapFunctionManager.cs), [API/ApiFunctionManager.cs](GameServerLib/API/ApiFunctionManager.cs) | 유닛 생성 함수, 스크립트용 API |
