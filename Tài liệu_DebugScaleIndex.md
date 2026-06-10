# Debug Scale Index Override

## Mục Tiêu

Tính năng `Scale Index Override` dùng để QC/dev test nhanh các công thức progression phụ thuộc vào `ScaleIndex` mà không cần đổi progression thật của player.

Debug chỉ thay đổi giá trị `ScaleIndex` trả về trong non-shipping build. Current city, progression save, city unlock, track progress và dữ liệu progression thật không bị đổi.

## Kết Quả Cuối Cùng

Trong debug cheat có nhóm `Scale Index` gồm:

```text
ScaleIndex
Override: ON/OFF
Set ScaleIndex
Reset
```

Hành vi cuối cùng:

- Dropdown hiển thị danh sách city và scale index tương ứng.
- Khi override OFF, dropdown tự chọn theo current city thật.
- Khi override ON, dropdown chọn theo city đang được dùng để override.
- Bấm `Set ScaleIndex` sẽ bật override bằng city đang chọn.
- Bấm `Reset` sẽ tắt override và đưa selection về current city thật.
- Text `Override: ON - City X / Y.YY` giúp user biết runtime đang bị override.

## Nguyên Tắc Quan Trọng

Scale index luôn được suy ra từ city bằng progression subsystem:

```cpp
ProgressionSubsystem->GetBaseScaleIndexByCityIndex(CityIndex)
```

Runtime vẫn gọi hàm chính:

```cpp
ProgressionSubsystem->GetScaleIndexByCityIndex(CityIndex)
```

Trong non-shipping build, hàm này sẽ hỏi `UProgressionDebugManager` trước. Nếu debug override đang bật thì trả về scale index của city override. Nếu không bật override thì trả về scale index thật theo city truyền vào.

## Dữ Liệu Được Lưu

Debug save chỉ lưu trạng thái override và city index được chọn:

```cpp
bool bHasScaleIndexOverride
int32 SelectedDebugScaleIndexCityIndex
```

Không lưu float `ScaleIndex`.

Lý do: scale index có thể suy ra từ city bằng `GetBaseScaleIndexByCityIndex()`, nên lưu thêm float sẽ dễ lệch dữ liệu khi công thức hoặc progression data thay đổi.

## Flow Build UI

Entry point:

```cpp
UDebugModule_Cheat::GetEntries()
```

Flow:

```text
GetEntries()
-> lấy UProgressionDebugManager
-> lấy UProgressionSubsystem
-> nếu override ON:
      PendingScaleIndexCityIndex = LoadSelectedDebugScaleIndexCityIndex()
-> nếu override OFF:
      PendingScaleIndexCityIndex = ProgressionSubsystem->GetCurrentCityPosition()
-> build dropdown ScaleIndex
-> build text trạng thái override
-> build button Set ScaleIndex
-> build button Reset
```

Điểm quan trọng: khi override OFF, dropdown không lấy city đã save trước đó. Nó hiển thị theo current city thật để khớp với runtime hiện tại.

## Flow Tạo Option Dropdown

Hàm:

```cpp
BuildScaleIndexDropdownOptions(...)
```

Flow:

```text
BuildScaleIndexDropdownOptions()
-> duyệt City 1 đến City 5
-> gọi ProgressionSubsystem->GetBaseScaleIndexByCityIndex(CityIndex)
-> tạo label "City X - ScaleIndex"
-> nếu CityIndex == PendingScaleIndexCityIndex thì thêm "(Current)"
```

Ví dụ:

```text
City 1 - 1.00
City 2 - 3.25
City 3 - 7.00 (Current)
City 4 - 13.33
City 5 - 26
```

Dropdown chỉ dùng công thức progression để hiển thị.

## Flow Chọn Dropdown

Khi user chọn city trong dropdown:

```text
HandleCarCREntry()
-> EntryId == CheatCarCRScaleIndexCity
-> PendingScaleIndexCityIndex = selected city
-> SaveSelectedDebugScaleIndexCityIndex(PendingScaleIndexCityIndex)
```

Việc chọn dropdown chỉ lưu option đang chọn. Nó chưa bật override.

Override chỉ bật khi user bấm `Set ScaleIndex`.

## Flow Bấm Set ScaleIndex

Khi user bấm `Set ScaleIndex`:

```text
HandleCarCREntry()
-> EntryId == CheatCarCRScaleIndexSet
-> SelectedCityIndex = PendingScaleIndexCityIndex
-> validate ProgressionSubsystem->GetBaseScaleIndexByCityIndex(SelectedCityIndex) > 0
-> enqueue/apply Manager->DebugSetScaleIndexCity(SelectedCityIndex)
```

Trong manager:

```text
DebugSetScaleIndexCity()
-> clamp selected city
-> validate GetBaseScaleIndexByCityIndex(selected city) > 0
-> SelectedDebugScaleIndexCityIndex = selected city
-> bHasDebugScaleIndexOverride = true
-> SaveScaleIndexToSave(true, SelectedDebugScaleIndexCityIndex)
```

Sau bước này, mọi flow gọi `GetScaleIndexByCityIndex()` trong non-shipping build sẽ nhận scale index override.

## Flow Bấm Reset

Khi user bấm `Reset`:

```text
HandleCarCREntry()
-> lấy current city từ ProgressionSubsystem
-> PendingScaleIndexCityIndex = current city
-> SaveSelectedDebugScaleIndexCityIndex(current city)
-> Manager->DebugResetScaleIndexToCurrentCity()
```

Trong manager:

```text
DebugResetScaleIndexToCurrentCity()
-> SelectedDebugScaleIndexCityIndex = current city
-> bHasDebugScaleIndexOverride = false
-> SaveScaleIndexToSave(false, current city)
```

Sau reset, runtime quay lại dùng scale index thật theo current city.

## Flow Runtime Lấy Scale Index

Hàm runtime:

```cpp
UProgressionSubsystem::GetScaleIndexByCityIndex(int CityIndex) const
```

Flow:

```text
GetScaleIndexByCityIndex(CityIndex)
-> nếu non-shipping:
      hỏi ProgressionDebugManager->TryResolveDebugScaleIndexOverride(...)
      nếu override hợp lệ thì return DebugScaleIndex
-> nếu không override:
      return GetBaseScaleIndexByCityIndex(CityIndex)
```

Hàm base:

```cpp
UProgressionSubsystem::GetBaseScaleIndexByCityIndex(int CityIndex) const
```

Flow:

```text
TargetRaceCount = GetTargetRaceCountByCityIndex(CityIndex)
BaseCityTargetRaceCount = GetTargetRaceCountByCityIndex(0)
ScaleIndex = TargetRaceCount / BaseCityTargetRaceCount * (CityIndex + 1)
```

Lưu ý implementation phải ép kiểu float trước khi chia để tránh integer division.

## Ý Nghĩa Override

Ví dụ:

```text
Current City = City 4
Override = City 3
```

Khi đó:

```text
Player vẫn ở City 4.
Progression thật vẫn là City 4.
Các công thức hỏi GetScaleIndexByCityIndex() sẽ nhận ScaleIndex của City 3.
```

Đây là hành vi đúng của debug override. Nó cho phép test tác động của scale index khác mà không cần nhảy city hoặc sửa save thật.

## Flow Upgrade Cost

Upgrade cost hiện căn cứ theo current city thật:

```text
GetCostForNextUpgradePerformance()
-> CurrentCity = ProgressionSubsystem->GetCurrentCityPosition()
-> ScaleIndex = ProgressionSubsystem->GetScaleIndexByCityIndex(CurrentCity)
-> BasePrice = RoundCashUpToMultiple(UpgradeBasePrice * ScaleIndex)
-> Cost = BasePrice * NextLevel
```

Nếu override đang bật, `GetScaleIndexByCityIndex(CurrentCity)` sẽ trả về scale index override. Vì vậy cost upgrade vẫn chạy trong context current city, nhưng dùng scale index debug đã chọn.

## Flow EXPECTED Cash

Trong progression debug, dòng `EXPECTED` dùng:

```text
GetExpectedMinimumUpgradeCash()
-> CarCustomizationManager->GetRequiredCashForLevel(3, CurrentCity.CityIndex)
-> ProgressionSubsystem->GetScaleIndexByCityIndex(CurrentCity)
```

Vì đi qua `GetScaleIndexByCityIndex()`, `EXPECTED` cũng bị ảnh hưởng khi override đang bật.

## File Liên Quan

```text
PrototypeRacing/Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Cheat.cpp
PrototypeRacing/Source/PrototypeRacing/Public/DebugSystem/Modules/DebugModule_Cheat.h
PrototypeRacing/Source/PrototypeRacing/Public/DebugSystem/ProgressionDebugManager.h
PrototypeRacing/Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp
PrototypeRacing/Source/PrototypeRacing/Public/DebugSystem/DebugSaveGame.h
PrototypeRacing/Source/PrototypeRacing/Public/BackendSubsystem/Progression/ProgressionSubsystem.h
PrototypeRacing/Source/PrototypeRacing/Private/BackendSubsystem/Progression/ProgressionSubsystem.cpp
PrototypeRacing/Source/PrototypeRacing/Private/CarCustomizationSystem/CarCustomizationManager.cpp
PrototypeRacing/Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Progression.cpp
```

## Tóm Tắt

```text
Debug UI chọn city index
-> Dropdown hiển thị scale index bằng GetBaseScaleIndexByCityIndex()
-> Override OFF thì dropdown bám current city thật
-> Override ON thì dropdown bám selected override city
-> Bấm Set ScaleIndex thì bật override
-> Runtime GetScaleIndexByCityIndex() ưu tiên override trong non-shipping
-> Upgrade cost / EXPECTED cash / các flow dùng ScaleIndex nhận giá trị override
-> Bấm Reset thì tắt override và quay lại current city thật
```
