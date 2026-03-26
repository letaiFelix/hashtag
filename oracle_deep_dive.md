# 🔌 ORACLE TRONG API — PHÂN TÍCH SÂU [oracle_pool.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py) & [database.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/database.py)

> Tài liệu này mổ xẻ chi tiết 2 file cốt lõi quản lý kết nối Oracle Database trong API,
> dùng ví dụ đời thường để giải thích từng khái niệm, từng hàm, từng con số.

---

## MỤC LỤC

1. [Bức tranh tổng thể](#1-bức-tranh-tổng-thể)
2. [File `database.py` — Tủ khóa chìa khóa chung](#2-file-databasepy)
3. [File `oracle_pool.py` — Toàn bộ hệ thống điện thoại](#3-file-oracle_poolpy)
   - [3.1 Hàm `build_dsn` — Địa chỉ kho](#31-hàm-build_dsn)
   - [3.2 Class `OracleConnectionPool.__init__` — Bản vẽ thiết kế](#32-class-__init__)
   - [3.3 Hàm `initialize` — Thi công xây dựng](#33-hàm-initialize)
   - [3.4 Keep-Alive Worker — Nhân viên kiểm tra đường dây](#34-keep-alive-worker)
   - [3.5 Hàm `fetch_one` — Gọi điện hỏi kho](#35-hàm-fetch_one)
   - [3.6 Hàm `fetch_all` — Gọi điện lấy danh sách](#36-hàm-fetch_all)
   - [3.7 Hàm `get_pool_stats` — Bảng theo dõi trực quan](#37-hàm-get_pool_stats)
   - [3.8 Hàm `close` — Đóng cửa nhà hàng](#38-hàm-close)
   - [3.9 Hàm `get_oracle_pool` — Xây dựng lần đầu tiên](#39-hàm-get_oracle_pool)
4. [Vòng đời đầy đủ của 1 Request](#4-vòng-đời-đầy-đủ-của-1-request)
5. [Bảng tra cứu nhanh: Lỗi → Nguyên nhân → Tham số liên quan](#5-bảng-tra-cứu-lỗi)

---

## 1. BỨC TRANH TỔNG THỂ

Hãy tưởng tượng hệ thống là **một nhà hàng lớn** cần liên lạc với **Kho Nguyên Liệu** (Oracle Database) qua điện thoại.

```
Nhà hàng (API Server)
  │
  ├── Phòng Quản Lý Chìa Khóa ← database.py (64 dòng)
  │       "Chìa khóa phòng điện thoại chỉ có 1 chiếc, ai muốn gọi thì mượn ở đây"
  │
  └── Phòng Điện Thoại Công Cộng ← oracle_pool.py (607 dòng)
          "Có 30 máy điện thoại (connections) gọi về Kho"
          "Có quy tắc: mỗi máy dùng tối đa 5 phút, ai chiếm máy quá 300ms phải trả lại"
          "Có nhân viên trực 24/7 kiểm tra máy còn sống không (keep-alive)"
```

---

## 2. FILE [database.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/database.py) — TỦ KHÓA CHÌA KHÓA CHUNG

**📁 Đường dẫn:** [app/core/database.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/database.py) | **64 dòng**

File này cực kỳ đơn giản nhưng cực kỳ **quan trọng về mặt kiến trúc**.
Nó chỉ làm 1 việc: **Đảm bảo toàn bộ ứng dụng chỉ có DUY NHẤT 1 Pool**
(Không ai được tự ý tạo pool riêng).

### Code thực tế và giải thích:

```python
# Dòng 16: Biến toàn cục
_db_pool: Optional[OracleConnectionPool] = None
```
**Ví dụ:** Đây là cái ngăn kéo duy nhất trong toàn nhà hàng để cất chìa khóa phòng điện thoại.
Lúc mới mở cửa: ngăn kéo TRỐNG (None). Chưa có chìa khóa nào.

---

```python
# Dòng 19-29: Hàm set_db_pool()
def set_db_pool(pool: OracleConnectionPool) -> None:
    global _db_pool
    _db_pool = pool
    logger.info("Database pool registered globally")
```
**Được gọi từ:** [app/main.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/main.py) dòng 89, chỉ gọi **1 lần duy nhất** khi khởi động.

**Ví dụ:** Người quản lý (main.py) mang chìa khóa phòng điện thoại đến bỏ vào ngăn kéo.
Từ giờ ai cũng lấy chìa khóa TỪ ĐÂY, không ai được khóa riêng.

---

```python
# Dòng 32-54: Hàm get_db_pool()
def get_db_pool() -> OracleConnectionPool:
    if _db_pool is None:
        raise RuntimeError("Database pool not initialized...")
    return _db_pool
```
**Được gọi từ:** Mọi file cần gọi DB:
- [app/api/routes/transaction_validator.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/api/routes/transaction_validator.py) dòng 98
- [app/services/validation_service.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/services/validation_service.py) dòng 128
- [app/core/ekyc_decision_engine.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/ekyc_decision_engine.py) (qua tham số [oracle_pool](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#527-590))

**Ví dụ:** Mỗi phụ bếp muốn gọi điện kho thì đến ngăn kéo lấy chìa khóa.
Nếu ngăn kéo trống (chưa khởi tạo) → Báo lỗi ngay: "Không có chìa khóa!".

---

```python
# Dòng 57-64: Hàm clear_db_pool()
def clear_db_pool() -> None:
    global _db_pool
    _db_pool = None
    logger.info("Database pool reference cleared")
```
**Được gọi từ:** [app/main.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/main.py) khi shutdown (app tắt).

**Ví dụ:** Nhà hàng đóng cửa → Quản lý lấy chìa khóa ra khỏi ngăn kéo, bỏ vào két sắt.

---

### Tại sao cần file riêng này?

Nếu không có [database.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/database.py), mỗi file muốn dùng pool phải tự import từ [oracle_pool.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py).
Khi cần đổi cách quản lý pool, phải sửa tất cả mọi file. Với [database.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/database.py):
- Chỉ cần sửa 1 chỗ duy nhất.
- Đảm bảo không ai vô tình tạo pool thứ 2.

---

## 3. FILE [oracle_pool.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py) — TOÀN BỘ HỆ THỐNG ĐIỆN THOẠI

**📁 Đường dẫn:** [app/core/oracle_pool.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py) | **607 dòng**

File này là TIM của toàn bộ kết nối Oracle. Nó chứa:
- Bản thiết kế hệ thống điện thoại (`class OracleConnectionPool`)
- Thi công xây dựng hệ thống ([initialize](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#142-202))
- Nhân viên trực 24/7 ([_keepalive_worker](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#224-274))
- Quy trình gọi điện ([fetch_one](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#319-376), [fetch_all](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#378-432))
- Cơ chế Singleton toàn ứng dụng ([get_oracle_pool](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#527-590))

---

### 3.1 Hàm [build_dsn](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#26-39) (Dòng 26-38)

```python
def build_dsn(host: str, port: str, service_name: str) -> str:
    return f"{host}:{port}/{service_name}"
```

**Ví dụ:** Đây là công thức viết địa chỉ Kho Nguyên Liệu:
```
10.122.121.126 : 1521 / ekyc
    (Địa chỉ IP)  (Cổng vào)  (Tên kho)
```
Giống như địa chỉ nhà: "Số 10, Đường 1521, Khu Công Nghiệp ekyc".

---

### 3.2 Class [__init__](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#65-141) — Bản vẽ thiết kế (Dòng 65-140)

```python
def __init__(self, *, host, port, user, password, db,
    min_size=20,           # Số máy điện thoại tối thiểu
    max_size=100,          # Số máy điện thoại tối đa
    increment=5,           # Thêm bao nhiêu máy mỗi lần mở rộng
    wait_timeout_ms=3000,  # Chờ máy rảnh tối đa bao lâu (ms)
    max_lifetime_session=3600,  # Mỗi máy dùng được tối đa bao lâu (s)
    timeout=5,             # Timeout kết nối pool
    stmtcachesize=100,     # Cache 100 câu SQL hay dùng
    ping_interval=60,      # Kiểm tra máy còn sống mỗi 60s
    enable_keepalive=True, # Có bật nhân viên trực không?
    keepalive_interval=30, # Nhân viên trực kiểm tra mỗi 30s
):
```

> ⚠️ **Lưu ý quan trọng:** [__init__](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#65-141) chỉ LƯU thông tin cấu hình vào biến, 
> KHÔNG tạo connection nào cả. Giống như bạn vẽ bản thiết kế nhà: 
> vẽ xong chưa có cục gạch nào được xây.

**Thực tế trong project** (ghi đè trong [main.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/main.py)):
```python
# main.py dòng 83-85 ghi đè giá trị mặc định:
min_size=MIN_SIZE      # 10 (từ .env) — thay vì default 20
max_size=MAX_SIZE      # 30 (từ .env) — thay vì default 100
increment=1            # thay vì default 5
wait_timeout_ms=1000   # 1 giây — thay vì default 3000ms
max_lifetime_session=300  # 5 phút — thay vì default 3600s (1 giờ)
```

---

### 3.3 Hàm [initialize](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#142-202) — Thi công xây dựng (Dòng 142-201)

Đây là hàm **THỰC SỰ TẠO CONNECTION** đến Oracle. Chỉ được gọi 1 lần.

```python
async def initialize(self):
    if self._initialized:
        logger.warning("Pool already initialized")  # Không được tạo 2 lần!
        return

    # DÒNG QUAN TRỌNG NHẤT (164): Tạo pool thật sự
    self.pool = oracledb.create_pool_async(
        user=self.user,
        password=self.password,
        dsn=self.dsn,

        min=self.min_size,        # Tạo sẵn 10 connection ngay lúc này
        max=self.max_size,        # Tối đa 30 connection
        increment=self.increment, # Mỗi lần mở rộng, tạo thêm 1

        # TIMEDWAIT = Chờ có thời hạn (không chờ mãi mãi + không fail ngay)
        getmode=oracledb.POOL_GETMODE_TIMEDWAIT,
        wait_timeout=self.wait_timeout_ms,  # Chờ tối đa 1000ms

        max_lifetime_session=self.max_lifetime_session,  # Mỗi conn sống 5 phút
        stmtcachesize=self.stmtcachesize,  # Cache 100 câu SQL

        ping_interval=self.ping_interval,  # Kiểm tra health mỗi 60s
        expire_time=2,                     # Connection idle 2s thì đánh dấu hết hạn

        params=oracledb.PoolParams(
            tcp_connect_timeout=5.0,  # Kết nối TCP tối đa 5s
            retry_count=2,            # Thử lại 2 lần nếu thất bại
            retry_delay=1             # Nghỉ 1s giữa các lần thử
        )
    )

    # Sau khi tạo pool, bật nhân viên trực keep-alive
    if self.enable_keepalive:
        await self._start_keepalive()
```

**Ví dụ:** Buổi sáng nhà hàng mở cửa:
1. Gọi công ty viễn thông, kéo 10 đường dây điện thoại vào nhà hàng (min=10).
2. Ký hợp đồng: "Tối đa được dùng 30 đường dây" (max=30).
3. Mỗi đường dây chỉ dùng 5 phút rồi phải trả lại và kéo đường mới (max_lifetime_session=300s).
4. Bật nhân viên trực (keep-alive) kiểm tra đường dây mỗi 30s.

---

### 3.4 Keep-Alive Worker — Nhân viên kiểm tra đường dây (Dòng 224-305)

Đây là **nhân viên chạy ngầm liên tục** trong suốt vòng đời ứng dụng.

#### [_start_keepalive](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#203-223) (Dòng 203-222)
```python
async def _start_keepalive(self):
    self._end_keepalive.clear()  # Reset cờ dừng
    # Tạo background task (chạy song song, không chặn request)
    self._keepalive_task = asyncio.create_task(self._keepalive_worker())
```

**Ví dụ:** Quản lý bấm nút "Bật" cho nhân viên tuần tra. Nhân viên này chạy ngầm, không ảnh hưởng gì đến việc phục vụ khách.

#### [_keepalive_worker](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#224-274) (Dòng 224-273) — Vòng lặp chính

```python
async def _keepalive_worker(self):
    while not self._end_keepalive.is_set():     # Còn đang chạy?
        try:
            # Chờ 30 giây (keepalive_interval)
            await asyncio.wait_for(
                self._end_keepalive.wait(),
                timeout=self.keepalive_interval  # 30 giây
            )
            break  # Nếu nhận tín hiệu dừng → thoát
        except asyncio.TimeoutError:
            pass   # 30 giây trôi qua không có tín hiệu → tiếp tục kiểm tra

        # Lấy 1 đường dây từ pool, gọi thử sang Oracle
        async with pool.acquire() as conn:
            with conn.cursor() as cur:
                await cur.execute("SELECT 1 FROM DUAL")  # Câu hỏi đơn giản
                await cur.fetchone()  # Oracle trả lời "1" → đường dây sống!
```

**Ví dụ timeline:**

```
8h00:00  App khởi động → Nhân viên trực bắt đầu ca
8h00:30  Nhân viên nhấc máy số 1: "Alô kho? Trả lời số 1!" → Kho: "1!" ✅
8h01:00  Nhân viên nhấc máy số 1: "Alô kho?" → Kho: "1!" ✅
8h01:30  Nhân viên nhấc máy: "Alô kho?" → ....Im lặng.... ❌
         → Log: "Keep-alive check failed: ..." (Cảnh báo nhưng tiếp tục)
8h02:00  Nhân viên thử lại: "Alô kho?" → Kho: "1!" ✅ (Kho hồi phục rồi)
...
(Cứ 30 giây 1 lần, mãi mãi cho đến khi app tắt)
```

**Tác dụng quan trọng:** Ngăn chặn lỗi `DPY-4011` (connection chết ngầm).
Nếu không có keep-alive, Oracle DB hoặc Firewall có thể cắt connection idle
mà pool không hay biết. Lần sau dùng → chết → lỗi.

#### [_stop_keepalive](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#275-306) (Dòng 275-305)
```python
async def _stop_keepalive(self):
    self._end_keepalive.set()  # Bật cờ "Dừng lại!"
    # Chờ nhân viên kết thúc lịch sự (tối đa 5 giây)
    await asyncio.wait_for(self._keepalive_task, timeout=5.0)
    # Nếu nhân viên không kết thúc trong 5s → Force cancel
```

**Ví dụ:** Nhà hàng sắp đóng cửa. Bảo nhân viên trực: "Xong việc thì về nhà đi." Nếu 5 giây sau vẫn chưa về → "Thôi về ngay đi, khóa cửa rồi!"

---

### 3.5 Hàm [fetch_one](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#319-376) — Gọi điện hỏi kho 1 dòng (Dòng 319-376)

Hàm này được gọi nhiều nhất trong toàn bộ API. Mỗi request gọi DB đều qua đây.

```python
async def fetch_one(self, sql, request_id='', call_timeout=0, params=None):
    self._check_initialized()  # Kiểm tra pool đã tạo chưa?

    # BƯỚC 1: Lấy connection từ pool (chờ tối đa wait_timeout_ms = 1000ms)
    async with self.pool.acquire() as conn:

        # BƯỚC 2: Set thời gian tối đa cho cuộc gọi này
        if call_timeout != 0:
            conn.call_timeout = call_timeout  # VD: 300ms hoặc 1000ms

        # BƯỚC 3: Tạo "bút" để viết và gửi câu hỏi
        with conn.cursor() as cur:
            try:
                # BƯỚC 4: Gửi câu SQL sang Oracle
                await cur.execute(sql, params or {})

                # BƯỚC 5: Đọc 1 dòng kết quả
                row = await cur.fetchone()
                return row   # Trả kết quả về cho người gọi

            except oracledb.Error as e:
                error_obj, = e.args
                logger.error(f"[{request_id}] Database error: {error_obj.message}")

                if error_obj.code == 4011:   # DPY-4011: Connection chết
                    logger.error(f"[{request_id}] Connection closed, closing connection")
                    await conn.close()  # Đóng connection bị chết để pool biết

                raise  # Ném lỗi lên tầng trên để xử lý

    # Khi ra khỏi "async with" → Connection TỰ ĐỘNG trả về pool
```

**Ví dụ đời thường (chi tiết từng bước):**

```
Phụ bếp A cần hỏi kho về hồ sơ khách hàng:

BƯỚC 1:  Phụ bếp đến Phòng Điện Thoại (pool.acquire())
         Phòng điện thoại: "Máy số 7 đang rảnh, xài đi!"
         ⏰ Nếu tất cả máy bận, chờ tối đa 1000ms → DPY-4005 nếu hết

BƯỚC 2:  Phụ bếp nhấc máy số 7 lên
         Dán label lên máy: "Máy này chỉ được nói chuyện tối đa 300ms"
         (conn.call_timeout = 300)

BƯỚC 3:  Phụ bếp lấy tờ giấy ghi câu hỏi (cursor)

BƯỚC 4:  Đọc câu hỏi cho kho nghe:
         "SELECT value FROM FACEPAY_FRAUD WHERE id = 'abc123'"
         (cur.execute)

BƯỚC 5:  Nghe kho trả lời, ghi vào tập (fetchone)
         Kho: "Đây là dữ liệu của abc123: [blob data...]"

    Nếu kho trả lời sau 300ms:
       → Đồng hồ kêu "tit tit" → DPY-4024 → Cúp máy cưỡng bức

    Nếu kho mất đường dây giữa chừng:
       → DPY-4011 → Phụ bếp bấm nút "Báo hỏng" (conn.close()) → Trả máy về

Ra khỏi phòng: Máy số 7 TỰ ĐỘNG trả lại pool (async with tự xử lý)
```

---

### 3.6 Hàm [fetch_all](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#378-432) — Gọi điện lấy danh sách (Dòng 378-431)

Giống hệt [fetch_one](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#319-376) nhưng ở BƯỚC 5 thay bằng `cur.fetchall()`.

```python
rows = await cur.fetchall()  # Lấy TẤT CẢ dòng, không chỉ 1
```

**Trong project hiện tại:** [fetch_all](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#378-432) ít được dùng hơn.
[fetch_one](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#319-376) chiếm ưu thế vì các query đều chỉ cần 1 dòng kết quả
(tìm 1 hồ sơ theo ID, có `ROWNUM = 1`).

---

### 3.7 Hàm [get_pool_stats](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#434-485) — Bảng theo dõi trực quan (Dòng 434-484)

```python
def get_pool_stats(self) -> Dict[str, Any]:
    return {
        "status": "active",
        "opened_connections": self.pool.opened,  # Đang có bao nhiêu máy mở
        "max_connections": self.pool.max,         # Tối đa bao nhiêu
        "min_connections": self.pool.min,         # Tối thiểu bao nhiêu
        "busy_connections": self.pool.busy,       # Đang có bao nhiêu máy có người dùng
        "available_connections": self.pool.opened - self.pool.busy,  # Bao nhiêu rảnh
        "dsn": self.dsn,
        "keepalive_enabled": self.enable_keepalive,
        "keepalive_running": self._keepalive_task is not None and not self._keepalive_task.done(),
    }
```

**Được gọi từ:** [transaction_validator.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/api/routes/transaction_validator.py) dòng 99 và 125 để log trạng thái pool trước/sau mỗi request.

**Ví dụ log bạn có thể thấy:**
```
[REQUEST_ID][BEFORE] Pool: opened=10, busy=3, available=7
[REQUEST_ID][AFTER]  Pool: opened=10, busy=2, available=8
```

**Bảng điện thoại thực tế:**

| Thông số | Ý nghĩa | Ví dụ |
|---|---|---|
| `opened=10` | 10 máy đang hoạt động (trong 30 tối đa) | Đang kéo đường dây từ min |
| `busy=8` | 8 máy đang có người gọi | 8 request đang chờ DB |
| `available=2` | 2 máy rảnh, sẵn sàng | 2 request tiếp theo vào ngay |

**Dấu hiệu nguy hiểm:** Nếu `busy = opened = max` → Pool cạn kiệt → DPY-4005 sắp xảy ra.

---

### 3.8 Hàm [close](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#486-520) — Đóng cửa nhà hàng (Dòng 486-519)

```python
async def close(self):
    # Bước 1: Dừng nhân viên keep-alive trước
    if self.enable_keepalive:
        await self._stop_keepalive()

    if self.pool:
        # Bước 2: In thống kê cuối cùng trước khi đóng
        stats = self.get_pool_stats()

        # Bước 3: Đóng tất cả connection
        await self.pool.close()

        # Bước 4: Xóa trạng thái
        self.pool = None
        self._initialized = False

        logger.info(f"Oracle pool closed. Final stats: {stats}")
```

**Được gọi từ:** [main.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/main.py) trong lifespan shutdown (khi uvicorn tắt):
```python
clear_db_pool()       # database.py: Xóa biến toàn cục
await close_oracle_pool()  # oracle_pool.py: Đóng pool thật sự
```

---

### 3.9 Hàm [get_oracle_pool](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#527-590) — Cơ chế Singleton (Dòng 527-589)

Đây là **cơ chế bảo vệ quan trọng** đảm bảo dù có bao nhiêu nơi gọi [get_oracle_pool()](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/oracle_pool.py#527-590),
vẫn chỉ có DUY NHẤT 1 pool được tạo ra.

```python
# Dòng 522-524: 2 biến singleton toàn cục
_pool_instance: Optional[OracleConnectionPool] = None
_pool_lock: Optional[asyncio.Lock] = None

async def get_oracle_pool(host, port, user, password, db, **kwargs):
    global _pool_instance, _pool_lock

    # Tạo khóa bảo vệ nếu chưa có
    if _pool_lock is None:
        _pool_lock = asyncio.Lock()

    # Kiểm tra lần 1 (không có khóa - nhanh)
    if _pool_instance is None:
        async with _pool_lock:            # Khóa lại để tránh 2 coroutine cùng tạo
            if _pool_instance is None:    # Kiểm tra lần 2 (trong khóa - an toàn)
                pool = OracleConnectionPool(host=host, ...)
                await pool.initialize()   # Tạo connection thật sự
                _pool_instance = pool     # Lưu singleton

    return _pool_instance  # Luôn trả về CÙNG 1 pool
```

**Ví dụ:** Tình huống 2 request vào cùng lúc khi app vừa khởi động:

```
Request A và B cùng gọi get_oracle_pool() tại t=0ms:

Request A: Kiểm tra _pool_instance → None → Vào khóa → Tạo pool → Lưu vào _pool_instance
Request B: Kiểm tra _pool_instance → None → Chờ khóa...
           ...A xong, mở khóa...
Request B: Kiểm tra lại _pool_instance → ĐÃ CÓ (A tạo rồi) → Trả luôn pool đó
           (Không tạo pool thứ 2!)

Kết quả: Chỉ 1 pool duy nhất, cả A và B cùng dùng chung.
```

---

## 4. VÒNG ĐỜI ĐẦY ĐỦ CỦA 1 REQUEST

Từ lúc request vào đến lúc trả về kết quả, Oracle được dùng như sau:

```
HTTP Request vào
    │
    ▼ transaction_validator.py dòng 98
oracle_pool = get_db_pool()     ← Lấy chìa khóa từ database.py
log_pool_status(...)            ← In stats: opened=10, busy=X, available=Y
    │
    ▼ validation_service.py dòng 226
result = await oracle_pool.fetch_one(
    "SELECT value FROM FACEPAY_FRAUD WHERE id = :id",
    call_timeout = CALL_TIMEOUT * 1000,   # 300ms (hoặc 1000ms)
    params = {'id': uid}
)
    │
    ├── oracle_pool.py: pool.acquire()    ← Lấy connection (chờ ≤ 1000ms)
    ├── oracle_pool.py: conn.call_timeout ← Set timer 300ms
    ├── oracle_pool.py: cur.execute()    ← Gửi SQL đến Oracle DB
    ├── oracle_pool.py: cur.fetchone()   ← Nhận kết quả
    └── oracle_pool.py: release conn     ← Trả connection về pool
    │
    ▼ validation_service.py dòng 291
is_fraud, sim, reason = await ekyc_decision_engine.check_vector_fraud(
    oracle_pool=oracle_pool,
    call_timeout=CALL_TIMEOUT,    # 300ms
    ...
)
    │
    ▼ ekyc_decision_engine.py dòng 40
result = await oracle_pool.fetch_one(
    "SELECT VECTOR, VECTOR_LASTEST FROM FACEPAY_LOGIN WHERE ID = :p_uid ...",
    call_timeout = call_timeout * 1000,
    params = {'p_uid': uid, 'p_bank_code': bank_code}
)
    │   ← Lần gọi DB thứ 2 cho cùng 1 request!
    ├── pool.acquire() ← Cần 1 connection nữa
    ├── cur.execute()  ← Gửi SQL
    └── fetchone()     ← Nhận kết quả (2 BLOB: VECTOR + VECTOR_LASTEST)
    │
    ▼ transaction_validator.py dòng 125
log_pool_status(...)   ← In stats sau khi xử lý xong
    │
    ▼
HTTP Response 200 OK
```

---

## 5. BẢNG TRA CỨU LỖI

| Lỗi | Nguyên nhân | Tham số liên quan | Giải pháp |
|---|---|---|---|
| **DPY-4005** | Pool hết connection, chờ `wait_timeout_ms` không có ai trả | `max_size`, `wait_timeout_ms` | Tăng `max_size` |
| **DPY-4011** | Connection chết ngầm (network cắt, DB idle timeout) | `max_lifetime_session`, `keepalive_interval` | Giảm `max_lifetime_session`, keep-alive hoạt động tốt |
| **DPY-4024** | Query mất lâu hơn `call_timeout` | `CALL_TIMEOUT` | Tăng `call_timeout` hoặc tối ưu query/index |
| **RuntimeError: Pool not initialized** | Gọi [get_db_pool()](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/database.py#32-55) trước khi app khởi động xong | [set_db_pool()](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/core/database.py#19-30) chưa được gọi | Kiểm tra thứ tự khởi động trong [main.py](file:///c:/Users/tainl/Documents/t1_v5/image-injection-detection-api/app/main.py) |
