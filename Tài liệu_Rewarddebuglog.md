# Reward Debug Log

## Mục Đích

Tính năng `Reward Debug Log` dùng để ghi lại lịch sử các batch reward được tạo ra trong game hoặc từ debug cheat.

Mục tiêu chính:

- QC xem được mỗi batch reward được tạo lúc nào.
- QC biết batch đến từ nguồn nào: `RaceRewards`, `GoalRewards`, `DebugCheat`.
- QC kiểm tra từng pull ra item gì, type/rarity nào.
- QC biết item được add vào inventory, bị duplicate, hay bị convert sang cash.
- QC xem được số unique item còn lại trong city pool tại thời điểm pull.
- QC xem được drop rate của item trước và sau khi pull: `before% -> after%`.
- QC xem được summary tổng số pull, drop theo type, duplicate, same-batch duplicate.
- QC xem được tỉ lệ roll item type hiện tại: Visual 40%, Performance 30%, Loot Crate 30%.

Tính năng này chỉ phục vụ debug/QC. Phần snapshot metric được bọc `#if !UE_BUILD_SHIPPING` để không chạy trong shipping build.

## Sản Phẩm Cuối

Blueprint/UI có một entry debug:

```cpp
ItemsRewardsTokens.Add({
	TEXT("CheatRewardLog"),
	FText::GetEmpty(),
	EDebugEntryType::RewardLog
});
```

Blueprint gọi hàm:

```cpp
CreateLogReward()
```

Hàm này trả về `FString` đã format sẵn để UI hiển thị trực tiếp.

Format log gồm:

```text
Batch #1: 20260615, 02:30 PM, DebugCheat, 3 Pulls
  - Pull 1: City1Pool -> CarVisual -> Common -> Basic Front Bumper x1 -> No Inventory Duplicate -> Add To Inventory -> 109 Unique Items Remaining in City1 Pools

Basic Front Bumper Rate Reduced to: 25.0% -> 20.0%.

SUMMARY BY CITY:
City 1
  - City Pool: 139
  - City Pool Remaining: 108
  - Total Pulls/Tokens: 3
  - Visual Drops: 2
  - Performance Drops: 1
  - LootBox Drops: 0
  - Duplicates: 0
  - SameBatch Duplicates: 1

TOTAL SUMMARY:
  - Total Pulls/Tokens: 3
  - Visual Drops: 2
  - Performance Drops: 1
  - LootBox Drops: 0
  - Duplicates: 0
  - SameBatch Duplicates: 1

ITEM TYPE DROP RATE:
  - Visual: 40%
  - Performance: 30%
  - Loot Crate: 30%
```

Log được lưu trong debug save:

```text
Debug/DebugSaveGame
```

## Flow Ngắn Gọn

1. `RewardCenterSubsystem` tạo reward batch.
2. Mỗi pull lưu snapshot debug vào `FRewardResult`.
3. Batch được gắn `DebugLogSource`.
4. `RewardCenterSubsystem` broadcast event reward calculated.
5. `ProgressionDebugManager::HandleItemRewardForLog(...)` nhận batch.
6. `MakeDebugRewardBatchLog(...)` convert `FRewardBatchResult` thành `FDebugRewardBatchLog`.
7. `RewardBatchLogs` được append và save vào `UDebugSaveGame`.
8. Blueprint gọi `CreateLogReward()` để lấy string log.
9. Blueprint gọi `ClearRewardBatchLogs()` nếu cần xóa log.

## Các Đoạn Code Liên Quan

### 1. Entry Type Cho Reward Log

File: `Source/PrototypeRacing/Public/DebugSystem/DebugSystemTypes.h`

```cpp
UENUM(BlueprintType)
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
	RewardLog,
};
```

### 2. Entry Trong Debug Cheat Module

File: `Source/PrototypeRacing/Private/DebugSystem/Modules/DebugModule_Cheat.cpp`

```cpp
ItemsRewardsTokens.Add({
	TEXT("CheatRewardLog"),
	FText::GetEmpty(),
	EDebugEntryType::RewardLog
});
```

Entry này chỉ tạo item để UI thấy và render phần reward log. Data log được Blueprint lấy qua `CreateLogReward()`.

### 3. Runtime Result Lưu Snapshot Debug

File: `Source/PrototypeRacing/Public/BackendSubsystem/RewardCenterSubsystem.h`

```cpp
UPROPERTY(BlueprintReadWrite, Category = "Reward")
float DropRateBeforePercent = 0.f;

UPROPERTY(BlueprintReadWrite, Category = "Reward")
float DropRateAfterPercent = 0.f;

UPROPERTY(BlueprintReadWrite, Category = "Reward")
int32 UniqueItemsRemaining = 0;
```

Các field này nằm trong `FRewardResult`.

### 4. Tính Snapshot Sau Khi Pull

File: `Source/PrototypeRacing/Private/BackendSubsystem/RewardCenterSubsystem.cpp`

```cpp
#if !UE_BUILD_SHIPPING
auto PopulateRewardSnapshot = [this](
	FRewardResult& RewardResult,
	const FName& SnapshotCityID,
	const float DropRateBeforePercent)
{
	if (RewardResult.ItemID.IsNone())
	{
		return;
	}

	RewardResult.DropRateBeforePercent = DropRateBeforePercent;
	RewardResult.DropRateAfterPercent = CalculateRewardItemDropRatePercent(
		SnapshotCityID,
		RewardResult.ItemID,
		RewardResult.ItemType,
		RewardResult.Rarity);
	RewardResult.UniqueItemsRemaining = GetRewardPoolUnpickedItemCount(SnapshotCityID);
};
#endif
```

Trước khi update weight, lưu rate trước pull:

```cpp
#if !UE_BUILD_SHIPPING
const float DropRateBeforePercent = CalculateRewardItemDropRatePercent(
	CityID,
	PickedItem.ItemMasterID,
	TargetType,
	TargetRarity);
#endif
```

Sau khi update weight, lưu snapshot sau pull:

```cpp
IncrementPickedCountAndUpdateWeight(PickedItem.CityID, PickedItem.ItemMasterID);

#if !UE_BUILD_SHIPPING
PopulateRewardSnapshot(FinalReward, PickedItem.CityID, DropRateBeforePercent);
#endif
```

### 5. Debug Source Của Batch

File: `Source/PrototypeRacing/Public/BackendSubsystem/RewardCenterSubsystem.h`

```cpp
UPROPERTY(BlueprintReadWrite, Category = "Reward")
FString DebugLogSource;
```

File: `Source/PrototypeRacing/Private/BackendSubsystem/RewardCenterSubsystem.cpp`

```cpp
BatchResult.DebugLogSource =
	StaticEnum<ERewardSource>()->GetNameStringByValue(static_cast<int64>(Source));
```

Các source override đang dùng:

```cpp
RewardBatchResult.DebugLogSource = TEXT("RaceRewards");
RewardBatchResult.DebugLogSource = TEXT("GoalRewards");
BatchResult.DebugLogSource = TEXT("DebugCheat");
```

### 6. Struct Lưu Debug Log

File: `Source/PrototypeRacing/Public/DebugSystem/DebugSaveGame.h`

```cpp
USTRUCT(BlueprintType)
struct FDebugRewardResultLog
{
	GENERATED_BODY()

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	bool bSuccess = false;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	FName ItemID = FName();

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	FString ItemName;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	EItemType ItemType = EItemType::Other;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	EItemRarity Rarity = EItemRarity::Common;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	int32 ItemCount = 1;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	bool bWasDuplicate = false;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	bool bConvertedToCash = false;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	int32 CashAmount = 0;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	float DropRateBeforePercent = 0.f;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	float DropRateAfterPercent = 0.f;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	int32 UniqueItemsRemaining = 0;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	FString FailureReason;
};
```

```cpp
USTRUCT(BlueprintType)
struct FDebugRewardBatchLog
{
	GENERATED_BODY()

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	FDateTime CreatedAt;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	FString SourceLabel;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	FName CityID = FName();

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	int32 TokenCount = 0;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	bool bSuccess = false;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	FString ErrorCode;

	UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
	TArray<FDebugRewardResultLog> Rewards;
};
```

```cpp
UPROPERTY(BlueprintReadWrite, Category = "Debug|Reward")
TArray<FDebugRewardBatchLog> RewardBatchLogs;
```

### 7. Bind Event Reward Calculated

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
RewardBatchLogs = LoadRewardBatchLogsFromSave();

if (RewardCenterSubsystem)
{
	RewardCenterSubsystem->OnItemRewardCalculated.AddUniqueDynamic(
		this,
		&UProgressionDebugManager::HandleItemRewardForLog);

	RewardCenterSubsystem->OnGoalItemRewardCalculated.AddUniqueDynamic(
		this,
		&UProgressionDebugManager::HandleItemRewardForLog);
}
```

### 8. Ghi Batch Vào Debug Save

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
void UProgressionDebugManager::HandleItemRewardForLog(
	const FRewardBatchResult& RewardBatchResult)
{
	if (RewardBatchResult.TokenCount <= 0 && RewardBatchResult.Rewards.Num() <= 0)
	{
		return;
	}

	RewardBatchLogs.Add(MakeDebugRewardBatchLog(RewardBatchResult));
	SaveRewardBatchLogsToSave(RewardBatchLogs);
}
```

### 9. Convert Runtime Batch Thành Debug Log

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
FDebugRewardBatchLog MakeDebugRewardBatchLog(const FRewardBatchResult& RewardBatchResult)
{
	FDebugRewardBatchLog Log;
	Log.CreatedAt = FDateTime::Now();
	Log.SourceLabel = GetRewardSourceLogLabel(RewardBatchResult);
	Log.CityID = RewardBatchResult.CityID;
	Log.TokenCount = RewardBatchResult.TokenCount;
	Log.bSuccess = RewardBatchResult.bSuccess;
	Log.ErrorCode = RewardBatchResult.ErrorCode;
	Log.Rewards.Reserve(RewardBatchResult.Rewards.Num());

	for (const FRewardResult& Reward : RewardBatchResult.Rewards)
	{
		FDebugRewardResultLog RewardLog;
		RewardLog.bSuccess = Reward.bSuccess;
		RewardLog.ItemID = Reward.ItemID;
		RewardLog.ItemName = Reward.ItemName.ToString();
		RewardLog.ItemType = Reward.ItemType;
		RewardLog.Rarity = Reward.Rarity;
		RewardLog.ItemCount = Reward.ItemCount;
		RewardLog.bWasDuplicate = Reward.bWasDuplicate;
		RewardLog.bConvertedToCash = Reward.bConvertedToCash;
		RewardLog.CashAmount = Reward.CashAmount;
		RewardLog.DropRateBeforePercent = Reward.DropRateBeforePercent;
		RewardLog.DropRateAfterPercent = Reward.DropRateAfterPercent;
		RewardLog.UniqueItemsRemaining = Reward.UniqueItemsRemaining;
		RewardLog.FailureReason = Reward.FailureReason.ToString();
		Log.Rewards.Add(RewardLog);
	}

	return Log;
}
```

### 10. Format Outcome Của Một Pull

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
FString BuildRewardOutcomeText(const FDebugRewardResultLog& Reward)
{
	if (Reward.bConvertedToCash)
	{
		return FString::Printf(TEXT("%s -> Convert to Cash -> %d$"),
			Reward.bWasDuplicate ? TEXT("Duplicate") : TEXT("No Inventory Space"),
			Reward.CashAmount);
	}

	if (!Reward.bSuccess)
	{
		return Reward.FailureReason.IsEmpty() ? TEXT("Failed") : Reward.FailureReason;
	}

	return Reward.bWasDuplicate
		? TEXT("Duplicate -> Add To Inventory")
		: TEXT("No Inventory Duplicate -> Add To Inventory");
}
```

### 11. Format Remaining Và Rate

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
if (Reward.UniqueItemsRemaining > 0)
{
	LogText += FString::Printf(
		TEXT(" -> %d Unique Items Remaining in City%s Pools"),
		Reward.UniqueItemsRemaining,
		*BatchLog.CityID.ToString());
}
```

```cpp
if (!Reward.ItemID.IsNone() &&
	(Reward.DropRateBeforePercent > 0.f || Reward.DropRateAfterPercent > 0.f))
{
	const FString ItemName = Reward.ItemName.IsEmpty()
		? Reward.ItemID.ToString()
		: Reward.ItemName;

	LogText += FString::Printf(
		TEXT("%s Rate Reduced to: %.1f%% -> %.1f%%.\n"),
		*ItemName,
		Reward.DropRateBeforePercent,
		Reward.DropRateAfterPercent);
}
```

### 12. Tính Summary

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
Summary.TotalPulls += BatchLog.TokenCount;
```

```cpp
const int32 SafeItemCount = FMath::Max(1, Reward.ItemCount);
switch (Reward.ItemType)
{
case EItemType::CarVisual:
	Summary.VisualDrops += SafeItemCount;
	break;
case EItemType::CarPerformance:
	Summary.PerformanceDrops += SafeItemCount;
	break;
case EItemType::LootCrate:
	Summary.LootBoxDrops += SafeItemCount;
	break;
default:
	break;
}
```

Duplicate chỉ tính khi duplicate bị convert sang cash:

```cpp
if (Reward.bConvertedToCash && Reward.FailureReason.Contains(TEXT("Duplicate")))
{
	Summary.Duplicates += SafeItemCount;
}
```

Same-batch duplicate tính phần lặp trong batch:

```cpp
if (Reward.FailureReason.Contains(TEXT("Duplicate In Batch")))
{
	Summary.SameBatchDuplicates += SafeItemCount;
}
else if (Reward.ItemCount > 1)
{
	Summary.SameBatchDuplicates += Reward.ItemCount - 1;
}
```

### 13. Format Summary Và Tỉ Lệ Item Type

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
SummaryText += TEXT("\nTOTAL SUMMARY:\n");
SummaryText += BuildRewardSummaryBullets(TotalSummary, false);
SummaryText += TEXT("\nITEM TYPE DROP RATE:\n");
SummaryText += TEXT("  - Visual: 40%\n");
SummaryText += TEXT("  - Performance: 30%\n");
SummaryText += TEXT("  - Loot Crate: 30%\n");
```

### 14. Tạo String Cho Blueprint

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
FString UProgressionDebugManager::CreateLogReward()
{
	RewardBatchLogs = LoadRewardBatchLogsFromSave();

	FString LogText;
	for (int32 BatchIndex = 0; BatchIndex < RewardBatchLogs.Num(); ++BatchIndex)
	{
		LogText += BuildRewardBatchLogString(RewardBatchLogs[BatchIndex], BatchIndex + 1);
	}

	LogText += BuildRewardSummaryString(RewardBatchLogs, RewardCenterSubsystem.Get());
	return LogText;
}
```

### 15. Xóa Log

File: `Source/PrototypeRacing/Private/DebugSystem/ProgressionDebugManager.cpp`

```cpp
void UProgressionDebugManager::ClearRewardBatchLogs()
{
	RewardBatchLogs.Reset();
	SaveRewardBatchLogsToSave(RewardBatchLogs);
}
```
