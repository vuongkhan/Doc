# Quick Match — Luồng code
---

## 1. File / class chính

| Layer | Class / file | Vai trò |
|-------|----------------|---------|
| Hub UI | `UMultiplayerHubWidget` | Bấm Quick Match → mở màn QM |
| QM UI | `UQuickMatchWidget` | Start/cancel, bind event, UI, gọi showroom |
| Online | `UMatchServiceSubsystem` | RPC join/leave, roster, notify, handoff |
| Online | `UNakamaServiceSubsystem` | Session, realtime, RP profile |
| Loadout | `FPlayerRaceLoadoutUtil` | Capture garage → join payload |
| Showroom | `AQMShowroomDirector` | Pad + spawn `ACustomizableCar` |
| Car | `ACustomizableCar` | `ConfigCar`, `ShowInformation` (BP) |
| Server | `go-runtime/qm_rooms.go` | Room memory, join/leave, notify |
| Server | `go-runtime/main.go` | Handoff match, countdown opcode 4, Edgegap |

---

## 2. Entry: bấm Quick Match

```text
UMultiplayerHubWidget::HandleQuickMatchClicked
  → ShowQuickMatchScreen(bAutoStartSearch=true)
       · SetVisibility Hub Collapsed
       · CreateWidget / AddToViewport UQuickMatchWidget (nếu chưa)
       · QuickMatchWidget->SetVisibility(Visible)
       · QuickMatchWidget->StartQuickMatch()
```

### `UQuickMatchWidget::StartQuickMatch`

```text
StartQuickMatch
  → CacheSubsystems()          // Nakama + MatchService
  → BindSubsystemEvents()
  → UpdateLocalRpDisplay / UpdateLocalCarDisplay
  → bFlowActive = true, bMatchLocked = false
  → SetUiState(Searching, "SEARCHING...")
  → SetLevelOnlyView(false)    // giữ UI hiện
  → AQMShowroomDirector::FindOrCreate(this)
  → UMatchServiceSubsystem::JoinQuickMatchRoom()
```

---

## 3. Client join room

```text
UMatchServiceSubsystem::JoinQuickMatchRoom
  → ApplyPartySizeFromSettings / WaitingRoomMaxPlayers
  → FPlayerRaceLoadoutUtil::TryCaptureLocalLoadout
       → payload JSON { baseCarId, configJson }  (hoặc "{}")
  → BindQuickMatchWaitingNotifications()
       · RealtimeClient->NotificationReceived
       · → HandleQuickMatchWaitingNotifications
  → NakamaClient->RPC("prototype_qm_join", payload)
       │
       ├─ success:
       │    TryParseQmRoomPayload(Rpc.Payload)
       │    ApplyWaitingRoomRoster(Roster)
       │         LastWaitingRoomRoster = Roster
       │         OnWaitingRoomRosterReceived.Broadcast(Roster)
       │    if matched + matchId:
       │         OnAddMatchmakerSuccess
       │         JoinHandoffMatch(MatchId)
       │    else:
       │         bQuickMatchRoomActive = true
       │         OnAddMatchmakerSuccess.Broadcast()
       │
       └─ fail:
            OnAddMatchmakerError.Broadcast(...)
```

### UI nhận success / roster

```text
OnAddMatchmakerSuccess
  → UQuickMatchWidget::HandleAddMatchmakerSuccess
       SetUiState(Searching, "SEARCHING...")
       UpdatePlayerSlots / UpdateLocal*
       SyncShowroom(GetWaitingRoomRoster())

OnWaitingRoomRosterReceived
  → UQuickMatchWidget::HandleWaitingRoomRoster
       (nếu bMatchLocked && roster rỗng → giữ UI/cars, return)
       UpdateLocalRpDisplay / UpdateLocalCarDisplay
       UpdatePlayerSlots(n, max)   // "Looking for players n/4"
       SyncShowroom(Roster)        // nếu Roster.Num() > 0
       nếu n >= max → EnterMatchLockedPhase()  // Cancel off, cars keep
```

### `SyncShowroom`

```text
UQuickMatchWidget::SyncShowroom(Roster)
  → AQMShowroomDirector::FindOrCreate (nếu null)
  → AQMShowroomDirector::SyncSeats(Roster, LocalUserId)
```

---

## 4. Showroom spawn (per client)

```text
AQMShowroomDirector::SyncSeats(Seats, LocalUserId)
  → EnsurePadsReady / CachePlayerStarts   // tag 01..0N
  → LEAVE: user không còn trong roster
       Destroy car, FreeSeat, Active.Remove
  → LOCAL first:
       ClaimSeat(userId, bIsLocal=true) → seat index 0 (pad 01)
       SpawnOrUpdateAtSeat(Seat, LocalUserId, 0)
  → REMOTES:
       ClaimSeat(userId, false, Seat.SeatIndex) → pad 02+
       SpawnOrUpdateAtSeat(...)
```

### `SpawnOrUpdateAtSeat`

```text
SpawnOrUpdateAtSeat(FPlayerInfo Seat, LocalUserId, SeatIndex)
  → ResolveLoadout(Seat, bIsLocal) → BaseCarID, ConfigJson
  → JsonStringToCarConfiguration (nếu có JSON)
  → UCarCustomizationManager::GetCarInformation / GetCarMeshesFromConfig / GetCarDecalsFromConfig
  → Nếu đã có car cùng seat + same skin:
       ShowInformation(username, rp, bLocal, userId)   // refresh label only
       return
  → Nếu same seat, skin đổi:
       ApplyShowroomCarConfig → ConfigCar (BP)
       ShowInformation(...)
  → Else spawn:
       World->SpawnActor<ACustomizableCar>(CustomizeClass, pad transform)
       bUseMultiplayerConfigOnly = true
       ApplyShowroomCarConfig → ConfigCar
       ShowInformation(...)
       Active[UserId] = { Car, SeatIndex, BaseCarID, ConfigJson }
```

`ShowInformation` = `BlueprintImplementableEvent` trên `ACustomizableCar` (nameplate BP).

---

## 5. Server: `prototype_qm_join`

File: `Server/Nakama/go-runtime/qm_rooms.go`

```text
rpcQmJoin(ctx, payload)
  → requireAuthenticatedUser
  → lookupUsername / readAuthoritativeRP
  → parseQmJoinLoadout(payload)
  → resolveJoinSkin (storage skin ưu tiên, payload fallback)
  → lock qmRooms:
       · user đã trong room? update member skin → encodeQmRoomResponse return
       · findBestOpenRoomLocked(joiner, party)  // RP window
       · null → new room "qm_<nano>_<user8>"
       · append member, qmUserToRoom[user] = roomID
       · full := len(members) >= partySize
  → if full:
       startQmRoomMatch → Nakama match (handoff broker)
       notifyQmRoomRoster (220)
       notifyQmMatchStarted (221 + matchId)
       return response { roster, matched, matchId }
  → else:
       notifyQmRoomRoster (220)  // mọi member
       return response { roster, roomId, max }
```

### Leave

```text
rpcQmLeave
  → remove member / room
  → notifyQmRoomRoster remaining peers
```

### Expand (background)

```text
startQmRoomFillLoop (~2s)
  → tryQmRoomFill
       // chỉ SOLO có thể chuyển vào room mở khi RP window cho phép
       // không merge room 2+ với room 2+
```

---

## 6. Notify realtime (client)

```text
HandleQuickMatchWaitingNotifications(NotificationList)
  · code 220 / subject "qm_roster"
       TryParseQmRoomPayload
       ApplyWaitingRoomRoster → UI + showroom

  · code 221 / subject "qm_match_started"
       parse matchId
       bQuickMatchRoomActive = false
       ClearWaitingRoomRoster()     // có thể roster rỗng
       JoinHandoffMatch(MatchId)
```

UI khi roster rỗng + `bMatchLocked`: **không** `SyncSeats([])` (tránh destroy xe sớm).

---

## 7. Room full → handoff → countdown → travel

```text
JoinHandoffMatch(MatchId)
  → (MatchService) join Nakama match / presence
  → handoff state machine broadcasts:

UQuickMatchWidget::HandleMatched / HandleJoinSuccess
  → EnterMatchLockedPhase()   // Cancel ẩn, UI + cars giữ

UMatchServiceSubsystem  (match loop / broker)
  → opcode 4 countdown seconds
  → OnMatchCountdown.Broadcast(Seconds)

UQuickMatchWidget::HandleMatchCountdown(Seconds)
  → SetUiState(Countdown, "START SAU Ns...")
  → EnterMatchLockedPhase()

Server main.go (handoff match, all joined):
  CountdownSeconds = 10 (Edgegap) | 30 (local handoff)
  broadcastBrokerCountdown mỗi tick đến remaining == 0
  → start Edgegap / local allocate

UQuickMatchWidget::HandleHandoffState
  NakamaMatchReady / AllocationStarting → lock, keep cars
  JoinStarting / DedicatedServerJoinStarted
       → ClearShowroom()          // XE BIẾN MẤT TẠI ĐÂY
       → Traveling UI
  JoinSucceeded → ClearShowroom (an toàn)
  fail states → ClearShowroom + RequestExitToHub
```

---

## 8. Cancel / exit

```text
CancelButton / Esc (nếu !bMatchLocked)
  → HandleCancelClicked
  → CancelQuickMatch
       ClearShowroom
       LeaveQuickMatchRoom
            RPC prototype_qm_leave
            ClearWaitingRoomRoster
            OnCancelMatchmakerSuccess
  → HandleCancelSuccess
       RequestExitToHub
            OnRequestExitToHub
            UMultiplayerHubWidget::HandleQuickMatchExitToHub
                 QuickMatchWidget Collapsed
                 ShowHub()
```

`bMatchLocked == true` → Cancel/Back **no-op**.

---

## 10. Sơ đồ call 

```text
Hub.HandleQuickMatchClicked
  └─ ShowQuickMatchScreen(true)
       └─ QuickMatchWidget.StartQuickMatch
            └─ MatchService.JoinQuickMatchRoom
                 ├─ TryCaptureLocalLoadout
                 ├─ BindQuickMatchWaitingNotifications
                 └─ RPC prototype_qm_join ──────────────────────┐
                                                                ▼
                                              qm_rooms.go rpcQmJoin
                                                find/create room
                                                notify 220 roster
                                         full? notify 221 + matchId
                                                                │
            ◄──── RPC payload / notify 220 ─────────────────────┘
  OnWaitingRoomRosterReceived
    └─ HandleWaitingRoomRoster
         └─ SyncShowroom → QMShowroomDirector.SyncSeats
              └─ SpawnOrUpdateAtSeat → ConfigCar + ShowInformation

  OnAddMatchmakerSuccess → HandleAddMatchmakerSuccess (UI + SyncShowroom)

  ── room full ──
  notify 221 / join payload matched
    └─ JoinHandoffMatch
         └─ countdown opcode 4 → HandleMatchCountdown
         └─ allocate → HandleHandoffState / JoinDedicatedServer
              └─ JoinStarting → ClearShowroom → travel race
```

---

## 11. Path file (absolute workspace-relative)

| Path |
|------|
| `PrototypeRacing/Source/.../UI/Multiplayer/Screens/MultiplayerHubWidget.cpp` |
| `PrototypeRacing/Source/.../UI/Multiplayer/Screens/QuickMatchWidget.cpp` |
| `PrototypeRacing/Source/.../BackendSubsystem/Online/MatchServiceSubsystem.cpp` |
| `PrototypeRacing/Source/.../Multiplayer/QMShowroomDirector.cpp` |
| `PrototypeRacing/Source/.../CarCustomizationSystem/CustomizableCar.h` |
| `PrototypeRacing/Source/.../Multiplayer/PlayerRaceLoadout.*` |
| `PrototypeRacing/Server/Nakama/go-runtime/qm_rooms.go` |
| `PrototypeRacing/Server/Nakama/go-runtime/main.go` |

---
