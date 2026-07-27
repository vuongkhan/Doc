# Lobby Flow

## Kiến trúc tổng quan

Client (Unreal) ←RPC→ Nakama Server (Go runtime) ←API→ Edgegap (dedicated server host)

Storage: Nakama Storage cho lobby records + match records.

---

## Test accounts

**File:** `Server/Nakama/go-runtime/main.go:71-84`

```go
var testerSeedEmails = []string{
    "doluan590@gmail.com",
    "huynhdongdn97@gmail.com",
    "loginson140521@gmail.com",
    "nguyencducthinh@dtu.edu.vn",
    "nguyenchanhtong@dtu.edu.vn",
    ...
}
```

**Được tạo khi nào?** — `InitModule()` gọi `seedTesterEmailAccounts()` (dòng 363). Chạy mỗi lần Nakama server start.

**Làm gì?** — `nk.AuthenticateEmail()` cho mỗi email (idempotent — nếu tài khoản đã tồn tại thì skip). Sau đó `seedTesterFriendships()` tạo friend graph toàn bộ (mỗi account là friend của mọi account khác).

**Để làm gì?** — Dev/QA có sẵn tài khoản + bạn bè để test invite lobby mà không cần tạo thủ công.

**Password chung:** `123456789` (dòng 65)

**Tắt:** env var `PR_SEED_TESTER_ACCOUNTS=false`

---

## Party size (số player mỗi match)

### Source of truth — client side

**`MultiplayerSessionSettings.h:23`** — Project Settings → Multiplayer Session

```cpp
int32 DefaultPartySize = 2;  // ClampMin=2, ClampMax=4
```

**`MultiplayerSessionSettings.cpp:8-14`**

```cpp
int32 UMultiplayerSessionSettings::GetDefaultPartySize()
{
    const int32 Size = Settings ? Settings->DefaultPartySize : 2;
    const int32 AbsMax = Settings ? Settings->AbsoluteMaxPlayers : 4;
    return FMath::Clamp(Size, 2, AbsMax);  // 2..4
}
```

### Source of truth — server side

**`main.go:48`**

```go
defaultExpectedPlayers = 2
```

Override bằng env: `PR_EXPECTED_PLAYERS=4` (`main.go:501`)

**`main.go:519-528`** — `partySizeBounds()`:

```go
func partySizeBounds() (minPlayers, maxPlayers int) {
    n := brokerCfg.ExpectedPlayers
    if n < 2 { n = defaultExpectedPlayers }
    if n > 4 { n = 4 }
    return n, n  // min = max (fixed size)
}
```

### Lobby server side

**`lobby.go:32-37`**

```go
func lobbyPartySize() int {
    minP, maxP := partySizeBounds()
    return maxP  // reuse Quick Match party size
}
```

### Lobby client side

**`LobbyServiceSubsystem.cpp:56-59`**

```cpp
const int32 Need = CurrentLobby.MaxSize > 0
    ? CurrentLobby.MaxSize
    : UMultiplayerSessionSettings::GetDefaultPartySize();
```

---

## Luồng Private Lobby

### 1. Create Lobby

```
User click "LOBBY" → MultiplayerHubWidget::HandleCreateLobbyClicked()
  → LobbyServiceSubsystem::CreateLobby()         [LobbyServiceSubsystem.cpp:143]
    → RPC: prototype_create_lobby ({})

Nakama Server: rpcCreateLobby()                    [lobby.go:79]
  → tạo lobbyRecord với:
      LobbyID = 6-char random (ABCDEFGHJKLMNPQRSTUVWXYZ23456789, dòng 23)
      MaxSize = lobbyPartySize() (default 2)
      Members = [host]
  → lưu vào Nakama Storage (collection: prototype_racing_lobbies)
  → trả về lobby + role="host"

Client: ApplyLobbyFromRpcPayload()                 [LobbyServiceSubsystem.cpp:434]
  → parse FLobbySnapshot → broadcast OnLobbySnapshotUpdated
  → UI switch sang LobbyWidget
```

### 2. Invite bạn

```
Host click invite → LobbyServiceSubsystem::InviteFriend()  [line 271]
  → RPC: prototype_invite_lobby ({lobbyId, targetUserId})

Nakama Server: rpcInviteLobby()                    [lobby.go:231]
  → kiểm tra host
  → nk.NotificationSend() với code 201 + type="lobby_invite"

Client (bạn): HandleNotificationList()             [LobbyServiceSubsystem.cpp:550]
  → tự động JoinLobbyById() (nếu chưa ở lobby nào)
```

### 3. Join Lobby

```
Bạn nhập code hoặc auto-join từ invite → LobbyServiceSubsystem::JoinLobbyById() [line 169]
  → RPC: prototype_join_lobby ({lobbyId})

Nakama Server: rpcJoinLobby()                      [lobby.go:121]
  → kiểm tra lobby tồn tại + chưa đầy (len(Members) < MaxSize)
  → thêm member → lưu
  → notify toàn bộ members code 202 (lobby_roster)

Client: HandleNotificationList()                   [LobbyServiceSubsystem.cpp:574]
  → parse lobby snapshot → broadcast OnLobbySnapshotUpdated
```

### 4. Start Match (host)

```
Host click "Start" (chỉ enabled khi đủ member) → LobbyServiceSubsystem::StartLobbyMatch() [line 309]
  → kiểm tra CanStartLobbyMatch():  [line 50]
      IsInLobby() && IsLocalHost() && MemberCount >= MaxSize
  → RPC: prototype_start_lobby ({lobbyId, mapId})

Nakama Server: rpcStartLobby()                      [lobby.go:277]
  → kiểm tra host + lobby full
  → createHandoffMatch()  →  giống Quick Match flow
  → notify members code 204 (lobby_match_started)
  → xóa lobby record

Client (host): nhận matchId từ RPC response → BeginHandoffMatch() [line 344]
Client (guest): nhận notification → tự động BeginHandoffMatch() [line 560]

BeginHandoffMatch():
  → clear lobby local
  → broadcast OnLobbyMatchStarted → UI chuyển sang matchmaking countdown
  → MatchServiceSubsystem::JoinHandoffMatch()       [MatchServiceSubsystem.cpp:370]
```

### 5. Edgegap handoff (== Quick Match)

Từ đây hoàn toàn giống Quick Match:

```
MatchServiceSubsystem::JoinHandoffMatch()           [MatchServiceSubsystem.cpp:370]
  → JoinMatch() → client join Nakama realtime match

Nakama handoffMatch.MatchLoop()                     [main.go:683]
  → đợi JoinedPresences == ExpectedPlayers
  → allocateDedicatedServer()
      ├─ [local] → allocateLocalHandoff() → 127.0.0.1:7777
      └─ [Edgegap] → POST /v2/deployments → poll cho Ready → host:port
  → broadcast opcode=1 (brokerSuccessPayload: host, port, joinToken, map)

Client MatchDataReceived(opcode=1)                  [MatchServiceSubsystem.cpp:1168]
  → StartDedicatedServerHandoff()
  → ClientTravel đến dedicated server

DedicatedServer MultiplayerWaitingRoomGameMode      [WaitingRoomGameMode.cpp:298]
  → server đọc PR_EXPECTED_PLAYERS từ env
  → đợi đủ player connect (GetConnectedPlayerCount == ExpectedPlayers)
  → ServerTravel vào TargetMap
  → RACE!
```

---

## File map

| File | Vai trò |
|---|---|
| `Server/Nakama/go-runtime/main.go` | Server: matchmaker, handoff match, Edgegap API, test accounts seed, party size |
| `Server/Nakama/go-runtime/lobby.go` | Server: lobby CRUD (create/join/leave/close/invite/start), storage + notifications |
| `Source/.../MultiplayerSessionSettings.h/cpp` | Client: DefaultPartySize + validation |
| `Source/.../LobbyServiceSubsystem.h/cpp` | Client: lobby RPC calls, snapshot parsing, notification handling |
| `Source/.../LobbyTypes.h` | Client: FLobbySnapshot, FLobbyRosterEntry, FLobbyFriendRow, enums |
| `Source/.../MatchServiceSubsystem.h/cpp` | Client: matchmaking, Nakama match data, Edgegap handoff, ClientTravel |
| `Source/.../MultiplayerHubWidget.h/cpp` | UI Hub: Quick Match, Create/Join Lobby buttons |
| `Source/.../MultiplayerLobbyWidget.h/cpp` | UI Lobby: roster, invite list, Start button |
| `Source/.../MultiplayerMatchmakingWidget.h/cpp` | UI Matchmaking: finding, countdown, race session |
| `Source/.../MultiplayerWaitingRoomGameMode.cpp` | Dedicated server: đợi đủ player → ServerTravel target map |
| `Source/PrototypeRacing.h` | Shared: env var key definitions |
| `Build/Edgegap/publish_edgegap_server.sh` | Deploy script: Edgegap app version + env defaults |
| `Server/Nakama/LOCAL_DEVELOPMENT.md` | Docs: local stack setup |

---

## Enum `ELobbySessionState`

```
Idle → Creating → InLobby
Idle → Joining → InLobby
InLobby → Leaving → Idle
InLobby → (host close) → Idle
```

## Nakama OpCodes (Match Data)

| OpCode | Ý nghĩa | Code |
|---|---|---|
| 0 | Player joined match | `main.go` brokerSuccessOpCode = 1 |
| 1 (brokerSuccessOpCode) | Edgegap allocation OK → host:port | `main.go:36` |
| 2 (brokerFailureOpCode) | Allocation failed / cancelled | `main.go:37` |
| 3 (brokerCompleteOpCode) | Match completed (MVP timeout) | `main.go:38` |

## Lobby Notification Codes

| Code | Type | Ý nghĩa |
|---|---|---|
| 201 | lobby_invite | Host mời bạn |
| 202 | lobby_roster | Roster thay đổi (có người vào/ra) |
| 203 | lobby_closed | Host close lobby |
| 204 | lobby_match_started | Trận bắt đầu → join handoff match |
