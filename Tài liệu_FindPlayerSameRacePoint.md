# Cách tìm player có cùng RacePoint (Quick Match)

## Ý tưởng

Khi bấm **Quick Match**, server ghép người chơi theo **RacePoint (RP)** — không tin RP do client gửi. RP lấy từ **profile trên Nakama storage**. Hai player “cùng tầm RP” nếu chênh lệch RP trong room nằm trong **window** cho phép.

---

## Flow ngắn

```text
Player ấn Quick Match
        │
        ▼
Client RPC prototype_qm_join
        │
        ▼
Server đọc RP thật (profile) + username
        │
        ├─ Đã trong room? → trả roster hiện tại
        │
        ├─ Phase A: tìm room còn chỗ, sau khi thêm:
        │     (maxRP − minRP) ≤ RPBaseWindow
        │   Ưu tiên: room đầy hơn → RP gần “tâm” room hơn
        │
        ├─ Phase B: room mở rộng theo min-window
        │     (maxRP − minRP) ≤ min(window của mọi member + người mới)
        │
        └─ Không có room phù hợp → tạo room mới (solo)
        │
        ▼
Notify roster (code 220): tên + RP cho mọi người trong room
        │
        ▼
Đủ N người (mặc định 4) → handoff match (code 221)
```

---

## Điều kiện “cùng RP / cùng room”

| Khái niệm | Ý nghĩa |
|-----------|---------|
| **spread** | `maxRP − minRP` trong room (sau khi thêm người) |
| **window** | Dung sai cho phép; nở dần theo thời gian chờ |
| **accept** | `spread ≤ window` → được vào cùng room |

- **Cùng RP** (spread = 0): luôn fit mọi window → join ngay nếu còn ghế.
- **RP gần** (vd 300 vs 320): fit base window → cùng room.
- **RP xa** (vd 300 vs 1500): room riêng; chờ lâu window nở mới có thể expand.

### Ai quyết định window khi mở rộng?

**Thằng ấn Quick Match trễ nhất** trong nhóm (JoinedAt mới nhất) → window thấp nhất vì window nở theo thời gian chờ. Hàm: `groupWindowLatestJoiner` trong `qm_rooms.go`.

---

## Expand (solo only)

Mỗi ~2s server quét:

- Chỉ **solo** (room 1 người) được kéo vào room khác còn ghế nếu RP fit min-window.

---

## Client làm gì

| Việc | Chi tiết |
|------|----------|
| Join | `JoinQuickMatchRoom` → RPC `prototype_qm_join` |
| Leave | `LeaveQuickMatchRoom` → RPC `prototype_qm_leave` |
| UI phòng chờ | Notify **220** → list `Tên \| RP`; **local luôn slot 0 (YOU)** |
| Bắt đầu trận | Notify **221** → `JoinHandoffMatch` |


---

## Ví dụ

```text
A (RP 300) QM → room mới
B (RP 300) QM → join room A
C (RP 320) QM → join room A  (spread trong base window)
D (RP 1500) QM → room riêng

Sau ~30s: window nở → solo D có thể vào room còn ghế (nếu fit)
```

---

## File chính

| File | Vai trò |
|------|---------|
| `Server/Nakama/go-runtime/qm_rooms.go` | Join / leave / expand / match theo RP |
| `MatchServiceSubsystem.*` | Client join/leave, parse notify |
| `QuickMatchWidget.*` | UI tên + RP |

---

**Một câu:** Ấn Quick Match → server đọc RP thật → join room RP tương tự nếu còn chỗ; không thì room mới; chờ thì nới window (chỉ solo re-seat); đủ N → vào trận.
