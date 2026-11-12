# League Sandbox Game Server (한글 코드 분석)

> **이 저장소는 보관(archived) 상태이며 더 이상 기여를 받지 않습니다.**
> 2022년 8월 Riot Games의 중지 요청(cease & desist) 이후 원 관리자가 프로젝트를 보관 처리했습니다.
> 원문 공지는 git 히스토리(`e8ea2227`)에서, 예전 설치 안내는 [README-old.md](README-old.md)에서 볼 수 있습니다.

리그 오브 레전드 **4.20 버전 클라이언트**와 통신하는 오픈소스 게임 서버 에뮬레이터입니다.
C# / **.NET 6** 으로 작성되었고, 클라이언트 패킷을 받아 게임 상태(챔피언, 미니언, 포탑, 스킬, 버프, 아이템 등)를
서버에서 시뮬레이션한 뒤 결과를 다시 클라이언트로 보냅니다.

---

## 1. 솔루션 구성

| 프로젝트 | 역할 |
| --- | --- |
| **GameServerConsole** | 실행 진입점(콘솔 앱). 명령줄 인자·설정 파일을 읽고 서버를 띄우며, 디버그 빌드에서는 게임 클라이언트를 자동 실행합니다. |
| **GameServerLib** | 서버 본체. 게임 루프, 네트워크, 게임 오브젝트, 맵, 스크립트 엔진, 콘텐츠 로딩, 채팅 명령어 등 핵심 로직 전부가 있습니다. |
| **GameServerCore** | 공용 계층. 열거형(Enums, 약 100개), 요청 패킷 정의(`ICoreRequest` 계열), 네트워크 핸들러 기반 클래스, 도메인 인터페이스. |
| **GameMaths** | 수학/기하 유틸리티. 다각형(Polygons), Clipper 라이브러리, 벡터 확장 메서드. |
| **QuadTree** | 충돌 검사용 공간 분할 자료구조. |
| **LeaguePackets** *(서브모듈)* | 4.20 클라이언트 패킷 정의(`PacketDefinitions420`) 직렬화 라이브러리. |
| **Content/LeagueSandbox-Default** *(서브모듈)* | 게임 데이터(챔피언/스펠/아이템 정의, 맵, 내비게이션 그리드 등). |
| **Content/LeagueSandbox-Scripts** | 챔피언 스킬, 버프, 아이템, 특성, 맵, AI를 C#으로 구현한 스크립트 패키지. `LeagueSandbox-Default`에 의존합니다. |
| **GameServerLibTests / GameServerConsoleTests** | MSTest 기반 단위 테스트 (스탯, 아이템, 채팅, 패킷, 명령줄 인자 등). |

프로젝트 참조 관계:

```text
GameServerConsole ──► GameServerLib ──► GameServerCore ──► LeaguePackets, QuadTree
                                    ├─► GameMaths
                                    ├─► QuadTree
                                    └─► LeaguePackets
Content/LeagueSandbox-Scripts  (런타임에 Roslyn으로 컴파일되어 로드됨)
```

주요 NuGet 패키지: `LENet`(ENet 기반 UDP 네트워킹), `Microsoft.CodeAnalysis.CSharp`(Roslyn, 스크립트 컴파일),
`log4net`(로깅), `Newtonsoft.Json`(설정/데이터 파싱), `CommandLineParser`(명령줄 인자), `Crc32.NET`.

---

## 2. 실행 흐름

```text
Program.Main  (GameServerConsole/Program.cs)
 ├─ ArgsOptions.Parse(args)            --config, --config-gameserver, --config-json, --port(기본 5119)
 ├─ LoadConfig(GameInfo.json)           파일이 없으면 내장 기본값으로 생성
 ├─ new GameServerLauncher(port, json)  (GameServerLib/Program.cs)
 │   └─ new Server(game, port, json)
 │       ├─ Config.LoadFromJson        플레이어별 Blowfish 키 수집
 │       └─ Server.Start()
 │           ├─ PacketServer.InitServer  ENet 호스트 생성, Blowfish 암호화 준비
 │           └─ Game.Initialize          콘텐츠·맵·매니저·패킷 핸들러·플레이어 초기화
 ├─ (DEBUG) GameServerSettings.json 의 autoStartClient 가 true 면 클라이언트 자동 실행
 └─ GameServerLauncher.StartNetworkLoop()
     └─ Game.GameLoop()                 메인 루프 진입
```

### `Game.Initialize` 에서 하는 일 ([GameServerLib/Game.cs](GameServerLib/Game.cs))

1. `Config.LoadContent` – 데이터 패키지 로드 및 스크립트 컴파일
2. `ChatCommandManager.LoadCommands` – 채팅 명령어 등록
3. `MapScriptHandler` 생성 – 맵 스크립트, 내비게이션 그리드, 충돌/경로 처리
4. `PacketNotifier`, `ObjectManager`, `ProtectionManager` 생성
5. 스크립트용 API(`ApiFunctionManager`, `ApiEventManager`, `ApiMapFunctionManager`, `ApiGameEvents`)에 Game 인스턴스 주입
6. `InitializePacketHandlers` – 요청 패킷 타입별 핸들러 등록 (이동, 스킬 시전, 아이템 구매/판매, 채팅, 핑, 일시정지, 항복 등 약 30종)
7. 설정 파일의 플레이어들을 `PlayerManager`에 추가

### 게임 루프 (`Game.GameLoop`)

- **60Hz**(`REFRESH_RATE = 1000/60 ms`)로 동작합니다.
- 매 틱마다 ENet 네트워크 이벤트(`PacketServer.NetLoop`: 연결/수신/해제)를 처리하고, 게임이 시작된 상태면 `Update(deltaTime)` 를 호출합니다.
- 모든 플레이어가 로딩을 마치지 않아도 `forcedStart` 초가 지나면 게임을 강제로 시작합니다.
- 일시정지 중에는 1초 주기로 돌면서 남은 일시정지 시간(최대 30분)을 줄입니다.

`Update(diff)` 의 처리 순서:

1. `Map.Update` – 충돌 처리(QuadTree), 맵 스크립트
2. `ObjectManager.Update` – 모든 게임 오브젝트 갱신, 시야(vision) 계산, 클라이언트 스폰/동기화
3. `ProtectionManager.Update` – 포탑·억제기·넥서스 보호 상태
4. 채팅 명령어 / 스크립트 타이머(`GameScriptTimer`) 갱신
5. 10초마다 클라이언트와 게임 시간 동기화(`NotifySynchSimTimeS2C`)

---

## 3. GameServerLib 내부 구조

| 폴더/파일 | 내용 |
| --- | --- |
| `Game.cs`, `Server.cs`, `Config.cs` | 게임 루프, 서버 시작, 설정 파싱 |
| `ObjectManager.cs` | 오브젝트 추가/제거/조회, 팀별 시야 제공자, 범위 내 유닛 검색, 포탑·억제기·챔피언 관리 |
| `ProtectionManager.cs` | 구조물 보호(앞 포탑이 살아있으면 무적 등) 규칙 |
| `Packets/` | `PacketServer`(ENet 서버), `PacketHandlerManager`(패킷 디스패치), `PacketNotifier`(서버→클라이언트 패킷 송신), `Blowfish`(암호화), `NetworkIdManager` |
| `Packets/PacketHandlers/` | 요청 패킷별 처리기 (`HandleMove`, `HandleCastSpell`, `HandleBuyItem` …) |
| `GameObjects/` | 게임 오브젝트 계층 (아래 참조) |
| `GameObjects/Spell/` | `Spell`, `CastInfo`, 투사체(`SpellMissile`·직선/원형/체인), 범위 판정(`SpellSector`·원뿔/다각형) |
| `GameObjects/Stats/` | `Stats`, `Stat`, `StatsModifier`, 클라이언트 복제용 `Replication*` 클래스 |
| `Handlers/` | `MapScriptHandler`, `CollisionHandler`(QuadTree), `PathingHandler`, `SurrenderHandler` |
| `Content/` | `ContentManager`(데이터 패키지 로드), `CharData`, `SpellData`, 아이템/특성 데이터, `GlobalData`(전역 상수), `Navigation/`(내비게이션 그리드 & A* 경로탐색) |
| `Inventory/` | 인벤토리, 아이템/조합법, 상점, 룬, 특성 |
| `Scripting/CSharp/` | `CSharpScriptEngine`(Roslyn 런타임 컴파일), 스크립트 인터페이스, 빈(Empty) 기본 스크립트, `GameScriptTimer` |
| `API/` | 스크립트에서 쓰는 API: `ApiFunctionManager`(유닛 생성, 파티클, 대시, 데미지 등), `ApiEventManager`(이벤트 리스너), `ApiMapFunctionManager`, `ApiGameEvents` |
| `Chatbox/` | 채팅 명령어 시스템 (`!` 접두사) |
| `Players/`, `NetInfo/` | 플레이어 정보·접속 상태 관리 |

### 게임 오브젝트 계층

```text
GameObject
 ├─ AttackableUnit                       체력·스탯·버프·이동을 가진 유닛
 │   ├─ ObjAIBase                        AI/스펠을 가진 유닛
 │   │   ├─ Champion
 │   │   ├─ Minion ─ LaneMinion, Pet
 │   │   ├─ Monster
 │   │   └─ BaseTurret ─ LaneTurret, AzirTurret
 │   └─ ObjBuilding
 │       └─ ObjAnimatedBuilding ─ Inhibitor, Nexus
 ├─ SpellMissile / SpellSector          스킬 투사체·범위 판정
 ├─ Particle, Region, LevelProp
 └─ Fountain, MonsterCamp, Buff 등
```

---

## 4. 스크립트 시스템

게임 콘텐츠(스킬, 버프, 아이템 효과, 맵 규칙, AI)는 서버 코드가 아니라
`Content/LeagueSandbox-Scripts` 의 C# 파일로 작성되고, **서버 시작 시 Roslyn으로 컴파일되어 로드**됩니다.

- `CSharpScriptEngine` 이 패키지 폴더의 `.cs` 파일(bin/obj 제외)을 모아 메모리에서 어셈블리로 컴파일한 뒤,
  네임스페이스 + 클래스 이름으로 스크립트 인스턴스를 생성합니다.
- 해당 스크립트가 없으면 `EmptyAIScript`, `BuffScriptEmpty` 같은 빈 기본 스크립트가 사용됩니다.
- **핫 리로드**: `!hotreload` 명령으로 `FileSystemWatcher` 를 켜면 스크립트 파일 변경 시 자동 재컴파일되고,
  살아있는 유닛의 캐릭터·스펠·버프 스크립트가 다시 로드됩니다(`!reloadscripts` 로 수동 재로드도 가능).

스크립트 인터페이스 (`GameServerCore.Scripting.CSharp`):

| 인터페이스 | 용도 | 예시 위치 |
| --- | --- | --- |
| `ISpellScript` | 스킬 (`OnActivate`, `OnSpellCast`, `OnSpellPostCast` …) | `Characters/Ezreal/Q.cs` |
| `ICharScript` | 챔피언 패시브 | `Characters/Ezreal/CharScriptEzreal.cs` |
| `IBuffGameScript` | 버프/디버프 | `Buffs/` |
| `IItemScript` | 아이템 액티브/패시브 | `Items/` |
| `ITalentScript` | 특성 | `Talents/` |
| `IMapScript` | 맵 규칙, 미니언/정글 스폰 | `Maps/Map1` (소환사의 협곡) 등 |
| `IAIScript` | 미니언·포탑·정글 몬스터 AI | `AIScripts/` |

스크립트는 `ApiEventManager.OnXxx.AddListener(...)` 로 게임 이벤트(스탯 갱신, 피격, 사망 등)를 구독하고,
`ApiFunctionManager` 의 정적 함수(`AddParticleTarget`, `SpellCast`, `FaceDirection` …)로 게임에 영향을 줍니다.

```csharp
// Content/LeagueSandbox-Scripts/Characters/Ezreal/Q.cs (발췌)
public class EzrealMysticShot : ISpellScript
{
    public void OnActivate(ObjAIBase owner, Spell spell)
    {
        ApiEventManager.OnUpdateStats.AddListener(this, owner, OnStatsUpdate, false);
    }

    public void OnSpellPostCast(Spell spell)
    {
        var owner = spell.CastInfo.Owner as Champion;
        FaceDirection(targetPos, owner);
        SpellCast(owner, 0, SpellSlotType.ExtraSlots, targetPos, targetPos, false, Vector2.Zero);
    }
}
```

구현된 맵 스크립트: `Map1`(소환사의 협곡), `Map8`(수정의 상처/도미니언), `Map10`(뒤틀린 숲), `Map11`(신 소환사의 협곡), `Map12`(칼바람 나락).

---

## 5. 네트워크

- **전송 계층**: `LENet` (리그 클라이언트용 ENet 포크), UDP, 기본 포트 **5119**
- **암호화**: 플레이어별 Blowfish 키 (`GameInfo.json` 의 `blowfishKey`)
- **수신**: `PacketServer.NetLoop` → `PacketHandlerManager.HandlePacket` → 패킷 타입별로 등록된 `RequestHandler` 콜백
- **송신**: `PacketNotifier` 가 `LeaguePackets` 의 패킷 구조체를 만들어 대상(개별/팀/전체/시야 내)에게 전송
- 오브젝트 스탯 변경은 `Replication` 클래스가 변경분만 묶어 클라이언트에 복제합니다.

---

## 6. 동기화 모델 분석

### 6.1 동기화 방식: lockstep이 아닌 서버 권한(server-authoritative) 상태 동기화

#### 근거

1. **시뮬레이션은 서버에서만 돕니다.** `Game.GameLoop` 는 60Hz로 `Update(deltaTime)` 를 직접 호출하고,
   클라이언트 입력이 모일 때까지 기다리지 않습니다. lockstep이라면 프레임마다 모든 플레이어의 입력이 모여야 다음 틱으로 넘어갑니다.
2. **클라이언트는 결과가 아니라 "요청"만 보냅니다.** `MovementRequest`, `CastSpellRequest`, `BuyItemRequest` 같은 명령을 보내면
   서버가 판정합니다. 예를 들어 [HandleMove.cs](GameServerLib/Packets/PacketHandlers/HandleMove.cs) 는 서버의 `NavigationGrid` 로 경로를 다시 계산합니다.
3. **서버가 계산한 상태를 내려보냅니다** ([PacketNotifier.cs](GameServerLib/Packets/PacketNotifier.cs)).
   - `NotifyWaypointGroup` / `NotifyWaypointList`: 이동 경로(웨이포인트). 클라이언트는 이를 보간해 이동을 그립니다.
   - `NotifyOnReplication`: 체력·마나·스탯의 변경분 복제(`Replication*` 클래스).
   - `NotifySpawn` / `NotifyEnterVisibilityClient`: 시야 안에 들어온 오브젝트만 해당 클라이언트에 보냅니다
     (`ObjectManager.UpdateVisionSpawnAndSync`). 모든 클라이언트가 전체 상태를 계산하는 lockstep과 달리 보이지 않는 정보는 전송하지 않습니다.
4. **시간 동기화는 주기적 보정 수준입니다.** 10초마다 `NotifySynchSimTimeS2C(GameTime)` 로 게임 시간을 보냅니다.

| 항목 | lockstep | 이 서버 |
| --- | --- | --- |
| 시뮬레이션 위치 | 모든 클라이언트 (결정론적) | 서버만 |
| 네트워크로 보내는 것 | 입력(명령) | 클라이언트→서버는 명령, 서버→클라이언트는 상태 |
| 틱 진행 | 모든 입력이 모일 때까지 대기 | 서버가 60Hz로 혼자 진행 |
| 시야 정보 | 전부 공유 | 시야 안의 것만 전송 |

### 6.2 애니메이션 동기화

별도의 애니메이션 동기화 채널이나, 프레임·재생 시간을 주기적으로 맞추는 처리는 **없습니다.**
서버는 "무엇을 하는지"와 "얼마나 걸리는지"를 담은 이벤트 패킷만 보내고, 클라이언트가 그 정보로 애니메이션을 직접 고르고 재생합니다.

#### 애니메이션에 영향을 주는 경로

1. **이동 → 달리기/대기 (암묵적)**
   - 웨이포인트를 받은 클라이언트가 이동을 보간하면서 RUN/IDLE 애니메이션을 직접 재생합니다.
   - 이동 패킷의 `SyncID` 에는 `Environment.TickCount` 가 들어가며, 오래된 이동 패킷을 걸러내는 용도로 보입니다.
   - 순간이동 시 `TeleportID` 를 1 올려([AttackableUnit.cs:845](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L845)) 클라이언트가 보간 없이 바로 위치를 옮기게 합니다.
2. **평타 / 스킬 시전 → 타이밍 값을 함께 전송**
   - [Spell.cs](GameServerLib/GameObjects/Spell/Spell.cs) 에서 공속과 캐릭터 데이터(`AttackDelayCastOffsetPercent` 등)로
     `DesignerCastTime`(선딜, 투사체 발사·데미지 시점)과 `DesignerTotalTime`(전체 동작 길이)을 계산합니다.
   - 이 값과 `AttackSpeedModifier` 를 `NotifyNPC_CastSpellAns`, `NotifyBasic_Attack` 에 실어 보내면, 클라이언트가 이에 맞춰 애니메이션 속도를 조절해 서버 판정 시점과 모션을 맞춥니다.
   - 평타 패킷의 `ExtraTime` 에는 서버 지연분을 음수로 넣어 보정합니다(`-CurrentDelayTime`). 다만 코드 주석에 `TODO: Find out what should go here` 라고 되어 있어 확정된 구현은 아닙니다.
   - 공격 취소 시 `NotifyNPC_InstantStop_Attack` 으로 클라이언트 모션을 끊습니다.
3. **스크립트에서 명시적으로 제어** ([GameObject.cs:424-447](GameServerLib/GameObjects/GameObject.cs#L424-L447))
   - `PlayAnimation(name, timeScale, startTime, speedScale, flags)`: 특정 애니메이션 재생. `startTime` 으로 중간부터 재생 가능.
   - `PauseAnimation`, `StopAnimation`(페이드, lock 무시 옵션).
   - `SetAnimStates`: 애니메이션 교체. 예) 대시 중 `{"RUN": "Spell3"}`, 종료 시 `{"RUN": ""}`
     ([AttackableUnit.cs:1550-1585](GameServerLib/GameObjects/AttackableUnits/AttackableUnit.cs#L1550-L1585)).
   - `NotifyFaceDirection`: 회전 방향과 회전 시간(`turnTime`, 기본 0.0833초).
4. **기타**: 스탯 복제로 보낸 이동속도·공속이 클라이언트 애니메이션 속도에 반영됩니다. 이모트, 사망, 레벨프롭 애니메이션은 각각 이벤트 패킷 하나로 처리합니다.

#### 한계 / 주의점

- 재생 위치를 주기적으로 보정하지 않습니다. 패킷이 늦으면 애니메이션도 그만큼 늦게 시작하고, 서버 판정은 서버 시간 기준이라
  지연이 크면 "모션은 아직인데 데미지는 이미 들어온" 상황이 생길 수 있습니다. `SyncSimTime` 은 게임 시계만 맞춥니다.
- `NotifyS2C_SetAnimStates` 는 시야 필터 없이 전체에 보냅니다(`BroadcastPacket`, [PacketNotifier.cs:3134](GameServerLib/Packets/PacketNotifier.cs#L3134)).
  다른 애니메이션 패킷은 대부분 `BroadcastPacketVision` 을 쓰므로, 안개 속 유닛 상태가 새는 버그일 가능성이 있습니다.
- 동기화 품질은 클라이언트에 크게 의존합니다. 서버는 원본 4.20 클라이언트 데이터(`CastFrame`, 공속 비율 등)를 읽어 같은 공식으로 시간을 계산할 뿐이며,
  리플레이를 분석해 맞춘 값이 섞여 있습니다(주석의 "Value range from replays" 등).

### 6.3 랙 보상(lag compensation)

랙 보상은 **구현되어 있지 않습니다.** 클라이언트 요청은 도착한 시점의 서버 상태에 그대로 적용됩니다.

- **과거 상태 되감기(rewind) 없음**: 위치·히트박스 기록 버퍼가 없습니다. `Champion.EventHistory` 는 사망 화면(death recap)용 피해·버프 기록입니다.
- **요청 즉시 적용**: [HandleCastSpell.cs:42](GameServerLib/Packets/PacketHandlers/HandleCastSpell.cs#L42) 는 `s.Cast(req.Position, req.EndPosition, targetUnit)` 를 바로 호출합니다. 요청에 타임스탬프가 없어 지연을 고려할 수 없습니다.
- **RTT 측정·활용 없음**: ENet 피어의 RTT를 읽는 코드가 없습니다.
  [HandleSyncSimTime.cs](GameServerLib/Packets/PacketHandlers/HandleSyncSimTime.cs) 는 클라이언트 시간이 앞서면 경고 로그만 남기고,
  `HandleLoadPing` 은 로딩 화면의 진행률·핑을 다른 플레이어에게 전달할 뿐입니다.
- **확인(confirm) 패킷 미처리**: `HandleMoveConfirm` 은 `// TODO: check movement cheat` 와 함께 `return true` 만 하고, `HandleStatsConfirm` 도 비어 있습니다.
  서버는 클라이언트가 어떤 상태까지 받았는지 추적하지 않습니다.

그나마 관련 있는 부분:

- **투사체 판정 여유**: `SpellMissile` 계열은 충돌 검사 범위를 이동량의 2배(`deltaMovement * 2`)로 잡습니다. 네트워크 지연이 아니라
  틱 사이에 대상을 건너뛰는 문제(터널링)를 막는 용도이며, 코드 리뷰 주석에도 서버 랙 시 문제가 된다고 적혀 있습니다.
- **시전 시간 정보**: `DesignerCastTime`, `ExtraTime` 은 클라이언트 모션을 맞추는 용도이고 서버 판정 시점은 바꾸지 않습니다.
- **클라이언트 보간**: 웨이포인트 기반 이동이라 원본 클라이언트가 경로를 부드럽게 보간합니다.

| 기법 | 구현 여부 |
| --- | --- |
| 서버 측 되감기 (rewind / 히트 판정 보정) | ✗ |
| RTT 측정 및 활용 | ✗ |
| 입력 타임스탬프 기반 처리 | ✗ |
| 클라이언트 예측 후 서버가 보정 | ✗ (서버 쪽 처리 없음, confirm 패킷 무시) |
| 게임 시계 동기화 | △ (10초마다 서버 시간을 보내기만 함) |

### 6.4 랙 보상이 없어도 문제가 잘 드러나지 않는 이유

두 가지 이유가 겹쳐 있습니다.

1. **실행 환경**: 설정 템플릿과 접속 예시가 모두 `127.0.0.1 5119` 로, 로컬/LAN 샌드박스를 전제로 합니다.
   로컬 RTT는 1ms 미만이라 서버 틱 간격(약 16.7ms)보다 짧고, 랙 보상 유무의 차이를 느낄 수 없습니다.
2. **MOBA 장르의 구조**: 레이턴시 크기와 무관하게 지연을 가려줍니다.
   - 스킬·평타에 0.2~0.5초 정도의 선딜이 있어, 그동안 패킷이 오갑니다.
   - 투사체 비행 시간이 있어 판정이 수십 ms 늦어도 묻힙니다(히트스캔 아님).
   - 목적지 클릭 방식 이동이라 클라이언트는 웨이포인트 보간만 하면 됩니다.
   - 원·직선 범위 판정이라 수십 ms의 위치 차이에 덜 민감합니다.

인터넷 환경(50~100ms)에서 운영하면 다음 문제가 실제로 생깁니다.

- **스킬샷 판정 불일치**: 내 화면은 RTT/2만큼 과거를 보여주므로, 피했는데 서버에서는 맞은 것으로 처리될 수 있습니다.
- **점멸·대시 반응 지연**: 입력이 서버에 반영되기까지 RTT만큼 걸리고, 그 사이 들어온 CC기가 먼저 적용됩니다.
- **막타·끊기 타이밍 어긋남**: 선딜 계산은 맞아도 시작 시점이 지연만큼 밀립니다.
- **핑에 따른 불공평**: 보정이 없으므로 핑이 낮은 쪽이 항상 유리합니다.

| 환경 | 영향 |
| --- | --- |
| 로컬/LAN (기본 환경) | 지연이 틱 간격보다 짧아 랙 보상 유무의 차이가 없습니다. |
| 인터넷 (50~100ms) | 장르 특성 덕분에 플레이는 가능하지만, 스킬샷 판정 불일치와 핑 차이로 인한 불공평이 드러납니다. |

원본 리그 오브 레전드도 FPS식 되감기는 거의 쓰지 않고, 지역별 서버로 핑을 낮게 유지하면서 선딜·투사체 같은 설계로 지연을 흡수합니다.
이 서버는 그 설계는 원본 데이터로 가져왔지만 핑을 낮게 유지하는 인프라는 없으므로, 원격 운영을 하지 않는다면 랙 보상 부재가 실제 문제가 될 일은 거의 없습니다.

---

## 7. 설정 파일

빌드 출력 폴더의 `Settings/` 에 생성됩니다 (템플릿: [GameServerConsole/Settings/](GameServerConsole/Settings/)).

**GameInfo.json** – 게임 구성

```jsonc
{
  "players": [{ "playerId": 1, "blowfishKey": "...", "name": "Test", "champion": "Ezreal",
                "team": "BLUE", "skin": 0, "summoner1": "SummonerHeal", "summoner2": "SummonerFlash",
                "runes": { ... }, "talents": { ... } }],
  "game":     { "map": 1, "gameMode": "CLASSIC", "dataPackage": "LeagueSandbox-Scripts" },
  "gameInfo": { "MANACOSTS_ENABLED": false, "COOLDOWNS_ENABLED": false, "CHEATS_ENABLED": true,
                "MINION_SPAWNS_ENABLED": false, "CONTENT_PATH": "../../../../../Content",
                "IS_DAMAGE_TEXT_GLOBAL": false },
  "forcedStart": 60   // 초. 이 시간이 지나면 로딩이 덜 끝나도 게임 시작
}
```

**GameServerSettings.json** – 디버그 실행 시 클라이언트 자동 실행

```json
{ "clientLocation": "C:\\LeagueSandbox\\League_Sandbox_Client", "autoStartClient": true }
```

명령줄 인자: `--config <경로>`, `--config-gameserver <경로>`, `--config-json <JSON 문자열>`, `--config-gameserver-json <JSON 문자열>`, `--port <번호>`

---

## 8. 채팅 명령어 (치트)

`CHEATS_ENABLED` 가 켜져 있으면 게임 채팅에서 `!` 로 시작하는 명령을 쓸 수 있습니다. `!help` 로 전체 목록 확인.

`ad`, `ap`, `ch`(챔피언 변경), `changeteam`, `clearslows`, `cooldownreduction`, `coords`, `debugmode`, `gold`, `health`,
`hotreload`, `inhib`, `junglespawn`, `kill`, `level`, `loadpackage`, `mana`, `mobs`, `model`, `packet`, `printpackagelist`,
`rainbow`, `reloadscripts`, `revive`, `size`, `skillpoints`, `spawn`, `spawnstate`, `speed`, `targetable`, `tp`, `xp` 등

---

## 9. 빌드 및 실행

```bash
git submodule update --init      # LeaguePackets, LeagueSandbox-Default 받기
dotnet build                      # .NET 6 SDK 필요
dotnet test                       # 단위 테스트
cd GameServerConsole/bin/Debug/net6.0
./GameServerConsole               # 포트 5119 에서 대기
```

- 서브모듈 원격 저장소가 내려갔을 수 있으므로 빌드가 안 되면 서브모듈 확보 여부를 먼저 확인하세요.
- 출력 폴더 위치에 따라 `GameInfo.json` 의 `CONTENT_PATH` 를 `Content` 폴더로 맞춰야 합니다.
- 클라이언트 수동 접속 (4.20 클라이언트의 `deploy` 폴더에서):

  ```bat
  "League of Legends.exe" "" "" "" "127.0.0.1 5119 17BLOhi6KZsTtldTsizvHg== 1"
  ```

- CI: [.github/workflows](.github/workflows) 에 Windows/Linux용 `dotnet restore → build → test` 워크플로가 있습니다.

---

## 라이선스

[AGPL-3.0](LICENSE). 이 코드를 수정해 사용하면 (네트워크 서버로 운영하는 경우 포함) 수정한 소스를 공개해야 합니다.
