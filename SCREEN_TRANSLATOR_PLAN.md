# Screen Translator (Android): Kế hoạch triển khai cho Agent

> Tài liệu này là **nguồn sự thật duy nhất** (single source of truth) cho agent thực hiện. Đọc toàn bộ trước khi viết code. Mọi quyết định kiến trúc đều có lý do (ADR) để agent hiểu và không tự ý đổi.

---

## 0. Cách agent phải làm việc

1. **Làm tuần tự theo mốc (M0 → M8).** Không bắt đầu mốc sau khi mốc trước chưa đạt toàn bộ *Acceptance Criteria* (AC).
2. **Mỗi mốc = 1 nhánh git + 1 Pull Request** (`feat/m1-overlay`, ...). Commit theo Conventional Commits (`feat:`, `fix:`, `refactor:`, `test:`, `docs:`).
3. **Trước khi code mỗi mốc:** đọc lại mục tương ứng, liệt kê task vào danh sách việc, ước lượng rủi ro.
4. **Sau khi code mỗi mốc:** chạy lint + unit test + build, cập nhật `docs/PROGRESS.md` (làm được gì, đo được gì, còn nợ gì).
5. **Không tự thêm thư viện** ngoài danh sách ở mục 3 mà không ghi ADR mới vào `docs/adr/`.
6. **Không bịa API.** Với API Android/ML Kit thay đổi theo phiên bản (đặc biệt MediaProjection, foreground service types), phải đối chiếu tài liệu chính thức tại thời điểm làm. Nếu không chắc, ghi vào `docs/OPEN_QUESTIONS.md` và hỏi người dùng thay vì đoán.
7. **Mọi số liệu hiệu năng phải đo trên máy thật**, không báo cáo ước lượng.
8. Khi gặp mâu thuẫn hoặc thiếu thông tin ảnh hưởng đến thiết kế: **dừng và hỏi**, không tự chọn hướng không thể đảo ngược.

---

## 1. Tổng quan sản phẩm

### 1.1 Mục tiêu
App Android dịch văn bản trên màn hình theo thời gian thực (ưu tiên phụ đề/hội thoại game), chạy nền qua bong bóng nổi.

### 1.2 Hành vi cốt lõi (giống ảnh mẫu của người dùng)
- **Bong bóng nổi** (kéo thả, ẩn/hiện được) điều khiển toàn bộ.
- **Khung 1, Khung chọn vùng:** viền đỏ, bên trong trong suốt. Xác định vùng màn hình cần đọc chữ.
- **Khung 2, Khung bản dịch:** nền bán trong suốt, hiển thị bản dịch (tùy chọn kèm chữ gốc).
- Luồng: chụp màn hình → cắt theo Khung 1 → OCR → dịch → ghi vào Khung 2.

### 1.3 Phạm vi (In scope)
- Android 10+ (API 29+), tối ưu cho API 34/35.
- Dịch văn bản qua OCR từ ảnh chụp màn hình.
- Dịch offline (ML Kit) và tùy chọn cloud/LLM.
- Cài đặt, preset vùng theo từng app, lịch sử, glossary.

### 1.4 Ngoài phạm vi (Non-goals), không làm ở v1
- iOS.
- Dịch audio/giọng nói.
- Dịch tại chỗ đè lên chữ gốc (để v2, xem M8).
- Tự động hóa thao tác trên app khác.

### 1.5 Mặc định ban đầu (người dùng có thể đổi)
- Cặp ngôn ngữ ưu tiên: **Anh → Việt**.
- Nội dung chính: phụ đề/hội thoại game.
- Phân phối ban đầu: APK trực tiếp (Google Play xử lý sau, xem mục 12).

---

## 2. Quyết định kiến trúc (ADR)

| ID | Quyết định | Lý do | Đánh đổi |
|---|---|---|---|
| ADR-01 | **Kotlin native 100%** | Mọi API cốt lõi (WindowManager, MediaProjection, Foreground Service) là API Android. Không cần cầu nối, không copy frame qua channel, một runtime duy nhất nên ít RAM, quan trọng vì chạy cạnh game. | Không dùng lại được cho iOS (iOS vốn không cho overlay/chụp màn hình toàn hệ thống nên mất mát thực tế rất nhỏ). |
| ADR-02 | **Overlay dùng View thuần + Canvas**, không dùng Compose | Compose trong Service phải tự dựng `LifecycleOwner`/`SavedStateRegistryOwner`, tốn RAM, dễ lỗi. Overlay chỉ gồm vài hình vẽ đơn giản. | Code UI overlay dài hơn một chút. |
| ADR-03 | **Màn hình cài đặt dùng Jetpack Compose** | Là Activity bình thường, Compose cho code ngắn, dễ bảo trì. | Thêm dependency Compose. |
| ADR-04 | **Pipeline dạng Kotlin Flow** | Luồng frame→text→dịch là stream; Flow có sẵn `debounce`, `distinctUntilChanged`, `mapLatest` (hủy tác vụ cũ khi có frame mới). | Cần hiểu coroutine cancellation. |
| ADR-05 | **Interface hóa OCR và Translation** (`OcrEngine`, `TranslationEngine`) | Thay engine không động đến pipeline; dễ test bằng fake. | Thêm một lớp trừu tượng. |
| ADR-06 | **MediaProjection là cơ chế chụp mặc định** | API chính thức, không cần quyền đặc biệt ngoài consent. | Android 14+ yêu cầu consent mỗi phiên. |
| ADR-07 | **Capture ở độ phân giải thấp (mặc định 0.5x)** | Phụ đề game đủ lớn; giảm ~4 lần băng thông/CPU. Chụp lại độ phân giải đầy đủ chỉ khi OCR kém tin cậy. | Chữ rất nhỏ có thể sai, cần tùy chọn nâng độ phân giải. |
| ADR-08 | **DI bằng Hilt**, lưu trữ bằng **Room + DataStore** | Chuẩn công nghiệp, dễ test. | Thời gian build tăng nhẹ. |

---

## 3. Công nghệ và phiên bản

| Hạng mục | Lựa chọn |
|---|---|
| Ngôn ngữ | Kotlin (bản ổn định mới nhất tại thời điểm làm) |
| Build | Gradle Kotlin DSL, **version catalog** (`libs.versions.toml`) |
| minSdk / targetSdk / compileSdk | 29 / mới nhất ổn định / mới nhất ổn định |
| UI cài đặt | Jetpack Compose + Material 3 |
| UI overlay | Android View + Canvas |
| Async | Coroutines + Flow |
| DI | Hilt |
| Lưu cấu hình | Jetpack DataStore |
| Lưu dữ liệu | Room (cache dịch, lịch sử, glossary, preset) |
| OCR | ML Kit Text Recognition v2 (Latin; thêm Chinese/Japanese/Korean theo nhu cầu) |
| Dịch offline | ML Kit Translation |
| Dịch cloud/LLM | Ktor Client hoặc OkHttp + kotlinx.serialization |
| Lưu API key | EncryptedSharedPreferences / Android Keystore |
| Log | Timber |
| Test | JUnit5/JUnit4, MockK, Turbine (test Flow), Robolectric, Espresso/UiAutomator |
| Chất lượng | ktlint, detekt, Android Lint |
| Đo hiệu năng | Macrobenchmark, Perfetto, Android Studio Profiler |
| CI | GitHub Actions |

> Lưu ý: số phiên bản cụ thể phải tra tại thời điểm thực hiện, không dùng số trong trí nhớ.

---

## 4. Kiến trúc hệ thống

### 4.1 Sơ đồ tổng thể

```
┌─────────────────────────── App process ────────────────────────────┐
│                                                                     │
│  UI (Compose): Onboarding · Settings · History · Glossary           │
│        │ (Hilt, Repository)                                         │
│        ▼                                                            │
│  TranslatorService  (Foreground Service, type = mediaProjection)    │
│   ├─ OverlayManager ── BubbleView / SelectionFrameView / ResultView │
│   ├─ CaptureController ── MediaProjection → VirtualDisplay →        │
│   │                         ImageReader                              │
│   └─ TranslationPipeline (Flow)                                     │
│        frame → crop → changeDetect → preprocess → OCR →             │
│        stabilize → cache → translate → emit(result)                 │
│                                                                     │
│  Data: Room (cache, history, glossary, preset) · DataStore (config) │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Cấu trúc module (đa module Gradle)

```
:app                    // Application, DI root, Manifest, UI Compose
:core:common            // Result, dispatcher, extensions, logging
:core:model             // Data class thuần (Region, TextBlock, Language...)
:core:data              // Room, DataStore, Repository
:feature:overlay        // View overlay, OverlayManager
:feature:capture        // MediaProjection, ImageReader, RegionCropper
:feature:pipeline       // ChangeDetector, TextStabilizer, Pipeline
:feature:ocr            // OcrEngine + MlKitOcrEngine
:feature:translation    // TranslationEngine + MlKit/Cloud/LLM + Glossary
:feature:settings       // Compose screens
```

Quy tắc phụ thuộc: `core:model` không phụ thuộc gì; `feature:*` chỉ phụ thuộc `core:*`; **`feature:*` không phụ thuộc lẫn nhau** (giao tiếp qua interface ở `core`). `:app` nối tất cả.

### 4.3 Interface bắt buộc (hợp đồng)

```kotlin
// core:model
data class Region(val left: Int, val top: Int, val right: Int, val bottom: Int)
data class TextBlock(val text: String, val bounds: Region?, val confidence: Float?)
data class OcrResult(val blocks: List<TextBlock>, val latencyMs: Long)
data class TranslationResult(val source: String, val translated: String,
                             val engineId: String, val latencyMs: Long, val fromCache: Boolean)

// feature:ocr
interface OcrEngine {
    suspend fun recognize(image: Bitmap, rotationDegrees: Int = 0): OcrResult
    fun close()
}

// feature:translation
interface TranslationEngine {
    val id: String
    suspend fun isReady(src: Lang, dst: Lang): Boolean
    suspend fun prepare(src: Lang, dst: Lang)                 // tải model nếu cần
    suspend fun translate(req: TranslationRequest): String    // 1 lần
    fun translateStream(req: TranslationRequest): Flow<String> = flowOf() // tùy chọn
}
data class TranslationRequest(
    val text: String, val src: Lang, val dst: Lang,
    val context: List<String> = emptyList(),                  // các câu trước
    val glossary: Map<String, String> = emptyMap()
)

// feature:capture
interface ScreenCapturer {
    fun start(config: CaptureConfig): Flow<Frame>
    fun stop()
}
```

### 4.4 Máy trạng thái của Service

```
IDLE ──(user bấm Start + consent)──► RUNNING_AUTO
RUNNING_AUTO ◄──(Pause/Resume)──► PAUSED
RUNNING_AUTO ──(Manual mode)──► RUNNING_MANUAL (chỉ dịch khi bấm)
* ──(Stop / revoke / lỗi nghiêm trọng)──► IDLE (giải phóng toàn bộ tài nguyên)
```
Mọi chuyển trạng thái phải được log và hiển thị ở notification.

---

## 5. Đặc tả kỹ thuật chi tiết

### 5.1 Quyền và Manifest

Khai báo (agent phải kiểm tra tên đúng theo tài liệu hiện hành):
- `SYSTEM_ALERT_WINDOW` (overlay), kiểm tra bằng `Settings.canDrawOverlays()`, điều hướng người dùng tới màn cấp quyền.
- `FOREGROUND_SERVICE` và `FOREGROUND_SERVICE_MEDIA_PROJECTION`.
- `POST_NOTIFICATIONS` (Android 13+).
- `INTERNET` (chỉ cho engine cloud/tải model).
- Service khai báo `android:foregroundServiceType="mediaProjection"`.

### 5.2 MediaProjection (điểm rủi ro cao nhất)

- Xin consent qua `MediaProjectionManager.createScreenCaptureIntent()` bằng Activity Result API.
- **Android 14+:** token dùng một lần; **phải đăng ký `MediaProjection.Callback` trước khi tạo `VirtualDisplay`**; phải gọi `startForeground()` với đúng type **trước** khi lấy `MediaProjection`.
- **Android 15+:** người dùng có thể chọn chia sẻ một phần màn hình hoặc một app; xử lý trường hợp vùng chọn nằm ngoài phần được chia sẻ (báo lỗi thân thiện).
- Xử lý `onStop()` của Callback (hệ thống/người dùng thu hồi): dừng pipeline, dọn dẹp, cập nhật UI.
- Xử lý xoay màn hình: tạo lại `VirtualDisplay` (hoặc `resize`) và tính lại tọa độ Region.
- `ImageReader` dùng `PixelFormat.RGBA_8888`, `maxImages = 2`, luôn `close()` Image sau khi dùng (tránh rò rỉ).
- Chỉ `acquireLatestImage()`, bỏ qua frame cũ (backpressure).

### 5.3 Overlay

**Cửa sổ:** `WindowManager.addView` với `TYPE_APPLICATION_OVERLAY`, `PixelFormat.TRANSLUCENT`.

| View | Flag chính | Hành vi |
|---|---|---|
| `BubbleView` | `FLAG_NOT_FOCUSABLE` | Kéo thả, tự dính cạnh, mờ khi không dùng, chạm mở menu hình quạt |
| `SelectionFrameView` | `FLAG_NOT_FOCUSABLE`; khi **khóa** thêm `FLAG_NOT_TOUCHABLE` | Viền đỏ, kéo 4 góc/4 cạnh, di chuyển; chế độ khóa cho chạm xuyên |
| `ResultView` | `FLAG_NOT_FOCUSABLE` | Nền bán trong suốt, kéo/resize, chỉnh cỡ chữ/màu/độ mờ, chạm = sao chép |

**Menu bong bóng:** ▶/⏸ Start/Pause · ⬚ Chọn vùng (mở khóa) · 👁 Ẩn/Hiện khung · ⚙ Cài đặt · ✕ Thoát.

**Quy tắc chống vòng lặp (bắt buộc):**
1. Vùng crop **thu nhỏ vào trong viền** `insetPx` (mặc định 4px) để không chụp viền đỏ.
2. `ResultView` mặc định **đặt ngoài** vùng chọn (phía trên hoặc dưới). Nếu người dùng cố đặt chồng lên, hiển thị cảnh báo.
3. Có cơ chế phát hiện: nếu text OCR giống bản dịch gần nhất (Levenshtein ratio cao) thì bỏ qua và đánh dấu nghi vòng lặp.
4. Phương án cuối (tùy chọn): ẩn tạm `ResultView` khi chụp (dễ nhấp nháy, mặc định tắt).

**Lưu vị trí/kích thước** từng khung vào DataStore, tính theo `packageName` của app foreground (nếu có quyền), nếu không thì theo preset toàn cục.

### 5.4 Pipeline xử lý

```
ImageReader frame
 → throttle (burst/idle, xem 5.5)
 → crop(Region - inset)
 → ChangeDetector  : thu nhỏ 32x32 xám, so sánh khoảng cách; dưới ngưỡng thì bỏ qua
 → Preprocess      : scale/độ tương phản (bật theo cấu hình)
 → OcrEngine.recognize (Dispatchers.Default / IO)
 → TextStabilizer  : debounce 300–500ms + Levenshtein so với lần trước; chỉ phát khi ổn định
 → TranslationCache (Room, key = hash(text, src, dst, engineId))
 → TranslationEngine.translate (kèm context + glossary)
 → ResultView.update
```

**Yêu cầu cài đặt:**
- Dùng `mapLatest`/`collectLatest` để hủy OCR/dịch đã lỗi thời khi có frame mới.
- `TextStabilizer` xử lý hiệu ứng chữ hiện dần (typewriter): không dịch khi văn bản còn đang tăng.
- Giữ bộ đệm `context` = N câu gần nhất (mặc định 5) để đưa vào engine hỗ trợ ngữ cảnh.
- Mọi lỗi được bọc `Result`, không để exception thoát khỏi coroutine làm sập Service.

### 5.5 Chiến lược chụp (tiết kiệm pin, giữ FPS game)

| Chế độ | Tốc độ chụp | Khi nào |
|---|---|---|
| **Burst** | 4–5 fps, ~2 giây | Ngay sau khi phát hiện thay đổi hoặc người dùng chạm |
| **Idle** | 0,5–1 fps | Văn bản không đổi |
| **Manual** | Chụp 1 lần | Người dùng bấm bong bóng |
| **Paused** | 0 | Người dùng tạm dừng / màn hình tắt |

Tự động về **Paused** khi màn hình tắt (`ACTION_SCREEN_OFF`), tiếp tục khi bật lại.

### 5.6 Dịch

| Engine | Vai trò | Ghi chú |
|---|---|---|
| `MlKitTranslationEngine` | **Mặc định**, offline | Quản lý tải/xóa model, báo tiến trình |
| `CloudTranslationEngine` | Tùy chọn (Google/DeepL) | Người dùng nhập key |
| `LlmTranslationEngine` | Tùy chọn "chất lượng cao" | Streaming, đưa `context` + `glossary` vào prompt |

**Glossary:** bảng `nguồn → đích` do người dùng sửa. Với engine không hỗ trợ prompt (ML Kit): thay tên riêng bằng placeholder trước khi dịch, thay lại sau khi dịch.

**Prompt cho LLM** (mẫu, lưu ở file resource, không hard-code trong logic):
> Bạn là biên dịch hội thoại game. Dịch sang {dst}, giữ nguyên tên riêng trong glossary, giữ giọng điệu nhân vật. Chỉ trả về bản dịch, không giải thích.

### 5.7 Bảo mật và quyền riêng tư

- **API key:** chỉ lưu bằng Keystore/EncryptedSharedPreferences; **cấm** commit key, cấm nhúng vào APK.
- **Ảnh chụp màn hình không bao giờ được ghi ra đĩa** (chỉ xử lý trong bộ nhớ) và không gửi lên mạng. Với cloud/LLM chỉ gửi **text đã OCR**, không gửi ảnh.
- Hiển thị rõ cho người dùng dữ liệu nào rời khỏi máy trước khi bật engine cloud.
- Cung cấp nút **xóa lịch sử và cache**.
- Không thu thập telemetry mặc định; nếu thêm crash reporting thì phải opt-in.
- Cảnh báo người dùng không dùng với nội dung nhạy cảm (ngân hàng, mật khẩu). `FLAG_SECURE` sẽ trả khung đen, app cần phát hiện và báo rõ.

---

## 6. Lộ trình và tiêu chí nghiệm thu

> Mỗi mốc gồm: **Mục tiêu · Task · Deliverables · AC (tiêu chí hoàn thành đo được)**.

### M0, Spike khả thi (2–3 ngày)
**Mục tiêu:** chứng minh chuỗi overlay + chụp + OCR chạy được trên máy thật và đo độ trễ.
**Task**
- Dựng project, version catalog, CI skeleton.
- Service tối thiểu + 1 overlay hình vuông.
- MediaProjection chụp 1 frame, hiển thị trong Activity debug.
- ML Kit OCR trên frame đó.
- Ghi số đo: thời gian chụp, OCR, RAM, trên ít nhất 1 máy thật.

**Deliverables:** nhánh `spike/m0`, `docs/spikes/M0-report.md` (số đo + kết luận + rủi ro phát hiện).
**AC:** OCR đọc đúng ≥ 90% ký tự trên 5 ảnh phụ đề mẫu tiếng Anh; báo cáo có số đo thật; liệt kê mọi khác biệt của Android 14/15 gặp phải.

### M1, Overlay hoàn chỉnh
**Task:** `BubbleView` (kéo, dính cạnh, mờ), menu hình quạt, `SelectionFrameView` (kéo/resize/khóa), `ResultView` (kéo/resize/style), `OverlayManager` (thêm/xóa/ẩn/hiện, lưu vị trí), xử lý xoay màn hình.
**AC:**
- Kéo/resize mượt (không rớt khung rõ rệt trên máy tầm trung).
- Chế độ khóa: chạm xuyên xuống app bên dưới hoạt động đúng, kiểm tra bằng UiAutomator.
- Ẩn/hiện khung không làm mất vị trí.
- Không rò rỉ View (kiểm tra bằng LeakCanary khi bật/tắt 20 lần).

### M2, Foreground Service và luồng quyền
**Task:** `TranslatorService` + notification (nút Pause/Stop), onboarding xin quyền (overlay → notification → MediaProjection), xử lý từ chối/thu hồi, máy trạng thái 4.4, hướng dẫn tắt tối ưu pin theo hãng.
**AC:** chạy liên tục ≥ 30 phút không bị kill trên máy thật; thu hồi quyền ở bất kỳ bước nào cũng về IDLE sạch (không crash, không còn overlay mồ côi); test thủ công trên Android 13, 14, 15.

### M3, Capture và OCR
**Task:** `ScreenCapturer`, `RegionCropper` (có inset), `ChangeDetector`, chiến lược burst/idle, `MlKitOcrEngine`, tiền xử lý ảnh tùy chọn.
**AC:** màn hình tĩnh → CPU trung bình do app gây ra < 3% (đo bằng Profiler); OCR đúng ≥ 90% trên bộ ảnh mẫu; không rò `Image`/`Bitmap` (chạy 30 phút, RAM ổn định).

### M4, Dịch và ổn định văn bản
**Task:** `TranslationEngine` + `MlKitTranslationEngine`, quản lý model, `TextStabilizer`, `TranslationCache` (Room), chống vòng lặp, ghép pipeline đầu-cuối, `ResultView` hiển thị bản dịch (+ chữ gốc tùy chọn).
**AC:** độ trễ đầu-cuối (chữ ổn định → hiện bản dịch) **p50 < 1,0s, p95 < 2,0s** trên máy tầm trung (offline); chữ gõ dần không bị dịch từng ký tự; không xuất hiện vòng lặp tự dịch lại bản dịch (test 10 phút liên tục).

### M5, UI cài đặt (Compose)
**Task:** màn chọn ngôn ngữ, tải/xóa model, cỡ chữ/màu/độ mờ, tần suất chụp, preset theo app, lịch sử, glossary, thông tin quyền riêng tư.
**AC:** đổi cài đặt có hiệu lực ngay trên overlay không cần khởi động lại service; cấu hình giữ nguyên sau khi tắt/mở app.

### M6, Engine nâng cao
**Task:** `CloudTranslationEngine`, `LlmTranslationEngine` (streaming), nhập/lưu key an toàn, ngữ cảnh hội thoại + glossary, chuyển engine nóng, fallback sang offline khi mất mạng/lỗi.
**AC:** chuyển engine không cần khởi động lại; mất mạng thì tự fallback và báo nhẹ nhàng; key không xuất hiện trong log/APK (kiểm tra bằng grep + decompile).

### M7, Tối ưu và kiểm thử toàn diện
**Task:** Macrobenchmark, tối ưu bộ nhớ/CPU/pin, kiểm thử ma trận thiết bị, sửa lỗi, tài liệu người dùng.
**AC:** đạt toàn bộ **ngân sách hiệu năng (mục 7)**; ma trận kiểm thử (mục 8) pass; không còn bug mức Blocker/Critical.

### M8, Phát hành và tính năng mở rộng
**Task:** ký APK/AAB, ProGuard/R8, icon, trang giới thiệu, chính sách quyền riêng tư, beta nội bộ. **Backlog v2** (chỉ làm khi M0–M7 xong và người dùng đồng ý): dịch tại chỗ, nguồn văn bản từ AccessibilityService, Shizuku, OCR chuyên biệt theo game, chia sẻ preset.
**AC:** bản beta cài và chạy được trên thiết bị sạch theo hướng dẫn; checklist phát hành (mục 12) hoàn tất.

---

## 7. Ngân sách hiệu năng (đo trên máy thật tầm trung)

| Chỉ số | Mục tiêu |
|---|---|
| Giảm FPS của game khi bật app (chế độ Idle) | < 5% |
| CPU do app gây ra khi màn hình tĩnh | < 3% |
| RAM của app khi chạy | < 150 MB (ghi rõ mức thực đo) |
| Độ trễ đầu-cuối, offline | p50 < 1,0s · p95 < 2,0s |
| Tiêu thụ pin | Ghi lại %/giờ ở Idle và Burst; không có ngưỡng cứng ở v1 nhưng phải báo cáo |
| Cold start Activity chính | < 1,5s |

Nếu không đạt: agent phải ghi nguyên nhân và đề xuất giảm tải (hạ độ phân giải chụp, giảm fps, đổi engine) trong `docs/PROGRESS.md`.

---

## 8. Chiến lược kiểm thử

**Unit test (bắt buộc, chạy trong CI):** `ChangeDetector`, `TextStabilizer`, Levenshtein/chống vòng lặp, `TranslationCache`, thay thế glossary, `RegionCropper` (inset, xoay), máy trạng thái service. Dùng fake cho `OcrEngine`/`TranslationEngine`.

**Instrumented test:** vòng đời Service, thêm/xóa overlay, chế độ khóa chạm xuyên (UiAutomator), luồng quyền.

**Bộ dữ liệu mẫu:** thư mục `testdata/` gồm ≥ 20 ảnh phụ đề (nhiều font, nền sáng/tối, có chữ gõ dần, chữ viền) kèm đáp án chuẩn → dùng đo độ chính xác OCR.

**Ma trận thiết bị thủ công:**

| Mục | Phạm vi |
|---|---|
| Phiên bản Android | 10, 12, 13, 14, 15 |
| Hãng | Samsung, Xiaomi, Oppo/Realme (tối thiểu 3 hãng) |
| Kịch bản | Game phụ đề ngang, manga/webtoon, video phụ đề cứng, xoay ngang/dọc, đa cửa sổ, màn hình tắt/bật |
| Ngoại lệ | App có `FLAG_SECURE` (phải báo lỗi rõ), thu hồi quyền giữa chừng, thiếu mạng (engine cloud) |

---

## 9. Chất lượng mã và quy trình

- **Definition of Done (mỗi PR):** build xanh · lint/ktlint/detekt sạch · unit test mới cho logic mới · không warning mới · cập nhật tài liệu · AC của mốc liên quan được tích hoàn thành trong `docs/PROGRESS.md`.
- **Quy ước code:** không hard-code chuỗi UI (dùng `strings.xml`, hỗ trợ vi/en); không dùng `GlobalScope`; mọi coroutine gắn vào scope có vòng đời; không giữ `Context` Activity trong singleton.
- **Xử lý lỗi:** dùng sealed `Result`/`Outcome`; mọi lỗi người dùng nhìn thấy phải có thông báo thân thiện và hành động khắc phục.
- **Logging:** Timber; bản release không log nội dung văn bản OCR/dịch.

**CI (GitHub Actions):** trên mỗi PR chạy `ktlintCheck detekt lintDebug testDebugUnitTest assembleDebug`; nightly chạy instrumented test trên emulator (nếu khả thi).

---

## 10. Cấu trúc tài liệu agent phải duy trì

```
docs/
  PROGRESS.md        // nhật ký theo mốc: đã làm / số đo / nợ kỹ thuật
  OPEN_QUESTIONS.md  // điều chưa chắc cần người dùng quyết
  adr/               // mỗi quyết định mới = 1 file ADR-XX.md
  spikes/            // báo cáo M0 và các spike khác
  USER_GUIDE.md      // hướng dẫn cấp quyền, tắt tối ưu pin theo hãng
```

---

## 11. Rủi ro và cách giảm thiểu

| Rủi ro | Mức | Giảm thiểu |
|---|---|---|
| Overlay bị OCR đọc lại → vòng lặp | Cao | Crop thu vào trong viền, đặt khung dịch ngoài vùng chọn, phát hiện tương tự, kiểm thử 10 phút |
| Game giảm FPS | Cao | Capture độ phân giải thấp, burst/idle, change detection, đo từng bước |
| Consent MediaProjection mỗi phiên (Android 14+) | TB | Onboarding rõ, nút bắt đầu lại nhanh, notification action |
| Service bị OEM kill | TB | Foreground notification, hướng dẫn tắt tối ưu pin theo hãng |
| OCR sai với font cách điệu | TB | Tiền xử lý, cho phép đổi engine/độ phân giải, v2: OCR theo game |
| Chất lượng dịch offline thấp | TB | Cho chọn engine LLM, glossary, ngữ cảnh hội thoại |
| API Android/ML Kit đổi theo phiên bản | TB | Tra tài liệu chính thức mỗi mốc, ghi vào OPEN_QUESTIONS |
| Rò rỉ bộ nhớ (Image/Bitmap/View) | TB | `close()` nghiêm ngặt, LeakCanary, chạy soak test 30 phút |
| Google Play từ chối | TB | Khai báo đúng mục đích foreground service; phát hành APK trước |

---

## 12. Checklist phát hành

- [ ] Bản release ký, bật R8/ProGuard, đã test lại sau thu gọn mã
- [ ] Chính sách quyền riêng tư, nêu rõ: không lưu ảnh, chỉ gửi text khi dùng cloud
- [ ] Khai báo lý do dùng `SYSTEM_ALERT_WINDOW` và `mediaProjection` (nếu lên Google Play)
- [ ] Hướng dẫn người dùng (cấp quyền, tắt tối ưu pin)
- [ ] Giấy phép các thư viện (open-source notices)
- [ ] Kiểm tra không có key/secret trong repo và APK
- [ ] Đã test trên ma trận thiết bị mục 8

---

## 13. Điểm cần người dùng xác nhận (agent hỏi trước M1)

1. Cặp ngôn ngữ ưu tiên và nội dung chính (game / manga / video)?
2. Có bắt buộc offline, hay cho phép cloud/LLM?
3. Phân phối: APK cá nhân hay Google Play?
4. minSdk có chốt API 29 không?
5. Có muốn đổi tên app/package name (`com.example.screentranslator` là mặc định tạm)?

> Nếu người dùng vẫn muốn giữ **Flutter cho phần UI cài đặt**: giữ nguyên toàn bộ lõi native ở trên, chỉ thay `:feature:settings` bằng module Flutter giao tiếp qua `MethodChannel`/`EventChannel`, và ghi một ADR mới nêu rõ đánh đổi (thêm một runtime, tăng kích thước APK).
