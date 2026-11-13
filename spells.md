# 스킬(Spell) 시스템 구조

챔피언 스킬·평타·아이템 액티브·소환사 주문이 서버에서 어떻게 정의되고, 시전 요청부터 피해가 들어가기까지 어떤 경로를 거치는지 코드 기준으로 정리한 문서입니다. AI가 스킬과 평타를 언제 쓰는지는 [ai.md](ai.md), 투사체 충돌의 기반이 되는 충돌 처리는 [pathfinding.md](pathfinding.md) 5장을 참고하세요.

> 요약
> - 스킬은 **데이터 + 스크립트** 두 부분으로 나뉩니다.
>   - **SpellData**: 클라이언트 inibin에서 추출한 JSON입니다. 쿨다운, 마나, 사거리, 시전 시간, 투사체 속도, 대상 플래그 같은 수치를 담습니다.
>   - **ISpellScript**: C# 스크립트입니다. 시전 시 실제로 무슨 일이 일어나는지(피해량 공식, 버프, 투사체 생성)를 정합니다.
> - `Spell` 객체 하나가 슬롯 하나를 맡고, `READY → CASTING → (CHANNELING) → COOLDOWN` 상태머신으로 돌아갑니다.
> - 평타도 같은 `Spell` 클래스입니다(슬롯 64~81).
> - 판정은 두 종류의 게임 오브젝트로 합니다. **SpellMissile**(투사체)과 **SpellSector**(장판·원뿔·다각형 범위)입니다. 둘 다 CollisionHandler의 겹침 검사로 대상을 찾고, `OnSpellHit` 이벤트를 받은 스크립트가 피해를 줍니다.
> - **피해량 공식은 스크립트에 하드코딩**되어 있습니다(예: `15 + 레벨*20 + AD계수`). SpellData의 Effect 수치를 자동으로 쓰지 않습니다.
> - 확인된 문제 중 영향이 큰 것은 **쿨감 부호 불일치**(아이템·버프 쿨감이 오히려 쿨다운을 늘림)와 **채널링 콜백 누락 조건**입니다. 10장에 정리했습니다.

---

## 1. 구성 요소

```
Spell (슬롯 1개)                        GameServerLib/GameObjects/Spell/Spell.cs
 ├─ SpellData        수치 데이터          GameServerLib/Content/SpellData.cs
 │                   ← Content/LeagueSandbox-Default/Spells/{이름}/{이름}.json  (약 3,400개)
 ├─ ISpellScript     동작 스크립트        Content/LeagueSandbox-Scripts/Characters/**  (namespace Spells)
 │   └─ SpellScriptMetadata              스크립트가 엔진에 알려 주는 설정 (투사체/섹터/채널링/시전시간…)
 ├─ CastInfo         이번 시전의 정보     대상, 위치, 레벨, 시전 시간, NetID …  (GameObjects/Spell/CastInfo.cs)
 └─ ToolTipData      툴팁 변수

시전 결과로 생기는 오브젝트
 ├─ SpellMissile     (Target)   유도 투사체 — 평타·지정형 스킬
 │   ├─ SpellCircleMissile (Circle)  직선 스킬샷 (이름과 달리 직선 비행 + 원형 충돌)
 │   │   └─ SpellLineMissile (Arc)
 │   └─ SpellChainMissile (Chained) 튕기는 투사체
 └─ SpellSector      (Area)     원형 장판
     ├─ SpellSectorCone    (Cone)
     └─ SpellSectorPolygon (Polygon)
```

### 1.1 슬롯 배치 — [ObjAIBase.cs:145-219](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L145-L219)

| 슬롯 | 용도 | 스크립트 네임스페이스 |
|---|---|---|
| 0–3 | Q/W/E/R (`CharData.SpellNames`) | `Spells` |
| 4–5 | 소환사 주문 (챔피언은 `clientInfo.SummonerSkills`) | `Spells` |
| 6–12 | 아이템 액티브 (12 = 장신구) | `ItemSpells` |
| 13 | 귀환(BluePill) | `Spells` |
| 14 | 임시 아이템 슬롯 (`SpellCastItem`) | `Spells` |
| 15–44 | 룬 | `Spells` |
| 45–60 | Extra Spells (`CharData.ExtraSpells`) — 투사체 전용 스펠 등. 생성 시 레벨 1 | `Spells` |
| 62 | 패시브 → **Spell 스크립트 대신 `CharScript{모델명}`** 을 로드 | `CharScripts` |
| 64–72 / 73–81 | 일반 평타 / 치명타 평타 (`CharData.BasicAttacks`). `SpellSlot >= 64` 면 `IsAutoAttack = true` | `Spells` |

비어 있는 슬롯은 `"BaseSpell"` 로 채웁니다.

### 1.2 스크립트 로드 — [Spell.LoadScript](GameServerLib/GameObjects/Spell/Spell.cs#L141-L173)

- 스크립트 엔진에서 `"{네임스페이스}.{SpellName}"` 클래스를 찾습니다. 없으면 `SpellScriptEmpty` 를 쓰고, 이때 `HasEmptyScript = true` 가 됩니다.
- 메타데이터에 따라 이벤트 리스너를 등록합니다.
  - `TriggersSpellCasts == true` → `OnSpellCast`, `OnSpellPostCast`
  - `ChannelDuration > 0` → `OnSpellChannel`, `OnSpellChannelCancel`, `OnSpellPostChannel`
- 마지막으로 `Script.OnActivate(owner, spell)` 를 호출합니다. 스크립트는 보통 여기서 `OnSpellHit` 같은 이벤트를 구독합니다.

---

## 2. SpellData: 수치 데이터

[SpellData.Load](GameServerLib/Content/SpellData.cs#L336)가 JSON(원본은 클라이언트 inibin)의 `SpellData` 섹션을 읽습니다.

- **레벨별 배열**은 `GetMultiFloat` 로 읽고 길이가 7입니다. `[0]` 은 기본값, `[1..6]` 은 `Cooldown1..6` 같은 키입니다. 인덱스로 `CastInfo.SpellLevel` 을 그대로 씁니다.
  `Cooldown`, `ManaCost`, `CastRange`, `ChannelDuration`, `CastRadius`, `LocationTargettingLength/Width` …
- **시전 관련**: `TargetingType`(Self / Target / Area / Location / DragDirection / SelfAOE / TargetOrLocation …), `CastType`, `OverrideCastTime`, `DelayCastOffsetPercent`, `CantCancelWhileWindingUp`, `CantCancelWhileChanneling`, `CanMoveWhileChanneling`, `DoesntBreakChannels`, `IsToggleSpell`, `ConsideredAsAutoAttack`, `UseAutoattackCastTime`
- **투사체**: `MissileSpeed`(기본 500), `LineWidth`(충돌 반경), `BounceRadius`(450), `MissilePerceptionBubbleRadius`(투사체 시야), `MissileEffect` 등. 가속, 중력, 최소/최대 속도도 읽지만 사용하지 않습니다.
- **충전형**: `MaxAmmo`, `AmmoRechargeTime[레벨-1]` (이 배열만 0부터 시작)
- **계수**: `AttackDamageCoefficient`, `MagicDamageCoefficient` (`Coefficient2`). 스크립트가 직접 읽어서 씁니다.
- **`Flags : SpellDataFlags`**: 대상 판정 비트입니다. `AffectEnemies / AffectFriends / AffectNeutral / AffectMinions / AffectHeroes / AffectTurrets / AffectBuildings / AffectWards / IgnoreLaneMinion / InstantCast …`
  - [`IsValidTarget(attacker, target, overrideFlags)`](GameServerLib/Content/SpellData.cs#L191)가 이 비트로 팀과 유닛 종류를 거릅니다. 투사체·섹터·채널링이 모두 이 함수를 씁니다.
  - `InstantCast` 는 시전 모션 없이 즉시 발동하고 이동도 멈추지 않습니다(점멸 류).

시전 시간 계산:
- 일반 스킬: [`GetCastTime()`](GameServerLib/Content/SpellData.cs#L290) = `0.5 × (1 + DelayCastOffsetPercent)` 초
- 우선순위: `SpellData.OverrideCastTime` > `ScriptMetadata.CastTime` (뒤에 적용되는 쪽이 덮어씀)
- 평타 판정 스킬(`ConsideredAsAutoAttack`/`UseAutoattackCastTime`, 예: 가렌 Q, 레오나 Q): `GetCharacterAttackCastDelay(공속 배율, …)` (10장 #3 참고)

---

## 3. ISpellScript: 스크립트 인터페이스

[ISpellScript.cs](GameServerLib/Scripting/CSharp/Interfaces/ISpellScript.cs) — 모든 메서드에 기본 구현이 있어서 필요한 것만 구현하면 됩니다.

| 콜백 | 호출 시점 | 호출 조건 |
|---|---|---|
| `OnActivate(owner, spell)` | Spell 생성 / 스크립트 교체 시 | 항상 |
| `OnDeactivate` | `Spell.Deactivate()` (스펠 교체 등) | 항상 |
| `OnSpellPreCast(owner, spell, target, start, end)` | 시전이 확정된 직후, 마나 차감 전 | 항상 (직접 호출) |
| `OnSpellCast(spell)` | 시전 시간 시작 시 | `TriggersSpellCasts` |
| `OnSpellPostCast(spell)` | 시전 완료 시 (`FinishCasting`) | `TriggersSpellCasts` |
| `OnSpellChannel` / `OnSpellChannelCancel(reason)` / `OnSpellPostChannel` | 채널링 시작 / 취소 / 완료 | **`ScriptMetadata.ChannelDuration > 0`** |
| `OnUpdate(diff)` | 매 틱 | 빈 스크립트가 아닐 때 |

[SpellScriptMetadata](GameServerLib/Scripting/CSharp/SpellScriptMetadata.cs) 중 엔진이 실제로 읽는 항목:

| 항목 | 효과 |
|---|---|
| `TriggersSpellCasts` | Cast/PostCast 콜백 등록 |
| `CastTime`, `ChannelDuration` | SpellData 값 덮어쓰기 |
| `MissileParameters` | 시전 완료 시 자동으로 투사체 생성 (`Type`: Target / Circle / Arc / Chained) |
| `SectorParameters` | 시전 완료 시 자동으로 섹터 생성 |
| `AutoFaceDirection` (기본 true) | 시전 시 대상 방향으로 회전 |
| `CooldownIsAffectedByCDR` (기본 true) | 쿨감 적용 여부 |
| `AmmoPerCharge` | 충전 1회당 충전량 |

그 밖의 항목(`IsDamagingSpell`, `NotSingleTargetSpell`, `CastingBreaksStealth`, `AutoCooldownByLevel`, `SpellDamageRatio` …)은 원본 스크립트 포맷을 옮겨 둔 것이고, 엔진에서는 쓰지 않습니다.

---

## 4. 시전 흐름

### 4.1 플레이어 시전 요청 — [HandleCastSpell.cs](GameServerLib/Packets/PacketHandlers/HandleCastSpell.cs)

```
CastSpellRequest(Slot, Position, EndPosition, TargetNetID)
  s = champion.Spells[Slot]
  if owner.CanCast(s) && (시전 중인 스펠 없음 || (s 가 InstantCast && 현재 시전이 취소 가능)):
      if s.Cast(Position, EndPosition, targetUnit):
          아이템 슬롯이고 소모품이면 인벤토리에서 제거
```

[`ObjAIBase.CanCast`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L345-L362): `OnCanCast` 이벤트(스크립트가 거부 가능)를 통과하고, `CanCast` 상태이며, 매혹·공포·침묵·기절·제압·도발·수면·Pacified 상태가 아니고, 시전 중이 아니고, 취소 불가 채널링이나 취소 불가 평타 선딜 중이 아니어야 합니다.

### 4.2 `Spell.Cast(start, end, unit)` — [Spell.cs:284-580](GameServerLib/GameObjects/Spell/Spell.cs#L284-L580)

```
1. 거부 조건
   - Target 타입인데 대상 없음 / 대시 중 (CanCastWhileDisabled 아니면)
   - 마나 부족 (평타 제외) / State != READY / 충전 0
2. Self·SelfAOE·TargetOrLocation 이고 대상이 없으면 대상 = 시전자
3. CastInfo 채우기: 새 SpellNetID·MissileNetID, TargetPosition(높이 포함), 쿨다운, 시전 시간(2장), 대상
4. ★ 사거리 검사 (Target/Area/Location/DragDirection, 평타 제외)
   사거리 밖이면 owner.SetSpellToCast(this, …) 로 예약하고 false 반환
   → 유닛이 걸어가서 사거리에 들어오면 ObjAIBase.UpdateTarget 이 다시 Cast 호출 (ai.md 2.4)
5. 시전 확정
   - 진행 중인 채널링 끊기 (DoesntBreakChannels 아니면)
   - InstantCast 가 아니면 owner.SetCastSpell(this)
   - Script.OnSpellPreCast(...)
   - 마나 차감 (FeatureFlags.EnableManaCosts 일 때만)
   - InstantCast 가 아니면 이동 정지 + MoveOrder = CastSpell + 대상 방향 회전
   - MissileType.Circle 이면 TargetPosition 을 "시전자 앞 최대 사거리 지점"으로 덮어씀
6. 패킷: 평타면 Basic_Attack(_Pos), 아니면 NPC_CastSpellAns. 대상이 있으면 UnitSetLookAt
   충전형(MaxAmmo > 1)이면 충전 1 소모
7. 시전 시간 > 0 → OnSpellCast 발행, State = CASTING
   시전 시간 = 0 (또는 InstantCast) → 즉시 FinishCasting()
```

### 4.3 상태머신 — [Spell.Update](GameServerLib/GameObjects/Spell/Spell.cs#L1572-L1649)

`ObjAIBase.Update` 가 매 틱 모든 슬롯의 `Spell.Update(diff)` 를 호출합니다.

```
          Cast() ──────────────┐ (시전 시간 0 / InstantCast)
             │                  ▼
   READY ──► CASTING ──► FinishCasting() ──► COOLDOWN ──(CurrentCooldown < 0)──► READY
                │             │
                │             └─ 채널 시간 > 0 이면 Channel()
                │                       ▼
                │                 CHANNELING ──(시간 종료)──► FinishChanneling() ──► COOLDOWN
                │                       │
                │                       └─(취소 조건)──► StopChanneling(Cancel) ──► READY  (쿨다운 없음)
                └─(CastCancelCheck 참)──► ResetSpellCast() ──► READY  (쿨다운 없음, 마나는 이미 소모됨)
```

- **CASTING**
  - 매 틱 [`CastCancelCheck`](GameServerLib/GameObjects/Spell/Spell.cs#L213-L282)로 취소 여부를 봅니다. 시전자 사망, 대상 사망·안 보임·타겟 불가, CC(매혹·공포·기절·제압·도발, 스킬이면 침묵, 평타면 무장해제)가 취소 사유입니다. `CantCancelWhileWindingUp` 이면 대상 관련 취소는 건너뜁니다.
  - 일반 스킬은 `CurrentCastTime` 이 0이 될 때까지 기다리고, 평타는 `CurrentDelayTime ≥ 선딜 / 공속배율` 이 될 때까지 기다립니다.
- **CHANNELING** — [`ChannelCancelCheck`](GameServerLib/GameObjects/Spell/Spell.cs#L836-L923)의 취소 사유: 사망, 대시, CC·침묵, 대상 상실, 이동 명령(`CanMoveWhileChanneling` 아닐 때), 공격 명령, 다른 스킬 시전. `CantCancelWhileChanneling` 이면 앞의 네 가지만 적용됩니다.
- **COOLDOWN**: Q~R(슬롯 0~3)만 `CHAR_SetCooldown` 패킷으로 클라이언트에 알립니다.
- **충전형**: `CurrentAmmo < MaxAmmo` 이면 상태와 상관없이 충전 타이머가 돌고, 다 되면 `AddAmmo(AmmoPerCharge)` 와 `S2C_AmmoUpdate` 패킷.

### 4.4 시전 완료 — [`FinishCasting`](GameServerLib/GameObjects/Spell/Spell.cs#L951-L1040)

```
평타(또는 UseAttackCastTime):
   OnLaunchAttack 발행
   원거리 & 빈 스크립트 → 유도 투사체(MissileType.Target) 생성
   근접               → 즉시 ApplyEffects + AutoAttackHit (피해)
   State = READY (평타 간격은 ObjAIBase 의 _autoAttackCurrentCooldown 이 관리)
일반 스킬:
   SpellData 채널 시간이 0 이면 State = COOLDOWN, 쿨다운 패킷
공통:
   ScriptMetadata.MissileParameters 있으면 → CreateSpellMissile()
   ScriptMetadata.SectorParameters  있으면 → CreateSpellSector()
   CastSpell/SpellToCast 정리
   MoveOrder 복구 (InstantCast: 이동 중이면 MoveTo, 대상 있으면 AttackTo / 그 외: Hold)
   OnSpellPostCast 발행 (TriggersSpellCasts)
```

### 4.5 스크립트에서 시전하기 — [ApiFunctionManager.SpellCast](GameServerLib/API/ApiFunctionManager.cs#L959-L1013)

스크립트가 다른 슬롯(주로 Extra 슬롯)의 스펠을 쏠 때 씁니다. 직접 `CastInfo` 를 만들어 `Spell.Cast(castInfo, cast)` 를 호출합니다.
- `fireWithoutCasting = false` → 정상 시전 흐름(마나·상태 검사, 시전 시간, 패킷)
- `fireWithoutCasting = true` → **검사와 시전 시간 없이** `OnSpellPreCast` 후 곧바로 투사체·섹터를 생성합니다(시전 패킷 없음).
- `overrideCastPos` 로 발사 위치를, `overrideForceLevel` 로 레벨을 바꿀 수 있습니다.
- `SpellCastItem` 은 임시 아이템 슬롯(14)에 스펠을 끼워 넣고 시전합니다.

---

## 5. 투사체 (SpellMissile)

모든 투사체는 `GameObject` 로 `ObjectManager` 에 추가되어 매 틱 `Update` 됩니다. 충돌은 CollisionHandler가 `OnCollision` 으로 알려 줍니다(미사일은 "충돌 영향 받음" 대상, pathfinding.md 5.2).

| 타입 | 클래스 | 이동 | 적중 판정 | 소멸 |
|---|---|---|---|---|
| `Target` | [SpellMissile](GameServerLib/GameObjects/Spell/Missile/SpellMissile.cs) | 매 틱 대상 현재 위치로 직선 이동(유도) | 대상 위치에 **정확히 도달**했을 때 `CheckFlagsForUnit` | 적중 시. 대상이 죽거나 타겟 불가가 되면 즉시 |
| `Circle` | [SpellCircleMissile](GameServerLib/GameObjects/Spell/Missile/SpellCircleMissile.cs) | 발사 지점 → `TargetPositionEnd` 방향으로 **사거리만큼** 직선 이동 (`OverrideEndPosition` 가능) | 이동 중 겹친 유닛마다 1회씩 (관통). 건물 무시 | 목적지 도달 시 |
| `Arc` | [SpellLineMissile](GameServerLib/GameObjects/Spell/Missile/SpellLineMissile.cs) | Circle 과 동일 계열 | 동일 | 동일 |
| `Chained` | [SpellChainMissile](GameServerLib/GameObjects/Spell/Missile/SpellChainMissile.cs) | 유도 | 적중 후 `BounceRadius` 안에서 가장 가까운 유효 대상으로 튕김. `MaximumHits`, `CanHitSameTarget(Consecutively)` 지원 | 최대 적중 수 또는 다음 대상 없음 |

- 투사체 속도 `SpellData.MissileSpeed`, 충돌 반경 `SpellData.LineWidth`, 시야 `MissilePerceptionBubbleRadius`.
- 적중하면 [`Spell.ApplyEffects`](GameServerLib/GameObjects/Spell/Spell.cs#L175-L207)가 호출됩니다.
  - 빈 스크립트면 `HitEffectName` 파티클을 자동 재생합니다.
  - 평타면 `OnBeingHit` 를 발행하고 `AutoAttackHit` 로 피해를 줍니다.
  - 스킬이면 `OnSpellHit(spell, (대상, 미사일, 섹터))`, `OnSpellMissileHit`, `OnBeingSpellHit` 를 발행합니다. 스크립트는 여기서 피해를 줍니다.
- 관통 여부는 엔진이 아니라 **스크립트**가 정합니다. 예를 들어 이즈리얼 Q는 적중 콜백에서 `missile.SetToRemove()` 로 첫 대상에서 멈춥니다.
- 지형 충돌은 처리하지 않습니다(TODO). 벽을 그대로 통과합니다.
- 생성하면 `OnLaunchMissile` 을, 소멸하면 `OnSpellMissileEnd` 와 `DestroyClientMissile` 패킷을 보냅니다. `SpellData.MissileEffect` 가 비어 있지 않으면 `IsServerOnly` 로 표시되어 생성 패킷을 보내지 않습니다([PacketNotifier.cs:1611](GameServerLib/Packets/PacketNotifier.cs#L1611)).

---

## 6. 범위 판정 (SpellSector)

[SpellSector.cs](GameServerLib/GameObjects/Spell/Sector/SpellSector.cs)

- 위치는 `CastInfo.TargetPositionEnd` 이고, 충돌 반경은 `max(Length, Width)` 인 원입니다. `BindObject` 를 지정하면 매 틱 그 오브젝트 위치로 순간이동합니다(따라다니는 장판).
- 생성 즉시(`OnAdded`) 충돌 검사와 첫 틱을 실행합니다.
- 흐름: CollisionHandler가 반경 안 유닛마다 `OnCollision` → `IsValidTarget` 이면 `_unitsToHit` 에 쌓음 → `Tickrate` 주기로 `ExecuteTick` → 하위 클래스의 `FilterCollisions`(원뿔 각도, 다각형 내부 판정)를 통과한 유닛에 `HitUnit` → `ApplyEffects`.
- 파라미터: `SingleTick`(한 번 판정 후 소멸), `Tickrate`(초당 판정 횟수), `Lifetime`(초, -1이면 무한), `MaximumHits`, `CanHitSameTarget`(범위를 벗어났다가 다시 들어오면 재적중), `CanHitSameTargetConsecutively`(매 틱 재적중), `OverrideFlags`(SpellData.Flags 대신 사용).
- 타입: `Area`(원), `Cone`(`ConeAngle`), `Polygon`(`PolygonVertices`), `Ring`(미구현).

---

## 7. 평타

평타도 `Spell` 이지만 흐름이 조금 다릅니다.

1. [`ObjAIBase.UpdateTarget`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L1233-L1266)이 사거리(`Range + 대상 반지름`) 안이고 평타 쿨다운이 끝났으면
   - 치명타를 굴리고(`CriticalChance`), 결과에 따라 일반(64~72) 또는 치명타(73~81) 평타 슬롯에서 무작위로 고릅니다(`GetNewAutoAttack`).
   - `AutoAttackSpell.Cast(대상)` 을 호출하고, 다음 평타까지 간격 = `1 / 공격속도`.
2. 선딜 = `1.6 × (1 + AttackDelayOffsetPercent) × (0.3 + AttackDelayCastOffsetPercent)` 를 공속 배율로 나눈 값. 첫 평타는 `Basic_Attack_Pos`, 이어지는 평타는 `Basic_Attack` 패킷을 보냅니다.
3. 선딜이 끝나면 `FinishCasting`: 근접은 즉시 피해, 원거리는 유도 투사체.
4. 피해: [`ObjAIBase.AutoAttackHit`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L265-L296) — AD(치명타면 × `CriticalDamage`) 물리 피해. 실명이면 MISS. `OnHitUnit` 이벤트로 적중 시 효과(온힛)를 붙일 수 있습니다.
5. 평타 스크립트가 있으면(예: 특수 평타) 투사체 생성을 스크립트가 맡습니다.

---

## 8. 스크립트 작성 패턴

### 8.1 스킬샷: 본 스펠 + Extra 슬롯 투사체 스펠 — [Ezreal/Q.cs](Content/LeagueSandbox-Scripts/Characters/Ezreal/Q.cs)

```csharp
public class EzrealMysticShot : ISpellScript            // Q 슬롯
{
    ScriptMetadata = { TriggersSpellCasts = true };
    OnSpellPostCast(spell) {
        // 시전이 끝나면 Extra 슬롯 0 의 투사체 스펠을 쏜다
        SpellCast(owner, 0, SpellSlotType.ExtraSlots, targetPos, targetPos, false, Vector2.Zero);
    }
}
public class EzrealMysticShotMissile : ISpellScript     // Extra 슬롯 (CharData.ExtraSpells)
{
    ScriptMetadata = { MissileParameters = new() { Type = MissileType.Circle } };  // 완료 시 자동 생성
    OnActivate(...) => ApiEventManager.OnSpellHit.AddListener(this, spell, TargetExecute, false);
    TargetExecute(spell, target, missile, sector) {
        var damage = 15 + spell.CastInfo.SpellLevel * 20 + AD * AttackDamageCoefficient + AP * MagicDamageCoefficient;
        target.TakeDamage(owner, damage, DAMAGE_TYPE_PHYSICAL, ...);
        owner.Spells[0..3].LowerCooldown(1);
        missile.SetToRemove();                         // 첫 적중에서 멈춤 (비관통)
    }
}
```

### 8.2 즉발 범위기: PostCast에서 섹터 생성 — [Annie/W.cs](Content/LeagueSandbox-Scripts/Characters/Annie/W.cs)

```csharp
OnSpellPostCast(spell) =>
    spell.CreateSpellSector(new SectorParameters { Type = SectorType.Cone, Length = 625, ConeAngle = 24.76f, SingleTick = true });
TargetExecute(...) => target.TakeDamage(owner, 70 + 레벨*45 + AP*0.8, DAMAGE_TYPE_MAGICAL, ...);
```

### 8.3 채널링 — [Global/Recall.cs](Content/LeagueSandbox-Scripts/Characters/Global/Recall.cs)

```csharp
ScriptMetadata = { ChannelDuration = 8.0f };     // 이 값이 있어야 채널 콜백이 등록됨
OnSpellChannel       → 파티클 + "Recall" 버프
OnSpellChannelCancel → 파티클·버프 제거
OnSpellPostChannel   → owner.Recall()
```

### 8.4 그 밖의 연결 지점

- **버프**: `AddBuff(이름, 시간, 스택, spell, 대상, 시전자)` — `Buffs/` 의 `IBuffGameScript` 와 연결됩니다.
- **패시브**: 슬롯 62는 `CharScript{모델명}`(`ICharScript`)이 담당합니다. 스크립트 19개.
- **스펠 교체**: `SetSpell(이름, 슬롯)` (변신, 재사용 스킬), `SetCooldown`, `LowerCooldown`, `SetSpellToggle`, `SetOverrideCastRange`, `SetToolTipVar`.
- **현황**: 캐릭터 폴더 51개, 스크립트 95개(챔피언 약 35명 + 정글·포탑·렐릭 등), 버프 폴더 35개. 나머지 챔피언 스킬은 `SpellScriptEmpty` 라서 시전 모션과 쿨다운만 돌고 효과는 없습니다.

### 8.5 스킬 관련 이벤트 ([ApiEventManager](GameServerLib/API/ApiEventManager.cs))

`OnCanCast` `OnSpellCast` `OnSpellPostCast` `OnSpellChannel` `OnSpellChannelCancel` `OnSpellPostChannel` `OnLaunchMissile` `OnSpellMissileUpdate` `OnSpellMissileHit` `OnSpellMissileEnd` `OnCreateSector` `OnSpellSectorHit` `OnSpellHit` `OnBeingSpellHit` `OnPreAttack` `OnLaunchAttack` `OnHitUnit` `OnBeingHit` `OnLevelUpSpell` `OnPreDealDamage` `OnDealDamage` `OnPreTakeDamage` `OnTakeDamage`

---

## 9. 스킬 레벨업

[HandleUpgradeSpell](GameServerLib/Packets/PacketHandlers/HandleUpgradeSpell.cs) → [`Champion.LevelUpSpell`](GameServerLib/GameObjects/AttackableUnits/AI/Champion.cs#L199-L208) → [`ObjAIBase.LevelUpSpell`](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L738-L750)

- 조건: 스킬 포인트가 있어야 하고, `CharData.SpellsUpLevels[슬롯][현재레벨] <= 챔피언 레벨` 이어야 합니다(궁극기 6/11/16 제한 등).
- `Spell.LevelUp()`: 레벨을 올리고 `Stats.ManaCost[슬롯]` 을 갱신합니다 → `OnLevelUpSpell` → `NPC_UpgradeSpellAns` 패킷.

---

## 10. 확인된 문제 / 미구현

| # | 내용 | 위치 |
|---|---|---|
| 1 | **쿨감 부호 불일치.** 엔진은 `쿨다운 × (1 + CooldownReduction)` 로 계산하고, 클라이언트로는 `mPercentCooldownMod`(음수 = 감소)로 보냅니다. 채팅 명령은 값을 빼서 맞게 동작하지만, **아이템·버프 스크립트는 `+0.1` 처럼 양수를 더해서 쿨다운이 오히려 10% 늘어납니다**(Nashor's Tooth, Stinger, 블루 버프, URF 등). 최대 쿨감(`PercentCooldownModMinimum = -0.4`) 제한도 적용하지 않습니다 | [Spell.cs:1289](GameServerLib/GameObjects/Spell/Spell.cs#L1289), [Nashor's Tooth.cs:13](Content/LeagueSandbox-Scripts/Items/Passives/Nashor's%20Tooth.cs#L13) |
| 2 | **채널링 콜백 누락.** 채널 콜백은 `ScriptMetadata.ChannelDuration > 0` 일 때만 등록됩니다. SpellData에만 채널 시간이 있는 스펠은 채널링 상태로는 들어가지만 `OnSpellChannel/PostChannel` 이 호출되지 않습니다 | [Spell.cs:157-162](GameServerLib/GameObjects/Spell/Spell.cs#L157-L162) |
| 3 | `GetCharacterAttackCastDelay` 가 `Math.Min(0.3 + offset, 0)` 을 씀 → 시전 비율이 항상 0 이하. 그래서 `ConsideredAsAutoAttack` 스킬(가렌 Q 등)의 시전 시간이 0 이하가 되어 즉시 발동합니다. `Max` 의 오타로 보입니다 | [SpellData.cs:324](GameServerLib/Content/SpellData.cs#L324) |
| 4 | 시전 시간이 0이거나 `InstantCast` 인 스펠은 `Cast()` 에서 바로 `FinishCasting` 하고 `Channel()` 은 부르지 않음 → 이 경로에서는 채널링이 시작되지 않습니다(`Channel()` 은 CASTING 상태의 Update에서만 호출) | [Spell.cs:545-577](GameServerLib/GameObjects/Spell/Spell.cs#L545-L577), [Spell.cs:1601-1607](GameServerLib/GameObjects/Spell/Spell.cs#L1601-L1607) |
| 5 | `Champion.LevelUpSpell` 이 레벨 제한을 확인하기 **전에** 스킬 포인트를 차감 → 조건 미달이면 포인트만 사라짐 | [Champion.cs:199-208](GameServerLib/GameObjects/AttackableUnits/AI/Champion.cs#L199-L208) |
| 6 | `HandleCastSpell` 의 "시전 중에도 InstantCast 허용" 분기가 실행될 수 없음. 앞에서 호출하는 `CanCast` 가 이미 `_castingSpell == null` 을 요구함 | [HandleCastSpell.cs](GameServerLib/Packets/PacketHandlers/HandleCastSpell.cs), [ObjAIBase.cs:359](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs#L359) |
| 7 | `SectorType.Ring` 미구현인데 `CreateSpellSector` 가 null을 `ObjectManager.AddObject` 에 넘김 → 예외 | [Spell.cs:1258-1265](GameServerLib/GameObjects/Spell/Spell.cs#L1258-L1265) |
| 8 | 섹터에서 `SingleTick = false` 이고 `Tickrate = 0` 이면 판정 주기가 무한대가 되어 한 번도 판정하지 않음 (기본 `Lifetime = -1` 이면 사라지지도 않음) | [SpellSector.cs Update](GameServerLib/GameObjects/Spell/Sector/SpellSector.cs) |
| 9 | Circle/Arc 투사체는 `MissileParameters.MaximumHits`·`CanHitSameTarget` 을 무시 (무제한 관통, 대상당 1회). 관통 제한은 스크립트가 직접 처리해야 함 | [SpellCircleMissile.cs](GameServerLib/GameObjects/Spell/Missile/SpellCircleMissile.cs) |
| 10 | 투사체 지형 충돌 미구현(벽 통과). 빠른 투사체의 터널링 미처리(한 틱에 대상을 지나치면 맞지 않음) | [SpellMissile.cs OnCollision](GameServerLib/GameObjects/Spell/Missile/SpellMissile.cs), pathfinding.md 7장 #5 |
| 11 | 유도 투사체 적중이 "위치가 정확히 같은가"로 판정됨. 대상이 `IsValidTarget` 을 통과하지 못하면 투사체가 대상에 붙은 채로 사라지지 않음 | [SpellMissile.cs Move/CheckFlagsForUnit](GameServerLib/GameObjects/Spell/Missile/SpellMissile.cs) |
| 12 | 차지형 스킬(바루스 Q, 제라스 Q 등) 요청 `SpellChargeUpdateReq` 미구현 (좌표를 채팅으로 출력만 함) | [HandleSpellChargeUpdateReq.cs](GameServerLib/Packets/PacketHandlers/HandleSpellChargeUpdateReq.cs) |
| 13 | `GetCurrentCastRange` 가 레벨별 사거리를 "기본값보다 클 때만" 쓰고 마지막 인덱스(레벨 6)는 보지 않음 → 레벨이 오를수록 사거리가 줄어드는 스킬은 반영 안 됨 | [Spell.cs:1318-1344](GameServerLib/GameObjects/Spell/Spell.cs#L1318-L1344) |
| 14 | 피해량이 스크립트에 하드코딩되어 SpellData의 Effect 수치와 따로 관리됨. 스크립트가 없는 스펠은 효과 없음 | 8장 |
| 15 | `SpellScriptMetadata` 의 상당수 항목과 SpellData의 투사체 가속·중력·최소/최대 속도는 읽기만 하고 미사용. `Deactivate` 는 스펠 제거 시 호출되지 않음(코드 주석) | 3장, [Spell.cs:164](GameServerLib/GameObjects/Spell/Spell.cs#L164) |

---

## 11. 관련 파일

| 파일 | 역할 |
|---|---|
| [GameObjects/Spell/Spell.cs](GameServerLib/GameObjects/Spell/Spell.cs) | 스펠 상태머신, 시전·채널·쿨다운, 투사체·섹터 생성 |
| [GameObjects/Spell/CastInfo.cs](GameServerLib/GameObjects/Spell/CastInfo.cs), [CastTarget.cs](GameServerLib/GameObjects/Spell/CastTarget.cs) | 시전 정보 |
| [GameObjects/Spell/Missile/](GameServerLib/GameObjects/Spell/Missile/) | 투사체 4종 |
| [GameObjects/Spell/Sector/](GameServerLib/GameObjects/Spell/Sector/) | 범위 판정 3종 |
| [Content/SpellData.cs](GameServerLib/Content/SpellData.cs) | 스펠 수치 데이터, 대상 판정(`IsValidTarget`) |
| [GameServerCore/Enums/SpellDataFlags.cs](GameServerCore/Enums/SpellDataFlags.cs) | 대상 플래그 |
| [Scripting/CSharp/Interfaces/ISpellScript.cs](GameServerLib/Scripting/CSharp/Interfaces/ISpellScript.cs), [SpellScriptMetadata.cs](GameServerLib/Scripting/CSharp/SpellScriptMetadata.cs) | 스크립트 인터페이스 |
| [Packets/PacketHandlers/HandleCastSpell.cs](GameServerLib/Packets/PacketHandlers/HandleCastSpell.cs), [HandleUpgradeSpell.cs](GameServerLib/Packets/PacketHandlers/HandleUpgradeSpell.cs) | 클라이언트 요청 |
| [API/ApiFunctionManager.cs](GameServerLib/API/ApiFunctionManager.cs), [API/ApiEventManager.cs](GameServerLib/API/ApiEventManager.cs) | 스크립트용 API·이벤트 |
| [AI/ObjAIBase.cs](GameServerLib/GameObjects/AttackableUnits/AI/ObjAIBase.cs) | 슬롯 구성, `CanCast`, 평타 루프, `AutoAttackHit` |
| [Content/LeagueSandbox-Scripts/Characters/](Content/LeagueSandbox-Scripts/Characters/) | 챔피언·몬스터 스펠/패시브 스크립트 |
| `Content/LeagueSandbox-Default/Spells/` | SpellData JSON |
