# tiktok-live-aquarium

# 🐠 TikTok Interactive Live Game: AQUARIUM WARS (Đại Chiến Thủy Cung)

## 1. Tổng Quan Dự Án (Project Overview)
**Aquarium Wars** là một trò chơi tương tác thời gian thực (Real-time Interactive Game) được phát sóng trực tiếp trên nền tảng TikTok LIVE thông qua TikTok LIVE Studio. Người xem (Viewers) sử dụng các thao tác tương tác trên livestream (Comment, Thả tim, Share, Tặng quà/Donate) để thả cá mang tên mình vào thủy cung, nuôi cá lớn lên, săn mồi và cạnh tranh thứ hạng trên Bảng Xếp Hạng (Leaderboard).

### 1.1. Mục tiêu Kinh doanh (Business Goals)
1. **Hook (Thu hút ban đầu):** Hình ảnh thủy cung sinh động, hiệu ứng bắt mắt và việc nhìn thấy tên + avatar của người xem xuất hiện ngay lập tức khi comment.
2. **Retention (Giữ chân người xem):** Cơ chế "Cá lớn nuốt cá bé", vòng lặp sinh tồn và sự kiện định kỳ buộc người xem phải ở lại tương tác liên tục để bảo vệ cá của mình.
3. **Algorithm Boost (Đẩy đề xuất TikTok):** Kích thích lượng Comment và Thả tim (Like) liên tục để thuật toán TikTok đánh giá phiên live có tương tác cao và đẩy lên xu hướng (FYP).
4. **Monetization (Tối ưu hóa Donate):** Cấp quyền năng đặc biệt (biến thành Cá Mập, bật Khiên bảo vệ, triệu hồi Boss) cho người xem tặng quà (Gift), đánh mạnh vào tâm lý muốn thể hiện (VIP/Spotlight) và tính hiếu thắng.

---

## 2. Vòng Lặp Gameplay & Business Logic (Core Business Logic)

### 2.1. Quy tắc Sinh tồn & Khởi tạo (Spawning & Lifecycle)
* **Giới hạn hiển thị (Max Concurrent Entities):** Tối đa **35 con cá** bơi trên màn hình cùng lúc để đảm bảo hiệu năng (60 FPS) và không gây rối mắt.
* **Hàng đợi (Waiting Queue):** Nếu hồ đã đủ 35 cá, người chơi mới comment sẽ được đưa vào `Waiting Queue`. Khi có 1 con cá trong hồ bị ăn thịt hoặc hết thời gian sống, người đầu hàng đợi sẽ được tự động thả vào hồ.
* **Chống trùng lặp (Anti-Duplicate):** Mỗi `uniqueId` (Username TikTok) chỉ được sở hữu tối đa **01 con cá** trên màn hình tại một thời điểm.
* **Thời gian sống (TTL - Idle Timeout):**
  * Mỗi con cá khi mới sinh ra có `TTL = 90 giây`.
  * Nếu người chơi không có bất kỳ tương tác nào (Comment/Like/Gift) trong 90 giây, cá sẽ mờ dần và rời khỏi hồ để nhường chỗ cho người khác.
  * Mỗi lần người chơi tương tác (Comment/Like), `TTL` được làm mới lại về `90 giây`.

---

### 2.2. Ma Trận Chuyển Đổi Tương Tác (TikTok Event Mapping)

| Nguồn Tương Tác (TikTok Event) | Điều kiện / Cú pháp | Hành động trong Game (Game Action) | Thay đổi Chỉ số (Stats Update) |
| :--- | :--- | :--- | :--- |
| **Comment (Chat)** | Comment bất kỳ (nếu chưa có cá) | Khởi tạo (Spawn) cá Cấp 1 tại vị trí ngẫu nhiên (hoặc đưa vào Queue nếu hồ đầy). | `Size = 1.0`, `Score = 10`, `TTL = 90s` |
| **Comment (Chat)** | Comment từ khóa: `up`, `an`, `boi` (khi đã có cá) | Cá bơi nhanh hơn (Dash) về phía thức ăn gần nhất + Hiện bong bóng chat trên đầu cá (tối đa 20 ký tự). | `Score += 5`, `TTL = 90s` |
| **Thả tim (Like)** | Mỗi đợt nhận Like từ TikTok (Batch 5-15 likes) | Rải các hạt thức ăn nhỏ từ mặt nước xuống gần vị trí cá của người đó. Cá tự bơi lại đớp. | `+1 Score / 1 Like`, tăng nhẹ kích thước (`Size += 0.02`), `TTL = 90s` |
| **Chia sẻ (Share)** | Chia sẻ phiên Live | Tạo vòng sáng "Khiên Bảo Vệ" (Shield) quanh con cá trong **20 giây** (miễn nhiễm bị cá lớn ăn). | `State = SHIELDED (20s)`, `Score += 20` |
| **Follow** | Follow kênh | Cá được tiến hóa ngoại hình lấp lánh (Golden Skin) trong 60 giây + Thông báo chào mừng. | `Score += 30` |

---

### 2.3. Hệ Thống Cấp Độ & Tiến Hóa (Tier & Evolution System)
Kích thước và hình dáng của cá thay đổi dựa trên tổng điểm (`Score`) hiện tại của người chơi trong phiên:

1. **Level 1 - Cá Con (Minnow):** `Score: 0 - 99` | Kích thước: `1.0x` | Chỉ ăn thức ăn rơi tự nhiên.
2. **Level 2 - Cá Nhiệt Đới (Tropical Fish):** `Score: 100 - 299` | Kích thước: `1.4x` | Có thể nuốt cá Level 1 (nếu được kích hoạt trạng thái Săn mồi).
3. **Level 3 - Cá Kiếm (Swordfish):** `Score: 300 - 999` | Kích thước: `1.8x` | Bơi nhanh, có hào quang nhẹ.
4. **Level 4 - Cá Mập / Thủy Quái (Apex Predator):** `Score >= 1000` hoặc **Kích hoạt trực tiếp qua Donate**.

---

### 2.4. Cơ Chế Donate / Quà Tặng (Monetization & PvP Mechanics)
Khi hệ thống nhận sự kiện `WebcastEvent.GIFT`:

| Nhóm Quà (Gift Tier) | Giá trị Xu (TikTok Coins) | Ví dụ Quà | Hiệu ứng trong Game (Pay-to-Win / Visual) |
| :--- | :--- | :--- | :--- |
| **Tier 1 (Nhỏ)** | 1 – 5 Xu | Hoa hồng (Rose), TikTok | **Bơm tăng trưởng & Săn mồi:** Cá lập tức cộng `+100 Score`, bật chế độ **Cuồng Nộ (Frenzy Mode)** trong **15 giây**: Cá hóa đỏ, bơi nhanh gấp đôi và **tự động lao tới đớp 1 con cá Level thấp hơn** gần nhất (Cướp 50% số điểm của nạn nhân). |
| **Tier 2 (Vừa)** | 10 – 99 Xu | Nước hoa, Mũ, Corgi | **Biến hình Cá Mập (Shark Mode):** Biến thành Cá Mập Khổng Lồ trong **45 giây** + Bật Khiên bất tử. Cá mập đi càn quét hồ, tối đa đớp được 5 cá nhỏ. Hiển thị banner vinh danh giữa màn hình. |
| **Tier 3 (Lớn / VIP)** | >= 100 Xu | Sư tử, Xe hơi, Pháo hoa | **Triệu hồi Thủy Thần (Poseidon / Leviathan):** Thay đổi thời tiết toàn bộ thủy cung trong 60 giây. Người tặng quà xuất hiện ở **Ngai Vàng Trung Tâm**, rải mưa ngọc trai cho toàn bộ người xem trong hồ cùng hưởng điểm + Đứng Top 1 BXH VIP. |

> **Quy tắc Bị Ăn Thịt (Death & Respawn Rule):**
> * Khi Cá A bị Cá B đớp: Cá A phát nổ thành hiệu ứng bong bóng, hiển thị thông báo: *"💀 [Tên A] đã bị [Tên B] nuốt chửng!"*.
> * Người chơi A mất con cá hiện tại trên màn hình nhưng **giữ lại được 50% tổng điểm tích lũy** trên BXH.
> * Trên màn hình hiện lời nhắc kích thích tương tác lại: *"Comment bất kỳ để hồi sinh và trả thù!"*.

---

### 2.5. Sự Kiện Định Kỳ Tự Động (Automated World Events)
Hệ thống tự động kích hoạt sự kiện mỗi **5 phút/lần** để duy trì tương tác:
* **Sự kiện 1: Cơn Mưa Thức Ăn (Feeding Frenzy - 30 giây):** Yêu cầu toàn bộ người xem spam comment từ khóa `"AN"` hoặc thả tim. Càng nhiều comment, rương kho báu ở đáy hồ càng mở rộng và phun ra ngọc trai tăng điểm cho tất cả cá trên màn hình.
* **Sự kiện 2: Câu Hỏi Sinh Tồn (Trivia Shark Attack - 30 giây):**
  * Màn hình chia làm 2 nửa: **Trái (Đáp án A)** và **Phải (Đáp án B)** kèm 1 câu đố vui đơn giản trên đỉnh màn hình.
  * Người xem comment `A` hoặc `B` để điều khiển cá của mình bơi sang bên tương ứng.
  * Hết 20 giây đếm ngược, một hàm răng Cá Mập khổng lồ sẽ ngoạm toàn bộ nửa màn hình chứa đáp án SAI. Những ai chọn đúng được sống sót và nhận `+100 Score`.

---

## 3. Thiết Kế Giao Diện Màn Hình Dọc (UI/UX Layout - 9:16)
Độ phân giải chuẩn: **1080 x 1920 px** (Tỷ lệ 9:16 tối ưu cho màn hình điện thoại TikTok).

```text
+---------------------------------------+
| [TOP BAR - 12% Height]                |
| 🏆 BXH ĐẠI GIA (Top 3 Gifters)        |
| 🥇 UserA   🥈 UserB   🥉 UserC        |
| ⏳ Sự kiện tiếp theo: 02:45           |
+---------------------------------------+
|                                       |
| [MAIN STAGE - 68% Height]             |
| (Khu vực Thủy Cung - PixiJS Canvas)   |
|                                       |
|      🐠 [Avatar]                      |
|        "Nguyễn Văn A (150đ)"          |
|                                       |
|                   🦈 [VIP SHARK]      |
|                     "Đại Gia B (990đ)"|
|                                       |
|   🐡 "User C"                         |
|                                       |
| [Kill Feed / Alert Box - Góc phải]    |
| ⚡ Đại Gia B vừa nuốt chửng User C!   |
+---------------------------------------+
| [BOTTOM BAR - 20% Height]             |
| (Vùng an toàn tránh bị UI TikTok che) |
| 📜 Hướng dẫn:                         |
| 💬 Comment bất kỳ: Thả cá vào hồ      |
| ❤️ Thả tim: Cho cá ăn lớn nhanh       |
| 🌹 Tặng Hoa Hồng: Hóa Cá Mập săn mồi! |
+---------------------------------------+
```
*Lưu ý UI/UX:* Phần góc dưới bên trái (nơi TikTok hiển thị khung Chat) và cột dọc sát mép phải (nơi có nút Tim, Share, Quà của TikTok) không được đặt thông tin quan trọng để tránh bị giao diện mặc định của app TikTok che khuất.

---

## 4. Kiến Trúc Kỹ Thuật & Tech Stack (Technical Architecture)

### 4.1. Sơ Đồ Luồng Dữ Liệu (System Data Flow)
```text
[TikTok LIVE Server]
        │ (Webcast Protobuf Stream)
        ▼
[Backend: Node.js + tiktok-live-connector]
        │ 1. Lắng nghe Chat, Like, Gift, Share
        │ 2. Chuẩn hóa dữ liệu (Normalize & Anti-spam/Throttle)
        ▼ (WebSocket / Socket.IO)
[Frontend: React + PixiJS Engine]
        ├── React Layer: Quản lý UI tĩnh (Leaderboard, Alert, Mock Dashboard)
        └── PixiJS Layer: Game Loop 60FPS (Di chuyển cá, Va chạm, Hiệu ứng)
        │
        ▼ (Render tại http://localhost:5173)
[TikTok LIVE Studio] (Add Source -> Link / Browser Source)
```

### 4.2. Chi Tiết Tech Stack Đề Xuất

| Layer | Công nghệ | Lý do lựa chọn & Vai trò |
| :--- | :--- | :--- |
| **Ngôn ngữ** | **TypeScript** | Đảm bảo đồng nhất kiểu dữ liệu (Event Payload, Fish State) giữa Backend và Frontend. |
| **Frontend UI** | **React + Vite + TailwindCSS** | Dựng nhanh các khung UI bên ngoài sân khấu (Bảng xếp hạng, Thông báo tặng quà, Bảng điều khiển Mock Test). |
| **Game Engine (2D)** | **PixiJS (v8)** | Xử lý chuyển động bơi lượn mượt mà (Sin-wave movement), tính toán khoảng cách săn mồi, hiệu ứng hạt (bong bóng, nổ) ở tốc độ 60 FPS. |
| **Sound Engine** | **Howler.js** | Phát âm thanh nền (tiếng nước chảy, nhạc nền) và hiệu ứng âm thanh (SFX) tức thì khi thả cá, cá đớp mồi, hoặc có quà tặng. |
| **Backend** | **Node.js + Express** | Chạy nền nhẹ nhàng trên cùng máy PC stream. |
| **TikTok Connector** | **tiktok-live-connector** | Thư viện bắt sự kiện trực tiếp từ phòng LIVE thông qua `TikTok Username`. |
| **Realtime** | **Socket.IO** | Truyền sự kiện thời gian thực từ Node.js sang trình duyệt. |
| **State & Storage** | **In-Memory + `lowdb` (JSON)** | Toàn bộ trạng thái cá lưu trên RAM. Dùng `lowdb` (lưu ra file JSON cục bộ) để backup điểm Bảng Xếp Hạng phòng trường hợp F5 trình duyệt. |

---

## 5. Cấu Trúc Thư Mục Dự Án (Project Structure)

```text
aquarium-live-game/
│
├── backend/                        # Node.js Event Forwarder
│   ├── src/
│   │   ├── connector/
│   │   │   └── tiktokClient.ts     # Kết nối tiktok-live-connector
│   │   ├── services/
│   │   │   └── eventNormalizer.ts  # Chuẩn hóa event & gom cụm (batch) lượt Like
│   │   ├── types/
│   │   │   └── events.ts           # Định nghĩa Interface dùng chung
│   │   └── server.ts               # Express + Socket.IO Server (Port 3001)
│   ├── package.json
│   └── tsconfig.json
│
├── frontend/                       # React + PixiJS Game Client (Port 5173)
│   ├── public/
│   │   ├── assets/
│   │   │   ├── fishes/             # Sprite sheet cá các cấp độ
│   │   │   ├── effects/            # Bong bóng, khiên, vụ nổ
│   │   │   └── sounds/             # SFX: splash.mp3, bite.mp3, horn.mp3
│   ├── src/
│   │   ├── components/             # React UI Overlays
│   │   │   ├── Leaderboard.tsx     # Bảng xếp hạng Top 5
│   │   │   ├── KillFeed.tsx        # Thông báo cá lớn nuốt cá bé
│   │   │   ├── EventBanner.tsx     # Banner thông báo Donate & Sự kiện
│   │   │   └── MockDashboard.tsx   # Bảng điều khiển giả lập (chỉ hiện khi ?debug=true)
│   │   ├── game/                   # PixiJS Core Logic (Độc lập với React re-render)
│   │   │   ├── AquariumApp.ts      # Khởi tạo Pixi Application & Game Loop
│   │   │   ├── entities/
│   │   │   │   ├── Fish.ts         # Class Fish: AI bơi lượn, đuổi bắt, vẽ tên/avatar
│   │   │   │   └── Food.ts         # Class Food: Rơi từ trên xuống
│   │   │   └── managers/
│   │   │       └── StageManager.ts # Quản lý danh sách cá, Queue, va chạm (Collision)
│   │   ├── hooks/
│   │   │   └── useSocket.ts        # Lắng nghe Socket.IO và đẩy lệnh vào StageManager
│   │   ├── App.tsx
│   │   └── main.tsx
│   ├── package.json
│   └── tsconfig.json
│
└── README.md
```

---

## 6. Đặc Tả Giao Tiếp API / Socket Events (Data Contract)

Backend chuẩn hóa mọi sự kiện từ TikTok (hoặc từ Mock Dashboard) về một định dạng chuẩn trước khi bắn qua Socket.IO xuống Frontend:

### 6.1. Event: `game:action` (Server -> Client)
```typescript
interface GameActionPayload {
  actionId: string;
  timestamp: number;
  type: 'SPAWN_OR_CHAT' | 'FEED_LIKE' | 'ACTIVATE_SHIELD' | 'GIFT_UPGRADE';
  user: {
    uniqueId: string;     // ID định danh duy nhất (VD: "nguyenvana_99")
    nickname: string;     // Tên hiển thị trên đầu cá (VD: "Nguyễn Văn A")
    avatarUrl: string;    // URL ảnh đại diện TikTok để vẽ tròn cạnh tên cá
  };
  data?: {
    comment?: string;     // Nội dung chat (nếu type === 'SPAWN_OR_CHAT')
    likeCount?: number;   // Số lượng tim trong đợt (nếu type === 'FEED_LIKE')
    giftId?: number;      // ID quà tặng
    giftName?: string;    // Tên quà
    diamondCount?: number;// Giá trị xu của quà (quyết định Tier 1, 2 hay 3)
  };
}
```

### 6.2. Event: `mock:trigger` (Client Mock Dashboard -> Server)
Dùng để đội Dev và QC tự kiểm thử không cần bật TikTok Live thật:
* **Spawn 1 Fish**: Giả lập 1 user ngẫu nhiên comment.
* **Stress Test (35 Fishes)**: Bắn liên tiếp 40 event comment để kiểm tra giới hạn 35 cá + Hàng đợi (Queue) + Đo FPS.
* **Simulate Rose Gift**: Giả lập 1 cá hóa Đỏ (Frenzy Mode) và đi săn cá khác.
* **Simulate Shark Gift**: Giả lập biến hình Cá Mập.
* **Trigger Trivia Event**: Kích hoạt ngay sự kiện Câu hỏi sinh tồn A/B.

---

## 7. Giai Đoạn Triển Khai & Tiêu Chí Nghiệm Thu (Phases & Acceptance Criteria)

### Phase 1: Core Visual & Mock Mode
* **Deliverables:**
  * Dựng khung màn hình 9:16 với nền thủy cung động.
  * Class `Fish.ts` trong PixiJS hoạt động mượt: cá tự bơi ngẫu nhiên (chạm mép màn hình tự quay đầu Flip ngang), hiển thị rõ Avatar tròn + Tên người chơi + Điểm số bên dưới bụng cá.
  * Tích hợp `MockDashboard` (`http://localhost:5173/?debug=true`).
* **Tiêu chí nghiệm thu (AC):** Bấm nút "Stress Test" thả 35 con cá cùng lúc, FPS giữ ổn định ở mức 55–60 FPS, cá thứ 36 trở đi nằm trong hàng đợi.

### Phase 2: Combat Logic & Sound Effects
* **Deliverables:**
  * Logic rải thức ăn khi nhận Like, cá tự tìm thức ăn gần nhất.
  * Logic Săn mồi (Cá lớn/Cá mập tìm tọa độ cá nhỏ gần nhất, tăng tốc lao tới, va chạm tạo hiệu ứng nổ và cộng điểm).
  * Ghép toàn bộ âm thanh (Howler.js) và Bảng xếp hạng Top 5.
* **Tiêu chí nghiệm thu (AC):** Khi bấm nút giả lập tặng quà trên Mock Dashboard, cá biến hình, phát âm thanh còi báo động, lao đi đớp cá nhỏ và cập nhật ngay lên Kill Feed + Bảng xếp hạng.

### Phase 3: TikTok Live Integration & Studio Setup
* **Deliverables:**
  * Tích hợp `tiktok-live-connector` vào Backend, có cơ chế tự động kết nối lại (Auto-reconnect) nếu rớt mạng.
  * Xử lý gom cụm (Debounce/Batching) sự kiện Like của TikTok để không làm nghẽn Socket.
  * Đưa link `http://localhost:5173` vào nguồn **Link (Browser Source)** trên TikTok LIVE Studio.
* **Tiêu chí nghiệm thu (AC):** Bật phiên LIVE thử nghiệm trên TikTok, dùng điện thoại phụ vào comment và thả tim, cá xuất hiện trên màn hình LIVE Studio với độ trễ dưới **2 giây**.
