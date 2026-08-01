# Logic: 4 Task Multiplayer (Token → Skin → Grid → AI thay thế)

> Tài liệu tóm tắt **logic ngắn gọn** cho 4 tính năng. Chi tiết trace từng dòng xem `Multiplayer_4Tasks_FlowReview.md`.

---

## 1. Nhận diện player hợp lệ (token Nakama + client khớp)

**Mục đích:** Chỉ player có trong trận (roster + token đúng) mới được vào server.

```text
[Broker Nakama] sinh mỗi player 1 join token (32B base64url)  →  main.go
[Client] nhận token qua MatchData → build URL ?JoinToken&PlayerId&ClientProtocol=1
[Server] PreLogin parse credentials từ Options                  → PlayerConnectionValidator
   validate theo thứ tự:
     1. Protocol  (ClientProtocol = "1")
     2. Roster    (PlayerId có trong danh sách trận)
     3. Per-player token (token khớp PlayerId)
     4. Shared token     (nếu không có token riêng)
[Sai]   → ErrorMessage (không lộ secret) → từ chối
[Đúng]  → PostLogin → cấp seat → spawn xe
```

**Nguồn:** `PlayerConnectionValidator.cpp:217-344`, `RacingCarGameMode.cpp:34-67`. Broker inject env kỳ vọng (`PR_PLAYER_JOIN_TOKENS`, `PR_PLAYER_ROSTER`). Chỉ enforce trên `NM_DedicatedServer`.

---

## 2. Đẩy skin mỗi player vào map lúc start race

**Mục đích:** Mọi client thấy đúng skin mỗi player từ snapshot chung, không đọc garage local của nhau.

```text
[Client] đọc garage LOCAL của mình → serialize JSON → ServerSubmitRaceLoadout (RPC)
[Server] validate + upsert vào PlayerLoadouts (ARaceGameState, replicated)
[Mọi client] OnRep_PlayerLoadouts → ApplyOnlineLoadoutToCar → ConfigCarOnlineVisual
[Sweep cuối] start race → apply lại toàn bộ xe (đỡ trường hợp loadout đến trễ)
```

**Nguồn sự thật duy nhất:** `PlayerLoadouts` trên `ARaceGameState` (replicate `OnRep_PlayerLoadouts`). Client submit ngay khi có `VehicleId` (`OnRep_VehicleId` / `OnPossess`).

---

## 3. Spawn xe player vào grid thay vì AI offline

**Mục đích:** Online, mỗi player chiếm 1 `PlayerStart` theo seat; không spawn AI trước.

```text
InitGame: collect + sort PlayerStarts theo tag 01,02,...
PostLogin: TryAllocatePlayerGridSlot → SeatIndex (reclaim nếu reconnect, còn không lấy Empty)
   Transform = PlayerStarts[SeatIndex]
   SetVehicleId("Player%02d") → trigger register + loadout
   SpawnPlayerCar(class từ replicated loadout, bIsAICar=false)
   AddCarIntoList(car, SeatIndex)
```

**Offline vẫn giữ AI:** `SetupAICar` spawn `PlayerStarts.Num()-1` AI (`bIsAICar=true`, `VehicleId="AICAR%d"`).

---

## 4. AI thay thế player không kết nối (~5s)

**Mục đích:** Tránh seat trống / trận thiếu người.

### Hai nguồn mất player

| Loại | Trigger | Xử lý hiện tại |
|---|---|---|
| **Disconnect** giữa trận | `Logout` (socket đóng tức thì / `ConnectionTimeout=5s`) | Unpossess, giữ xe → log đỏ 5s |
| **No-show** không vào trận | `StartOnlineWaitTimerIfNeeded` timeout | mở gate race start, seat trống |

### Luồng log màn hình (đã có)

```text
Server phát hiện mất player (disconnect/no-show)
   → SignalPlayerDisconnected(SeatIndex, AccountName)  (NetMulticast)
   → debug timer 5s → DebugLogPlayerDisconnect
   → in màn hình 5 giây: "Player n:username disconnect, AI will replace this player"
```

**Nguồn:** `RaceTrackManager.cpp:1847-1872` (`SignalPlayerDisconnected`, `DebugLogPlayerDisconnect`), `RacingCarGameMode.cpp:940-1019` (`Logout`), `DefaultEngine.ini:512` (`ConnectionTimeout=5.0`).



## Tóm tắt mối liên hệ

```text
Player vào → (1) validate token → (2) submit skin → (3) seat + spawn xe grid
Player mất → (4) log sau 5s
```

**File chính:** `PlayerConnectionValidator.cpp` (gate), `RacingCarGameMode.cpp` (seat/spawn/Logout), `RaceGameState.cpp` (skin replicated), `RaceTrackManager.cpp` (AI + disconnect signal), `RacingCarController.cpp` (RPC loadout), `MultiplayerWaitingRoomGameMode.cpp` (no-show).
