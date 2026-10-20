# VIIS AI Skills

Kết nối trợ lý AI (Claude, ChatGPT, Cursor) với dữ liệu nông trại trên VIIS: công việc, kế hoạch sản xuất, phân bổ, lô môi trường, QC, tồn kho, thiết bị. Trợ lý làm việc bằng chính tài khoản VIIS của bạn, với đúng quyền và dữ liệu bạn thấy trên web.

*English: Claude Code plugin marketplace with the VIIS MCP server connection and the `viis-mcp` skill. The server is `https://iot.viis.tech/api/v2/mcp`, and sign-in is OAuth with your VIIS account.*

> **Trạng thái:** kết nối MCP trên production đang được mở dần cho từng khách hàng. Nếu khi kết nối bạn bị từ chối đăng ký hoặc không thấy trang đăng nhập, hãy liên hệ VIIS để được bật quyền.

## Có gì trong repo

| Thành phần | Dùng cho | Nội dung |
|---|---|---|
| Plugin `viis` | Claude Code | Server MCP `viis` + skill `viis-mcp`, cài một lần là có cả hai |
| Skill `viis-mcp` | Claude Code, Claude Desktop, claude.ai | Dạy Claude dùng tool VIIS cho đúng: tìm id trước, ghi theo thứ tự xem trước rồi mới xác nhận, giờ Việt Nam, những thao tác phải hỏi lại |
| [`docs/connect.md`](docs/connect.md) | Mọi client | Hướng dẫn kết nối từng bước cho Claude Desktop, Claude Code, ChatGPT, Cursor |

## Cài nhanh

**Claude Code**

```text
/plugin marketplace add VIIS-IOT/viis-ai-skills
/plugin install viis@viis
```

Sau đó gõ `/mcp`, chọn `viis`, chọn **Authenticate** rồi đăng nhập tài khoản VIIS.

**Claude Desktop / claude.ai**

1. Thêm connector: Settings → Connectors → Add custom connector → URL `https://iot.viis.tech/api/v2/mcp` → Connect → đăng nhập VIIS.
2. Cài skill: tải `viis-mcp.zip` ở mục [Releases](https://github.com/VIIS-IOT/viis-ai-skills/releases), rồi vào Settings → Capabilities → Skills → Upload skill.

**ChatGPT, Cursor:** xem [`docs/connect.md`](docs/connect.md).

## An toàn dữ liệu

- Trợ lý chỉ thấy dữ liệu mà tài khoản của bạn được phép xem.
- Mọi thao tác ghi đều chạy hai bước: **xem trước** (chưa lưu gì) → bạn đồng ý → **xác nhận**.
- Gửi lệnh thiết bị, xóa, duyệt QC và ghi sổ kho luôn được hỏi lại trước khi thực hiện.
- Không có API key nào cần sao chép. Bạn có thể thu hồi kết nối bất kỳ lúc nào bằng cách ngắt connector.

## Hỗ trợ

Liên hệ đội VIIS qua kênh hỗ trợ khách hàng của bạn. Mọi góp ý về skill xin gửi qua mục Issues của repo.

---

© 2026 VIIS. All rights reserved. Xem [LICENSE](LICENSE).
