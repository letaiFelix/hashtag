# T1 Model - Tài liệu Chi Tiết

## 📋 Mục Lục
1. [Tổng Quan](#tổng-quan)
2. [Input/Output](#inputoutput)
3. [Các Rule Chính](#các-rule-chính)
4. [Logic Flow](#logic-flow)
5. [Integration với Hệ Thống](#integration-với-hệ-thống)
6. [Cách Chạy](#cách-chạy)

---

## 🎯 Tổng Quan

### T1_Model là gì?
- **T-1 Model**: Model **huấn luyện hàng ngày** để phát hiện fraud patterns
- **Input**: Dữ liệu từ T (hôm nay) + T-1 (hôm qua) + T-2 (hôm kia) = 3 ngày dữ liệu
- **Output**: 
  - Danh sách UID gian lận (Blacklist + Suspicious)
  - Danh sách IP nguy hiểm (High-risk + Low-risk + Blacklisted)
  - Checksum patterns
- **Frequency**: Chạy hàng ngày (thường là T+1 hoặc T+0 tối)
- **Update To**: Redis (cache real-time cho validation API)

---

## 📥 Input/Output

### Input Data

| Thành phần | Nguồn | Format | Ghi chú |
|-----------|-------|--------|---------|
| **Bank List** | Command line arg | List `['vcb', 'tcb', 'bid']` | Danh sách bank cần train model |
| **Infer Date** | Env var / Command arg | `YYYY-MM-DD` | Ngày inference (thường là hôm nay) |
| **Training Period** | Code (3 days) | T-2, T-1, T (3 ngày) | Query 3 ngày dữ liệu từ Trino |
| **BANK_CODE_MAP** | Database (CONFIG_FRAUD) | `VCB:vcb,TCB:tcb` | Map từ bank code sang bank name |
| **Config Rules** | Database (CONFIG_FRAUD) | `ACTIVE_HOUR=16`, `BLACKLIST_DURATION=30` | Các threshold để xác định fraud |

### Output Data

| Key | Lưu Ở | Format | Nội dung |
|-----|----------|--------|---------|
| **BLACKLISTED_UID_KEY** | Redis | Hash `{bank_code: [uid1, uid2...]}` | UID có 3+ ảnh cùng size hoặc active ≥16h |
| **SUS_UID_KEY** | Redis | Hash `{bank_code: [uid1, uid2...]}` | UID có 5+ liveness/v2/v6 errors |
| **HIGH_RISK_IP_KEY** | Redis | Set `{ip1, ip2, ...}` | IP có 10+ transactions, 2+ sender |
| **LOW_RISK_IP_KEY** | Redis | Set `{ip1, ip2, ...}` | Permanent (30 ngày) |
| **BLACKLISTED_IP_KEY** | Redis | Set `{ip1, ip2, ...}` | IP từ nước ngoài |

TTL: Tất cả **30 ngày** (BLACKLIST_DURATION)

---

## 🔐 Các Rule Chính

### 1. UID Blacklist Rules (Gian Lận Xác Định)

**Rule 1.1: High Frequency UID**
```
Nếu: UID có ≥10 transactions/ngày (trung bình 3 ngày)
      AND không có liveness error (error-liveness-12 = NA)
Thì: Đánh dấu High Frequency
```
**Code:** Line 424-435
```python
high_frequency_uid = training_data[training_data['error_1'].isna()]
high_frequency_uid = high_frequency_uid.groupby(['uid', 'bank_code'])['request_id'].nunique()
high_frequency_uid['count'] = high_frequency_uid['count'] / num_training_date  # ÷3
high_frequency_uid = high_frequency_uid[high_frequency_uid['count'] >= 10]
```

**Rule 1.2: Active High Hour**
```
Nếu: UID active ≥16 giờ/ngày (default ACTIVE_HOUR=16)
     AND không phải High Frequency
Thì: Đánh dấu Active
```
**Code:** Line 445-449
```python
active_uid = training_data.groupby(['uid', 'bank_code', 'date'])['hour'].nunique()
active_uid = active_uid[active_uid['hour_count'] >= ACTIVE_HOUR]  # 16 giờ
```

**Rule 1.3: Same Size Images**
```
Nếu: UID có 2+ images với cùng size
Thì: Đánh dấu Same Size (có khả năng reuse image)
```
**Code:** Line 451-465
```python
aggregated_uid = training_data.groupby(['uid', 'bank_code', 'size'])['request_id'].nunique()
same_size_image_pairs = aggregated_uid[aggregated_uid['num_txn'] >= 2]  # 2+ txn same size
```

**Rule 1.4: UID Blacklist Final**
```
Blacklist UID = (High Frequency ∩ (Same Size ∪ Active))
```
**Code:** Line 476-484
```python
uid_blacklist = high_frequency_uid.merge(uid_blacklist, how='inner')
uid_blacklist = pd.concat([uid_blacklist, same_size_image_pairs])
uid_blacklist = pd.concat([uid_blacklist, active_uid])
```

---

### 2. Suspicious UID Rules (Nghi Ngờ)

```
Nếu: UID có:
     - ≥5 liveness errors (error-liveness-12), HOẶC
     - ≥5 v2 signal errors, HOẶC  
     - ≥5 v6 signal errors
Thì: Đánh dấu Suspicious
```
**Code:** Line 558-570
```python
sus_uid = aggregated_data[
    (
        (aggregated_data['num_liveness'] >= 5) |
        (aggregated_data['num_v2'] >= 5) |
        (aggregated_data['num_v6'] >= 5)
    )
]
```

**Device Type Analysis:**
- **iOS**: Chỉ check v1, v2, v3, v4 signal
- **Android**: Check v1-v9, d1-d9, bd1/bd2/bd4/bd5 signals

---

### 3. IP Risk Rules

**Rule 3.1: Blacklisted IP** (Nước ngoài)
```
Nếu: IP có usage_type = "DCH" OR country ≠ "VIETNAM"
Thì: Blacklist IP (permanent block)
```
**Code:** Line 580-583

**Rule 3.2: High Risk IP**
```
Nếu: IP có usage_type = "DCH" OR country ≠ "VIETNAM"
Thì: High Risk IP (cảnh báo)
```
**Code:** Line 585-588

**Rule 3.3: Low Risk IP** (Reuse Pattern)
```
Nếu: UID+IP anomaly có:
     - ≥10 transactions, AND
     - ≥2 different senders (UID)
Thì: Low Risk IP (cảnh báo mức độ thấp)
```
**Code:** Line 602-605
```python
low_risk_ip = aggregated_ip[
    (aggregated_ip['num_txn'] >= 10) &
    (aggregated_ip['num_sender'] >= 2)
]['ip'].unique().tolist()
```

---

## 🔄 Logic Flow

```
┌─────────────────────────────────────────┐
│  1. LOAD CONFIG từ DATABASE             │
│     - BANK_CODE_MAP (VCB→vcb, TCB→tcb) │
│     - ACTIVE_HOUR (16)                  │
│     - BLACKLIST_DURATION (30 ngày)      │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  2. QUERY DATA từ TRINO (3 ngày)        │
│     - For each bank in bank_list        │
│     - Format: facepay_{bank}            │
│     - Lấy: uid, device, ip, checksum    │
│     - Filter: size != NA, ip != 10.28   │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  3. DATA PREPROCESSING                  │
│     - Merge BANK_CODE_MAP               │
│     - Split iOS vs Android              │
│     - Parse SecKiller signals (v1-v9)   │
│     - Fill NA values                    │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  4. FEATURE ENGINEERING                 │
│     - High Frequency UID (≥10 txn/day)  │
│     - Active UID (≥16h/day)             │
│     - Same Size Images (2+)             │
│     - Signal Detection (Liveness/v2/v6) │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  5. BUILD BLACKLISTS                    │
│     - uid_blacklist                     │
│     - sus_uid                           │
│     - blacklisted_ip                    │
│     - high_risk_ip, low_risk_ip         │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  6. SAVE TO REDIS                       │
│     - BLACKLISTED_UID_KEY {bank: [...]} │
│     - SUS_UID_KEY {bank: [...]}         │
│     - HIGH_RISK_IP_KEY {ip...}          │
│     - LOW_RISK_IP_KEY {ip...}           │
│     - BLACKLISTED_IP_KEY {ip...}        │
│     - TTL: 30 ngày                      │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│  7. VALIDATION API LOADS                │
│     - Cache vào memory từ Redis         │
│     - Dùng trong process_validation()   │
│     - Check UID/IP status               │
└─────────────────────────────────────────┘
```

---

## 🔗 Integration với Hệ Thống

### Validation Service Sử Dụng Output

```python
# Load từ cache
blacklisted_uid_dict = get_blacklisted_uid_from_cache()  # {bank: [uid...]}
sus_uid_dict = get_sus_uid_from_cache()                  # {bank: [uid...]}
blacklisted_ip_set = get_blacklisted_ip_from_cache()     # {ip...}
sus_ip_set = get_sus_ip_from_cache()                     # {ip...}

# Dùng trong decision engine
if uid in blacklisted_uid_dict.get(bank_code, []):
    decision = fraudulent  # Xác định gian lận

if ip in blacklisted_ip_set:
    decision = fraudulent  # IP bị cấm
```

### T1_Model + Device Model Logging

```
[Mới thêm - feature/v5-log]

if verbose_log == 1 and bank_code in log_bank_filter:
    log_request()  # Log chi tiết request từ bank theo dõi
    
→ Giúp T1_Model team debug/verify model output
```

---

## ▶️ Cách Chạy

### Command Line

```bash
# Cách 1: Cơ bản (3 bank)
python scripts/t1_model.py \
    --infer-date 2024-01-15 \
    --bank-list vcb tcb bid

# Cách 2: Single bank
python scripts/t1_model.py \
    --infer-date 2024-01-15 \
    --bank-list vcb

# Cách 3: Environment variable
export INFER_DATE=2024-01-15
python scripts/t1_model.py --bank-list vcb tcb bid

# Cách 4: Scheduled (Cron job - chạy hàng ngày)
0 1 * * * cd /path/to/api && python scripts/t1_model.py \
    --infer-date $(date -d 'yesterday' +%Y-%m-%d) \
    --bank-list vcb tcb bid vpb
```

### Expected Output Logs

```
[t1_model_logger] Bank code map: {'970405': 'vcb', '970407': 'tcb', ...}
[t1_model_logger] Active hour threshold: 16 hours.
[t1_model_logger] Blacklist duration: 30 days.

[t1_model_logger] Querying VCB for data
[t1_model_logger] Retrieving data on 2024-01-13
[t1_model_logger] Retrieved 15234 rows for VCB on 2024-01-13

[t1_model_logger] Number of UID with a pair of images with same size: 342
[t1_model_logger] Number of UID being active >= 16 hours: 156
[t1_model_logger] Number of suspicious IP: 89
[t1_model_logger] Number of blacklisted UID: 245
[t1_model_logger] Number of suspicious UIDs: 167

[t1_model_logger] Saving new features to Redis
[t1_model_logger] Successfully updated features and sets in Redis.
```

### Error Handling

```
Lỗi                          | Giải pháp
---------------------------|---------------------------
No data from Trino          | Check bank name format
Config missing              | Add BANK_CODE_MAP to DB
Redis connection failed     | Check Redis server status
Insufficient data (< 3 days)| Wait for more data
Model results not valid     | Retry next day
```

---

## 📊 Metrics & Statistics

### Output Metrics

```
├─ Total UIDs analyzed:      50,000+
├─ Blacklisted UIDs:         ~500
├─ Suspicious UIDs:          ~1,500
├─ High Risk IPs:            ~200
├─ Low Risk IPs:             ~100
├─ Blacklisted IPs:          ~50
└─ Processing time:          5-10 minutes
```

### Per-Bank Breakdown (Log)
```
label: 'blacklisted_uid'
bank_code: 'VCB'
before: 234
after: 478
added: 244
removed: 0
```

---

## 🎯 Use Cases & Examples

### Use Case 1: Detect Image Reuse
```
UID 12345@VCB:
- Day 1: Upload selfie (checksum: abc123, size: 2048KB)
- Day 2: Upload selfie (checksum: abc123, size: 2048KB)
- Day 3: Upload selfie (checksum: abc123, size: 2048KB)

Result: same_size_image_pairs → Blacklist UID
```

### Use Case 2: Detect Bot Activity
```
UID 67890@TCB:
- Transactions: 15/day (≥10) → High Frequency ✓
- Active hours: 0-23h (≥16h) → Active ✓
- SecKiller v2 errors: 12 (≥5) → Suspicious v2 ✓

Result: uid_blacklist ∩ sus_uid → Fraud Score HIGH
```

### Use Case 3: Detect Foreign IP Abuse
```
IP 203.xxx.xxx.xxx:
- Country: "Singapore"
- Transactions: 20 (≥10)
- Unique UIDs: 5 (≥2)

Result: high_risk_ip + country ≠ VN → Blacklisted IP
```

---

## 🔧 Configuration Keys (CONFIG_FRAUD)

| Key | Type | Default | Ý Nghĩa |
|-----|------|---------|--------|
| BANK_CODE_MAP | String | - | Map code→name |
| ACTIVE_HOUR | Int | 16 | Threshold giờ hoạt động |
| BLACKLIST_DURATION | Int | 30 | TTL Redis (ngày) |
| VERBOSE_LOG | Int | 0 | Ghi log chi tiết |
| LOG_BANK_FILTER | String | 970419 | Bank cần ghi log |

---

## 📝 Notes & Best Practices

1. **Always run 3-day roll**: T1_Model cần 3 ngày data để tìm pattern
2. **Monitor Redis TTL**: Blacklist hết hạn sau 30 ngày
3. **Per-bank analysis**: Mỗi bank có rule riêng (device type khác)
4. **Error recovery**: Nếu model fail, Redis giữ 72h retry buffer
5. **Logging**: Enable VERBOSE_LOG=1 khi debug model issues

---

## 🚀 Summary

**T1_Model** là backbone của fraud detection:
- Chạy **hàng ngày** để phát hiện patterns
- Gửi blacklist/whitelist vào **Redis**
- Validation API dùng để **reject fraud requests**
- Integration với **verbose logging** để debug/verify

Key Metrics:
- ✓ 3-day data analysis
- ✓ Multi-rule detection (High Freq, Active, Same Size)
- ✓ Per-bank granularity
- ✓ Real-time Redis cache (30-day TTL)
