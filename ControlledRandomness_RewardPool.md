# Controlled Randomness — Reward Pool

## Tổng quan

Thay thuật toán pick reward cũ (decay weight) bằng **controlled randomness** — đảm bảo tỷ lệ drop theo `BaseWeight` trong dài hạn, tránh streak ngẫu nhiên quá lệch.

- **Thuật toán gốc:** Jason Frank & Malte Skarupke (2019) — [blog post](https://probablydance.com/2019/08/28/a-new-algorithm-for-controlled-randomness/)
- **Tham chiếu:** Radoshaka (Unity/C# port + QoL features)
- **UE implementation:** `FControlledRandomness`, tích hợp trong `URewardCenterSubsystem`

---

## Tóm tắt cơ chế

Mỗi item có **vị trí ảo** (`Mark`) trên một đường thẳng.

1. **Init:** `Multiplier = maxWeight / BaseWeight` — item weight thấp → multiplier cao → nhảy xa hơn mỗi lần.
2. **Pick:** Chọn item có `(Recycle, Mark)` **thấp nhất** trong candidates.
3. **Sau pick:** Đẩy mark item đó lên: `Mark += random(Scale..1) × Multiplier` → tạm thời khó bị pick lại.
4. **Wrap:** `Mark` quá lớn → `Recycle++`, mark quay vòng — roll mãi không overflow.

**Kết quả:** Vẫn random, nhưng item vừa pick nhiều sẽ lùi, item lâu chưa pick sẽ lên — tỷ lệ dài hạn theo `BaseWeight`, không streak lệch, không "chết" như decay.

**Reset:** `ForceResetRewardPool()` vẫn tồn tại (`BlueprintCallable`) nhưng **không được gọi** từ gameplay hay debug — pool tự cân, chỉ dùng thủ công khi cần fresh start (DT đổi, test, v.v.).

---

## Cách hoạt động

### Ý tưởng cốt lõi

Mỗi item trong pool được gán một **vị trí ảo** (`Mark`) trên một đường thẳng. Mỗi lần cần roll reward:

1. Chọn item có vị trí **thấp nhất** (đi chậm nhất → sắp đến lượt).
2. Sau khi pick, **đẩy** vị trí item đó lên phía trước (nhảy xa hơn → tạm thời khó bị pick lại).
3. Item có `BaseWeight` thấp hơn nhảy **xa hơn** mỗi lần → ít được pick hơn, nhưng vẫn đúng tỷ lệ dài hạn.

Khác với random thuần (`rand % totalWeight`), thuật toán này **có nhớ** — item vừa pick nhiều sẽ bị "đẩy lùi", item lâu chưa pick sẽ dần lên đầu.

### Bước 1 — Tính Multiplier (lúc init pool)

```
BiggestWeight = max(BaseWeight của tất cả item trong city pool)
Multiplier[i] = BiggestWeight / BaseWeight[i]
```

| Item | BaseWeight | Multiplier | Ý nghĩa |
|------|-----------|------------|---------|
| A | 60 | 1.0 | Weight cao nhất → nhảy ngắn nhất → pick nhiều nhất |
| B | 30 | 2.0 | Weight bằng nửa → nhảy gấp đôi → pick ít hơn ~2x |
| C | 0 | 0 (ignored) | Không tham gia roll |

Item có `BaseWeight <= 0` bị bỏ qua hoàn toàn.

### Bước 2 — Khởi tạo Mark

```
Mark[i] = random(0..1) × Multiplier[i]
Recycle[i] = 0
```

Mark ban đầu random để lần pick đầu tiên cũng công bằng, không luôn chọn cùng một item.

### Bước 3 — Pick (chọn item)

Trong danh sách candidates (đã filter type/rarity), chọn item có **(Recycle, Mark) nhỏ nhất**:

- So sánh `Recycle` trước (vòng wrap thấp hơn = đi trước).
- Nếu `Recycle` bằng nhau → so sánh `Mark`.

```
Ví dụ candidates:
  Item A: Recycle=0, Mark=1.2
  Item B: Recycle=0, Mark=0.8  ← pick (Mark thấp nhất)
  Item C: Recycle=0, Mark=3.5
```

### Bước 4 — Push Mark (sau khi pick)

Item vừa được pick sẽ nhảy về phía trước:

```
jump = random(Scale, 1.0) × Multiplier
Mark += jump
```

- `Scale` (`ControlledRandomnessScale`, default `0.1`): điều chỉnh độ ngẫu nhiên của bước nhảy.
  - `0` → jump gần cố định, spacing đều.
  - `1` → jump hoàn toàn random trong khoảng `[Multiplier, Multiplier]`.

Item weight thấp (Multiplier cao) nhảy xa hơn → tạm thời tụt xuống cuối hàng.

### Bước 5 — Recycle (wrap mark)

Khi `Mark >= 1,048,576` (`RecycleMark`):

```
Recycle += floor(Mark / RecycleMark)
Mark     = Mark mod RecycleMark
```

Cho phép mark chạy vô hạn mà không overflow float, vẫn so sánh đúng thứ tự qua cặp `(Recycle, Mark)`.

### Ví dụ đơn giản (2 item, Scale = 0.1)

Pool: **Red** (weight 50), **Blue** (weight 50) → cả hai Multiplier = 1.0.

```
Lần 1: Red Mark=0.3, Blue Mark=0.7  → pick Red  → Red Mark += ~0.5 → Red=0.8
Lần 2: Red Mark=0.8, Blue Mark=0.7  → pick Blue → Blue Mark += ~0.6 → Blue=1.3
Lần 3: Red Mark=0.8, Blue Mark=1.3  → pick Red  → ...
```

Kết quả: xen kẽ Red/Blue thay vì streak 5 Red liên tiếp như random thuần.

Pool lệch weight: **Common** (60) vs **Rare** (20) → Rare Multiplier = 3.0, mỗi lần pick Rare nhảy gấp 3 lần Common → Rare xuất hiện ~1/3 số lần Common trong dài hạn.

---

## Thay đổi struct `FRewardPoolEntryItem`

| Bỏ | Thêm |
|----|------|
| `TimesPicked` | `Mark` (float) — vị trí trên distribution line |
| `EffectiveWeight` | `Recycle` (int32) — số vòng wrap mark |
| | `Multiplier` (float, runtime) — `BiggestWeight / BaseWeight` |

**Giữ nguyên:** `PoolEntryID`, `CityID`, `ItemMasterID`, `BaseWeight`

`FCityLootPoolRow` và các struct reward khác **không đổi**.

---

## Files chính

| File | Vai trò |
|------|---------|
| `ControlledRandomness.h/.cpp` | Core algorithm: `Initialize`, `PickByLowestMark`, `PushMarkForward`, `UpdateDistribution` |
| `RewardCenterSubsystem.h/.cpp` | Tích hợp pool, save/load, roll reward |
| `RewardCenterSaveGame.h` | Lưu `RewardPoolEntries` (Mark + Recycle qua struct) |

---

## Flow pick item

```
BuildRewardPoolFromDataTable()
  └─ FControlledRandomness::Initialize()   // tính Multiplier, gán Mark ngẫu nhiên

RollItemFromRewardPool()
  └─ GetRewardPoolCandidates()             // filter type/rarity từ DataTable
  └─ PickItemByLowestMark()                // chọn item có (Recycle, Mark) nhỏ nhất

Sau khi grant reward
  └─ PushMarkForward(CityID, ItemMasterID) // đẩy mark item vừa pick
```

**Config:** `ControlledRandomnessScale` (default `0.1`) — `0` = spacing gần deterministic, `1` = random hơn.

---

## Save / migration

- **Lưu mới:** `Mark`, `Recycle`, `Multiplier` trên từng `FRewardPoolEntryItem`
- **Save cũ** (chỉ có `TimesPicked` / `EffectiveWeight`): `SyncRewardPoolEntries` bỏ qua entry có `Multiplier == 0` → dùng marks mới từ `Initialize()`
- `ForceResetRewardPool()` — rebuild pool + init marks từ đầu (API giữ lại, hiện không gọi từ code)

---

## API đã thay đổi

| Bỏ | Thêm |
|----|------|
| `PickItemByProbabilityPercent` | `PickItemByLowestMark` |
| `IncrementPickedCountAndUpdateWeight` | `PushMarkForward` |
| `CalculateEffectiveWeight` | — |
| `RecalculateEffectiveWeight` | — |
| `DecayFactor` | `ControlledRandomnessScale` |

---

## So sánh hành vi

| | Cũ (Decay) | Mới (Controlled Randomness) |
|--|-----------|----------------------------|
| Cơ chế | `EffectiveWeight = BaseWeight × DecayFactor^TimesPicked` | Pick mark thấp nhất, push mark sau mỗi lần pick |
| Item pick nhiều | Weight giảm vĩnh viễn | Mark tăng tạm thời, tỷ lệ dài hạn vẫn theo `BaseWeight` |
| Randomness | Roll % theo weight | Mark + scale + multiplier |
