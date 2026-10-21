# BrewLite

Ứng dụng đặt đồ uống **không dùng tiền mặt**: khách chọn món, đặt hàng, thanh toán bằng ví/thẻ (giả lập) và theo dõi trạng thái đơn.
Bài tập lớn môn Công nghệ Phần mềm, làm theo quy trình Agile Scrum.

## Công nghệ

| Thành phần | Công nghệ |
|---|---|
| Frontend | Next.js (React, TypeScript), TailwindCSS, Zustand |
| Backend | NestJS (TypeScript), REST, class-validator, JWT (Passport), bcrypt |
| CSDL | PostgreSQL + Prisma 7 |
| Thanh toán | Mock payment (mô phỏng Momo/VNPay/Stripe), không trừ tiền thật |
| Kiểm thử | Vitest + Supertest |
| Đóng gói | Docker, Docker Compose |

## Yêu cầu

- **Docker Desktop** (để chạy cả hệ thống bằng một lệnh).
- Nếu chạy chế độ phát triển: **Node.js 24 LTS** (bắt buộc từ 22.12 trở lên) và Git.

## Chạy nhanh bằng Docker

Ở thư mục gốc của dự án:

```bash
docker compose up -d --build
```

Lần đầu mất vài phút để tải image và build. Khi xong:

| Dịch vụ | Địa chỉ |
|---|---|
| Ứng dụng web | http://localhost:3000 |
| API backend | http://localhost:3001 |
| PostgreSQL | localhost:5432 (user `brewlite`, mật khẩu `brewlite`, database `brewlite`) |

Khi khởi động, backend tự tạo bảng (`prisma migrate deploy`) và nạp dữ liệu mẫu (`prisma db seed`).

Các lệnh thường dùng:

```bash
docker compose ps                # xem trạng thái 3 dịch vụ
docker compose logs -f backend   # xem log backend
docker compose down              # tắt, giữ nguyên dữ liệu
docker compose down -v           # tắt và xóa dữ liệu (về trạng thái ban đầu)
```

## Dữ liệu mẫu

- **4 sản phẩm**: Cà phê sữa 35.000đ, Americano 40.000đ, Cappuccino 45.000đ, Trà đào 39.000đ (mỗi món tồn kho 50).
- **Mã giảm giá** `BREW10`: giảm 10%.
- **Tài khoản**: tự đăng ký ở trang `/register` (mật khẩu tối thiểu 6 ký tự).

## Chạy chế độ phát triển

Chỉ chạy database bằng Docker, còn backend và frontend chạy trực tiếp để sửa code thấy kết quả ngay.

```bash
# 1. Database
docker compose up -d db

# 2. Backend (terminal 1)
cd backend
cp .env.example .env        # rồi điền JWT_SECRET
npm install
npx prisma generate
npx prisma migrate deploy
npx prisma db seed
npm run start:dev           # http://localhost:3001

# 3. Frontend (terminal 2)
cd frontend
cp .env.example .env.local
npm install
npm run dev                 # http://localhost:3000
```

Mọi lệnh `prisma` phải chạy trong thư mục `backend` và dùng `npx prisma ...` (dự án dùng Prisma 7).

## Biến môi trường

| Biến | Nơi dùng | Ý nghĩa |
|---|---|---|
| `DATABASE_URL` | backend | Chuỗi kết nối PostgreSQL |
| `JWT_SECRET` | backend | Khóa ký token đăng nhập, phải đặt chuỗi dài và bí mật |
| `PORT` | backend | Cổng backend (mặc định 3001) |
| `NEXT_PUBLIC_API_URL` | frontend | Địa chỉ backend mà **trình duyệt** gọi tới |
| `INTERNAL_API_URL` | frontend | Địa chỉ backend mà **máy chủ Next.js** gọi tới (trong Docker là `http://backend:3001`) |

Không đưa file `.env` lên Git. Dùng các file `.env.example` làm mẫu.

## Chạy test

Cần database đang chạy (`docker compose up -d db`).

```bash
cd backend
npm run test        # test đơn vị
npm run test:e2e    # test end-to-end trên database thật
```

Các test chứng minh yêu cầu của Task 10:

| Yêu cầu | Test |
|---|---|
| (a) Chặn chuyển trạng thái sai | `src/orders/order-state-machine.spec.ts`, ca hủy đơn trong `test/payments.e2e-spec.ts` |
| (b) Gửi 2 lần cùng `Idempotency-Key` chỉ tạo 1 thanh toán | `test/payments.e2e-spec.ts` (gửi liên tiếp và gửi đồng thời) |
| (c) Đặt đồng thời không vượt tồn kho | `test/inventory.e2e-spec.ts` |
| Khuyến mãi và điểm thưởng | `src/promotions/discount.spec.ts`, `test/promotions.e2e-spec.ts` |

## API

| Phương thức | Đường dẫn | Chức năng | Đăng nhập |
|---|---|---|---|
| GET | `/products` | Danh sách sản phẩm | Không |
| GET | `/products/:id` | Chi tiết sản phẩm, kèm bảng size/topping | Không |
| POST | `/auth/register` | Đăng ký, trả JWT | Không |
| POST | `/auth/login` | Đăng nhập, trả JWT | Không |
| GET | `/auth/me` | Thông tin người dùng hiện tại | Có |
| POST | `/promotions/validate` | Kiểm tra mã giảm giá | Không |
| POST | `/orders` | Tạo đơn `PENDING` (kèm `voucherCode` tùy chọn) | Có |
| GET | `/orders/me` | Lịch sử đơn của tôi | Có |
| GET | `/orders/:id` | Chi tiết một đơn | Có |
| POST | `/orders/:id/cancel` | Hủy đơn | Có |
| POST | `/payments` | Thanh toán, bắt buộc header `Idempotency-Key` | Có |

Gọi cửa cần đăng nhập bằng header `Authorization: Bearer <accessToken>`.

## Luồng sử dụng

1. Chọn món ở menu, chọn size và topping ở trang chi tiết, thêm vào giỏ.
2. Ở giỏ hàng, nhập mã giảm giá (nếu có) và bấm **Đặt hàng** (cần đăng nhập). Đơn được tạo ở trạng thái `PENDING` và tồn kho bị trừ.
3. Ở trang đơn hàng, chọn Ví hoặc Thẻ rồi **Thanh toán**, hoặc **Hủy đơn**.
4. Thanh toán thành công thì đơn chuyển `PAID` và khách được cộng điểm. Thanh toán lỗi thì đơn chuyển `PAYMENT_FAILED`, vẫn thanh toán lại hoặc hủy được.

Ô "Giả lập thanh toán lỗi" ở trang đơn hàng chỉ dùng để demo nhánh lỗi.

## Quy tắc nghiệp vụ

**Giá:** giá menu là giá size S. Size M +5.000đ, size L +10.000đ, mỗi topping +5.000đ. Backend luôn tự tính lại giá khi tạo đơn, không tin giá từ trình duyệt.

**Máy trạng thái đơn hàng:** `PENDING → PAID → PREPARING → READY → COMPLETED`. Ngoại lệ: `PENDING → PAYMENT_FAILED → PENDING/PAID` (thử lại), và `PENDING`/`PAYMENT_FAILED → CANCELLED`. Mọi chuyển tiếp khác bị `assertTransition` chặn. Đơn đã thanh toán không hủy được vì hệ thống chưa có bước hoàn tiền. Các trạng thái `PREPARING`, `READY`, `COMPLETED` thuộc thao tác của nhân viên quầy, nằm ngoài phạm vi MVP.

**Thanh toán idempotent:** mỗi yêu cầu thanh toán kèm `Idempotency-Key`. Cùng key chỉ xử lý một lần, các lần sau trả lại kết quả cũ. Dòng đơn được khóa (`SELECT ... FOR UPDATE`) nên hai yêu cầu đồng thời không gọi cổng thanh toán hai lần.

**Tồn kho:** tạo đơn và trừ kho nằm trong cùng một giao dịch. Mỗi lần trừ kho kiểm tra cột `version` (khóa lạc quan) và điều kiện `stock >= số lượng`, nên đặt đồng thời không bán vượt số còn lại. Đơn nhiều món thiếu một món thì cả đơn bị từ chối. Hủy đơn thì hoàn kho.

**Khuyến mãi và điểm thưởng:** mã giảm giá tính theo phần trăm, có hạn dùng và có thể tắt. Khách được cộng 1 điểm cho mỗi 10.000đ đã trả, cộng trong cùng giao dịch với việc chuyển đơn sang `PAID`, nên gửi lại cùng `Idempotency-Key` không cộng thêm.

## Cấu trúc thư mục

```
brewlite/
├── docker-compose.yml
├── backend/              NestJS
│   ├── prisma/           schema, migrations, seed
│   ├── src/              auth, users, products (kèm inventory), promotions,
│   │                     orders (kèm state machine), payments, common, prisma
│   └── test/             test end-to-end
└── frontend/             Next.js
    └── src/
        ├── app/          các trang
        ├── features/     auth, catalog, cart, orders, payments
        ├── components/   thành phần dùng chung
        └── lib/          gọi API, định dạng, kiểu dữ liệu
```

## Giới hạn đã biết

- Thanh toán là giả lập, không có hoàn tiền.
- Đơn `PENDING` bỏ dở vẫn giữ kho cho tới khi bị hủy (chưa có tác vụ tự hủy đơn quá hạn).
- Mã giảm giá chưa giới hạn số lần dùng.
- Token đăng nhập lưu trong `localStorage` cho đơn giản.
- Chưa có giao diện cho nhân viên quầy (ngoài phạm vi MVP).
