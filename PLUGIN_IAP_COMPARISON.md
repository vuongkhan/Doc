# So sánh ClickStudioIAP vs MonetizationGoodies

Tài liệu tổng hợp so sánh hai plugin IAP trong project, bao gồm metadata, kiến trúc, cấu trúc thư mục, build integration, artifact zip và khuyến nghị sử dụng nội bộ.

---

## 1) Tổng quan nhanh

| Hạng mục | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| Tác giả | Click Studio | Nineva Studios |
| Version | 1.0.0 | 2.2.3 |
| Triết lý thiết kế | Unified shop API (1 Subsystem) | Low-level native wrappers |
| Entry point chính | `UClickStudioIAPSubsystem` | Nhiều class `MGAndroid*`, `MGIos*`, `MGIosStoreKit2*` |
| Public headers | 4 | 29 |
| Source files (không tính Intermediate/Binaries) | ~72 | ~90 |
| Có demo content | Không | Có (Levels + UI demo) |
| Trạng thái trong project | Plugin active | Giữ để tham chiếu / disabled |

---

## 2) Metadata plugin (`.uplugin`)

### ClickStudioIAP

- **Path:** `Plugins/ClickStudioIAP/ClickStudioIAP.uplugin`
- **FriendlyName:** Click Studio IAP
- **Description:** In-app purchases for Android (Google Play Billing) and iOS (StoreKit 2)
- **CanContainContent:** `false`
- **EngineVersion:** 5.6.0
- **Platforms:** Win64, Mac, Android, IOS

### MonetizationGoodies

- **Path:** `Plugins/Monetiza63ff578473cbV7/MonetizationGoodies.uplugin`
- **FriendlyName:** MonetizationGoodies
- **Description:** Handle in-app purchases and monetization with ease
- **CanContainContent:** `true`
- **EngineVersion:** 5.6.0
- **Platforms:** Win64, Mac, Android, IOS
- **Extras:** DocsURL, MarketplaceURL, SupportURL (plugin marketplace Nineva)

---

## 3) Kiến trúc runtime

### ClickStudioIAP

```
Blueprint / C++
  └── UClickStudioIAPSubsystem          (Public — GameInstanceSubsystem)
        └── ICSIAPPlatform              (Private — interface thống nhất)
              ├── FCSIAPPlatformAndroid
              ├── FCSIAPPlatformIOS
              └── FCSIAPPlatformStub
                    └── Legacy Bridge   (Private/Legacy — internal only)
                          ├── Android: CSIAPAndroidPlatformBridge
                          └── iOS:     CSIAPIosPlatformBridge
                                └── StoreKit2 wrappers
```

**Đặc điểm:**

- Game logic chỉ cần biết Subsystem + unified types (`FCSIAPProduct`, `FCSIAPPurchase`, ...).
- Legacy wrappers (`CSIAPAndroidBillingClient`, `CSIAPIosStoreKitManager`, ...) nằm trong `Private/Legacy/*`, không expose ra Blueprint.
- Có product cache, owned purchases cache, auto-init queue, auto-finish policy.

### MonetizationGoodies

```
Blueprint / C++
  └── Trực tiếp gọi wrapper objects
        ├── Android: UMGAndroidBillingClient, UMGAndroidBillingLibrary, ...
        ├── iOS StoreKit1: UMGIosPaymentQueue, UMGIosProduct, ...
        └── iOS StoreKit2: UMGIosStoreKit2StoreKitManager, ...
              └── Java/ObjC bridge
```

**Đặc điểm:**

- Expose rộng toàn bộ native wrapper ra `Public/*`.
- Dev tự orchestrate flow: connect billing client → query products → launch flow → consume/acknowledge.
- Không có subsystem-level automation.

---

## 4) Cấu trúc thư mục chi tiết

### ClickStudioIAP

```text
Plugins/ClickStudioIAP/
├── ClickStudioIAP.uplugin
├── Config/
│   └── FilterPlugin.ini
├── Docs/
│   ├── CLICKSTUDIO_IAP_API.md
│   ├── CLICKSTUDIO_IAP_CHANGES_FROM_MONETIZATION.md
│   ├── CLICKSTUDIO_IAP_PHASE2_PLAN.md
│   ├── CLICKSTUDIO_IAP_VS_MONETIZATIONGOODIES.md
│   ├── MONETIZATION_GOODIES_ANALYSIS.md
│   └── PLUGIN_IAP_COMPARISON.md          ← tài liệu này
└── Source/
    ├── ClickStudioIAP/
    │   ├── ClickStudioIAP.Build.cs
    │   ├── ClickStudioIAP_Android_UPL.xml
    │   ├── ClickStudioIAP_IOS_UPL.xml
    │   ├── Public/                        ← 4 headers (API duy nhất cho game)
    │   │   ├── ClickStudioIAP.h
    │   │   ├── ClickStudioIAPSubsystem.h
    │   │   ├── ClickStudioIAPTypes.h
    │   │   └── ClickStudioIAPSettings.h
    │   └── Private/
    │       ├── ClickStudioIAP.cpp
    │       ├── ClickStudioIAPSubsystem.cpp
    │       ├── ClickStudioIAPSettings.cpp
    │       ├── ClickStudioIAPLog.h
    │       ├── Platform/                  ← Unified platform adapters
    │       │   ├── ICSIAPPlatform.h
    │       │   ├── CSIAPPlatformFactory.cpp
    │       │   ├── CSIAPPlatformAndroid.*
    │       │   ├── CSIAPPlatformIOS.*
    │       │   ├── CSIAPPlatformStub.*
    │       │   ├── CSIAPAndroidConverters.*
    │       │   └── CSIAPIosConverters.*
    │       ├── Legacy/                    ← Internalized legacy bridge (không public)
    │       │   ├── Android/
    │       │   │   ├── CSIAPAndroidPlatformBridge.*
    │       │   │   ├── CSIAPAndroidBillingLibrary.*
    │       │   │   ├── CSIAPAndroidBillingClient.*
    │       │   │   ├── CSIAPAndroidBillingFlowParameters.*
    │       │   │   ├── CSIAPAndroidBillingResult.*
    │       │   │   ├── CSIAPAndroidProductDetails.*
    │       │   │   ├── CSIAPAndroidPurchase.*
    │       │   │   └── CSIAPAndroidWrapperObject.*
    │       │   └── StoreKit2/
    │       │       ├── CSIAPIosPlatformBridge.*
    │       │       ├── CSIAPIosStoreKitManager.*
    │       │       ├── CSIAPIosStoreKit2Product.*
    │       │       ├── CSIAPIosStoreKit2Transaction.*
    │       │       ├── CSIAPIosStoreKit2SubscriptionInfo.*
    │       │       ├── CSIAPIosStoreKit2SubscriptionOffer.*
    │       │       ├── CSIAPIosStoreKit2SubscriptionPeriod.*
    │       │       ├── CSIAPIosStoreKit2PurchaseOption.*
    │       │       └── CSIAPIosWrapperObject.*
    │       ├── Android/
    │       │   ├── Java/
    │       │   │   └── CSBilling.java
    │       │   └── Utils/
    │       │       ├── CSIAPJavaConvertor.*
    │       │       └── CSIAPMethodCallUtils.*
    │       └── IOS/
    │           ├── Bridge/
    │           │   └── CSIAPStoreKitManagerDelegateProxy.*
    │           └── Utils/
    │               └── CSIAPIosUtils.*
    └── ThirdParty/
        └── IOS/
            └── StoreKitWrapper.embeddedframework.zip
```

### MonetizationGoodies

```text
Plugins/Monetiza63ff578473cbV7/
├── MonetizationGoodies.uplugin
├── Content/                               ← Demo levels + UI (CSIAP không có)
│   ├── Levels/
│   │   ├── MonetizationGoodiesDemo.umap
│   │   ├── MonetizationGoodiesDemoAndroid.umap
│   │   ├── MonetizationGoodiesDemoIOS.umap
│   │   └── MonetizationGoodiesDemoIOS-StokeKit2.umap
│   └── UI/
│       ├── UI_MonetizationDemoAndroid.uasset
│       ├── UI_MonetizationDemoIOS.uasset
│       └── UI_MonetizationDemoIOS-StoreKit2.uasset
├── Resources/
│   └── Icon128.png
└── Source/
    ├── MonetizationGoodies/
    │   ├── MonetizationGoodies.Build.cs
    │   ├── MonetizationGoodies_Android_UPL.xml
    │   ├── MonetizationGoodies_IOS_UPL.xml
    │   ├── Public/                        ← 29 headers (expose rộng)
    │   │   ├── MonetizationGoodies.h
    │   │   ├── MonetizationGoodiesSettings.h
    │   │   ├── MGAndroidBillingClient.h
    │   │   ├── MGAndroidBillingLibrary.h
    │   │   ├── MGAndroidBillingFlowParameters.h
    │   │   ├── MGAndroidBillingResult.h
    │   │   ├── MGAndroidProductDetails.h
    │   │   ├── MGAndroidPurchase.h
    │   │   ├── MGAndroidPurchaseHistoryRecord.h
    │   │   ├── MGAndroidSkuDetails.h
    │   │   ├── MGAndroidWrapperObject.h
    │   │   ├── MGIosPayment.h
    │   │   ├── MGIosPaymentQueue.h
    │   │   ├── MGIosPaymentTransaction.h
    │   │   ├── MGIosPaymentDiscount.h
    │   │   ├── MGIosProduct.h
    │   │   ├── MGIosProductDiscount.h
    │   │   ├── MGIosProductsRequest.h
    │   │   ├── MGIosStoreFront.h
    │   │   ├── MGIosTransactionObserver.h
    │   │   ├── MGIosDownload.h
    │   │   ├── MGIosWrapperObject.h
    │   │   ├── MGStoreKit2Example.h
    │   │   └── StoreKit2/
    │   │       ├── MGIosStoreKit2StoreKitManager.h
    │   │       ├── MGIosStoreKit2Product.h
    │   │       ├── MGIosStoreKit2Transaction.h
    │   │       ├── MGIosStoreKit2SubscriptionInfo.h
    │   │       ├── MGIosStoreKit2SubscriptionOffer.h
    │   │       ├── MGIosStoreKit2SubscriptionPeriod.h
    │   │       └── MGIosStoreKit2PurchaseOption.h
    │   └── Private/
    │       ├── MonetizationGoodies.cpp
    │       ├── MGAndroid*.cpp               ← Android wrappers (public headers)
    │       ├── MGIos*.cpp                   ← iOS StoreKit1 wrappers
    │       ├── MGStoreKit2Example.cpp
    │       ├── StoreKit2/                   ← StoreKit2 implementation
    │       ├── Java/
    │       │   └── MGBilling.java
    │       ├── Android/Utils/
    │       │   ├── MGJavaConvertor.*
    │       │   └── MGMethodCallUtils.*
    │       └── IOS/
    │           ├── MGPaymentQueueDelegate.*
    │           ├── MGProductsRequestDelegate.*
    │           ├── MGTransactionObserverDelegate.*
    │           ├── StoreKit2/
    │           │   └── MGStoreKit2ManagerDelegateProxy.*
    │           └── Utils/
    │               └── MGIosUtils.*
    └── ThirdParty/
        └── IOS/
            └── StoreKitWrapper.embeddedframework.zip
```

### So sánh cấu trúc thư mục

| Khía cạnh | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| Public surface | 4 headers gọn | 29 headers rộng |
| Legacy/internal | `Private/Legacy/*` (ẩn) | Không có — wrappers ở Public |
| Platform layer | `Private/Platform/*` (unified) | Không có — gọi trực tiếp wrappers |
| Demo content | Không | 4 levels + 3 UI widgets |
| Java bridge | `Private/Android/Java/CSBilling.java` | `Private/Java/MGBilling.java` |
| ThirdParty zip | `Source/ThirdParty/IOS/StoreKitWrapper.embeddedframework.zip` | Cùng path tương đối |

---

## 5) So sánh API Blueprint / C++

### ClickStudioIAP — High-level unified API

**Functions:**

| Function | Mô tả |
|---|---|
| `InitializeIAP` | Khởi tạo kết nối store |
| `FetchProducts` | Lấy thông tin sản phẩm theo ProductIds |
| `Purchase` | Mua sản phẩm với options |
| `RestorePurchases` | Khôi phục entitlement |
| `FinishTransaction` | Hoàn tất transaction (consume/ack/finish) |
| `GetOwnedPurchases` | Danh sách purchase đang sở hữu |
| `GetCachedProduct` | Lấy product đã cache |
| `IsInitialized` | Kiểm tra trạng thái init |

**Events:**

| Event | Payload |
|---|---|
| `OnInitialized` | `bool bSuccess` |
| `OnProductsFetched` | `TArray<FCSIAPProduct>` |
| `OnPurchaseResult` | `FCSIAPPurchaseResult` (Pending/Completed/Failed/Cancelled/Deferred) |
| `OnTransactionUpdated` | `FCSIAPTransaction` |
| `OnRestoreCompleted` | `TArray<FCSIAPPurchase>` |

**Unified types:** `FCSIAPProduct`, `FCSIAPPurchase`, `FCSIAPTransaction`, `FCSIAPError`, `FCSIAPPurchaseResult`, `FCSIAPPurchaseOptions`

### MonetizationGoodies — Low-level wrapper API

Dev phải tự lắp flow từ nhiều node:

- **Android:** `CreateAndroidBillingClient` → `StartConnection` → `QueryProductDetails` → `LaunchBillingFlow` → `Consume` / `AcknowledgePurchase`
- **iOS StoreKit1:** `PaymentQueue` → `ProductsRequest` → `AddPayment` → `TransactionObserver`
- **iOS StoreKit2:** `StoreKit2Manager` → `FetchProducts` → `Purchase` → `FinishTransaction`

Không có unified struct cho cả Android + iOS theo style shop-level.

---

## 6) Settings

| Setting | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| Class | `UClickStudioIAPSettings` (DeveloperSettings) | `UMonetizationGoodiesSettings` (UObject, rỗng) |
| Auto init on startup | `bAutoInitOnStartup = true` | Không có |
| Auto finish transactions | `bAutoFinishTransactions = true` | Không có |
| Verbose logging | `bEnableVerboseLogging` | Không có |
| Android auto reconnect | `bAutoReconnect`, `ReconnectDelaySeconds` | Không có |

---

## 7) Build integration

### Module dependencies

| Module | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| Core | Public | Public |
| CoreUObject, Engine, Projects | Private | Private |
| DeveloperSettings | Private | Không |
| Launch (Android) | Public | Public |

### Android UPL

| Hạng mục | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| Java package | `com.clickstudio.iap` | `com.ninevastudios.monetization` |
| Java class | `CSBilling.java` | `MGBilling.java` |
| Billing dependency | `billing:8.0.0` | `billing:8.0.0` |
| AndroidX/Jetifier | Có | Có |
| Proguard keep rules | `com.clickstudio.**` | `com.ninevastudios.**` |
| AndroidX migration block | Không | Có (`baseBuildGradleAdditions` rewrite imports) |

### iOS build

| Hạng mục | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| StoreKit framework | Có | Có |
| StoreKitWrapper zip | Có (với logic co-exist MG) | Có (luôn unzip) |
| Co-exist logic | Detect MG enabled → dùng shared extracted framework | Không có |

---

## 8) File zip — ý nghĩa và vị trí

Cả hai plugin đều có file:

```
Source/ThirdParty/IOS/StoreKitWrapper.embeddedframework.zip
```

### StoreKitWrapper.embeddedframework.zip là gì?

- **Embedded framework** chứa Objective-C/Swift bridge code để gọi StoreKit 2 APIs từ C++ Unreal.
- UBT (Unreal Build Tool) **tự động unzip** file này vào `Engine/Intermediate/UnzippedFrameworks/StoreKitWrapper/` khi build iOS.
- Framework này wrap các API StoreKit 2 (products, transactions, subscriptions) thành interface mà C++ plugin có thể gọi qua delegate proxy.

### Vì sao cả hai plugin dùng cùng zip?

- Cả hai plugin đều cần StoreKit 2 bridge cho iOS.
- Zip này là artifact native (ObjC framework) do Nineva/Click Studio ship kèm plugin.
- **Xung đột:** UBT không thể register hai unzip action vào cùng path `Engine/Intermediate/UnzippedFrameworks/StoreKitWrapper`.
- **Giải pháp CSIAP:** Nếu MG đang bật, CSIAP detect và dùng shared extracted framework thay vì unzip lại.

### Khi nào cần refresh zip?

- Khi Apple cập nhật StoreKit API yêu cầu bridge mới.
- Khi cần thêm/bỏ capability trong framework (ví dụ: subscription offers, family sharing).
- Khi muốn tách zip riêng cho CSIAP (đổi tên framework, đổi bundle ID) để không phụ thuộc artifact MG.

---

## 9) Mapping class MG → CSIAP (legacy layer)

ClickStudioIAP giữ legacy wrappers ở `Private/Legacy/*` với naming tương ứng:

| MonetizationGoodies (Public) | ClickStudioIAP (Private/Legacy) |
|---|---|
| `UMGAndroidBillingLibrary` | `UCSIAPAndroidBillingLibrary` |
| `UMGAndroidBillingClient` | `UCSIAPAndroidBillingClient` |
| `UMGAndroidBillingFlowParameters` | `UCSIAPAndroidBillingFlowParameters` |
| `UMGAndroidBillingResult` | `UCSIAPAndroidBillingResult` |
| `UMGAndroidProductDetails` | `UCSIAPAndroidProductDetails` |
| `UMGAndroidPurchase` | `UCSIAPAndroidPurchase` |
| `UMGAndroidWrapperObject` | `UCSIAPAndroidWrapperObject` |
| `MGBilling.java` | `CSBilling.java` |
| `UMGIosStoreKit2StoreKitManager` | `UCSIAPIosStoreKitManager` |
| `UMGIosStoreKit2Product` | `UCSIAPIosStoreKit2Product` |
| `UMGIosStoreKit2Transaction` | `UCSIAPIosStoreKit2Transaction` |
| `UMGIosStoreKit2SubscriptionInfo` | `UCSIAPIosStoreKit2SubscriptionInfo` |
| `UMGIosWrapperObject` | `UCSIAPIosWrapperObject` |
| `MGStoreKit2ManagerDelegateProxy` | `CSIAPStoreKitManagerDelegateProxy` |
| `MGJavaConvertor` | `CSIAPJavaConvertor` |
| `MGMethodCallUtils` | `CSIAPMethodCallUtils` |
| `MGIosUtils` | `CSIAPIosUtils` |

**Lưu ý:** CSIAP không expose StoreKit1 wrappers (`MGIosPaymentQueue`, `MGIosProduct`, ...) — chỉ dùng StoreKit2 path.

---

## 10) Luồng chức năng so sánh

### Purchase flow

| Bước | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| 1. Init | `InitializeIAP()` (auto hoặc manual) | Dev tự `CreateBillingClient` + `StartConnection` |
| 2. Fetch products | `FetchProducts(ProductIds)` | Dev tự `QueryProductDetails` theo sku type |
| 3. Purchase | `Purchase(ProductId, Options)` | Dev tự build `BillingFlowParams` + `LaunchBillingFlow` |
| 4. Nhận kết quả | `OnPurchaseResult` (1 callback, enum status) | Nhiều callback rời (`OnPurchasesUpdated`, `OnLaunchBillingFlowStarted`, ...) |
| 5. Grant reward | Khi `Status == Completed` | Dev tự quyết định timing |
| 6. Finish | Auto (trừ Android one-time) hoặc `FinishTransaction` | Dev tự gọi `Consume` / `AcknowledgePurchase` |

### Restore flow

| Bước | ClickStudioIAP | MonetizationGoodies |
|---|---|---|
| Gọi restore | `RestorePurchases()` | Dev tự `QueryPurchases` theo từng `EMGSkuType` |
| Kết quả | `OnRestoreCompleted(Purchases[])` | Dev tự merge và xử lý entitlement |
| Android query | Built-in query cả `inapp` + `subs` | Dev query từng loại riêng |

---

## 11) Ưu / Nhược từng plugin

### ClickStudioIAP

**Ưu:**

- API gọn, dễ onboard dev mới.
- Unified data model cho Android + iOS.
- Event purchase chuẩn hóa (`OnPurchaseResult` với enum status).
- Auto-init, auto-finish, product cache built-in.
- Legacy code ẩn — Blueprint không thấy node low-level.
- Settings phong phú hơn (auto-init, auto-finish, reconnect).

**Nhược:**

- Ít low-level control hơn nếu cần behavior rất custom.
- Vẫn phụ thuộc legacy internal bridge ở runtime.
- Chưa có demo content.
- Chưa có low-level API surface (đang trong kế hoạch Phase 2).

### MonetizationGoodies

**Ưu:**

- Full low-level control qua native wrappers.
- Expose cả StoreKit1 và StoreKit2.
- Có demo levels + UI widgets sẵn.
- Marketplace plugin với docs/support chính thức.

**Nhược:**

- Blueprint graph dễ rối (nhiều node, nhiều callback).
- Dev phải tự ráp flow, dễ sai logic.
- Không có unified struct cho cross-platform shop flow.
- Settings rỗng — không có automation.

---

## 12) Kết luận và khuyến nghị nội bộ

### Khi nào dùng ClickStudioIAP

- Production shop flow (coins, no-ads, VIP, subscription).
- Team gameplay/UI cần API đơn giản, ít node Blueprint.
- Cần chuẩn hóa purchase/restore flow across platforms.
- Đang migration từ MG sang CSIAP.

### Khi nào tham chiếu MonetizationGoodies

- Cần hiểu low-level native API behavior.
- Debug sâu billing client / StoreKit callback.
- Tham khảo demo content hoặc StoreKit1 flow (CSIAP không expose SK1).

### Trạng thái hiện tại

- **ClickStudioIAP** là lớp refactor + productization từ nền MonetizationGoodies.
- Chức năng core (init, fetch, purchase, restore, finish) tương đương MG nhưng qua API gọn hơn.
- Legacy bridge vẫn còn ở `Private/Legacy/*` — kế hoạch rewrite để tách hoàn toàn khỏi pattern MG đang được lên (xem `CLICKSTUDIO_IAP_PHASE2_PLAN.md`).

### Khuyến nghị

1. Dùng **ClickStudioIAP** cho production.
2. Giữ **MonetizationGoodies disabled** để tham chiếu.
3. Nếu cần feature low-level chưa có trong CSIAP, bổ sung vào CSIAP thay vì mở lại MG nodes.
4. Refresh `StoreKitWrapper.embeddedframework.zip` khi cần tách artifact riêng cho CSIAP.
