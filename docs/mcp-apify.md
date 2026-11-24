# Apify MCP Server

Nguồn: https://github.com/apify/apify-mcp-server

## Cấu hình

File `.mcp.json` ở gốc repo khai báo server `apify` qua endpoint hosted:

```
https://mcp.apify.com?tools=actors,runs,storage,docs
```

Transport: Streamable HTTP. Mỗi session Claude Code mở trong repo này sẽ tự nạp server.

### Vì sao pin tham số `tools`

README của Apify cảnh báo: bộ tool mặc định có thể thay đổi giữa các phiên bản, nên
khai báo `tools` tường minh nếu cần giao diện ổn định. Bốn nhóm đang pin:

| Nhóm | Dùng để |
|---|---|
| `actors` | Tìm Actor, đọc chi tiết, gọi chạy Actor (`search-actors`, `fetch-actor-details`, `call-actor`) |
| `runs` | Theo dõi / huỷ lần chạy (`get-actor-run`, `abort-actor-run`) |
| `storage` | Đọc kết quả (`get-dataset-items`, `get-key-value-store-record`) |
| `docs` | Tra tài liệu Apify (`search-apify-docs`, `fetch-apify-docs`) |

Muốn thêm nhóm khác: `tasks`, `schedules`, `builds`, `versions`, `api`, `dev`.
Muốn nạp sẵn một Actor cụ thể thì thêm tên Actor vào danh sách, ví dụ
`?tools=actors,runs,storage,docs,apify/instagram-scraper`.

## Xác thực

Endpoint hosted KHÔNG lưu token trong repo. Có hai đường:

**1. Máy cá nhân (Claude Code local) — OAuth**

Mở Claude Code trong repo, chạy `/mcp`, chọn `apify`, đăng nhập. Token do Claude Code
giữ, không nằm trong file nào của repo.

**2. Cloud session (claude.ai/code) — network secret**

Cloud session không mở được luồng OAuth. Thay vào đó lưu Apify token làm network secret
cho host `mcp.apify.com` với header `Authorization: Bearer <APIFY_TOKEN>`. Proxy của
environment sẽ tự chèn header vào request. Token lấy ở Apify Console →
Settings → API & Integrations.

**Tuyệt đối không** commit token vào `.mcp.json`, và không đặt token làm environment
variable thường (giá trị lưu dạng plain text).

## Bản stdio (tuỳ chọn, chạy local)

Nếu muốn chạy server tại máy thay vì dùng endpoint hosted — cần Node.js 22+:

```
npx -y @apify/actors-mcp-server --tools actors,runs,storage,docs
```

Token đọc từ biến môi trường `APIFY_TOKEN`, hoặc từ phiên `apify login`.

## Chi phí

Kết nối MCP không mất phí. Nhưng mỗi lần `call-actor` là một lần chạy Actor thật trên
nền tảng Apify và bị tính vào tài khoản Apify theo giá của Actor đó. Gói free có
khoảng 5 USD usage/tháng.

Hai tham số trong `callOptions` của `call-actor` dùng để chặn trần chi phí:

- `maxTotalChargeUsd` — trần tiền cho một lần chạy
- `maxItems` — trần số bản ghi thu về

Nên luôn đặt một trong hai khi gọi Actor scrape dữ liệu lớn.

Giới hạn kỹ thuật: server hosted nhận tối đa 20 request/giây mỗi tài khoản.

## Lưu ý về network của cloud environment

Environment `Default` hiện ở chế độ `limited` nhưng bật `allow_mcp_servers`, nghĩa là
host của MCP server khai báo cho session được phép truy cập mà không cần thêm vào
allowlist. Nếu session báo `ERR_PROXY_TUNNEL 403` với `mcp.apify.com` thì phải thêm
host này vào allowed hosts của environment.

Riêng `docs.apify.com` hiện KHÔNG truy cập được từ cloud session (`ENOTFOUND`) — nếu
cần đọc tài liệu Apify bằng web fetch thì phải mở thêm host đó.
