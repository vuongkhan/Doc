# Cơ chế Random Map cho Quick Match (QM)

Tài liệu này mô tả chi tiết cách hệ thống Prototype Racing hiện tại xử lý việc chọn ngẫu nhiên map (đường đua) cho chế độ Quick Match, cũng như hướng dẫn cách cấu hình và thêm map mới vào map pool.

## 1. Luồng hoạt động chính (Nakama Go-Runtime)

Việc chọn random map cho chế độ Quick Match được xử lý hoàn toàn trên **Backend (Nakama Go-runtime)**. Client C++ hay Dedicated Server không tự random mà chỉ nhận URL map do Backend chỉ định.

Luồng hoạt động:
1. **Kích hoạt**: Khi một phòng Quick Match đã đủ người (`qm_rooms.go`), server tiến hành chuẩn bị map để chuyển người chơi vào trận đấu bằng cách gọi hàm `pickRandomVNTourMap()`.
2. **Xác định Map Pool**:
   - Hệ thống ưu tiên đọc cấu hình từ database Storage của Nakama (Collection: `system`, Key: `race_map_pool`).
   - Nếu trong database không có hoặc dữ liệu trống, server sẽ dùng hàm fallback `vnTourMapPool()` dựa theo biến môi trường `PR_QM_MAP_POOL` (được cấu hình lúc chạy server).
3. **Thuật toán Random**:
   - Nếu pool chỉ có 1 map, trả về map đó.
   - Nếu có nhiều map, server dùng hàm random an toàn (`crypto/rand`) sinh ra một byte ngẫu nhiên, sau đó chia lấy dư: `random_byte % số_lượng_map_trong_pool` để pick map.
4. **Khởi tạo trận đấu**: Đường dẫn (URL) map ngẫu nhiên vừa chọn được đưa vào payload của trận đấu. Dedicated Server (Unreal) sẽ nhận thông tin này và tự động load đúng map đó.

## 2. Cách thêm và sửa Map Pool

Hệ thống hỗ trợ cập nhật danh sách map linh hoạt (hot-reload không cần tắt server). Dưới đây là 3 cách để thay đổi hoặc thêm map mới vào tính năng Random Quick Match:

### Cách 1: Thông qua Nakama RPC (Code / API)
- Backend cung cấp một RPC có tên `prototype_update_map_pool` dành riêng cho Admin/Server.
- Gọi RPC này kèm theo payload định dạng JSON chứa danh sách URL của các map mới.
- Dữ liệu sẽ được ghi thẳng vào Storage của Nakama. Các trận Quick Match được tạo ngay sau đó sẽ tự động pick từ danh sách map mới này.

### Cách 2: Chỉnh sửa thủ công qua Nakama Developer Console
Đây là cách trực quan nhất để dev/admin thay đổi map pool:
1. Đăng nhập vào giao diện web của **Nakama Console**.
2. Chuyển sang tab **Storage**.
3. Tìm bản ghi (record) với thông tin sau:
   - Collection: `system`
   - Key: `race_map_pool`
4. Cập nhật file JSON bên trong bằng cách thêm đường dẫn URL của map mới.
   - *Ví dụ:* `"/Game/Maps/BanDoMoi/LV_Test?RaceMode=1"`

### Cách 3: Cấu hình qua Biến môi trường (Environment Variable)
Được dùng làm fallback mặc định khi Storage Nakama không có data:
- Tên biến môi trường: `PR_QM_MAP_POOL`
- Cú pháp: Cung cấp danh sách đường dẫn map ngăn cách nhau bởi dấu phẩy `,` hoặc chấm phẩy `;`.
- *Ví dụ:* `PR_QM_MAP_POOL="/Game/Maps/Map_Test_1?RaceMode=1,/Game/Maps/Map_Test_2?RaceMode=2"`
- **Lưu ý các giá trị đặc biệt**:
  - Bỏ trống hoặc điền `"vntour"`, `"all"`, `"full"`: Hệ thống sẽ lấy toàn bộ danh sách 21 map chuẩn (Sơn Trà, Downtown, NHS, Đông Giang, Hội An, Huế - mỗi nơi có 3 mode Circuit, Sprint, TA).
  - Điền `"safe"`, `"nightlight"`, `"validated"`: Chỉ random trong tập map an toàn mặc định (hiện là `RacingTest_NightLight_Features`).

## 3. Chi tiết Implementation Code

Dưới đây là các đoạn code thực thi cơ chế random map (nằm trong thư mục `Server/Nakama/go-runtime/`).

### 3.1. Kích hoạt Random khi phòng đầy (`qm_rooms.go`)
Khi Quick Match Room đầy người, hệ thống gọi `pickRandomVNTourMap` để lấy 1 URL map ngẫu nhiên trước khi tạo trận:
```go
// [qm_rooms.go] - Khi phòng đủ người (tryRoomHandoff)
// Room is full and about to travel: pick a random VN Tour map (pool from DT_Map / PR_QM_MAP_POOL). Falls back to PR_TARGET_MAP when the pool is empty.
raceMap := pickRandomVNTourMap(ctx, logger, nk, brokerCfg.TargetMap)
matchID, err := createHandoffMatch(ctx, logger, nk, roster, raceMap)
```

### 3.2. Thuật toán lấy Map Pool và chọn ngẫu nhiên (`main.go`)
Hàm `pickRandomVNTourMap` phụ trách lấy danh sách map và áp dụng random:
```go
// [main.go]
func pickRandomVNTourMap(ctx context.Context, logger runtime.Logger, nk runtime.NakamaModule, fallback string) string {
	// 1. Ưu tiên lấy pool từ Nakama Storage
	pool := fetchMapPoolFromStorage(ctx, logger, nk)
	if len(pool) == 0 {
		// 2. Nếu trống thì dùng fallback từ biến môi trường
		pool = vnTourMapPool()
	}
	if len(pool) == 0 {
		// Fallback cuối cùng
		fb := strings.TrimSpace(fallback)
		if fb != "" {
			return fb
		}
		return defaultTargetMap
	}
	if len(pool) == 1 {
		return pool[0] // Không cần random nếu chỉ có 1 map
	}
	
	// 3. Random an toàn bằng thư viện crypto
	var b [1]byte
	if _, err := rand.Read(b[:]); err != nil {
		return pool[0]
	}
	return pool[int(b[0])%len(pool)]
}
```

### 3.3. Đọc danh sách Map từ Storage (`main.go`)
```go
// [main.go]
func fetchMapPoolFromStorage(ctx context.Context, logger runtime.Logger, nk runtime.NakamaModule) []string {
	records, err := nk.StorageRead(ctx, []*runtime.StorageRead{
		{
			Collection: "system",
			Key:        "race_map_pool",
			UserID:     "", // System record
		},
	})
	// ... (Xử lý lỗi) ...
	if len(records) == 0 {
		return nil
	}

	var pool RaceMapPoolStorage
	if err := json.Unmarshal([]byte(records[0].Value), &pool); err != nil {
		return nil
	}
	return pool.Maps
}
```

### 3.4. Cơ chế RPC để hot-reload thêm Map (`main.go`)
Hàm RPC `rpcUpdateMapPool` cho phép đẩy payload JSON lên Nakama để ghi đè danh sách map hiện tại vào Database mà không cần tắt server:
```go
// [main.go]
func rpcUpdateMapPool(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, payload string) (string, error) {
	// ... (Check quyền Admin/Server) ...
	var req RaceMapPoolStorage
	if err := json.Unmarshal([]byte(payload), &req); err != nil {
		return "", runtime.NewError("Invalid JSON payload", 3)
	}

	value, err := json.Marshal(req)
	// Ghi vào Nakama Storage
	_, err = nk.StorageWrite(ctx, []*runtime.StorageWrite{
		{
			Collection:      "system",
			Key:             "race_map_pool",
			UserID:          "",
			Value:           string(value),
			PermissionRead:  2,
			PermissionWrite: 0,
		},
	})
	// ...
	return "{\"success\": true}", nil
}
```
