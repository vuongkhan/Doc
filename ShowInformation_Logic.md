# ShowInformation — logic ngắn

Nameplate / info plate trên xe waiting-room (`AMultiplayerCarShowroom`). **Không** nằm trên `ACustomizableCar` (garage).

## Ai gọi

`AQMShowroomDirector::ApplySeatPlayerInfo` sau khi spawn/update seat.

Nguồn data: `FPlayerInfo` (roster QM / lobby) + car DT.

```
roster seat (FPlayerInfo)
        │
        ▼
ResolveSeatCarDisplayInfo  →  BaseCarID, CarName, Brand, CarType
        │                       (loadout JSON / BaseCarID → CarCustomizationManager::GetCarInformation)
        ▼
ShowInformation(...)
        │
        ├─ C++ Implementation  →  ghi UPROPERTY Showroom*
        └─ BP override (nếu có) →  set Text / style / visibility
        │
        ▼
next tick: gọi lại 1 lần (ConfigCar rebuild widget)
```

## Input → cache C++

| Param | Nguồn | Property |
|---|---|---|
| `DisplayName` | `Username`, fallback `UserId` 8 ký tự, rồi `"Player"` | `ShowroomDisplayName` |
| `RacePoint` | `Info.RacePoint` | `ShowroomRacePoint` |
| `bIsLocalPlayer` | so sánh `UserId` local | `bShowroomIsLocalPlayer` |
| `UserId` | roster | `ShowroomUserId` |
| `CarRating` | roster / Nakama | `ShowroomCarRating` |
| `PlayerRank` | RP leaderboard (1 = cao nhất, **0 = unknown**) | `ShowroomPlayerRank` |
| `RpTotal` | tổng người trên bảng RP | `ShowroomRpTotal` |
| `bIsFriend` | server stamp theo viewer | `bShowroomIsFriend` |
| `BaseCarID` + `CarName` / `BrandName` / `CarTypeName` | DT + tag `Car.Brand` | `Showroom*` tương ứng |

C++ **chỉ cache**. UI text/style: **Blueprint** đọc `Showroom*` (local đổi style, mọi seat đều hiện).

## Rank

Không fetch trong `ShowInformation`. Rank đi sẵn trên roster (`FPlayerInfo.PlayerRank`). Fetch riêng qua `UNakamaServiceSubsystem::RequestPlayerRank` / `GetPlayerRank` nếu roster = 0.

## BP bind

Event `Show Information` (NativeEvent): bind text Name, RP (`RacePoint`), rank (`PlayerRank` / `RpTotal`), rating, tên xe / brand / type. `bIsLocalPlayer` / `bIsFriend` chỉ đổi style, không ẩn plate.
