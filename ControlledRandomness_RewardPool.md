# Controlled Randomness — Reward Pool

## Tổng quan

Thay thuật toán pick reward cũ (decay weight) bằng **controlled randomness** — đảm bảo tỷ lệ drop theo `BaseWeight` trong dài hạn, tránh streak ngẫu nhiên quá lệch.

| | |
|---|---|
| Thuật toán gốc | Jason Frank & Malte Skarupke (2019) — [blog post](https://probablydance.com/2019/08/28/a-new-algorithm-for-controlled-randomness/) |
| Tham chiếu | Radoshaka (Unity/C# port + QoL features) |
| UE implementation | `FControlledRandomness` + `URewardCenterSubsystem` |

---

## Tóm tắt cơ chế

Mỗi item có **vị trí ảo** (`Mark`) trên một đường thẳng. Mỗi nhóm **(ItemType + Rarity)** là một **randomize riêng** — không gom init cả city.

1. **Init** — `Multiplier = maxWeight / BaseWeight` **trong từng nhóm (Type + Rarity)**. Item weight thấp → multiplier cao → nhảy xa hơn.
2. **Pick** — Chọn item có `(Recycle, Mark)` **thấp nhất** trong candidates (đã filter type + rarity).
3. **Sau pick** — `Mark += random(Scale..1) × Multiplier` → item vừa trúng tạm lùi.
4. **Wrap** — `Mark >= 1,048,576` → `Recycle++`, mark quay vòng.

**Kết quả:** Random có nhớ, tỷ lệ dài hạn theo `BaseWeight`, item không "chết" như decay.

---

## Hai lớp random (post race)

```
Lớp 1 — Chọn rarity (random thuần, không nhớ)
  70% Common | 20% Uncommon | 10% Rare
        ↓
Lớp 2 — Chọn item trong nhóm (controlled randomness, có nhớ)
  filter Type + Rarity → pick mark thấp nhất → push mark
```

Goal reward (T1/T2/T3) **chỉ định rarity** rồi roll trong nhóm đó — vẫn dùng cùng 1 pool/city.

---

## Nhóm randomize (Type + Rarity)

**Yêu cầu:** mỗi rarity tier một randomize, không dồn 1 cục.

**Cách làm:** Vẫn **1 pool data/city**, nhưng `Initialize` / `UpdateDistribution` chạy **từng nhóm**:

```
City pool (1 list item)
  ├── Common + Visual      → InitializePerTypeRarityGroup
  ├── Common + Performance → InitializePerTypeRarityGroup
  ├── Rare + Visual        → InitializePerTypeRarityGroup
  └── ...
```

| Bước | Phạm vi |
|------|---------|
| Init / recalc Multiplier | Per nhóm (Type + Rarity) |
| Pick / push mark | Per nhóm (filter khi roll) |
| Lưu save | Per item (`Mark`, `Recycle`); `Multiplier` **recalc** sau load |

**Tại sao (Type + Rarity) chứ không chỉ Rarity?** — `RollItemFromRewardPool` luôn filter cả type lẫn rarity. Nhóm khớp đúng flow roll.

---

## Chi tiết thuật toán

### Tính Multiplier (trong từng nhóm)

```
BiggestWeight = max(BaseWeight)   // CHỈ trong nhóm (Type, Rarity)
Multiplier[i] = BiggestWeight / BaseWeight[i]
```

| Item | BaseWeight | Multiplier (trong nhóm) |
|------|-----------|-------------------------|
| A | 60 | 1.0 — pick nhiều nhất |
| B | 30 | 2.0 — pick ~ít hơn 2x |
| C | 0 | ignored |

### Khởi tạo Mark

```
Mark[i] = random(0..1) × Multiplier[i]
Recycle[i] = 0
```

### Pick

Chọn item có **(Recycle, Mark) nhỏ nhất** trong candidates:

1. So sánh `Recycle` trước
2. Nếu bằng nhau → so sánh `Mark`

### Push Mark (sau grant)

```
jump = random(ControlledRandomnessScale, 1.0) × Multiplier
Mark += jump
```

`ControlledRandomnessScale` (default `0.1`): `0` = spacing đều, `1` = random hơn.

### Recycle (wrap)

```
RecycleMark = 1,048,576
Recycle += floor(Mark / RecycleMark)
Mark     = Mark mod RecycleMark
```

### Ví dụ trong 1 nhóm (2 item, weight 50/50)

```
Lần 1: Red Mark=0.3, Blue Mark=0.7  → pick Red  → Red += ~0.5
Lần 2: Red Mark=0.8, Blue Mark=0.7  → pick Blue → Blue += ~0.6
Lần 3: Red Mark=0.8, Blue Mark=1.3  → pick Red  → ...
```

Xen kẽ thay vì streak 5 Red liên tiếp.

---

## Struct `FRewardPoolEntryItem`

| Bỏ (decay cũ) | Thêm (controlled randomness) |
|---------------|------------------------------|
| `TimesPicked` | `Mark` (float) |
| `EffectiveWeight` | `Recycle` (int32) |
| | `Multiplier` (float, runtime) |

**Giữ nguyên:** `PoolEntryID`, `CityID`, `ItemMasterID`, `BaseWeight`

`FCityLootPoolRow` và struct reward khác **không đổi**.

---

## Files

| File | Vai trò |
|------|---------|
| `ControlledRandomness.h/.cpp` | Core: init/pick/push, **group per Type+Rarity** |
| `RewardCenterSubsystem.h/.cpp` | Pool, save/load, roll reward |
| `RewardCenterSaveGame.h` | Lưu `RewardPoolEntries` |

---

## Flow runtime

```
SetupRewardCenter / ForceResetRewardPool
  └─ BuildRewardPoolFromDataTable()
       └─ InitializePerTypeRarityGroup(Items, CityLootPoolDataTable)

SetupRewardCenter + có save
  └─ SyncRewardPoolEntries()
       ├─ load Mark + Recycle (skip nếu Multiplier == 0 → save cũ)
       └─ UpdateDistributionPerTypeRarityGroup(Items, CityLootPoolDataTable)

Mỗi token reward
  └─ RollItemFromRewardPool(City, Type, Rarity, Source)
       ├─ GetRewardPoolCandidates()     // filter Type + Rarity
       ├─ PickByLowestMark()
       └─ (sau grant) PushMarkForward()
```

---

## Review nhanh (checklist)

Đọc theo thứ tự dưới đây để verify implementation hợp lý. **3 file chính:**

| File | Vai trò |
|------|---------|
| `ControlledRandomness.cpp` | Core algorithm + group per (Type, Rarity) |
| `RewardCenterSubsystem.cpp` | Build pool, sync save, roll, push |
| `RewardCenterSubsystem.h` | Struct `FRewardPoolEntryItem` |

### 1. Fix yêu cầu “mỗi nhóm 1 randomize” ⭐

**Gom nhóm** — `BuildTypeRarityGroupIndices` (`ControlledRandomness.cpp` ~L14–41):

```
Key = (ItemCategory, RarityTier) từ CityLootPoolDataTable
→ TMap<FTypeRarityKey, TArray<int32>>
```

**Init per nhóm** — `InitializePerTypeRarityGroup` (~L73–82):

- `RunOnEachTypeRarityGroup` tách subset → gọi `Initialize(GroupItems)` → ghi lại vào pool gốc
- `CalculateMultipliers` chỉ thấy item **trong nhóm** → `max(BaseWeight)` đúng scope

**Gọi lúc build** — `BuildRewardPoolFromDataTable` (`RewardCenterSubsystem.cpp` ~L90–94):

```
for each city pool:
  InitializePerTypeRarityGroup(Items, CityLootPoolDataTable)
```

**Sync save** — `SyncRewardPoolEntries` (~L99–132):

```
load Mark + Recycle (skip nếu save cũ)
→ UpdateDistributionPerTypeRarityGroup  // recalc Multiplier per nhóm
```

### 2. Core algorithm ⭐

| Hàm | File | Cần check |
|-----|------|-----------|
| `CalculateMultipliers` | `ControlledRandomness.cpp` ~L178–191 | `Multiplier = maxWeight / BaseWeight`; weight 0 → ignored |
| `Initialize` | ~L95–112 | Random mark `0..1 × Multiplier`; ignored → `Mark = Max` |
| `PickByLowestMark` | ~L114–140 | So `(Recycle, Mark)` thấp nhất; bỏ ignored |
| `PushMarkForward` | ~L142–161 | `jump = random(Scale..1) × Multiplier`; wrap tại `RecycleMark` |
| `UpdateDistribution` | ~L164–176 | Recalc multiplier; **không** reset mark/recycle |

### 3. Roll flow ⭐

| Hàm | File | Cần check |
|-----|------|-----------|
| `GetRewardPoolCandidates` | `RewardCenterSubsystem.cpp` ~L135–155 | Filter `ItemCategory == RewardType` **và** `RarityTier == RarityTier` |
| `RollItemFromRewardPool` | ~L163–172 | Candidates → `PickByLowestMark` |
| `PushMarkForward` | ~L785–798 | Tìm item **trong `RewardPoolEntries` persistent** → push mark |

**Lưu ý:** `PickByLowestMark` chạy trên **copy** candidates; state chỉ đổi khi `PushMarkForward` ghi vào pool thật.

**Callers push mark** — `GenerateRewardsByRequests` (~L361) và path grant khác (~L408): gọi `PushMarkForward(CityID, ItemMasterID)` sau khi item được grant thành công.

### 4. Struct & save

**Struct** — `RewardCenterSubsystem.h` ~L71–99:

```
BaseWeight  — từ DataTable (config)
Mark        — runtime, lưu save
Recycle     — runtime, lưu save
Multiplier  — runtime, KHÔNG load từ save (recalc per nhóm)
```

**Migration save cũ** — `SyncRewardPoolEntries` ~L121–124:

```
if (loaded Multiplier <= SMALL_NUMBER) → skip load Mark/Recycle
→ giữ mark fresh từ InitializePerTypeRarityGroup
```

### Checklist review

| # | Câu hỏi | Pass nếu |
|---|---------|----------|
| 1 | Init có per (Type+Rarity) không? | `InitializePerTypeRarityGroup` gọi từ `BuildRewardPoolFromDataTable` |
| 2 | Multiplier max trong nhóm, không cả city? | `CalculateMultipliers` chỉ qua `RunOnEachTypeRarityGroup` |
| 3 | Pick filter đúng nhóm? | `GetRewardPoolCandidates` check cả Type **và** Rarity |
| 4 | Push mark trên pool persistent? | `PushMarkForward` sửa slot trong `RewardPoolEntries` |
| 5 | Save load Mark/Recycle, recalc Multiplier? | `Sync` → `UpdateDistributionPerTypeRarityGroup` |
| 6 | Save cũ xử lý ok? | `Multiplier == 0` → skip, dùng init mới |
| 7 | Item weight 0 không pick? | `IsIgnoredItem` → `Multiplier <= 0` |
| 8 | Scale configurable? | `ControlledRandomnessScale` (default 0.1) truyền vào push |

### Điểm cần chú ý khi review

1. **Nhóm khớp flow roll** — Key `(Type, Rarity)` khớp filter trong `GetRewardPoolCandidates`, không chỉ Rarity.
2. **Không reset pool trong debug** — `ForceResetRewardPool` giữ API nhưng không gọi từ `ResetCityProgression` (pool tự cân bằng).
3. **Hai lớp random** — Lớp 1 (rarity %) và lớp 2 (controlled) độc lập; doc trên mô tả đúng flow post-race.

---

## Save / migration

| Tình huống | Hành vi |
|------------|---------|
| Save mới | Lưu `Mark`, `Recycle` per item (`Multiplier` có trong struct nhưng **recalc** khi load) |
| Save cũ (`Multiplier == 0`) | Skip load Mark/Recycle → giữ mark từ `InitializePerTypeRarityGroup` |
| Sau sync | `UpdateDistributionPerTypeRarityGroup` — recalc `Multiplier` per nhóm, giữ `Mark`/`Recycle` |
| `ForceResetRewardPool()` | Rebuild + init lại (API giữ, **không gọi** từ gameplay/debug) |

---

## API thay đổi (`URewardCenterSubsystem`)

| Bỏ | Thêm |
|----|------|
| `PickItemByProbabilityPercent` | `PickItemByLowestMark` |
| `IncrementPickedCountAndUpdateWeight` | `PushMarkForward` |
| `CalculateEffectiveWeight` | — |
| `RecalculateEffectiveWeight` | — |
| `DecayFactor` | `ControlledRandomnessScale` |

### API mới (`FControlledRandomness`)

| Hàm | Mục đích |
|-----|----------|
| `InitializePerTypeRarityGroup` | Init mark/multiplier per (Type, Rarity) |
| `UpdateDistributionPerTypeRarityGroup` | Recalc multiplier per nhóm sau load save |
| `Initialize` | Init 1 nhóm (gọi nội bộ) |
| `PickByLowestMark` | Chọn mark thấp nhất |
| `PushMarkForward` | Đẩy mark sau pick |
| `UpdateDistribution` | Recalc multiplier 1 nhóm (gọi nội bộ) |

---

## So sánh hành vi

| | Cũ (Decay) | Mới (Controlled Randomness) |
|--|-----------|----------------------------|
| Cơ chế | `BaseWeight × DecayFactor^TimesPicked` | Pick mark thấp nhất, push mark |
| Item pick nhiều | Weight → 0, gần "chết" | Mark tăng tạm, quay lại được |
| Init scope | Cả city (effective weight) | Per nhóm (Type + Rarity) |
| Cần reset cứu pool | Có (debug) | Không bắt buộc |
| Randomness | Roll % mỗi lần | Mark + multiplier + scale |
