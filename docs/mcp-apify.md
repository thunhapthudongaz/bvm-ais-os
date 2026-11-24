# Apify MCP Server

Nguồn: https://github.com/apify/apify-mcp-server

## Cấu hình

File `.mcp.json` ở gốc repo khai báo server `apify` qua endpoint hosted, transport
Streamable HTTP. Mỗi session Claude Code mở trong repo này sẽ tự nạp server.

```
https://mcp.apify.com?tools=actors,runs,storage,docs
  ,compass/Google-Maps-Reviews-Scraper
  ,apify/facebook-reviews-scraper
  ,apify/facebook-comments-scraper
```

### Nhóm tool (categories)

README của Apify cảnh báo bộ tool mặc định có thể đổi giữa các phiên bản, nên pin
tường minh. Bốn nhóm đang dùng:

| Nhóm | Dùng để |
|---|---|
| `actors` | Tìm Actor, đọc chi tiết, gọi chạy bất kỳ Actor nào (`search-actors`, `fetch-actor-details`, `call-actor`) |
| `runs` | Theo dõi / huỷ lần chạy (`get-actor-run`, `abort-actor-run`) |
| `storage` | Đọc kết quả (`get-dataset-items`, `get-key-value-store-record`) |
| `docs` | Tra tài liệu Apify (`search-apify-docs`, `fetch-apify-docs`) |

Nhóm chọn thêm được nếu cần: `tasks`, `schedules`, `builds`, `versions`, `api`, `dev`.

### Ba Actor nạp sẵn

Nạp sẵn nghĩa là Actor có tool riêng với schema input đầy đủ, gọi trực tiếp không cần
tìm. Ba Actor này phục vụ việc theo dõi uy tín — việc chạy thường xuyên nhất:

| Actor | Dùng để |
|---|---|
| `compass/Google-Maps-Reviews-Scraper` | Review Google Maps của bệnh viện và đối thủ |
| `apify/facebook-reviews-scraper` | Đánh giá trên fanpage Facebook |
| `apify/facebook-comments-scraper` | Comment dưới bài đăng Facebook |

Không nạp sẵn nhiều hơn vì mỗi Actor thêm một tool với schema input lớn, làm phình
danh sách tool. Các Actor dưới đây gọi qua `call-actor` với đúng ID, không cần pin:

| Actor | Dùng để | Số lần chạy trên Store |
|---|---|---|
| `apify/facebook-posts-scraper` | Bài đăng fanpage | 57,4M |
| `clockworks/tiktok-scraper` | Video TikTok theo profile / hashtag / từ khoá | 153M |
| `clockworks/tiktok-comments-scraper` | Comment video TikTok | 20,7M |
| `clockworks/tiktok-profile-scraper` | Thông tin profile TikTok | 22M |

## Xác thực

File `.mcp.json` KHÔNG chứa token. Hai đường tuỳ môi trường:

**1. Máy cá nhân (Claude Code local) — OAuth**

Mở Claude Code trong repo, chạy `/mcp`, chọn `apify`, đăng nhập. Token do Claude Code
giữ, không nằm trong file nào của repo.

**2. Cloud session (claude.ai/code) — network secret**

Cloud session không mở được luồng OAuth. Lưu Apify token làm network secret cho host
`mcp.apify.com`, header `Authorization: Bearer <APIFY_TOKEN>`. Proxy của environment
sẽ tự chèn header. Token lấy ở Apify Console → Settings → API & Integrations.

**Tuyệt đối không** commit token vào `.mcp.json`, và không đặt token làm environment
variable thường (giá trị lưu dạng plain text).

## Chi phí — đọc trước khi chạy

Kết nối MCP miễn phí. Nhưng mỗi lần `call-actor` là một lần chạy Actor thật trên nền
tảng Apify, tính tiền vào tài khoản Apify. Gói free có 5 USD usage/tháng.

Giá dưới đây là bậc FREE, lấy từ `api.apify.com/v2/acts/<actor>` ngày 2026-11-24.
Bậc trả phí (BRONZE → DIAMOND) rẻ hơn 20–80%.

| Actor | Giá mỗi bản ghi | Phí khởi động |
|---|---|---|
| `compass/Google-Maps-Reviews-Scraper` | $0.0006 / review | $0.00005 |
| `apify/facebook-reviews-scraper` | $0.0025 / review | $0.001 |
| `apify/facebook-comments-scraper` | $0.0025 / comment | $0.001 |
| `apify/facebook-posts-scraper` | $0.005 / post | $0.001 |
| `clockworks/tiktok-scraper` | $0.0037 / video | $0.001 |
| `clockworks/tiktok-comments-scraper` | $0.00125 / comment | — |

Ước tính một lần quét điển hình:

| Việc | Khối lượng | Chi phí |
|---|---|---|
| Review Google Maps bệnh viện | 500 review | ~$0.30 |
| Đánh giá fanpage Facebook | 500 review | ~$1.25 |
| Comment một chiến dịch Facebook | 1.000 comment | ~$2.50 |
| Bài đăng fanpage đối thủ | 100 post | ~$0.50 |
| Video TikTok theo hashtag | 100 video | ~$0.37 |
| Comment video TikTok | 1.000 comment | ~$1.25 |

Nghĩa là gói free 5 USD/tháng chỉ đủ khoảng **hai lần quét 1.000 comment Facebook**.
Quét Facebook là loại đắt nhất trong bộ này.

### Bắt buộc chặn trần

Luôn đặt ít nhất một trong hai tham số trong `callOptions` của `call-actor`:

- `maxTotalChargeUsd` — trần tiền cho một lần chạy
- `maxItems` — trần số bản ghi thu về

Giới hạn kỹ thuật: server hosted nhận tối đa 20 request/giây mỗi tài khoản.

## Network

Repo này dùng environment `bvm-ais-os` (personal), network `limited`, mở 3 host:

| Host | Vì sao |
|---|---|
| `mcp.apify.com` | MCP server kết nối |
| `api.apify.com` | Tra ID và giá Actor qua REST API |
| `docs.apify.com` | Đọc tài liệu Apify bằng web fetch |

`allow_package_managers` và `allow_mcp_servers` đều bật. Environment `Default` không
được sửa, nên các repo khác không bị ảnh hưởng.

Khi mở session mới cho repo này, chọn environment `bvm-ais-os`. Nếu chạy trên
`Default`, `mcp.apify.com` sẽ bị proxy chặn 403.

## Bản stdio (tuỳ chọn, chạy local)

Nếu muốn chạy server tại máy thay vì dùng endpoint hosted — cần Node.js 22+:

```
npx -y @apify/actors-mcp-server --tools actors,runs,storage,docs
```

Token đọc từ biến môi trường `APIFY_TOKEN`, hoặc từ phiên `apify login`.
