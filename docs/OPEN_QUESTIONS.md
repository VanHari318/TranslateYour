# Danh sách câu hỏi và quyết định cần xác nhận (OPEN_QUESTIONS.md)

| ID | Câu hỏi | Lựa chọn / Đề xuất | Trạng thái | Quyết định của người dùng |
|---|---|---|---|---|
| Q1 | Công nghệ UI Cài đặt: 100% Kotlin Native (Jetpack Compose) hay Flutter cho phần UI Settings? | Đề xuất: 100% Kotlin Native theo ADR-01 (tiết kiệm RAM khi chạy cạnh game, kích thước APK nhỏ). Nếu dùng Flutter: sẽ tạo bridge MethodChannel sang native service. | Chờ xác nhận | |
| Q2 | Package name & Tên App chính thức? | Mặc định: `com.example.screentranslator` / Tên: "Screen Translator" | Chờ xác nhận | |
| Q3 | Cặp ngôn ngữ ưu tiên ban đầu? | Mặc định: Anh → Việt (Phụ đề/hội thoại game) | Chờ xác nhận | |
| Q4 | Chế độ dịch v1: Có bắt buộc chỉ offline (ML Kit) hay hỗ trợ thêm Cloud/LLM? | Đề xuất: Mặc định offline, có tuỳ chọn LLM/Cloud ở các mốc sau (M6). | Chờ xác nhận | |
| Q5 | Kênh phân phối ban đầu? | Mặc định: Build file APK cài trực tiếp. | Chờ xác nhận | |

