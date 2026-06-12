# Debug System Feature Guide

Tài liệu này tóm tắt toàn bộ flow debug hiện tại và các điểm code cần xem khi thêm UI mới, nút mới, module mới hoặc tính năng debug mới.

## Phạm vi build

Debug system chỉ chạy trong non-shipping build.

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Private/DebugSystem/DebugToolsSubsystem.cpp
bool UDebugToolsSubsystem::ShouldCreateSubsystem(UObject* Outer) const
{
#if (!UE_BUILD_SHIPPING)
	return true;
#else
	return false;
#endif
}
```

Lưu ý:

- Không assume debug có trong Shipping.
- Tính năng mới nên bọc bằng `#if !UE_BUILD_SHIPPING` hoặc `#if (!UE_BUILD_SHIPPING)`.
- Nếu thêm code gọi debug từ gameplay, phải xử lý trường hợp subsystem trả về `nullptr`.

## Kiến trúc tổng quan

Flow chính:

```text
Touch gesture / menu / Blueprint
-> UDebugToolsSubsystem::RequestToggleDebugPanel()
-> EnsureDebugPanelWidget()
-> WBP_DebugPanel
-> UDebugPanelWidget::RefreshFromRegistry()
-> UDebugToolsSubsystem::GetCategories()
-> UDebugToolsSubsystem::GetEntriesForCategory(Category)
-> UDebugModuleBase::GetEntries()
-> Blueprint build UI theo FDebugEntry.Type
-> user bấm / kéo / chọn
-> UDebugToolsSubsystem::ExecuteEntry(Category, EntryId, Value, Key, Toggle)
-> UDebugModuleBase::ExecuteEntry()
-> subsystem/gameplay thật được gọi
```

Các file lõi:

- `Source/PrototypeRacing/Public/DebugSystem/DebugSystemTypes.h`: định nghĩa `EDebugEntryType`, `FDebugEntry`, `FProgressionBox`, `FProgressionGroup`.
- `Source/PrototypeRacing/Public/DebugSystem/DebugModuleBase.h`: interface bắt buộc của mọi debug module.
- `Source/PrototypeRacing/Public/DebugSystem/DebugToolsSubsystem.h`
- `Source/PrototypeRacing/Private/DebugSystem/DebugToolsSubsystem.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/DebugPanelWidget.h`
- `Source/PrototypeRacing/Private/DebugSystem/DebugPanelWidget.cpp`
- Blueprint UI chính: `/Game/UI/Debug/WBP_DebugPanel.WBP_DebugPanel_C`

## Entry type đang hỗ trợ

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Public/DebugSystem/DebugSystemTypes.h
enum class EDebugEntryType : uint8
{
	Button,
	Slider,
	Toggle,
	Dropdown,
	CheckBox,
	Text,
	ProgressionBox,
	Nested,
	Pending,
	Log,
};
```

Ý nghĩa:

- `Button`: bấm một lần để chạy action.
- `Slider`: kéo giá trị float.
- `Toggle`: bật/tắt bằng bool.
- `Dropdown`: chọn một item, thường trả về `ID` qua `Value` hoặc command qua `Key`.
- `CheckBox`: bool giống toggle, đang dùng trong cheat reward.
- `Text`: chỉ hiển thị, không nên có side effect khi render.
- `ProgressionBox`: nhóm nhiều dòng debug dạng box.
- `Nested`: nhóm nhiều `FProgressionGroup`, dùng nhiều trong Cheat.
- `Pending`: bảng pending change của cheat.
- `Log`: bảng log change đã confirm.

Lưu ý khi thêm type mới:

- Thêm enum trong `DebugSystemTypes.h`.
- Thêm widget class hoặc logic render trong `WBP_DebugPanel`.
- Thêm property class nếu cần ở `UDebugPanelWidget`.
- Cập nhật các Blueprint entry widget để gọi đúng `ExecuteEntry`.

## Flow mở debug panel

Flow gesture:

```text
APrototypeRacingPlayerControllerBase forward touch
-> UDebugGestureSubsystem::ReportTouch()
-> đủ 3 ngón tay, 2 lần trong 1 giây
-> nếu race không đang Running
-> UDebugToolsSubsystem::RequestToggleDebugPanel()
```

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Private/DebugSystem/DebugGestureSubsystem.cpp
static constexpr int32 RequiredTouchCount = 3;
static constexpr int32 RequiredGestureCount = 2;
static constexpr double GestureWindowSeconds = 1.0;
```

```cpp
// Source/PrototypeRacing/Private/DebugSystem/DebugToolsSubsystem.cpp
void UDebugToolsSubsystem::OpenDebugPanel()
{
	PC->SetPause(true);
	DebugPanelWidget->AddToViewport(15);
	PC->bShowMouseCursor = true;
	DebugPanelWidget->SyncDebugPresetFromCurrentProfileOnPanelOpen();
	DebugPanelWidget->RefreshDisplayedCategoryOnOpen();
}
```

Lưu ý:

- Khi close bằng nút X trong UI, phải gọi `UDebugPanelWidget::CloseDebugPanelFromUI()`, không chỉ hide widget trong Blueprint.
- `OpenDebugPanel()` pause game và bật mouse cursor.
- `CloseDebugPanel()` unpause game nhưng hiện tại không reset `bShowMouseCursor`; nếu UI cần cursor state chuẩn, kiểm tra thêm chỗ này.

## Flow register module

Flow:

```text
UDebugToolsSubsystem::Initialize()
-> GetDerivedClasses(UDebugModuleBase::StaticClass())
-> NewObject từng class con không abstract
-> RegisterModule(Module)
-> CategoryRegistry[Module->GetCategoryName()] = Module
```

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Private/DebugSystem/DebugToolsSubsystem.cpp
GetDerivedClasses(UDebugModuleBase::StaticClass(), DerivedClasses);
for (UClass* Class : DerivedClasses)
{
	if (!Class->HasAnyClassFlags(CLASS_Abstract))
	{
		UDebugModuleBase* Module = NewObject<UDebugModuleBase>(this, Class);
		RegisterModule(Module);
	}
}
```

Checklist khi thêm module mới:

- Tạo `.h/.cpp` kế thừa `UDebugModuleBase`.
- Class không được `Abstract`.
- Implement đủ `GetCategoryName`, `GetCategoryDisplayName`, `GetEntries`, `ExecuteEntry`, `GetCurrentValues`.
- Category name phải unique.
- Build non-shipping để module tự được register.

Template ngắn:

```cpp
FName UDebugModule_NewFeature::GetCategoryName() const
{
#if !UE_BUILD_SHIPPING
	return FName(TEXT("NewFeature"));
#else
	return NAME_None;
#endif
}

TArray<FDebugEntry> UDebugModule_NewFeature::GetEntries() const
{
	TArray<FDebugEntry> Entries;
#if !UE_BUILD_SHIPPING
	FDebugEntry Entry;
	Entry.Id = FName(TEXT("DoSomething"));
	Entry.DisplayName = FText::FromString(TEXT("Do Something"));
	Entry.Type = EDebugEntryType::Button;
	Entries.Add(Entry);
#endif
	return Entries;
}

void UDebugModule_NewFeature::ExecuteEntry(FName EntryId, const FVariant& Value, const FString& Key, const bool Toggle)
{
#if !UE_BUILD_SHIPPING
	if (EntryId == FName(TEXT("DoSomething")))
	{
		// Gọi subsystem/gameplay thật tại đây.
	}
#endif
}
```

## Flow render UI

Flow C++ sang Blueprint:

```text
UDebugPanelWidget::NativeConstruct()
-> BindToSubsystem()
-> RefreshFromRegistry()
-> BP_OnRefreshFromRegistry(CategoryNames)
-> user chọn category
-> PopulateCategory(Category)
-> BP_OnPopulateCategory(Category, Entries)
```

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Private/DebugSystem/DebugPanelWidget.cpp
void UDebugPanelWidget::PopulateCategory(FName InCategoryName)
{
	CurrentCategory = InCategoryName;
	TArray<FDebugEntry> Entries = DebugToolsSubsystem->GetEntriesForCategory(InCategoryName);
	BP_OnPopulateCategory(InCategoryName, Entries);
}
```

Blueprint/UI cần kiểm tra:

- `WBP_DebugPanel`: tab/category render từ `BP_OnRefreshFromRegistry`.
- `WBP_DebugPanel`: entry render từ `BP_OnPopulateCategory`.
- Entry widget cho `SliderEntryWidgetClass`, `ButtonEntryWidgetClass`, `ToggleEntryWidgetClass`, `DropdownEntryWidgetClass`, `CheckBoxEntryWidgetClass`, `TextEntryWidgetClass`, `ProgressionEntryWidgetClass`.
- Mỗi widget phải gọi `DebugToolsSubsystem->ExecuteEntry(Category, EntryId, Value, Key, Toggle)` đúng tham số.

## Thêm nút mới trong module có sẵn

Flow:

```text
Thêm FDebugEntry trong GetEntries()
-> đảm bảo Entry.Id unique trong category
-> xử lý EntryId trong ExecuteEntry()
-> gọi subsystem/gameplay thật
-> nếu UI cần refresh text, broadcast OnRegistryChanged hoặc OnDebugTextUpdated
```

Checklist:

- `Entry.Id` trong `GetEntries()` phải trùng với `EntryId` trong `ExecuteEntry()`.
- `DisplayName` là text user thấy trên UI.
- `Type` phải có widget render tương ứng trong Blueprint.
- Nếu action thay đổi save/game state lớn, cân nhắc đưa vào pending queue thay vì apply ngay.
- Nếu action cần `World`, `GameInstance`, subsystem khác: luôn check null.

Ví dụ button:

```cpp
FDebugEntry Entry;
Entry.Id = FName(TEXT("MyButton"));
Entry.DisplayName = FText::FromString(TEXT("My Button"));
Entry.Type = EDebugEntryType::Button;
Entries.Add(Entry);
```

```cpp
if (EntryId == FName(TEXT("MyButton")))
{
	// Apply logic.
	return;
}
```

## Thêm slider mới

Flow:

```text
GetEntries() tạo Entry.Type = Slider, MinValue, MaxValue, CurrentValue
-> Blueprint slider hiển thị current
-> user kéo
-> ExecuteEntry(Category, EntryId, Value, Key, Toggle)
-> module đọc Value.GetValue<float>()
-> apply vào runtime object
-> GetCurrentValues() trả lại current state nếu UI cần sync
```

Code mẫu từ Vehicle:

```cpp
// Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Vehicle.cpp
Entries.Add(MakeSlider(
	FName("TopSpeed"),
	FText::FromString(TEXT("Top Speed")),
	0.f,
	500.f,
	CarStats.TopSpeed,
	CarStats.TopSpeed));
```

Điểm cần kiểm tra:

- `InitializeActions()` có map `EntryId -> field` chưa.
- `ExecuteEntry()` có apply `Value.GetValue<float>()` chưa.
- `GetCurrentValues()` có trả lại `EntryId` đó chưa.
- Runtime target có tồn tại không, ví dụ player pawn có phải `ASimulatePhysicsCar` không.

## Thêm dropdown mới

Flow:

```text
GetEntries() tạo DropBoxItems { ID, DisplayName }
-> user chọn item
-> UI truyền ID qua Value hoặc command qua Key
-> ExecuteEntry() lưu pending selection hoặc apply trực tiếp
-> nếu cần giữ selection qua lần mở panel, lưu vào DebugSaveGame
```

Code mẫu:

```cpp
FDropBoxItem Item;
Item.ID = 1;
Item.DisplayName = FText::FromString(TEXT("Option 1"));

FDebugEntry Entry;
Entry.Id = FName(TEXT("MyDropdown"));
Entry.DisplayName = FText::FromString(TEXT("My Dropdown"));
Entry.Type = EDebugEntryType::Dropdown;
Entry.DropBoxItems.Add(Item);
```

Lưu ý:

- Với Cheat nested UI, dropdown là `FProgressionBox` và field type là `ItemsEntry`.
- Nếu label cần đánh dấu current, xem các helper `MakeCurrentLabel` trong `DebugModule_Cheat.cpp`.
- Nếu selection ảnh hưởng nhiều bước, nên lưu pending value rồi chỉ apply khi bấm nút `Set`.

## Thêm text debug mới

Flow:

```text
GetEntries() trả Entry.Type = Text
-> UI hiển thị DisplayName hoặc row text
-> nếu value thay đổi runtime, module broadcast OnDebugTextUpdated hoặc refresh panel
```

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Public/DebugSystem/DebugToolsSubsystem.h
DECLARE_DYNAMIC_MULTICAST_DELEGATE_ThreeParams(
	FOnDebugTextUpdated,
	FName, Category,
	FName, EntryId,
	FText, NewValue);
```

Lưu ý:

- Text entry không nên tự mutate state trong `GetEntries()`.
- Nếu text cần live update, dùng `OnDebugTextUpdated` hoặc gọi `RefreshDebugPanel()`.

## Pending queue và confirm/revert

Các cheat nguy hiểm không apply ngay mà đi qua pending queue.

Flow:

```text
Cheat button
-> DebugModuleCheatPending::EnqueueOrApply()
-> UDebugCheatSubsystem::AddPending()
-> UI Pending hiển thị GetPendingQueue()
-> ConfirmAll()
-> chạy từng ApplyFunc
-> ghi ChangeLogs
-> phân tích thay đổi cash/inventory
-> refresh text cheat
```

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Public/DebugCheatSubsystem.h
USTRUCT(BlueprintType)
struct FPendingChange
{
	FString Description;
	TFunction<void()> ApplyFunc;
	TFunction<void()> RevertFunc;
	FDateTime CreatedAt;
	bool bNotifyPendingQueueOnAdd = true;
};
```

```cpp
// Source/PrototypeRacing/Private/DebugCheatSubsystem.cpp
void UDebugCheatSubsystem::ConfirmAll()
{
	for (FPendingChange& Change : PendingQueue)
	{
		if (Change.ApplyFunc)
		{
			Change.ApplyFunc();
		}
		ChangeLogs.Add(FChangeLogEntry(Change.Description));
	}
	PendingQueue.Empty();
	OnPendingQueueModified.Broadcast();
	OnChangeLogsModified.Broadcast();
}
```

Lưu ý:

- Lambda trong pending queue phải không capture object có thể invalid mà không check lại.
- Description phải rõ để log/export đọc được.
- Nếu action chỉnh cash/inventory, confirm sẽ log phân tích trước/sau.
- `RevertAll()` hiện tại chỉ clear pending, không gọi `RevertFunc`.

## Export log

Flow:

```text
UI Log
-> UDebugCheatSubsystem::ExportLogToTXT(ScriptURL, ChangeLog)
-> build text
-> HTTP POST text/plain
-> clear ChangeLogs khi dispatch xong
```

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Private/DebugCheatSubsystem.cpp
Request->SetURL(ScriptURL);
Request->SetVerb(TEXT("POST"));
Request->SetHeader(TEXT("Content-Type"), TEXT("text/plain"));
Request->SetContentAsString(OutputText);
Request->ProcessRequest();
```

Lưu ý:

- Cần URL script hợp lệ.
- Network fail chỉ log lỗi, không retry.
- Sau khi dispatch, `ChangeLogs.Empty()` được gọi.

## Debug save

File save debug:

```cpp
// Source/PrototypeRacing/Public/DebugSystem/DebugSaveGame.h
class UDebugSaveGame : public USaveGame
{
	bool bHasScaleIndexOverride = false;
	int32 SelectedDebugScaleIndexCityIndex = 0;
	int32 PendingRewardCityId = 1;
	int32 PendingRewardSourceId = 1;
	int32 PendingRewardTokenAmount = 1;
	bool bPendingRewardGrantToInventory = false;
};
```

Đang lưu:

- Scale index override.
- City index đang chọn cho scale index.
- Pending reward city/source/amount/grant-to-inventory.

Checklist khi thêm selection/persistent state:

- Thêm field vào `UDebugSaveGame`.
- Thêm hàm load/save ở manager/module đang sở hữu state.
- Không lưu dữ liệu suy ra được nếu có thể tính lại từ source data.
- Validate save cũ, vì field mới có default.

## Module Camera

Category: `Camera`

Flow:

```text
Button SwitchCamera
-> UDebugModule_Camera::ExecuteEntry()
-> UDebugToolsSubsystem::OnSwitchCamera.Broadcast()

Button CameraLeft
-> UDebugToolsSubsystem::OnLeftCamera.Broadcast()
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Camera.cpp`
- Listener Blueprint/C++ đang bind `OnSwitchCamera`, `OnLeftCamera`.

Lưu ý:

- Module chỉ broadcast delegate, không trực tiếp đổi camera.
- Nếu thêm camera action mới, cần thêm delegate ở `UDebugToolsSubsystem` hoặc gọi thẳng camera component/subsystem.

## Module Overlay

Category: `SOverlay`, display name `Overlay`

Flow:

```text
Toggle StatsOverlay
-> UDebugModule_Overlay::SetStatsOverlayEnabled()
-> EnsureStateOverlayWidget()
-> /Game/UI/TrackTest/WBP_DebugStatsOverlay
-> show/collapse widget
-> timer cập nhật runtime state
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Overlay.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/StateOverlayWidget.h`
- Blueprint `/Game/UI/TrackTest/WBP_DebugStatsOverlay`

Lưu ý:

- Widget được reset khi travel world.
- Nếu đổi overlay widget path, sửa `StatsOverlaySoftClass_TrackTest`.
- `GetCurrentValues()` trả `StatsOverlay` để UI sync toggle.

## Module TrackLogic

Category: `TrackLogic`

Tính năng chính:

- Restart race.
- Lose race.
- Win race.
- Freeze/UnFreeze AI.
- Increase/Decrease lap.
- PSO implement.
- Toggle draw checkpoint/collision/environment/AI/airborne/incline/impact/status/racing line.

Flow button:

```text
Button RestartRace
-> URaceSessionSubsystem::RestartRace()
```

Flow toggle draw:

```text
Toggle DrawCollision
-> UDebugToolsSubsystem::DrawCollision.Broadcast(ECarDebugTypes::CollisionCar, bToggle)
-> listener gameplay vẽ debug tương ứng
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_TrackLogic.cpp`
- Delegates draw trong `DebugToolsSubsystem.h`
- Các listener đang bind delegate draw.

Lưu ý:

- `InitializeActions()` cache weak pointer tới subsystem/actor.
- Nếu map reload hoặc actor đổi, cần đảm bảo action không giữ pointer chết; hiện tại dùng `TWeakObjectPtr`.

## Module Test Maps

Category: `Test Maps`

Flow mở map:

```text
Button OpenTrackTest
-> UGameplayStatics::OpenLevel(World, "/Game/Maps/Map_TrackTest/Map_TrackTest2")
```

Flow mở widget:

```text
Button ConnectToMap
-> LoadClass<UUserWidget>()
-> CreateWidget()
-> AddToViewport(500)
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_TestMaps.cpp`

Lưu ý khi thêm map:

- Thêm `EntryActions.Add("Id", { OpenMap(TEXT("/Game/...")) })`.
- Thêm button tương ứng trong `GetEntries()`.
- Đảm bảo path level đúng asset path.

## Module Tutorial

Category: `Tutorial`

Tính năng chính:

- Reset Tutorials.
- Trigger Tutorial.
- Show Tooltip.
- Enable Tutorial.
- First Player reset.

Flow command:

```text
Dropdown TriggerTutorial
-> UI truyền Key là console command
-> UKismetSystemLibrary::ExecuteConsoleCommand(GetWorld(), Key, nullptr)
```

Flow toggle:

```text
Toggle EnableTutorial
-> UTutorialManagerSubsystem::SetTutorialIsEnable(bToggle)
```

Flow First Player:

```text
Button FirstPlayer
-> TutorialManager->ResetTutorials()
-> CarCustomizationManager->ResetToDefaultConfiguration()
-> ProgressionSubsystem->ResetProgression()
-> ProfileManager->ResetProfileData()
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Tutorial.cpp`
- Console command implementation trong tutorial system.
- `UTutorialManagerSubsystem`.

Lưu ý:

- `ResetTutorials`, `TriggerTutorial`, `ShowTooltip` đang phụ thuộc `Key` từ UI/dropdown.
- Nếu thêm tutorial mới, thêm item dropdown và đảm bảo command tồn tại.

## Module Progression

Category: `Progression`

Đây là view read-only tổng hợp progression.

Flow:

```text
GetEntries()
-> UProgressionDebugManager lấy dữ liệu player/city/economy/garage/inventory
-> UProgressionSubsystem lấy current city goals/scale index
-> UCarRatingSubsystem resolve CR level
-> trả các ProgressionBox dạng Text
```

Box hiện có:

- Player overview: city, scale index, playtime, races, cars, items.
- Wallet: cash, fuel, click.
- Economy: earned, expected, ratio, spent, tokens.
- City goals.
- Garage.
- Inventory.

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Progression.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/ProgressionDebugManager.h`
- `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

Lưu ý:

- `ExecuteEntry()` đang rỗng, module này chủ yếu để hiển thị.
- Nếu thêm metric mới, ưu tiên thêm hàm đọc ở `UProgressionDebugManager`, sau đó add text row trong `GetEntries()`.
- Kiểm tra null `ProgressionSubsystem`, `DebugManager`, `CarRatingSubsystem` nếu mở trên map thiếu subsystem.

## Module Cheat

Category: `Cheat`

Module này có UI nested và nhiều nhóm:

- Wallet: cash/fuel.
- Inventory: full item/clear all.
- City & Progression: jump city, goal toggle, reset progression, unlock all.
- Track Cheat: win/lose từng track trong current city.
- Car & CR: global CR, stat per slot, scale index override.
- Car Stats: text runtime car stats.
- Rewards & tokens.
- Preset.
- Pending.
- Log.

Flow build nested UI:

```text
UDebugModule_Cheat::GetEntries()
-> tạo FProgressionBox cho từng row
-> tạo FProgressionGroup cho từng group
-> tạo FDebugEntry Type = Nested
-> Entries.Add(...)
```

Code cần kiểm tra:

```cpp
// Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Cheat.cpp
FDebugEntry Entry;
Entry.Id = Id;
Entry.DisplayName = DisplayName;
Entry.Type = EDebugEntryType::Nested;
Entry.ProgressionGroups = Groups;
```

Flow execute:

```text
UDebugModule_Cheat::ExecuteEntry()
-> HandleCityProgressionEntry()
-> HandleTextEntry()
-> HandleWalletEntry()
-> HandleInventoryEntry()
-> HandleTrackEntry()
-> HandleRewardTokenEntry()
-> HandleCarCREntry()
```

Lưu ý:

- Khi thêm cheat mới, nên thêm helper handler riêng nếu nhóm lớn.
- Những thay đổi lớn nên dùng pending queue.
- Nếu entry là dropdown/slider pending, chỉ lưu pending value; nút apply riêng sẽ gọi manager.
- Nếu cần refresh text, dùng `BroadcastByEntryId()` hoặc gọi `CheatRefreshTexts`.

## Cheat Wallet

Flow:

```text
Button CheatCash1K / CheatCash10K
-> pending "Add cash"
-> ConfirmAll
-> UProgressionDebugManager::SetEconomy(CurrentCash + Amount)

Button CheatCashMax
-> pending set cash 999999999

Button CheatCashReset
-> pending SetEconomy(0)

Button CheatFuelFull
-> pending AddFuel(999999)

Button CheatFuelZero
-> pending RemoveAllFuel()
```

Code cần kiểm tra:

- `UDebugModule_Cheat::HandleWalletEntry`
- `UProgressionDebugManager::SetEconomy`
- `UProgressionDebugManager::AddFuel`
- `UProgressionDebugManager::RemoveAllFuel`
- `UDebugCheatSubsystem::ConfirmAll`

## Cheat Inventory

Flow:

```text
Button CheatInventoryFullItem
-> pending Manager->SetFullInventory()

Button CheatInventoryClearAll
-> pending Manager->ClearInventory()
```

Code cần kiểm tra:

- `UDebugModule_Cheat::HandleInventoryEntry`
- `UProgressionDebugManager::SetFullInventory`
- `UProgressionDebugManager::ClearInventory`
- `UInventoryManager`

## Cheat City Progression

Flow goal toggle:

```text
Toggle CheatGoalT1/T2/T3
-> ProgressionSubsystem->DebugGetCurrentCityGoals()
-> nếu state khác hiện tại
-> pending ProgressionSubsystem->DebugSetCurrentCityGoalTierCompleted(Tier, bToggle)
```

Flow jump city:

```text
Button CheatJumpCityC{n}
-> validate city count từ TourData
-> pending Manager->JumpToCity(n)
```

Flow reset/unlock:

```text
CheatResetCityProgression
-> pending Manager->ResetCityProgression()

CheatUnlockAll
-> pending Manager->UnlockAllCities()
```

Code cần kiểm tra:

- `UDebugModule_Cheat::HandleCityProgressionEntry`
- `UProgressionSubsystem::DebugGetCurrentCityGoals`
- `UProgressionSubsystem::DebugSetCurrentCityGoalTierCompleted`
- `UProgressionDebugManager::JumpToCity`
- `UProgressionDebugManager::ResetCityProgression`
- `UProgressionDebugManager::UnlockAllCities`

## Cheat Track

Flow:

```text
GetEntries()
-> lấy current city
-> tạo Win/Lose button cho từng track

Button CheatTrackWin{TrackId}
-> pending Manager->DebugCompleteTrackAsFirstPlace(TrackId)

Button CheatTrackLose{TrackId}
-> pending Manager->DebugCompleteTrackAsLose(TrackId)
```

Code cần kiểm tra:

- `UDebugModule_Cheat::HandleTrackEntry`
- `UProgressionSubsystem::GetTrackById`
- `UProgressionDebugManager::DebugCompleteTrackAsFirstPlace`
- `UProgressionDebugManager::DebugCompleteTrackAsLose`

Lưu ý:

- EntryId parse bằng prefix `CheatTrackWin` / `CheatTrackLose`; không đổi format nếu UI còn phụ thuộc.

## Cheat Car & CR

Flow global CR:

```text
Slider CheatCarCRGlobalValue
-> lưu PendingGlobalCR
-> Button CheatCarCRGlobalSet
-> apply/reset CR theo logic trong HandleCarCREntry
```

Flow per-slot stats:

```text
Slider CheatCarCRSpd/Acc/Hdl/Nos
-> lưu pending stat
-> apply qua manager khi entry apply được bấm
```

Code cần kiểm tra:

- `UDebugModule_Cheat::HandleCarCREntry`
- `UDebugModule_Cheat::SyncPendingCarCRValues`
- `UProgressionDebugManager`
- `UCarCustomizationManager`
- `UCarRatingSubsystem`

Lưu ý:

- Slider pending không nhất thiết apply ngay.
- Sau khi apply CR/stats, cần refresh text `CheatCarCRText`.

## Cheat Scale Index Override

Chi tiết riêng có ở `Docs/DebugScaleIndexOverride.md`.

Flow ngắn:

```text
Dropdown CheatCarCRScaleIndexCity
-> PendingScaleIndexCityIndex = selected city
-> SaveSelectedDebugScaleIndexCityIndex()

Button CheatCarCRScaleIndexSet
-> validate GetBaseScaleIndexByCityIndex(selected city) > 0
-> Manager->DebugSetScaleIndexCity(selected city)
-> save bHasScaleIndexOverride = true

Button CheatCarCRScaleIndexReset
-> PendingScaleIndexCityIndex = current city
-> Manager->DebugResetScaleIndexToCurrentCity()
-> save bHasScaleIndexOverride = false

Runtime formula
-> UProgressionSubsystem::GetScaleIndexByCityIndex()
-> hỏi UProgressionDebugManager override trước
-> nếu có override hợp lệ thì trả scale index debug
-> nếu không thì trả base scale index
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Cheat.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/ProgressionDebugManager.h`
- `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/DebugSaveGame.h`
- `Source/PrototypeRacing/Private/BackendSubsystem/Progression/ProgressionSubsystem.cpp`
- `Docs/DebugScaleIndexOverride.md`

Lưu ý:

- Không lưu float scale index trong save; chỉ lưu city index và flag override.
- Dropdown chọn city chưa bật override; phải bấm `Set ScaleIndex`.
- Override chỉ dùng non-shipping.

## Cheat Rewards & Tokens

Flow selection:

```text
Dropdown CheatRewardCityID
-> PendingRewardCityId
-> SavePendingRewardSelection()

Dropdown CheatRewardSource
-> PendingRewardSourceId
-> SavePendingRewardSelection()

Dropdown CheatRewardTokenAmount
-> PendingRewardTokenAmount
-> SavePendingRewardSelection()

CheckBox CheatRewardGrantToInventory
-> bPendingRewardGrantToInventory
-> SavePendingRewardSelection()
```

Flow action:

```text
Button CheatRewardSpawnToken
-> resolve ERewardSource từ PendingRewardSourceId
-> Manager->DebugSpawnTokenBySource(CityID, Amount, Source, bGrantToInventory)

Button CheatRewardResetPool
-> Manager->DebugResetRewardPool()
```

Code cần kiểm tra:

- `UDebugModule_Cheat::HandleRewardTokenEntry`
- `UDebugSaveGame`
- `UProgressionDebugManager::DebugSpawnTokenBySource`
- `UProgressionDebugManager::DebugResetRewardPool`
- Reward center / inventory subsystem nếu grant vào inventory.

## Cheat Car Stats

Flow:

```text
GetEntries()
-> tìm player ASimulatePhysicsCar
-> ResolveCarStatRowText() từng stat
-> hiển thị Text rows
```

Code cần kiểm tra:

- Namespace/helper `DebugModuleCheatCarStats` trong `DebugModule_Cheat.cpp`
- `ASimulatePhysicsCar`
- Các field runtime trên car.

Lưu ý:

- Nếu không có player physics car, UI hiển thị status "not found".
- Đây là read-only text, không apply stat.

## Module Vehicle

Category: `Vehicle`

Tính năng:

- Slider tuning performance/runtime car physics.
- Apply trực tiếp lên `ASimulatePhysicsCar`.
- Broadcast `OnApplyCarSettings`.
- Preset manager có thể save/load values.

Flow:

```text
GetEntries()
-> InitializeActions()
-> SyncWithPlayerVehicle() nếu chưa có user override
-> tạo slider từ CarStats

Slider changed
-> ExecuteEntry()
-> EntryActions[EntryId].SetValue(Value)
-> ApplyPerformanceToCar()
-> DebugToolsSubsystem->OnApplyCarSettings.Broadcast(CarStats)
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Vehicle.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/Modules/DebugModule_Vehicle.h`
- `Source/PrototypeRacing/Public/DebugSystem/CarProfileTypes.h`
- `Source/PrototypeRacing/Private/DebugSystem/DebugPresetManager.cpp`
- `ASimulatePhysicsCar::SetGamePerformance`

Lưu ý quan trọng:

- `UDebugToolsSubsystem::IsVehicleDebugApplyBlocked()` block apply trong `ARacingCarGameMode` city progression.
- TrackTest/dev maps được phép apply.
- Nếu thêm slider mới, phải cập nhật đủ 4 chỗ: `InitializeActions`, `GetEntries`, `GetCurrentValues`, `ApplyPerformanceToCar`.
- Nếu field cũng cần lưu preset, kiểm tra `DebugPresetManager` và struct profile.

## Preset manager

Flow:

```text
DebugPanelWidget::SavePreset(PresetName)
-> UDebugPresetManager::SavePreset()

DebugPanelWidget::LoadPreset(PresetName)
-> nếu Vehicle apply không bị block
-> UDebugPresetManager::LoadPreset()
-> PopulateCategory(CurrentCategory)
```

Code cần kiểm tra:

- `Source/PrototypeRacing/Private/DebugSystem/DebugPresetManager.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/DebugPresetManager.h`
- `UDebugPanelWidget::SyncDebugPresetFromCurrentProfileOnPanelOpen`

Lưu ý:

- Khi mở panel, preset theo current car profile có thể auto-load nếu chưa có vehicle user override.
- Nếu đang ở city progression race, vehicle preset load bị block.

## Data export và race data

Các file liên quan:

- `Source/PrototypeRacing/Public/DebugSystem/DataExportManager.h`
- `Source/PrototypeRacing/Private/DebugSystem/DataExportManager.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/RaceDataCollector.h`
- `Source/PrototypeRacing/Private/DebugSystem/RaceDataCollector.cpp`
- `Source/PrototypeRacing/Public/DebugSystem/RaceDataTypes.h`
- `Source/PrototypeRacing/Public/DebugSystem/ExportTypes.h`

Flow tổng quát:

```text
RaceDataCollector thu dữ liệu race
-> DataExportManager format/export
-> ExportTypes/RaceDataTypes định nghĩa struct dữ liệu
```

Khi thêm metric race:

- Thêm field vào `RaceDataTypes`.
- Ghi dữ liệu trong `RaceDataCollector`.
- Cập nhật export format trong `DataExportManager`.
- Kiểm tra nơi gọi export trong UI hoặc test flow.

## Checklist thêm UI/tính năng debug mới

1. Xác định loại tính năng:
   - Chỉ hiển thị: dùng `Text` / `ProgressionBox`.
   - Bấm chạy ngay: dùng `Button`.
   - Chọn trạng thái: dùng `Toggle` / `CheckBox`.
   - Chọn option: dùng `Dropdown`.
   - Tuning số: dùng `Slider`.
   - Nhiều nhóm con: dùng `Nested`.

2. Chọn module:
   - Camera: action camera.
   - Overlay: overlay/debug HUD.
   - TrackLogic: race state/draw debug.
   - Test Maps: mở map/widget test.
   - Tutorial: tutorial/tooltip.
   - Progression: read-only progression info.
   - Cheat: mutate wallet/inventory/progression/reward/car CR.
   - Vehicle: tuning physics/performance.
   - Nếu không thuộc nhóm nào, tạo module mới.

3. Sửa C++:
   - Add entry trong `GetEntries()`.
   - Add handling trong `ExecuteEntry()`.
   - Add current state trong `GetCurrentValues()` nếu UI cần sync.
   - Add helper/manager method nếu logic không nên nằm trong module.
   - Add save field nếu cần persist pending selection.

4. Sửa Blueprint UI nếu cần:
   - `WBP_DebugPanel` render entry mới đúng `EDebugEntryType`.
   - Entry widget gọi `ExecuteEntry` đúng `Category`, `EntryId`, `Value`, `Key`, `Toggle`.
   - Nếu type mới, thêm widget class/property.

5. Kiểm tra side effect:
   - Có cần pending queue không?
   - Có cần refresh text/panel không?
   - Có cần block trong race/city progression không?
   - Có cần null check subsystem/actor không?
   - Có cần save/load debug state không?

6. Test:
   - Mở panel bằng gesture hoặc gọi `OpenDebugPanel()`.
   - Category xuất hiện.
   - Entry render đúng.
   - Button/slider/toggle/dropdown gọi đúng `EntryId`.
   - Không crash khi thiếu player car, thiếu race manager, hoặc mở ở map khác.
   - Confirm/revert pending chạy đúng nếu có.
   - Log `LogDebugTools` rõ khi action bị ignore.

## Các lỗi hay gặp

- `Entry.Id` trong UI không trùng với string xử lý trong `ExecuteEntry()`.
- Thêm slider nhưng quên `GetCurrentValues()`, UI mở lại bị sai value.
- Thêm vehicle field nhưng quên `ApplyPerformanceToCar()`.
- Capture raw pointer trong pending lambda, đến lúc confirm object đã invalid.
- Thêm dropdown nhưng UI truyền `Key` trong khi C++ đọc `Value`, hoặc ngược lại.
- Quên bọc non-shipping khiến Shipping build fail hoặc chứa code debug.
- Category name trùng, module sau overwrite module trước.
- Blueprint `WBP_DebugPanel` chưa gán widget class cho entry type mới.
- Thay đổi save debug nhưng không có default hợp lệ cho save cũ.

## Quy tắc đặt tên đề xuất

- Category: ngắn, unique, ví dụ `Vehicle`, `Cheat`, `TrackLogic`.
- EntryId: prefix theo module/nhóm, ví dụ `CheatRewardSpawnToken`, `CheatTrackWin{TrackId}`.
- Với entry parse bằng prefix, giữ format ổn định.
- Display text ngắn, dễ đọc trong panel.
- Log dùng prefix rõ: `[Cheat]`, `[Overlay]`, `[DebugTools]`.

