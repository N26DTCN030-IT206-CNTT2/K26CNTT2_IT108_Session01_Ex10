# BÀI 10: TÍNH TOÁN CHI PHÍ API DOANH NGHIỆP & ENTERPRISE SECURITY

**Môn học:** Kỹ năng Prompting với AI (IT206-K26) - Session 01 - Ex10.
**Tình huống:** Giải quyết bài toán tính toán chi phí Token chính xác và thiết kế hạ tầng API an toàn cho chiến dịch gửi 50.000 tin nhắn Zalo của phòng Marketing MartX.
---

## PHẦN 1: BẢNG TÍNH TOÁN CHI PHÍ API (API COST CALCULATION)

Phòng Marketing ước tính chi phí thô là **$37.5 USD**, tuy nhiên họ đã mắc 2 sai lầm nghiêm trọng trong tư duy tính toán chi phí LLM:
1. **Bỏ qua hệ số BPE (Byte-Pair Encoding) của Tiếng Việt:** AI không đếm theo "từ" (word). Tiếng Việt là ngôn ngữ có dấu, thường bị chia nhỏ thành nhiều subwords. Hệ số thực tế phải nhân với **2.5 Tokens/từ**.
2. **Gộp chung đơn giá Input và Output:** Chi phí Output (AI sinh ra) của GPT-4o đắt gấp 4 lần Input (User nạp vào). Marketing đã tính nhầm toàn bộ 15M từ theo một mức đơn giá rẻ ($2.50).

### 1. Phân tích Chi tiết Số lượng Token thực tế
- **Tổng số lượng tin nhắn:** 50,000 tin
- **Hệ số BPE Tiếng Việt:** 2.5 Tokens/từ

| Thành phần | Số từ/tin | Tổng số từ (50k tin) | Quy đổi Token (Nhân 2.5) | Đơn giá GPT-4o (per 1M) |
| :--- | :--- | :--- | :--- | :--- |
| **Input Prompt** | 200 từ | 10,000,000 từ | **25,000,000 Tokens** | $2.50 |
| **Output Response** | 100 từ | 5,000,000 từ | **12,500,000 Tokens** | $10.00 |
| **TỔNG CỘNG** | **300 từ** | **15,000,000 từ** | **37,500,000 Tokens** | N/A |

### 2. Tính toán Chi phí Thực tế & Khoảng hụt Ngân sách
- **Chi phí Input:** 25 triệu tokens × ($2.50 / 1,000,000) = **$62.50**
- **Chi phí Output:** 12.5 triệu tokens × ($10.00 / 1,000,000) = **$125.00**
- **Tổng chi phí thực tế (USD):** $62.50 + $125.00 = **$187.50 USD**
- **Tổng chi phí thực tế (VNĐ):** $187.50 × 25,000 = **4,687,500 VNĐ**

> ⚠️ **KẾT LUẬN RỦI RO TÀI CHÍNH:** 
> - Số tiền Marketing ước tính: **$37.50** (~937,500 VNĐ).
> - Số tiền Marketing tính thiếu (Thâm hụt): **$150.00** (Tương đương **3,750,000 VNĐ**). 
> - Chi phí thực tế bị đội lên gấp **5 lần** so với kế hoạch thô ban đầu.

---

## PHẦN 2: THIẾT KẾ HẠ TẦNG ENTERPRISE API GATEWAY

### 1. Tại sao KHÔNG được dùng tài khoản ChatGPT Plus cá nhân chạy Auto Web Scraping?
- **Rủi ro PII (Rò rỉ dữ liệu cá nhân):** Phiên bản ChatGPT Web UI (Tài khoản Plus $20/tháng) mặc định thu thập dữ liệu hội thoại để huấn luyện mô hình (Model Training). Việc tự động đẩy thông tin của 50.000 khách hàng (Tên, tuổi, hành vi mua sắm) lên bản Web sẽ dẫn đến thảm họa rò rỉ dữ liệu (Data Breach) diện rộng.
- **Vi phạm ToS (Điều khoản dịch vụ):** Việc dùng tool tự động (Selenium/Puppeteer) để cào (scrape) Web UI của ChatGPT vi phạm chính sách của nhà cung cấp. Tài khoản sẽ bị block ngay lập tức bởi cơ chế chống bot (HTTP 429 / Cloudflare CAPTCHA), làm gián đoạn toàn bộ chiến dịch.

### 2. Giải pháp: Chuyển đổi sang Enterprise API Gateway
Để bảo vệ dữ liệu PII và kiểm soát chi phí, hệ thống bắt buộc phải giao tiếp thông qua một cổng trung gian **(Enterprise API Gateway)** và gọi các dịch vụ LLM dành cho doanh nghiệp (như Azure OpenAI).

**Bản chất kỹ thuật (Kiến trúc Zero Data Retention - ZDR):**
- **Zero Data Retention:** Trái ngược với bản Web, API của OpenAI và Azure OpenAI cam kết **KHÔNG** lưu trữ dữ liệu payload của khách hàng quá 30 ngày và **TUYỆT ĐỐI KHÔNG** dùng dữ liệu API để huấn luyện lại mô hình ngôn ngữ. Dữ liệu của MartX hoàn toàn thuộc về MartX.
- **Private Endpoint (Mạng kín):** Dữ liệu truyền tải giữa server của MartX và Azure OpenAI không đi qua mạng Internet công cộng mà được bảo mật thông qua mạng riêng ảo (VNet).

### 3. Luồng hoạt động của Kiến trúc Gateway (Workflow)

1. **Tầng Request (Hệ thống Marketing):** Khởi tạo yêu cầu tạo 50.000 tin nhắn.
2. **Tầng Trung gian (Enterprise API Gateway):**
   - *Tính năng DLP (Data Loss Prevention) / Masking:* Quét Regex để che mờ thông tin PII nhạy cảm trước khi gói tin ra ngoài mạng nội bộ (Ví dụ: Chuyển SĐT `0901234567` thành token `[PHONE_REDACTED]`).
   - *Rate Limiting & Cost Control:* Khống chế quota token và budget giới hạn cho chiến dịch; tự động ngắt kết nối (Circuit Breaker) nếu vòng lặp sinh tin nhắn bị lỗi, ngăn chặn hóa đơn API phình to ngoài kiểm soát.
3. **Tầng Xử lý (Enterprise LLM):** Nhận Prompt đã ẩn danh hóa, thực hiện sinh ngôn ngữ tự nhiên an toàn.
4. **Tầng Response:** Trả kết quả về API Gateway để giải mã (unmasking) và đẩy về hệ thống Marketing gửi cho khách hàng qua Zalo ZNS.
