# Hướng dẫn kết nối VIIS MCP

Địa chỉ server: **`https://iot.viis.tech/api/v2/mcp`**

Khi kết nối, client mở trang **"Authorize VIIS MCP"**. Bạn đăng nhập bằng email và mật khẩu VIIS như trên web. Không cần API key.

| Mục | Chi tiết |
|---|---|
| Tài khoản | Tài khoản VIIS của bạn. Tài khoản có bước OTP hiện chưa dùng được. |
| Quyền | Giống trên web: chỉ thấy và sửa được dữ liệu tài khoản đó được phép. |
| Thời hạn | Phiên tự gia hạn. Để yên 30 ngày (tối đa 90 ngày) thì cần kết nối lại. |
| Giới hạn | 60 lượt gọi/phút, 1000 lượt/ngày cho mỗi kết nối. |

## Claude Desktop / claude.ai

Thêm một lần là dùng được trên app Desktop, web và app điện thoại.

1. Mở **Settings** → **Connectors** → **Add custom connector**.
2. Đặt tên `VIIS`, dán URL `https://iot.viis.tech/api/v2/mcp` → **Add**.
3. Bấm **Connect** → đăng nhập VIIS → cho phép.
4. Mở chat mới, bật connector VIIS trong menu công cụ rồi hỏi thử: "Hôm nay có bao nhiêu công việc đang thực hiện?"

Gói Team/Enterprise: có thể cần Owner thêm connector cho cả tổ chức trước, sau đó từng người bấm Connect bằng tài khoản VIIS của mình.

**Cài thêm skill `viis-mcp`** (khuyến nghị): tải `viis-mcp.zip` ở [Releases](https://github.com/VIIS-IOT/viis-ai-skills/releases) → Settings → Capabilities → Skills → Upload skill.

## Claude Code

Cách nhanh nhất là cài plugin (gồm cả server và skill):

```text
/plugin marketplace add VIIS-IOT/viis-ai-skills
/plugin install viis@viis
```

Hoặc chỉ thêm server:

```bash
claude mcp add --transport http viis https://iot.viis.tech/api/v2/mcp
```

Sau đó gõ `/mcp` → chọn `viis` → **Authenticate** → đăng nhập trên trình duyệt.

## ChatGPT

Cần gói có Developer mode (OpenAI thay đổi gói hỗ trợ và tên menu theo thời gian).

1. **Settings** → **Apps & Connectors** → **Advanced settings** → bật **Developer mode**.
2. Quay lại Apps & Connectors → **Create**.
3. Name `VIIS`, MCP Server URL `https://iot.viis.tech/api/v2/mcp`, Authentication **OAuth**. Để trống Client ID và Client secret.
4. Tích ô xác nhận → **Create** → đăng nhập VIIS → cho phép.
5. Trong chat mới, chọn connector VIIS rồi đặt câu hỏi.

ChatGPT không cài được skill. Muốn trợ lý làm theo các quy tắc an toàn, hãy dán phần [An toàn](#an-toàn) bên dưới vào Custom instructions.

## Cursor

1. Thêm vào `~/.cursor/mcp.json` (mọi dự án) hoặc `.cursor/mcp.json` (một dự án):

```json
{"mcpServers": {"viis": {"url": "https://iot.viis.tech/api/v2/mcp"}}}
```

2. Mở **Cursor Settings** → **MCP**. `viis` hiện "Needs login" → bấm vào → đăng nhập VIIS.
3. Khi trạng thái chuyển xanh, dùng trong Agent chat.

Muốn trợ lý làm theo quy tắc an toàn, dán phần [An toàn](#an-toàn) vào Rules.

## An toàn

Mọi thao tác ghi chạy qua đúng nghiệp vụ của web: gửi thông báo cho người được giao, ghi sổ kho, thay đổi phân bổ. Vì vậy tool ghi luôn có hai bước:

1. **Xem trước** (`confirm` = false): server kiểm tra dữ liệu và quyền, trả về những gì sẽ thay đổi. Chưa lưu gì.
2. **Xác nhận** (`confirm` = true): chỉ chạy sau khi bạn đồng ý đúng nội dung xem trước.

| Thao tác | Vì sao cần cẩn thận |
|---|---|
| Gửi lệnh thiết bị | Lệnh đi thẳng xuống thiết bị thật và không có phản hồi xác nhận. Hãy kiểm tra lại thiết bị sau khi gửi. |
| Xóa | Không hoàn tác được từ trợ lý. |
| Duyệt QC, nhập kho, submit/cancel phiếu kho | Ghi hoặc đảo sổ kho. |
| Tạo nhân viên | Tạo tài khoản đăng nhập mới. |
| In tem | Đánh dấu đã in, có thể in ra máy in thật. |

Không có tool bật/tắt lịch chạy thiết bị. Việc đó vẫn làm trên web hoặc app.

## Xử lý sự cố

| Hiện tượng | Cách xử lý |
|---|---|
| Tool biến mất hoặc báo 401 | Kết nối lại (Connect / Authenticate). |
| Không thấy tool mới | Ngắt kết nối rồi kết nối lại, hoặc khởi động lại app. |
| Bị từ chối đăng ký (403) hoặc không thấy trang đăng nhập | Tổ chức của bạn chưa được bật MCP. Liên hệ VIIS. |
| Trang đăng nhập từ chối tài khoản | Tài khoản cần OTP hoặc chưa được cấp quyền. Liên hệ quản trị VIIS. |
| Ghi không được dù đã xác nhận | Kết nối chỉ có quyền đọc, hoặc tài khoản thiếu quyền mục đó trên web. |
| "Rate limit" | Đợi một lúc rồi thử lại. |
