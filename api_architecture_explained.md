# 🏗️ KIẾN TRÚC API IMAGE INJECTION DETECTION — GIẢI THÍCH CHI TIẾT

> Tài liệu này giải thích toàn bộ cách API hoạt động bên trong: từ cách nhận request, xử lý đa luồng, async/await, đến cách kết nối Oracle Database. Mỗi khái niệm đều có ví dụ đời thường và chú thích file/hàm cụ thể trong code.

---

## MỤC LỤC

1. [Tổng quan: API hoạt động như thế nào?](#1-tổng-quan)
2. [Async / Await / Event Loop — Giải thích bằng ví dụ Nhà Hàng](#2-async--await--event-loop)
3. [ThreadPoolExecutor — 200 Workers](#3-threadpoolexecutor)
4. [Oracle Connection Pool — Bể Đường Dây Điện Thoại](#4-oracle-connection-pool)
5. [Các Cơ Chế Timeout — Hệ Thống Đồng Hồ Hẹn Giờ](#5-các-cơ-chế-timeout)
6. [Kịch bản thực tế: 500 Request khi DB chết](#6-kịch-bản-thực-tế)

---

## 1. TỔNG QUAN

### API của bạn giống một Nhà Hàng

Hãy tưởng tượng API là một **nhà hàng cao cấp**:

| Thành phần trong code | Vai trò trong nhà hàng |
|---|---|
| **Uvicorn** (web server) | Cửa chính nhà hàng — tiếp nhận khách |
| **FastAPI event loop** | **1 bếp trưởng duy nhất** — điều phối mọi thứ |
| **ThreadPoolExecutor (200 workers)** | 200 phụ bếp — nhận đơn hàng |
| **Oracle Connection Pool (10 conn)** | 10 đường dây điện thoại gọi đến nhà cung cấp nguyên liệu (DB) |
| **Redis** | Tủ lạnh cache — lưu nguyên liệu hay dùng |
| **VALIDATION_TIMEOUT = 5s** | Đồng hồ bếp: món nào quá 5 phút thì hủy |

### Sơ đồ tổng quan

```
Khách hàng (HTTP Request)
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  UVICORN (Cửa nhà hàng)                                 │
│  📁 File: uvicorn chạy app.main:app                     │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│  FASTAPI EVENT LOOP (Bếp trưởng - CHỈ CÓ 1 NGƯỜI)      │
│  📁 File: app/api/routes/transaction_validator.py        │
│  🔧 Hàm: validate_transaction()                         │
│                                                          │
│  Bếp trưởng nhận đơn → Giao cho phụ bếp → Bấm đồng hồ │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │  THREADPOOLEXECUTOR (200 phụ bếp)                  │  │
│  │  📁 File: transaction_validator.py dòng 34          │  │
│  │  [Phụ bếp 1] [Phụ bếp 2] ... [Phụ bếp 200]       │  │
│  └────────────────────────────────────────────────────┘  │
│                       │                                  │
│                       ▼                                  │
│  ┌────────────────────────────────────────────────────┐  │
│  │  VALIDATION SERVICE (Quy trình nấu món)            │  │
│  │  📁 File: app/services/validation_service.py        │  │
│  │  🔧 Hàm: process_validation()                      │  │
│  │                                                     │  │
│  │  1. Lấy nguyên liệu từ tủ lạnh (Redis cache)      │  │
│  │  2. Kiểm tra whitelist, blacklist                   │  │
│  │  3. Gọi điện thoại đến kho (Oracle DB)             │  │
│  │  4. Quyết định (decision_engine)                    │  │
│  │  5. Kiểm tra vector fraud (nếu có)                 │  │
│  └────────────────────┬───────────────────────────────┘  │
│                       │                                  │
│                       ▼                                  │
│  ┌────────────────────────────────────────────────────┐  │
│  │  ORACLE CONNECTION POOL (10 đường dây điện thoại)  │  │
│  │  📁 File: app/core/oracle_pool.py                   │  │
│  │  🔧 Class: OracleConnectionPool                     │  │
│  │  [☎️1] [☎️2] [☎️3] ... [☎️10]                      │  │
│  └────────────────────┬───────────────────────────────┘  │
│                       │                                  │
└───────────────────────┼──────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │   Oracle DB       │
              │   (Nhà kho)       │
              │   10.122.121.126  │
              └──────────────────┘
```

---

## 2. ASYNC / AWAIT / EVENT LOOP

### Ví dụ đời thường: BẾP TRƯỞNG NHÀ HÀNG

Trong Python bình thường (**synchronous / đồng bộ**), mọi thứ hoạt động kiểu:

> Bếp trưởng nhận đơn món A → Tự tay nấu món A từ đầu đến cuối → Xong → Mới nhận đơn món B

Nếu món A cần đợi nước sôi 3 phút, bếp trưởng **đứng đó chờ 3 phút**, không làm gì cả. Lãng phí!

---

Trong Python **async / bất đồng bộ** (cách FastAPI hoạt động):

> Bếp trưởng nhận đơn món A → Cho nước lên bếp → **Trong lúc chờ nước sôi** → Nhận đơn món B → Cắt rau cho món B → Nước sôi rồi! → Quay lại món A

**Bếp trưởng không bao giờ đứng yên.** Khi có việc phải chờ (I/O: gọi DB, đọc file, gọi API), anh ấy chuyển sang làm việc khác.

#### Ánh xạ vào code:

```python
# 📁 File: app/services/validation_service.py
# 🔧 Hàm: process_validation()

async def process_validation(payload, api_version):
    # Bếp trưởng bắt đầu nấu món
    
    # "await" = "Tôi cần chờ kết quả này, nhưng trong lúc chờ, hãy đi làm việc khác"
    sus_device_model_list = await get_sus_device_model_list()
    #                        ^^^^^
    #                        "Đi lấy danh sách thiết bị nghi ngờ từ config"
    #                        "Nếu config chưa load xong → chờ → làm việc khác"
    
    whitelist = await get_whitelist()
    #           ^^^^^
    #           "Đi lấy whitelist, nếu cần chờ thì chờ"
    
    # ...
    
    result = await oracle_pool.fetch_one(sql, params)
    #        ^^^^^
    #        "Gọi điện thoại đến kho (DB), TRONG LÚC CHỜ KHO TRẢ LỜI"
    #        "→ Bếp trưởng đi xử lý request của khách hàng khác"
    #        "→ Kho trả lời → Quay lại xử lý tiếp request này"
```

#### Từ khóa quan trọng:

| Từ khóa | Ý nghĩa | Ví dụ đời thường |
|---|---|---|
| `async def` | "Hàm này CÓ THỂ bị tạm dừng để nhường chỗ" | "Món này có bước chờ (nước sôi, gọi kho)" |
| `await` | "Tạm dừng ở đây, đi làm việc khác, quay lại khi có kết quả" | "Cho nước lên bếp, đi cắt rau cho món khác, nước sôi thì quay lại" |
| Event Loop | "Vòng lặp điều phối" — quyết định lúc nào quay lại món nào | "Bếp trưởng nhìn quanh: nước sôi chưa? Kho gọi lại chưa?" |

#### Event Loop hoạt động chi tiết:

```
Bếp trưởng (Event Loop) xoay vòng liên tục:

Vòng 1:
  → Đơn A: Bắt đầu nấu → Gọi DB → CHỜ (await) → Tạm dừng A
  → Đơn B: Bắt đầu nấu → Kiểm tra whitelist → Gọi DB → CHỜ → Tạm dừng B
  → Đơn C: Bắt đầu nấu → ...

Vòng 2:
  → Kiểm tra: DB đã trả kết quả cho đơn A chưa? → CHƯA → Bỏ qua
  → Kiểm tra: DB đã trả kết quả cho đơn B chưa? → RỒI! → Tiếp tục xử lý B
  → Đơn D: Bắt đầu nấu → ...

Vòng 3:
  → Kiểm tra: DB đã trả kết quả cho đơn A chưa? → RỒI! → Tiếp tục xử lý A
  → ...

(Lặp lại mãi mãi cho đến khi tắt server)
```

> **Key insight:** Event loop giống bếp trưởng siêu nhân có thể nấu 1000 món cùng lúc, MIỄN LÀ anh ấy không phải đứng chờ ở bất kỳ món nào.

---

## 3. THREADPOOLEXECUTOR

### Ví dụ đời thường: ĐỘI PHỤ BẾP

Bếp trưởng (Event Loop) giỏi điều phối, nhưng anh ấy chỉ có **2 tay**. Nếu có việc **nặng** (tính toán CPU, xử lý ảnh, giải mã...), anh ấy không thể tự làm vì sẽ bị **block** — không xoay vòng được.

Giải pháp: Thuê **200 phụ bếp** (ThreadPoolExecutor).

```python
# 📁 File: app/api/routes/transaction_validator.py, dòng 34-37
validation_executor = ThreadPoolExecutor(
    max_workers=200,                    # 200 phụ bếp
    thread_name_prefix="validation_worker"  # Tên tag: "validation_worker_1", "validation_worker_2"...
)
```

#### Cách bếp trưởng giao việc cho phụ bếp:

```python
# 📁 File: transaction_validator.py, dòng 109-116
task = await loop.run_in_executor(
    validation_executor,           # Giao cho đội phụ bếp
    functools.partial(
        validation_service.process_validation,  # Công thức nấu
        payload,                               # Nguyên liệu (request data)
        settings.API_VERSION                   # Phiên bản menu
    )
)
```

#### Quy trình:

```
Bếp trưởng:
  "Phụ bếp số 47! Nấu đơn hàng này theo công thức process_validation!"
  
Phụ bếp 47:
  "Vâng!" → Nhận nguyên liệu → Bắt đầu nấu

Bếp trưởng (không đợi):
  → Quay sang xử lý đơn hàng khác
  → Khi phụ bếp 47 xong → Bếp trưởng nhận kết quả → Trả cho khách
```

#### ⚠️ Lưu ý quan trọng về code hiện tại:

Trong code của bạn, [process_validation](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/services/validation_service.py#89-402) là hàm **`async def`**. Nhưng `run_in_executor` được thiết kế cho hàm **đồng bộ** (bình thường). Khi gọi hàm async qua `run_in_executor`:

1. Phụ bếp chỉ **mở sách công thức** (tạo coroutine) → mất ~0ms → xong ngay
2. **Việc nấu thực sự** (chạy coroutine) do bếp trưởng (event loop) tự làm

```
Phụ bếp 47: "Ơ... tôi chỉ mở sách công thức ra thôi à? OK xong rồi!" (0ms)
            → Trả sách công thức (coroutine) cho bếp trưởng
            → Phụ bếp rảnh ngay

Bếp trưởng: "Sách đây rồi, để TÔI tự nấu" 
            → Bếp trưởng nấu trên event loop
```

→ Hệ quả: 200 phụ bếp **gần như không làm gì nặng** trong kiến trúc hiện tại. Nhưng cũng không gây hại.

#### Khi nào ThreadPoolExecutor THỰC SỰ hữu ích?

Nếu [process_validation](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/services/validation_service.py#89-402) là hàm **đồng bộ** (`def` thay vì `async def`):

```python
# Ví dụ hàm ĐỒNG BỘ (nặng, sẽ block event loop nếu không dùng executor)
def process_validation_sync(payload, api_version):
    # Tính toán CPU nặng (giải mã ảnh, so sánh vector...)
    result = heavy_computation(payload)
    return result

# Dùng executor để tránh block bếp trưởng
task = await loop.run_in_executor(validation_executor, process_validation_sync, payload)
```

Lúc này phụ bếp **thực sự nấu** (chạy code nặng), bếp trưởng rảnh tay xử lý đơn khác.

---

## 4. ORACLE CONNECTION POOL

### Ví dụ đời thường: 10 ĐƯỜNG DÂY ĐIỆN THOẠI ĐẾN KHO

Nhà hàng cần gọi điện đến **Kho nguyên liệu** (Oracle Database) để lấy thông tin. Nhưng đường dây điện thoại đắt tiền, nên nhà hàng chỉ có thể thuê **tối đa 10 đường dây**.

### 4.1 Khởi tạo Pool (Thuê đường dây)

```python
# 📁 File: app/main.py, dòng 77-88
# 🔧 Hàm: lifespan() — chạy khi app khởi động
db_pool = await get_oracle_pool(
    host=settings.ORACLE_HOST,       # Địa chỉ kho: 10.122.121.126
    port=settings.ORACLE_PORT,       # Cổng: 1521
    user=settings.ORACLE_USER,       # Tên đăng nhập
    password=settings.ORACLE_PASSWORD,
    db=settings.ORACLE_DB,           # Tên kho: ekyc
    min_size=MIN_SIZE,               # Thuê sẵn 5 đường dây lúc mở cửa
    max_size=MAX_SIZE,               # Tối đa chỉ được thuê 10 đường dây
    increment=1,                     # Khi cần thêm, thuê từng đường 1
    max_lifetime_session=300,        # Mỗi đường dây sống tối đa 5 phút rồi đổi mới
    wait_timeout_ms=1000,            # Chờ đường dây rảnh tối đa 1 giây
)
```

#### Ví dụ:

```
🏬 Nhà hàng mở cửa (App khởi động)

Bước 1: Thuê sẵn 5 đường dây (min_size=5)
  ☎️ Đường dây 1: SẴN SÀNG
  ☎️ Đường dây 2: SẴN SÀNG
  ☎️ Đường dây 3: SẴN SÀNG
  ☎️ Đường dây 4: SẴN SÀNG
  ☎️ Đường dây 5: SẴN SÀNG

→ "OK, 5 đường dây sẵn sàng phục vụ!"
```

### 4.2 Sử dụng Connection (Gọi điện đến kho)

```python
# 📁 File: app/core/oracle_pool.py, dòng 319-375
# 🔧 Hàm: fetch_one()
async def fetch_one(self, sql, request_id, call_timeout, params):
    
    # Bước 1: Nhấc máy (lấy connection từ pool)
    async with self.pool.acquire() as conn:   # "Cho tôi 1 đường dây rảnh"
        
        if call_timeout != 0:
            conn.call_timeout = call_timeout  # "Nếu kho không trả lời trong Xs thì cúp máy"
        
        # Bước 2: Nói yêu cầu (chạy SQL)
        with conn.cursor() as cur:
            await cur.execute(sql, params)    # "Kho ơi, cho tôi hồ sơ khách hàng X"
            
            # Bước 3: Nghe câu trả lời
            row = await cur.fetchone()        # "Kho trả lời: Đây là hồ sơ..."
            return row
    
    # Bước 4: Cúp máy (connection tự động trả về pool)
    # "Đường dây rảnh, người khác dùng được"
```

#### Ví dụ đời thường:

```
Phụ bếp 47 cần gọi kho:

1. Phụ bếp đến phòng điện thoại
2. Nhìn bảng: "Đường dây rảnh: 3/5"  
3. Nhấc máy đường dây số 2
4. Quay số: "SELECT VECTOR FROM FACEPAY_LOGIN WHERE ID = 'abc123'"
5. Kho trả lời: "Đây là vector: [0.12, 0.45, ...]"
6. Phụ bếp ghi nhận kết quả
7. Cúp máy → Đường dây số 2 rảnh cho người khác

Toàn bộ quá trình: ~50-200ms (nếu kho hoạt động bình thường)
```

### 4.3 Pool mở rộng tự động (Thuê thêm đường dây)

```
Tình huống: 6 phụ bếp cùng cần gọi kho, nhưng chỉ có 5 đường dây

Phụ bếp 1-5: Nhấc máy ngay (5 đường dây sẵn có) ✅
Phụ bếp 6:   "Hết đường dây rồi!"
             → Pool tự động: "Thuê thêm 1 đường dây mới" (increment=1)
             → Tạo TCP connection mới đến DB
             → Mất ~50ms nếu DB khỏe
             → Phụ bếp 6 dùng đường dây mới ✅

(Pool có thể thuê thêm tối đa đến 10 đường dây = max_size)
```

### 4.4 Keep-Alive (Kiểm tra đường dây còn sống không)

```python
# 📁 File: app/core/oracle_pool.py, dòng 224-273
# 🔧 Hàm: _keepalive_worker()

# Mỗi 30 giây, hệ thống nhấc 1 đường dây lên kiểm tra:
async with pool.acquire() as conn:
    with conn.cursor() as cur:
        await cur.execute("SELECT 1 FROM DUAL")  # "Kho ơi, anh còn sống không?"
        await cur.fetchone()                      # Kho: "Sống!" → OK
```

#### Ví dụ:

```
Mỗi 30 giây, quản lý nhà hàng nhấc từng đường dây:

⏰ 10:00:00  Nhấc đường dây 1: "Alô kho? Còn sống chứ?" → "Sống!" ✅
⏰ 10:00:30  Nhấc đường dây 2: "Alô kho?" → "Sống!" ✅
⏰ 10:01:00  Nhấc đường dây 3: "Alô kho?" → ....im lặng.... ❌ → Log cảnh báo!
```

### 4.5 Singleton Pattern (Chỉ có DUY NHẤT 1 bể đường dây)

```python
# 📁 File: app/core/database.py
# 🔧 Hàm: get_db_pool() và set_db_pool()

# Toàn bộ nhà hàng chỉ có 1 phòng điện thoại duy nhất
# Mọi phụ bếp đều đến CÙNG 1 PHÒNG để gọi
# Không ai được tự thuê đường dây riêng

_db_pool = None  # Biến toàn cục giữ pool

def set_db_pool(pool):     # Gọi 1 lần duy nhất khi mở cửa nhà hàng
    global _db_pool
    _db_pool = pool

def get_db_pool():         # Mọi người gọi hàm này để lấy pool
    return _db_pool        # Luôn trả về cùng 1 pool
```

---

## 5. CÁC CƠ CHẾ TIMEOUT

### Ví dụ: HỆ THỐNG ĐỒNG HỒ HẸN GIỜ TRONG NHÀ HÀNG

Nhà hàng có **4 loại đồng hồ hẹn giờ** khác nhau, mỗi loại bảo vệ một khâu:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ⏰ ĐỒNG HỒ 1: VALIDATION_TIMEOUT = 5 giây                │
│  📁 transaction_validator.py dòng 40                        │
│  "Toàn bộ quy trình nấu món tối đa 5 giây"                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                      │   │
│  │  ⏰ ĐỒNG HỒ 2: call_timeout = 5000ms                │   │
│  │  📁 config.py: CALL_TIMEOUT = 5                      │   │
│  │  "Mỗi cuộc gọi đến kho tối đa 5 giây"              │   │
│  │  ┌──────────────────────────────────────────────┐   │   │
│  │  │                                               │   │   │
│  │  │  ⏰ ĐỒNG HỒ 3: wait_timeout = 1000ms         │   │   │
│  │  │  📁 main.py dòng 87                           │   │   │
│  │  │  "Chờ đường dây rảnh tối đa 1 giây"          │   │   │
│  │  │                                               │   │   │
│  │  │  ⏰ ĐỒNG HỒ 4: tcp_connect_timeout = 5s      │   │   │
│  │  │  📁 oracle_pool.py dòng 181                   │   │   │
│  │  │  "Kết nối TCP đến kho tối đa 5 giây"         │   │   │
│  │  │                                               │   │   │
│  │  └──────────────────────────────────────────────┘   │   │
│  │                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Chi tiết từng đồng hồ:

---

### ⏰ Đồng hồ 1: VALIDATION_TIMEOUT = 5 giây

```python
# 📁 File: app/api/routes/transaction_validator.py, dòng 40
VALIDATION_TIMEOUT = 5

# Dòng 118: Bấm đồng hồ
msg = await asyncio.wait_for(task, timeout=VALIDATION_TIMEOUT)
#                                          ^^^^^^^^^^^^^^^^^
#     "Bất kể bên trong đang làm gì, hết 5 giây là DỪNG HẾT"

# Dòng 149-169: Khi đồng hồ hết
except asyncio.TimeoutError:
    raise HTTPException(status_code=504, detail="Request timeout")
```

**Ví dụ:** Đồng hồ treo trên tường bếp. Khi bếp trưởng giao đơn cho phụ bếp, bếp trưởng bấm đồng hồ. Hết 5 giây, dù phụ bếp đang cắt hành hay đang chờ kho gọi lại, bếp trưởng la lên: "HẾT GIỜ! Hủy đơn!" → Trả khách HTTP 504.

**Tác dụng:** Lưới an toàn cuối cùng. Đảm bảo khách KHÔNG BAO GIỜ phải chờ quá 5 giây.

---

### ⏰ Đồng hồ 2: call_timeout = 5000ms (5 giây)

```python
# 📁 File: app/config.py, dòng 35
CALL_TIMEOUT: int = Field(5, env='CALL_TIMEOUT')

# 📁 File: app/services/validation_service.py, dòng 226-234
result = await oracle_pool.fetch_one(
    "SELECT value FROM FACEPAY_FRAUD WHERE id = :id",
    request_id,
    CALL_TIMEOUT * 1000,  # 5 * 1000 = 5000ms
    {'id': uid}
)

# 📁 File: app/core/oracle_pool.py, dòng 346-347
async with self.pool.acquire() as conn:
    if call_timeout != 0:
        conn.call_timeout = call_timeout  # Nói với đường dây: "5 giây không trả lời thì cúp"
```

**Ví dụ:** Phụ bếp nhấc máy gọi kho: "Cho tôi hồ sơ khách X". Rồi bấm đồng hồ cầm tay: "Nếu kho không trả lời trong 5 giây thì tôi cúp máy". Kho vẫn đang tìm hồ sơ... 5 giây... CÚP! → `oracledb.DatabaseError` → Phụ bếp quyết định "Không có thông tin → Không gian lận (False)".

**Khác biệt với Đồng hồ 1:** Đồng hồ 1 tính TOÀN BỘ quy trình (lấy config + kiểm tra whitelist + gọi DB + tính toán). Đồng hồ 2 chỉ tính riêng 1 cuộc gọi DB.

---

### ⏰ Đồng hồ 3: wait_timeout = 1000ms (1 giây)

```python
# 📁 File: app/main.py, dòng 87
wait_timeout_ms=1000

# 📁 File: app/core/oracle_pool.py, dòng 171-172
# Được dùng khi tạo pool
getmode=oracledb.POOL_GETMODE_TIMEDWAIT,  # "Chờ có thời hạn"
wait_timeout=self.wait_timeout_ms,         # 1000ms
```

**Ví dụ:** Phụ bếp đến phòng điện thoại, nhưng 10 đường dây đều đang có người dùng. Phụ bếp đứng chờ. Quản lý nói: "Chờ tối đa 1 giây. Hết 1 giây mà chưa có đường dây rảnh thì thôi, quay về bếp."

```
Phụ bếp: "Cho tôi 1 đường dây!"
Pool:    "10 đường dây đều bận. Xin chờ."

[...0.5 giây...]  Có đường dây rảnh! → Phụ bếp dùng ngay ✅

HOẶC

[...1.0 giây...]  Hết thời gian chờ! → Pool ném exception ❌
                   → Phụ bếp quay về: "Không gọi được kho"
```

**Quan trọng:** Đây là timeout ngắn nhất (1 giây), nên nó thường NỔ TRƯỚC các đồng hồ khác.

---

### ⏰ Đồng hồ 4: tcp_connect_timeout = 5 giây

```python
# 📁 File: app/core/oracle_pool.py, dòng 180-184
params=oracledb.PoolParams(
    tcp_connect_timeout=5.0,  # Kết nối TCP tối đa 5 giây
    retry_count=2,            # Thử lại 2 lần
    retry_delay=1             # Nghỉ 1 giây giữa các lần thử
)
```

**Ví dụ:** Khi pool cần **thuê thêm đường dây mới** (tạo connection mới đến DB), nó phải gọi công ty viễn thông (TCP handshake). Nếu công ty viễn thông không trả lời trong 5 giây → hủy. Thử lại 2 lần, mỗi lần nghỉ 1 giây.

```
Pool: "Cần thêm 1 đường dây!"
  → Gọi công ty viễn thông... chờ... 5 giây... không trả lời ❌
  → Nghỉ 1 giây
  → Thử lần 2... chờ... 5 giây... không trả lời ❌
  → "Không thuê được đường dây mới"
  Tổng: 5 + 1 + 5 = 11 giây (nhưng bị VALIDATION_TIMEOUT=5s cắt trước)
```

---

### Bảng tổng hợp tất cả timeout:

| Đồng hồ | Giá trị | File & dòng | Khi nào kích hoạt | Kết quả khi hết giờ |
|---|---|---|---|---|
| `VALIDATION_TIMEOUT` | **5s** | `transaction_validator.py:40` | Bao trùm toàn bộ request | HTTP 504 Gateway Timeout |
| `call_timeout` | **5s** (5000ms) | `config.py:35` | Khi gửi SQL query đến DB | `oracledb.DatabaseError` → decision=False |
| `wait_timeout` | **1s** (1000ms) | `main.py:87` | Khi chờ connection rảnh từ pool | Exception → decision=False |
| `tcp_connect_timeout` | **5s** | `oracle_pool.py:181` | Khi tạo connection MỚI đến DB | Tạo connection thất bại |
| `retry_count × retry_delay` | **2 × 1s** | `oracle_pool.py:182-183` | Sau mỗi lần tạo conn thất bại | Thử lại tối đa 2 lần |
| `max_lifetime_session` | **300s** (5 phút) | `main.py:86` | Connection tồn tại quá 5 phút | Connection bị đóng & tạo mới |
| `keepalive_interval` | **30s** | `oracle_pool.py:23` | Định kỳ kiểm tra sức khỏe pool | Chạy `SELECT 1 FROM DUAL` |
| `ping_interval` | **60s** | `oracle_pool.py:80` | Kiểm tra connection trước khi dùng | `oracledb` tự ping nếu idle > 60s |

---

## 6. KỊCH BẢN THỰC TẾ: 500 REQUEST KHI DB CHẾT

### Bối cảnh nhà hàng:

> Nhà hàng đang hoạt động bình thường. Bỗng nhiên **kho nguyên liệu bị cháy** (Database chết).
> 500 khách hàng đang xếp hàng chờ đặt món.

### Timeline chính xác:

```
t=0ms   🔥 Kho bị cháy! (DB chết)
        500 khách đang chờ xuất món

        Bếp trưởng (Event Loop): "Nhận 500 đơn, giao cho phụ bếp!"
        → 200 phụ bếp nhận đơn ngay (nhưng chỉ tạo coroutine ~ 0ms)
        → 300 đơn xếp hàng nhưng phụ bếp rảnh ngay → nhận tiếp
        → Tất cả 500 coroutine được tạo trong vài ms

        ⏰ ĐỒNG HỒ 1 (VALIDATION_TIMEOUT=5s): BẮT ĐẦU ĐẾM cho tất cả 500
```

#### Nhóm A: 5 đơn đầu tiên (Connection có sẵn)

```
t=0ms     5 coroutine gọi pool.acquire() → Nhận connection ngay (min=5 sẵn)
          → Nhấc máy gọi kho: "SELECT VECTOR FROM FACEPAY_LOGIN..."
          → Kho im lặng... (cháy rồi mà)
          
          ⏰ ĐỒNG HỒ 2 (call_timeout=5s): Bắt đầu đếm cho cuộc gọi

t=5000ms  ⏰ ĐỒNG HỒ 1 hết! (VALIDATION_TIMEOUT=5s)
          ⏰ ĐỒNG HỒ 2 cũng gần hết! (call_timeout=5s)
          
          → Đồng hồ 1 cắt ngang trước (nó ở tầng ngoài cùng)
          → 5 request này: HTTP 504 Gateway Timeout ❌
          
          Thời gian: ~5 giây
```

#### Nhóm B: 5 đơn tiếp theo (Cần tạo connection mới)

```
t=0ms     5 coroutine gọi pool.acquire()
          → Pool: "Hết connection sẵn, thuê thêm đường dây mới"
          → Gọi công ty viễn thông (TCP connect) → Kho cháy → không kết nối được
          
          ⏰ ĐỒNG HỒ 3 (wait_timeout=1s): "Chờ tối đa 1s cho connection"

t=1000ms  ⏰ ĐỒNG HỒ 3 hết! (wait_timeout=1s)
          → Pool ném exception: "Không lấy được connection trong 1 giây"
          → Phụ bếp bắt exception:
            "Không gọi được kho → Quyết định: Không gian lận (decision=False)"
          → Build response → Trả cho bếp trưởng
          
          ⏰ ĐỒNG HỒ 1 mới đếm được 1 giây → CÒN THỪA 4 GIÂY → Không cắt
          
          → 5 request này: HTTP 200, isFraud=False ✅
          
          Thời gian: ~1 giây
```

#### Nhóm C: 490 đơn còn lại (Pool hết connection)

```
t=0ms     490 coroutine gọi pool.acquire()
          → Pool: "10 đường dây đều đang bận!"
          → 490 coroutine xếp hàng chờ đường dây rảnh
          
          ⏰ ĐỒNG HỒ 3 (wait_timeout=1s): "Chờ tối đa 1s"

t=1000ms  ⏰ ĐỒNG HỒ 3 hết đồng loạt cho 490 coroutine!
          → 490 exceptions đổ vào event loop
          → Event loop (bếp trưởng) phải xử lý LẦN LƯỢT:
          
            Coroutine 11:  exception → decision_engine → build response (~2ms)
            Coroutine 12:  exception → decision_engine → build response (~2ms)
            ...
            Coroutine 200: exception → decision_engine → build response (~2ms)
            ...
            Coroutine 500: exception → decision_engine → build response (~2ms)

          490 × ~2ms = ~980ms để xử lý hết

t=~2000ms Coroutine cuối cùng (500) hoàn thành
          
          ⏰ ĐỒNG HỒ 1 mới đếm được ~2 giây → CÒN THỪA 3 GIÂY
          
          → 490 request này: HTTP 200, isFraud=False ✅
          
          Thời gian: ~1-2 giây
```

### 🔴 Khi nào Nhóm C bị 504 (trường hợp xấu nhất)?

Chỉ xảy ra khi **event loop bị quá tải nghiêm trọng**, ví dụ:

```
Tình huống cực đoan: 5000 request cùng lúc (thay vì 500)

t=0ms     5000 coroutine đổ vào
t=1000ms  4990 exceptions đổ vào event loop cùng lúc
          
          Event loop xử lý lần lượt:
          4990 × 2ms = ~9980ms ≈ 10 giây

t=5000ms  ⏰ ĐỒNG HỒ 1 hết cho những coroutine chưa được xử lý!
          → Coroutine thứ ~2500 trở đi: chưa kịp xong → HTTP 504 ❌
```

Hoặc khi **mỗi coroutine mất thời gian xử lý lâu hơn bình thường**:

```
Tình huống: Mỗi coroutine cần gọi vector check (query DB lần 2)

t=1000ms  490 coroutine bắt exception từ FACEPAY_FRAUD
          → Mỗi coroutine TIẾP TỤC gọi check_vector_fraud()
          → Gọi pool.acquire() LẦN 2 cho FACEPAY_LOGIN
          → Lại chờ wait_timeout=1s
          
t=2000ms  490 exceptions LẦN 2 đổ vào
          → Event loop xử lý:
            - Decode vector
            - Tính cosine similarity
            - Build response
          → Mỗi coroutine: ~5-10ms

          490 × 10ms = ~4900ms

t=~5000ms ⏰ ĐỒNG HỒ 1 hết cho những coroutine ở cuối hàng
          → HTTP 504 ❌
```

### Bảng tóm tắt cuối cùng:

| Nhóm | Số lượng | Điểm nghẽn | Thời gian | Kết quả |
|---|---|---|---|---|
| A (conn sẵn, query DB) | 5 | `call_timeout` ≈ `VALIDATION_TIMEOUT` | ~5s | **504** |
| B (tạo conn mới thất bại) | 5 | `wait_timeout` = 1s | ~1s | **200** ✅ |
| C (pool hết conn) | 490 | `wait_timeout` = 1s + event loop xử lý | ~1-2s | **200** ✅ |
| C (nếu có vector check) | 490 | 2× `wait_timeout` + CPU tính toán | ~3-5s | **200** hoặc **504** |
| Cực đoan (5000+ request) | ~2500+ | Event loop quá tải | >5s | **504** ❌ |
