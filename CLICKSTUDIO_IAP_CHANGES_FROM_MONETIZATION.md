# ClickStudioIAP - Những thay đổi so với MonetizationGoodies

Tài liệu này ghi lại **các thay đổi thực tế trong codebase hiện tại** khi so `ClickStudioIAP` với `MonetizationGoodies` (plugin gốc của Nineva).

## 1) Thay đổi về kiến trúc tổng thể

### Từ mô hình low-level wrapper sang mô hình Subsystem thống nhất

- `MonetizationGoodies`: expose rất nhiều class native wrapper trực tiếp ra public API (Android + iOS + StoreKit2).
- `ClickStudioIAP`: gom về một entry point duy nhất `UClickStudioIAPSubsystem` để game logic gọi từ Blueprint/C++.

Kết quả:

- Giảm độ phức tạp graph Blueprint.
- Chuẩn hóa flow mua hàng/restore giữa Android và iOS.
- Che giấu phần legacy bridge vào `Private/Legacy/*` thay vì lộ ra public surface.

---

## 2) Thay đổi bề mặt API public (rất lớn)

### Số lượng public headers

- `ClickStudioIAP`: **4** file public header
  - `ClickStudioIAP.h`
  - `ClickStudioIAPSubsystem.h`
  - `ClickStudioIAPTypes.h`
  - `ClickStudioIAPSettings.h`
- `MonetizationGoodies`: **29** file public header (nhiều wrapper class Android/iOS/StoreKit2).

### API theo hướng gameplay/shop thay vì native primitives

`ClickStudioIAPSubsystem` cung cấp các hàm cấp cao:

- `InitializeIAP`
- `FetchProducts`
- `Purchase`
- `RestorePurchases`
- `FinishTransaction`
- `GetOwnedPurchases`
- `GetCachedProduct`
- `IsInitialized`

và event thống nhất:

- `OnInitialized`
- `OnProductsFetched`
- `OnPurchaseResult`
- `OnTransactionUpdated`
- `OnRestoreCompleted`

Trong khi đó, `MonetizationGoodies` yêu cầu lắp ghép flow từ các object low-level (billing client, params, storekit managers, ...).

---

## 3) Unified data model mới (thay cho model phân tán theo platform)

`ClickStudioIAP` đưa vào bộ kiểu thống nhất:

- `FCSIAPProduct`
- `FCSIAPPurchase`
- `FCSIAPTransaction`
- `FCSIAPError`
- `FCSIAPPurchaseResult`
- `FCSIAPPurchaseOptions`

với enum thống nhất:

- `ECSIAPProductType`
- `ECSIAPPurchaseState`
- `ECSIAPInitializationState`
- `ECSIAPPurchaseResultStatus`

Điểm khác biệt chính:

- Trước đây (`MonetizationGoodies`) dev làm việc với object wrappers khác nhau theo Android/iOS.
- Nay (`ClickStudioIAP`) game logic làm việc trên cùng một data contract cho cả 2 nền tảng.

---

## 4) Thay đổi hành vi runtime và lifecycle

### Bổ sung cơ chế auto-init + hàng đợi callback chờ init

`ClickStudioIAPSubsystem` có:

- `bAutoInitOnStartup` (qua `UDeveloperSettings`)
- `EnsureInitialized(...)` + `PendingReadyCallbacks`

=> Các lệnh `FetchProducts/Purchase/Restore` có thể tự chờ IAP ready thay vì fail cứng nếu chưa init.

### Chuẩn hóa purchase result về một callback duy nhất

`OnPurchaseResult` trả `FCSIAPPurchaseResult` với status:

- `Pending`
- `Completed`
- `Failed`
- `Cancelled`
- `Deferred`

Điểm mới:

- Có `ActivePurchaseProductId` để map lỗi/trạng thái với request hiện tại.
- Có logic `ResolvePurchaseResultStatus(...)` để convert trạng thái platform về enum dùng chung.

### Thêm auto-finish có điều kiện

Qua setting `bAutoFinishTransactions`:

- Tự finish cho các trường hợp phù hợp.
- **Không tự finish Android one-time (`ECSIAPProductType::OneTime`)** để game tự quyết định consume/ack theo business logic.

---

## 5) Thay đổi build/integration (ModuleRules)

So với `MonetizationGoodies.Build.cs`, `ClickStudioIAP.Build.cs` có các thay đổi đáng chú ý:

- Thêm dependency `DeveloperSettings`.
- Mở rộng private include cho các layer:
  - `Private/Platform`
  - `Private/Legacy/Android`
  - `Private/Legacy/StoreKit2`
  - `Private/Legacy/IOS`
- Thêm logic detect plugin cùng tồn tại:
  - `IsPluginEnabledForTarget("MonetizationGoodies", Target)`
  - Tránh xung đột unzip `StoreKitWrapper.embeddedframework.zip` khi 2 plugin cùng bật.
- Nếu `MonetizationGoodies` đang bật trên iOS, `ClickStudioIAP` dùng shared extracted framework từ `Engine/Intermediate/UnzippedFrameworks/...`.

Ý nghĩa:

- Có lớp tương thích để chuyển đổi dần từ MG sang CSIAP trong cùng project.
- Giảm rủi ro build conflict khi chưa tắt hẳn plugin cũ.

---

## 6) Thay đổi Android UPL

### Namespace/package Java đổi từ Nineva sang Click Studio

- `MonetizationGoodies`: copy Java vào `com/ninevastudios/monetization`
- `ClickStudioIAP`: copy Java vào `com/clickstudio/iap`

### Proguard rules đổi theo namespace mới

- Từ `com.ninevastudios.**` sang `com.clickstudio.**`

### Giữ lại dependency billing core

- Cả hai đang dùng: `com.android.billingclient:billing:8.0.0`

### Lược bỏ block rewrite AndroidX mapping cũ

`MonetizationGoodies_Android_UPL.xml` có `baseBuildGradleAdditions` để duyệt `.java` và replace import support libs -> AndroidX.

`ClickStudioIAP_Android_UPL.xml` **không còn block này**, chỉ giữ phần cần thiết (copy Java, gradleProperties, proguard, dependency).

---

## 7) Thay đổi iOS UPL

- `ClickStudioIAP_IOS_UPL.xml` rất gọn (init log), còn xử lý framework chính nằm trong Build.cs.
- Vẫn dùng `StoreKit` framework và `StoreKitWrapper` như nền tảng tương thích StoreKit2 bridge.

---

## 8) Thay đổi metadata plugin (`.uplugin`)

### ClickStudioIAP

- `FriendlyName`: `Click Studio IAP`
- `VersionName`: `1.0.0`
- `Description`: tập trung Android Google Billing + iOS StoreKit2
- `CanContainContent`: `false`
- `CreatedBy`: `Click Studio`

### MonetizationGoodies

- `FriendlyName`: `MonetizationGoodies`
- `VersionName`: `2.2.3`
- Có đầy đủ URL docs/marketplace/support của Nineva
- `CanContainContent`: `true`

---

## 9) Mapping nhanh: MG -> CSIAP

- `MGAndroidBillingClient` / `MGAndroidBillingLibrary` / `MGIos*` wrappers  
  -> gọi qua `UClickStudioIAPSubsystem`
- `Query theo sku type + parse thủ công`  
  -> `FetchProducts` + nhận `FCSIAPProduct[]`
- `Purchase callback phân tán`  
  -> `OnPurchaseResult` (1 kênh chuẩn)
- `Manual restore orchestration`  
  -> `RestorePurchases` + `OnRestoreCompleted`
- `Manual finish all cases`  
  -> `bAutoFinishTransactions` + manual cho Android one-time khi cần

---

## 10) Kết luận thay đổi cốt lõi

`ClickStudioIAP` là một lớp tái cấu trúc mạnh từ `MonetizationGoodies`:

- Giữ phần bridge/native cần thiết ở private legacy layer.
- Thu gọn public API theo hướng product/gameplay team dễ dùng.
- Bổ sung cơ chế lifecycle/compatibility để chạy ổn trong project đang chuyển đổi.
- Giảm đáng kể coupling của gameplay code với platform-specific wrappers.

---

## 11) Kết luận sự khác nhau giữa hai plugin

### Khi nhìn từ góc độ team gameplay

- `ClickStudioIAP` phù hợp hơn cho production flow cần ổn định và dễ maintain: một subsystem, một bộ event, một data model.
- `MonetizationGoodies` phù hợp khi cần can thiệp sâu low-level vào từng API native và chấp nhận độ phức tạp cao hơn ở Blueprint/C++.

### Khác nhau cốt lõi theo mục tiêu thiết kế

- `MonetizationGoodies` ưu tiên **độ mở low-level** (nhiều wrapper public, dev tự ráp flow).
- `ClickStudioIAP` ưu tiên **chuẩn hóa high-level** (ẩn legacy internals, tập trung vào use case shop thực tế).

### Tác động thực tế khi vận hành dự án

- Với `ClickStudioIAP`, chi phí onboarding dev mới thấp hơn do API surface nhỏ.
- Với `MonetizationGoodies`, flexibility cao hơn nhưng dễ phát sinh sai khác logic giữa các màn/shop flow nếu không chuẩn hóa nội bộ.
- Việc `ClickStudioIAP` có compatibility logic build với MG giúp migration theo từng giai đoạn thay vì bắt buộc cut-over một lần.

---

## 12) Cấu trúc thư mục hai plugin (để đối chiếu nhanh)

### ClickStudioIAP

```text
Plugins/ClickStudioIAP/
  ClickStudioIAP.uplugin
  Docs/
  Source/ClickStudioIAP/
    ClickStudioIAP.Build.cs
    ClickStudioIAP_Android_UPL.xml
    ClickStudioIAP_IOS_UPL.xml
    Public/
      ClickStudioIAP.h
      ClickStudioIAPSubsystem.h
      ClickStudioIAPTypes.h
      ClickStudioIAPSettings.h
    Private/
      ClickStudioIAPSubsystem.cpp
      ClickStudioIAPSettings.cpp
      Platform/                   # Unified platform adapters
      Legacy/Android/             # Internalized Android legacy bridge/wrappers
      Legacy/StoreKit2/           # Internalized StoreKit2 legacy bridge/wrappers
      Legacy/IOS/
      Android/Utils/
      IOS/Bridge/
      IOS/Utils/
```

### MonetizationGoodies

```text
Plugins/Monetiza63ff578473cbV7/
  MonetizationGoodies.uplugin
  Source/MonetizationGoodies/
    MonetizationGoodies.Build.cs
    MonetizationGoodies_Android_UPL.xml
    MonetizationGoodies_IOS_UPL.xml
    Public/
      MGAndroid*.h               # Billing client/library/result/purchase/details...
      MGIos*.h                   # StoreKit1 wrappers
      StoreKit2/*.h              # StoreKit2 wrappers/managers
      MonetizationGoodies.h
      MonetizationGoodiesSettings.h
    Private/
      MGAndroid*.cpp
      MGIos*.cpp
      StoreKit2/*.cpp
      Android/Utils/
      IOS/
```

Ghi chú: điểm khác biệt nổi bật nhất về cấu trúc là `ClickStudioIAP` đẩy phần wrapper cũ vào `Private/Legacy/*`, còn `MonetizationGoodies` để nhiều wrapper ở `Public/*`.

