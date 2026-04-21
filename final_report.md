# Final Report — Image Injection Detection Pipeline QA

> **Scope**: `scripts/t1_model.py` + `scripts/data_validators.py`  
> **Date**: 2026-04-21  
> **Test Suite**: 159 tests | **159 PASSED** | 0 failed  
> **Environment**: Python 3.8.20, conda `image_injection`  

---

## Phần 1 — Requirement & Business Rules

### 1.1 Requirement Summary

| REQ | Mô tả | File & Line |
|-----|--------|-------------|
| REQ-001 | Validate data Trino: `size > 0`, `uid` not null/empty, `ip` not blacklisted internal, `checksum` not null/empty | `data_validators.py:114-120` |
| REQ-002 | Phân loại iOS (iPhone) vs Android dựa trên `device_model` | `t1_model.py:464-483` |
| REQ-003 | SecKiller signal cho iOS: v1≠0, v2≠0, v3∉{0,21,33}, v4∉{0,66,75} | `t1_model.py:466-474` |
| REQ-004 | SecKiller signal cho Android: v1-v9≥900, d1-d9≥900, bd1/2/4/5≥900, v6=822, d9≥900&≠941 | `t1_model.py:487-510` |
| REQ-005 | High-frequency UID: avg ≥ 10 request/ngày qua 3 ngày (chỉ txn không lỗi) | `t1_model.py:529-544` |
| REQ-006 | Active UID: active ≥ ACTIVE_HOUR giờ distinct/ngày | `t1_model.py:550-554` |
| REQ-007 | Same-size image pairs: ≥ MIN_SAME_SIZE_PAIRS sizes khác nhau, mỗi size ≥ 2 txn | `t1_model.py:577-591` |
| REQ-008 | Blacklist UID: high_freq ∩ (same_size ≥ 3) | `t1_model.py:594-606` |
| REQ-009 | Tổng hợp blacklist = REQ-008 ∪ REQ-007 ∪ REQ-006 (loại trừ đã blacklist) | `t1_model.py:607-624` |
| REQ-010 | Suspicious UID: signal aggregated ≥ 5 cho liveness/v2/v6 | `t1_model.py:689-701` |
| REQ-011 | IP rủi ro cao: usage_type='DCH' HOẶC country≠'VIET NAM' | `t1_model.py:716-719` |
| REQ-012 | IP blacklist: usage_type='DCH' VÀ country≠'VIET NAM' | `t1_model.py:711-714` |
| REQ-013 | IP rủi ro thấp: ≥ 10 txn VÀ ≥ 2 sender unique từ giao dịch nghi ngờ | `t1_model.py:733-736` |
| REQ-014 | Lưu kết quả Redis với TTL = BLACKLIST_DURATION ngày | `t1_model.py:773-885` |
| REQ-015 | Error handling: renew Redis expire khi pipeline lỗi | `t1_model.py:888-910` |
| REQ-016 | Config từ Oracle: BANK_CODE_MAP, ACTIVE_HOUR, BLACKLIST_DURATION, MIN_SAME_SIZE_PAIRS | `t1_model.py:46-110` |

### 1.2 Business Rules (22 rules)

| BR | Rule | Verified By |
|----|------|-------------|
| BR-001 | `size > 0.0` AND `notna` | TC-001→TC-004 ✅ |
| BR-002 | `uid notna` AND `uid != ''` | TC-005→TC-006 ✅ |
| BR-003 | `ip notna` AND `ip != '10.28.100.4'` | TC-007→TC-008 ✅ |
| BR-004 | `checksum notna` AND `checksum != ''` | TC-009→TC-010 ✅ |
| BR-005 | iPhone prefix → iOS path | TC-014 ✅ |
| BR-006 | iOS v1≠'0' → signal=1 | TC-017→TC-018 ✅ |
| BR-007 | iOS v3∉{0,21,33} → signal=1 | v3_safe_21, v3_safe_33 ✅ |
| BR-008 | iOS v4∉{0,66,75} → signal=1 | v4_safe_66, v4_safe_75 ✅ |
| BR-009 | Android v≥'900' → signal=1 | TC-019→TC-020 ✅ |
| BR-010 | Android v6=='822' only | TC-021→TC-022 ✅ |
| BR-011 | Android d9≥'900' AND ≠'941' | TC-023→TC-024 ✅ |
| BR-012 | High-freq: count/3 ≥ 10 | TC-026→TC-029 ✅ |
| BR-013 | Active: hour_count ≥ ACTIVE_HOUR | TC-030→TC-032 ✅ |
| BR-014 | Same-size: num_size ≥ MIN_SAME_SIZE_PAIRS | TC-033→TC-036 ✅ |
| BR-015 | Blacklist = high_freq ∩ size≥3 | TC-037→TC-039 ✅ |
| BR-016 | Active loại trừ đã blacklist | TC-040→TC-042 ✅ |
| BR-017 | Sus: liveness≥5 OR v2≥5 OR v6≥5 | TC-043→TC-046 ✅ |
| BR-018 | IP high_risk: DCH OR foreign | TC-047→TC-049 ✅ |
| BR-019 | IP blacklist: DCH AND foreign | TC-049→TC-050 ✅ |
| BR-020 | IP low_risk: txn≥10 AND sender≥2 | TC-053→TC-055 ✅ |
| BR-021 | Redis TTL = BLACKLIST_DURATION * 24h | Code review ✅ |
| BR-022 | Pipeline fail → renew 72h | Code review ✅ |

---

## Phần 2 — Kết quả Test Chi Tiết

### 2.1 Test Suite 1: Business Logic (80 tests)

| # | Test | KC Report | Kết quả thực | Khớp? |
|---|------|-----------|-------------|-------|
| 1 | `TestTrinoLogRecordModel::test_tc011_valid_minimal_record` | size=100.5 → accept | PASSED | ✅ |
| 2 | `TestTrinoLogRecordModel::test_tc012_size_zero_raises_error` | size=0 → ValidationError | PASSED | ✅ |
| 3 | `TestTrinoLogRecordModel::test_tc013_size_negative_raises_error` | size=-1 → ValidationError | PASSED | ✅ |
| 4 | `TestFilterDataframe::test_tc001_valid_record_accepted` | Valid row → kept | PASSED | ✅ |
| 5 | `TestFilterDataframe::test_tc002_size_zero_rejected` | size=0.0 → dropped | PASSED | ✅ |
| 6 | `TestFilterDataframe::test_tc003_size_negative_rejected` | size=-5.0 → dropped | PASSED | ✅ |
| 7 | `TestFilterDataframe::test_tc004_size_nan_rejected` | size=NaN → dropped | PASSED | ✅ |
| 8 | `TestFilterDataframe::test_tc005_uid_none_rejected` | uid=None → dropped | PASSED | ✅ |
| 9 | `TestFilterDataframe::test_tc006_uid_empty_rejected` | uid='' → dropped | PASSED | ✅ |
| 10 | `TestFilterDataframe::test_tc007_ip_none_rejected` | ip=None → dropped | PASSED | ✅ |
| 11 | `TestFilterDataframe::test_tc008_ip_internal_blacklist_rejected` | ip=10.28.100.4 → dropped | PASSED | ✅ |
| 12 | `TestFilterDataframe::test_tc009_checksum_none_rejected` | checksum=None → dropped | PASSED | ✅ |
| 13 | `TestFilterDataframe::test_tc010_checksum_empty_rejected` | checksum='' → dropped | PASSED | ✅ |
| 14 | `TestIosSignals::test_tc017_all_safe` | All 0 → total=0 | PASSED | ✅ |
| 15 | `TestIosSignals::test_tc018_all_flagged` | All non-0 → total=4 | PASSED | ✅ |
| 16 | `TestAndroidSignals::test_tc019_v_at_boundary_900` | '900' → signal=1 | PASSED | ✅ |
| 17 | `TestAndroidSignals::test_tc020_v_below_boundary` | '899' → signal=0 | PASSED | ✅ |
| 18 | `TestAndroidSignals::test_tc021_v6_822_flagged` | v6='822' → 1 | PASSED | ✅ |
| 19 | `TestAndroidSignals::test_tc022_v6_900_not_flagged` | v6='900' → 0 (override) | PASSED | ✅ |
| 20 | `TestAndroidSignals::test_tc023_d9_941_excluded` | d9='941' → 0 | PASSED | ✅ |
| 21 | `TestAndroidSignals::test_tc024_d9_950_flagged` | d9='950' → 1 | PASSED | ✅ |
| 22 | `TestAndroidSignals::test_tc025_string_comparison_single_char` | '9'≥'900'→False | PASSED | ✅ |
| 23 | `TestHighFrequencyUid::test_tc026_uid_30_requests_flagged` | 30/3=10 → flagged | PASSED | ✅ |
| 24 | `TestHighFrequencyUid::test_tc027_uid_29_requests_not_flagged` | 29/3=9.67 → not | PASSED | ✅ |
| 25 | `TestHighFrequencyUid::test_tc028_errors_excluded` | Errors filtered | PASSED | ✅ |
| 26 | `TestActiveUid::test_tc030_active_exact_threshold` | 8h=ACTIVE_HOUR → flagged | PASSED | ✅ |
| 27 | `TestActiveUid::test_tc031_below_threshold` | 7h < 8 → not | PASSED | ✅ |
| 28 | `TestSameSizePairs::test_tc033_flagged` | 2 sizes × 2 txn → flagged | PASSED | ✅ |
| 29 | `TestSameSizePairs::test_tc034_only_one_size` | 1 size → not | PASSED | ✅ |
| 30 | `TestIpClassification::test_tc047_dch_vietnam_high_risk_not_blacklisted` | DCH+VN → high only | PASSED | ✅ |
| 31 | `TestIpClassification::test_tc049_dch_and_foreign_both` | DCH+foreign → both | PASSED | ✅ |
| 32 | `TestSuspiciousUid::test_tc043_liveness_flagged` | liveness≥5 → sus | PASSED | ✅ |
| 33 | `TestSuspiciousUid::test_tc046_all_below_threshold` | All<5 → not | PASSED | ✅ |

> **Tất cả 80/80 tests trong report.md đều PASSED và khớp kết quả dự kiến.**

### 2.2 Test Suite 2: Edge Cases (79 tests)

| # | Test | Dự kiến trong report_2 | Kết quả thực | Khớp? |
|---|------|----------------------|-------------|-------|
| 1 | `test_ec002_size_infinity_passes` | EC-002: Inf lọt qua ❌ | PASSED (Inf accepted) | ✅ |
| 2 | `test_ec003_uid_whitespace_only_passes` | EC-003: Whitespace lọt ❌ | PASSED (whitespace accepted) | ✅ |
| 3 | `test_ec004_ip_with_trailing_spaces_bypasses_blacklist` | EC-004: Spaces bypass ❌ | PASSED (bypass confirmed) | ✅ |
| 4 | `test_ec005_checksum_whitespace_only_passes` | EC-005: Whitespace lọt ❌ | PASSED (whitespace accepted) | ✅ |
| 5 | `test_ec006_missing_column_raises_keyerror` | EC-006: Missing col → KeyError | PASSED (KeyError raised) | ✅ |
| 6 | `test_ec008_bank_none_causes_error_in_log` | EC-008: bank=None → AttributeError | PASSED (error raised) | ✅ |
| 7 | `test_ec009_start_after_end_returns_empty` | EC-009: start>end → [] | PASSED (empty list) | ✅ |
| 8 | `test_ec010_invalid_format_raises_valueerror` | EC-010: Bad format → ValueError | PASSED (error raised) | ✅ |
| 9 | `test_ec022_multiple_dots` | EC-022: '12.34.56' → float crash | PASSED (ValueError confirmed) | ✅ |
| 10 | `test_ec025_bd3_bd6_remain_nan` | EC-025: bd3,bd6 not filled | PASSED (NaN confirmed) | ✅ |
| 11 | `test_ec030_uppercase_iphone` | EC-030: 'IPHONE' → Android | PASSED (misclassified) | ✅ |
| 12 | `test_ec038_two_digit_91_bug` | EC-038: '91'≥'900'=True → BUG | PASSED (bug confirmed) | ✅ |
| 13 | `test_ec040_four_digit_1000_bug` | EC-040: '1000'≥'900'=False → BUG | PASSED (bug confirmed) | ✅ |
| 14 | `test_ec056_datetime_class_vs_instance` | EC-056: datetime class passed | PASSED (class confirmed) | ✅ |
| 15 | `test_ec062_no_space` | EC-062: 'VIETNAM'→foreign | PASSED (false positive) | ✅ |
| 16 | `test_vietnam_unicode` | EC-064: 'VIỆT NAM'→foreign | PASSED (false positive) | ✅ |
| 17 | `test_ec066_high_risk_ip_no_ttl` | EC-066: No TTL | PASSED (gap confirmed) | ✅ |
| 18 | `test_ec012_invalid_json_in_redis` | EC-012: Bad JSON → crash | PASSED (JSONDecodeError) | ✅ |

> **Tất cả 79/79 edge case tests đều PASSED và khớp kết quả dự kiến trong report_2.md.**

---

## Phần 3 — Tổng hợp Bugs & Issues

### 🔴 CRITICAL — Có thể crash production

| ID | Vị trí | Bug | Test xác nhận |
|----|--------|-----|---------------|
| BUG-001 | `t1_model.py:393` | Trino query fail → `daily_log` chưa gán → `UnboundLocalError` | Code review (không thể unit test do cần mock Trino) |
| BUG-002 | `t1_model.py:420-421` | `size="12.34.56kb"` → `float("12.34.56")` → `ValueError` crash (thiếu `errors='coerce'`) | `test_ec022_multiple_dots` ✅ |
| BUG-003 | `t1_model.py:489-490` | String comparison: `'91' >= '900'` = True (91 < 900 as integer) | `test_ec038_two_digit_91_bug` ✅ |
| BUG-004 | `t1_model.py:489-490` | String comparison: `'1000' >= '900'` = False (1000 > 900 as integer) | `test_ec040_four_digit_1000_bug` ✅ |
| BUG-005 | `t1_model.py:894` | `'timestamp': datetime` truyền class thay vì instance | `test_ec056_datetime_class_vs_instance` ✅ |
| BUG-006 | `t1_model.py:913` | `cursor.close()` trong finally — `cursor` có thể chưa tồn tại khi `bank_list` rỗng | Code review |

### 🟡 HIGH — Kết quả sai nhưng không crash

| ID | Vị trí | Bug | Test xác nhận |
|----|--------|-----|---------------|
| BUG-007 | `data_validators.py:117` | `uid="   "` (whitespace) lọt qua validation | `test_ec003_uid_whitespace_only_passes` ✅ |
| BUG-008 | `data_validators.py:118` | `ip="  10.28.100.4  "` (spaces) bypass blacklist | `test_ec004_ip_with_trailing_spaces_bypasses_blacklist` ✅ |
| BUG-009 | `data_validators.py:119` | `checksum="   "` (whitespace) lọt qua | `test_ec005_checksum_whitespace_only_passes` ✅ |
| BUG-010 | `t1_model.py:432` | `bd3`, `bd6` không được fillna → NaN trong output | `test_ec025_bd3_bd6_remain_nan` ✅ |
| BUG-011 | `t1_model.py:465` | `device_model="IPHONE 15"` bị phân vào Android (case-sensitive) | `test_ec030_uppercase_iphone` ✅ |
| BUG-012 | `t1_model.py:527` | `"VIETNAM"` (không space) bị coi là nước ngoài | `test_ec062_no_space` ✅ |
| BUG-013 | `t1_model.py:527` | `"VIỆT NAM"` (diacritics) bị coi là nước ngoài | `test_vietnam_unicode` ✅ |
| BUG-014 | `t1_model.py:800-801` | `high_risk_ip_key` không có TTL → data tồn tại vĩnh viễn | `test_ec066_high_risk_ip_no_ttl` ✅ |
| BUG-015 | `t1_model.py:882-883` | `blacklisted_ip_key` không có TTL → tương tự BUG-014 | `test_ec067_blacklisted_ip_no_ttl` ✅ |
| BUG-016 | `data_validators.py:116` | `size=Inf` lọt qua validation (giá trị phi lý) | `test_ec002_size_infinity_passes` ✅ |

### 🟢 LOW — Rủi ro thấp hoặc edge case hiếm

| ID | Vị trí | Issue | Test xác nhận |
|----|--------|-------|---------------|
| LOW-001 | `t1_model.py:189-193` | `start_date > end_date` → trả `[]` im lặng | `test_ec009_start_after_end_returns_empty` ✅ |
| LOW-002 | `t1_model.py:138` | Invalid JSON trong Redis → `JSONDecodeError` crash | `test_ec012_invalid_json_in_redis` ✅ |
| LOW-003 | `t1_model.py:445` | `data_list < 3` check sai scope (log nhầm bank) | Code review |
| LOW-004 | `t1_model.py:393-416` | Duplicate `len(daily_log)==0` check — code thừa | Code review |
| LOW-005 | `t1_model.py:476,512` | `'signal' in col` filter — fragile nếu thêm cột | Code review |

---

## Phần 4 — Traceability Matrix (REQ → BR → Test → Kết quả)

| REQ | BR | Test File | Tests | All Pass? |
|-----|----|-----------|-------|-----------|
| REQ-001 | BR-001→004 | `test_data_validators.py` | TC-001→010, EC-002→006 | ✅ 22/22 |
| REQ-002 | BR-005 | `test_data_validators.py` + `test_edge_cases.py` | TC-014→016, EC-030→031 | ✅ 11/11 |
| REQ-003 | BR-006→008 | `test_data_validators.py` | TC-017→018 + safe values | ✅ 7/7 |
| REQ-004 | BR-009→011 | `test_data_validators.py` + `test_edge_cases.py` | TC-019→025, EC-037→041 | ✅ 19/19 |
| REQ-005 | BR-012 | `test_data_validators.py` | TC-026→029 | ✅ 4/4 |
| REQ-006 | BR-013 | `test_data_validators.py` | TC-030→032 | ✅ 3/3 |
| REQ-007 | BR-014 | `test_data_validators.py` | TC-033→036 | ✅ 4/4 |
| REQ-010 | BR-017 | `test_data_validators.py` | TC-043→046 | ✅ 6/6 |
| REQ-011 | BR-018 | `test_data_validators.py` + `test_edge_cases.py` | TC-047→050, EC-061→065 | ✅ 12/12 |
| REQ-012 | BR-019 | `test_data_validators.py` | TC-049→050 | ✅ 2/2 |
| REQ-013 | BR-020 | `test_data_validators.py` | TC-053→055 | ✅ 3/3 |
| REQ-014 | BR-021 | `test_edge_cases.py` | EC-066→068, low_risk_ttl | ✅ 4/4 |
| REQ-015 | BR-022 | `test_edge_cases.py` | EC-056 | ✅ 2/2 |
| Utility | — | Both | date_range, build_dsn, read_uid_hash, log_diff | ✅ 20/20 |

**Tổng cộng: 159/159 PASSED — 100% trùng khớp giữa report và kết quả thực.**

---

## Phần 5 — Khuyến Nghị

### 🔴 Ưu tiên cao — Sửa ngay trước khi chạy dữ liệu thật

#### 5.1 Chuyển SecKiller signal từ string comparison sang integer comparison

```diff
# t1_model.py — Line 489-491
- android_txn[f'v{idx}_signal'] = (
-     android_txn[f'v{idx}'] >= '900'
- ).astype('int')
+ android_txn[f'v{idx}_signal'] = (
+     pd.to_numeric(android_txn[f'v{idx}'], errors='coerce').fillna(0) >= 900
+ ).astype('int')
```

**Lý do**: String comparison `'91' >= '900'` = `True`, `'1000' >= '900'` = `False` — hoàn toàn ngược logic số. Nếu SecKiller codes có chiều dài không đồng nhất (1-4 chữ số), pipeline sẽ tạo **false positive** (flag UID vô tội) và **false negative** (bỏ sót UID gian lận).

#### 5.2 Sửa `UnboundLocalError` khi Trino query fail

```diff
# t1_model.py — Line 371-393
  try:
      cursor.execute(query)
      ...
      daily_log = pd.DataFrame(data, columns=columns)
  except Exception as e:
      cursor.close()
      ...
+     continue  # ← Thêm continue để skip ngày bị lỗi
  
  if len(daily_log) == 0:
```

**Lý do**: Nếu `cursor.execute()` fail, biến `daily_log` chưa bao giờ được gán → dòng 393 crash `UnboundLocalError`. Production sẽ crash giữa chừng mỗi khi Trino timeout hoặc có lỗi query.

#### 5.3 Thêm `errors='coerce'` cho size casting

```diff
# t1_model.py — Line 421
- daily_log['size'] = daily_log['size'].astype('float')
+ daily_log['size'] = pd.to_numeric(daily_log['size'], errors='coerce')
```

**Lý do**: Nếu `size` chứa giá trị không parse được (ví dụ: `"12.34.56"` sau regex) → crash cả pipeline. `errors='coerce'` chuyển giá trị lỗi thành `NaN`, filter sẽ loại bỏ an toàn.

#### 5.4 Sửa timestamp bug trong error handler

```diff
# t1_model.py — Line 894
- 'timestamp': datetime,
+ 'timestamp': datetime.now(
+     pytz.timezone("Asia/Ho_Chi_Minh")
+ ).strftime('%Y-%m-%d %H:%M:%S'),
```

#### 5.5 Bảo vệ `cursor.close()` trong finally block

```diff
# t1_model.py — Line 912-914
  finally:
-     cursor.close()
-     db_conn.close()
+     try:
+         cursor.close()
+     except (NameError, UnboundLocalError):
+         pass
+     try:
+         db_conn.close()
+     except (NameError, UnboundLocalError):
+         pass
      logger.info('T-1 model job finished.')
```

---

### 🟡 Ưu tiên trung bình — Nên sửa sớm

#### 5.6 Thêm `str.strip()` vào validation mask

```diff
# data_validators.py — Line 115-120
  mask = (
      df['size'].notna() & df['size'].gt(0.0) &
+     (~df['size'].isin([float('inf'), float('-inf')])) &
-     df['uid'].notna() & (df['uid'] != '') &
+     df['uid'].notna() & (df['uid'].str.strip() != '') &
-     df['ip'].notna() & (df['ip'] != '10.28.100.4') &
+     df['ip'].notna() & (df['ip'].str.strip() != '10.28.100.4') &
-     df['checksum'].notna() & (df['checksum'] != '')
+     df['checksum'].notna() & (df['checksum'].str.strip() != '')
  )
```

**Lý do**: Whitespace-only `uid/checksum` và IP có spaces hiện tại bypass validation. Dữ liệu thực từ Trino có thể chứa khoảng trắng thừa do JSON parsing.

#### 5.7 Bổ sung `bd3`, `bd6` vào fillna loop

```diff
# t1_model.py — Line 432
- for idx in [1, 2, 4, 5]:
+ for idx in [1, 2, 3, 4, 5, 6]:
      daily_log[f'bd{idx}'] = daily_log[f'bd{idx}'].fillna('0')
```

**Lý do**: `bd3`, `bd6` hiện không được fill → NaN trong DataFrame sau concat. Mặc dù không tham gia signal tính toán hiện tại, nhưng gây inconsistency và lỗi nếu thêm rule tương lai.

#### 5.8 Thêm case-insensitive cho device classification

```diff
# t1_model.py — Line 464-465
  ios_txn = training_data[training_data['device_model'].notna()]
- ios_txn = ios_txn[ios_txn['device_model'].str.startswith('iPhone')]
+ ios_txn = ios_txn[ios_txn['device_model'].str.lower().str.startswith('iphone')]
```

```diff
# t1_model.py — Line 480-483
  android_txn = training_data[training_data['device_model'].notna()]
  android_txn = android_txn[
-     ~android_txn['device_model'].str.startswith('iPhone')
+     ~android_txn['device_model'].str.lower().str.startswith('iphone')
  ]
```

#### 5.9 Thêm TTL cho `high_risk_ip_key` và `blacklisted_ip_key`

```diff
# t1_model.py — After line 801
  if len(high_risk_ip) > 0:
+     pipe.delete(high_risk_ip_key)  # Xóa data cũ trước
      pipe.sadd(high_risk_ip_key, *high_risk_ip)
+     pipe.expire(high_risk_ip_key, timedelta(hours=BLACKLIST_DURATION * 24))
```

```diff
# t1_model.py — After line 883
  if len(blacklisted_ip_list) > 0:
      pipe.sadd(blacklisted_ip_key, *blacklisted_ip_list)
+     pipe.expire(blacklisted_ip_key, timedelta(hours=BLACKLIST_DURATION * 24))
```

**Lý do**: Hiện tại 2 key này **không bao giờ expire** và **không bao giờ bị xóa** → data cũ tích tụ vĩnh viễn. IP đã hết rủi ro vẫn nằm trong danh sách đen.

---

### 🟢 Ưu tiên thấp — Cải thiện code quality

#### 5.10 Xóa code trùng lặp

```diff
# t1_model.py — Line 393-416: Xóa block check len(daily_log)==0 thứ hai (dòng 406-416)
# Block này không bao giờ thực thi vì nếu daily_log==0, dòng 393 đã continue.
```

#### 5.11 Refactor `compare_score` casting

```diff
# t1_model.py — Line 439-441
- daily_log['compare_score'] = daily_log['compare_score'].astype('float')
+ daily_log['compare_score'] = pd.to_numeric(
+     daily_log['compare_score'], errors='coerce'
+ )
```

#### 5.12 Cân nhắc chuẩn hóa country_name

```python
# Thêm normalize function:
def normalize_country(name):
    if pd.isna(name):
        return name
    name = name.strip().upper()
    # Map các variant phổ biến
    VIETNAM_ALIASES = {'VIETNAM', 'VIET NAM', 'VN', 'VIỆT NAM'}
    return 'VIET NAM' if name in VIETNAM_ALIASES else name
```

#### 5.13 Tách logic thành functions riêng để dễ test

Các block logic lớn trong `aggregate_daily_features()` nên tách thành functions riêng:
- `compute_ios_signals(df)` → trả DataFrame với signal columns
- `compute_android_signals(df)` → trả DataFrame với signal columns
- `detect_high_frequency_uids(df, num_days, threshold)` → trả DataFrame
- `detect_same_size_pairs(df, min_pairs)` → trả DataFrame
- `classify_ip(df)` → trả dict {high_risk, low_risk, blacklisted}

**Lý do**: `aggregate_daily_features()` hiện tại có 670 dòng — quá dài, không thể unit test trực tiếp mà không mock Trino/Redis. Tách ra giúp test từng logic riêng lẻ với dữ liệu giả.

---

### Tóm tắt ưu tiên

| # | Khuyến nghị | Effort | Impact | Ưu tiên |
|---|-------------|--------|--------|---------|
| 5.1 | Integer comparison cho SecKiller | Thấp | **Rất cao** — sai kết quả fraud detection | 🔴 P0 |
| 5.2 | Fix UnboundLocalError | Thấp | **Rất cao** — crash production | 🔴 P0 |
| 5.3 | `errors='coerce'` cho size | Thấp | **Cao** — crash khi data bẩn | 🔴 P0 |
| 5.4 | Fix timestamp bug | Rất thấp | Trung bình — log sai | 🔴 P0 |
| 5.5 | Protect finally block | Thấp | **Cao** — crash edge case | 🔴 P0 |
| 5.6 | Strip whitespace validation | Thấp | Trung bình — data quality | 🟡 P1 |
| 5.7 | Fill bd3/bd6 | Rất thấp | Thấp — consistency | 🟡 P1 |
| 5.8 | Case-insensitive device | Thấp | Trung bình — phân loại sai | 🟡 P1 |
| 5.9 | TTL cho IP keys | Thấp | **Cao** — data bloat | 🟡 P1 |
| 5.10 | Xóa duplicate code | Rất thấp | Thấp — code quality | 🟢 P2 |
| 5.11 | Coerce compare_score | Rất thấp | Thấp — edge case | 🟢 P2 |
| 5.12 | Normalize country | Trung bình | Trung bình — false positives | 🟢 P2 |
| 5.13 | Refactor thành functions | Cao | **Rất cao** — testability | 🟢 P2 |
