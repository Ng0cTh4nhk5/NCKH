# SOP - Người D: Transaction Features (Banking Transaction Data)

> **Vai trò**: Synthetic Data Generator - Transaction Features  
> **Output**: 8 Transaction features cho 3,000 mẫu  
> **Dependencies**: KHÔNG - Sử dụng revenue TẠM để độc lập

---

## 🎯 TỔNG QUAN CÔNG VIỆC

Tạo 8 features về giao dịch ngân hàng dưới dạng synthetic.

**8 Transaction Features**:
1. avg_daily_balance (VNĐ)
2. min_balance_3m (VNĐ)
3. cash_flow_volatility (%)
4. avg_monthly_deposits (VNĐ)
5. avg_monthly_withdrawals (VNĐ)
6. net_cash_flow (VNĐ)
7. num_transactions_3m (count)
8. overdraft_count (count)

**Đặc điểm**:
- **100% synthetic** (không có real data)
- Dựa trên business logic
- Làm việc độc lập HOÀN TOÀN

---

## 📋 CĂN CỨ VÀ ASSUMPTIONS

### Tham Khảo Documentation

File `SOP_Luong3_CIC_GiaoDich.md` có:
- Section về căn cứ sinh dữ liệu
- Parameters cho transaction features
- Business logic về cash flow

**Key insights**:
- Hầu hết features correlate với revenue
- avg_daily_balance ∝ revenue (r=0.7)
- avg_monthly_deposits ≈ revenue/12 × (1.05-1.15)
- num_transactions ∝ √revenue

---

## 📋 GIAI ĐOẠN 1: THIẾT KẾ SCHEMA

### Công Việc 1.1: Create Feature Schema

**Tạo file**: `schemas/transaction_schema.yaml`

Cho MỖI feature:
- Distribution type
- Parameters
- Range
- Dependencies
- Business logic

**Ví dụ**:

```yaml
avg_daily_balance:
  distribution: Lognormal
  mean_log: 17.0  # ~25M VNĐ
  std: 1.2
  range: [0, 5_000_000_000]
  dependencies: revenue (correlation 0.7)
  
cash_flow_volatility:
  distribution: Lognormal
  mean_log: 15.0
  std: 1.0
  range: [0, inf]
  dependencies: null

overdraft_count:
  distribution: Binomial
  n: 90  # 3 months
  p: 0.05  # 5% chance/day
  dependencies: null
```

---

## 📋 GIAI ĐOẠN 2: GENERATE INDEPENDENT FEATURES

### Công Việc 2.1: Simple Independent Features

**2 features truly independent**:
1. **cash_flow_volatility**: Lognormal(15.0, 1.0)
2. **overdraft_count**: Binomial(90, 0.05)

**Generate**:

Excel:
```
Column A: sample_id (1-3000)
Column B: =LOGNORM.INV(RAND(), 15, 1)  # volatility
Column C: ... # Binomial - dùng tool
```

Python:
```python
n = 3000
volatility = np.random.lognormal(15, 1, n)
overdraft = np.random.binomial(90, 0.05, n)
```

---

## 📋 GIAI ĐOẠN 3: REVENUE-DEPENDENT FEATURES

### Công Việc 3.1: Create Temporary Revenue

**GIỐNG Người C**: Generate revenue_temp để độc lập

```
revenue_temp = Lognormal(18.5, 1.0)
```

**3000 samples**

---

### Công Việc 3.2: Generate Correlated Features

**Feature 1: avg_daily_balance** (correlation 0.7)

```python
# Base generation
balance_base = np.random.lognormal(17, 1.2, 3000)

# Apply correlation
revenue_factor = revenue_temp / revenue_temp.mean()
avg_daily_balance = balance_base * (0.3 + 0.7 * revenue_factor)
```

Verify: corr(revenue_temp, avg_daily_balance) ≈ 0.6-0.8

---

**Feature 2: avg_monthly_deposits**

Logic: Deposits ≈ Revenue mỗi tháng (với variation)

```python
avg_monthly_deposits = (revenue_temp / 12) * np.random.uniform(1.05, 1.15, 3000)
```

Deposits cao hơn revenue 1 chút (5-15%) là reasonable.

---

**Feature 3: num_transactions_3m**

Logic: Lớn hơn nhưng không tỷ lệ thuận với revenue (economies of scale)

```python
# Square root relationship
revenue_factor_sqrt = np.sqrt(revenue_temp / revenue_temp.mean())
base_transactions = np.random.poisson(50, 3000)  # Base ~50/month
num_transactions_3m = base_transactions * 3 * revenue_factor_sqrt
num_transactions_3m = np.clip(num_transactions_3m, 30, 500)
```

---

## 📋 GIAI ĐOẠN 4: DERIVED FEATURES

### Công Việc 4.1: Calculate Dependent Features

**Feature: min_balance_3m**

Derived from avg_daily_balance:
```python
# Min balance = some fraction of average
min_balance_3m = avg_daily_balance * np.random.uniform(0.3, 0.7, 3000)
```

Logic: Min luôn thấp hơn average

---

**Feature: avg_monthly_withdrawals**

Similar to deposits but slightly less:
```python
# Withdrawals ≈ Deposits, slightly less (profitable companies)
avg_monthly_withdrawals = avg_monthly_deposits * np.random.uniform(0.95, 1.05, 3000)
```

---

**Feature: net_cash_flow**

```python
net_cash_flow = avg_monthly_deposits - avg_monthly_withdrawals
```

**Note**: Có thể âm (unprofitable months) → OK!

---

## 📋 GIAI ĐOẠN 5: QA & VALIDATION

### QA Checklist

**Range checks**:
- [ ] avg_daily_balance > 0
- [ ] min_balance_3m ≤ avg_daily_balance (always)
- [ ] cash_flow_volatility ≥ 0
- [ ] num_transactions > 0
- [ ] overdraft_count ≥ 0

**Logic checks**:
- [ ] deposits and withdrawals reasonable (not 100x revenue)
- [ ] ~50% samples có net_cash_flow > 0 (profitable)

**Correlations**:
- [ ] balance vs revenue_temp: 0.6-0.8 ✓
- [ ] deposits vs revenue_temp: 0.7-0.9 ✓
- [ ] transactions vs revenue_temp: 0.4-0.6 ✓

**Distributions**:
- [ ] Histograms look reasonable
- [ ] No extreme outliers (> mean + 5*std)

---

## 📦 DELIVERABLES

**Main Output**:
1. **`synthetic_transaction_3k.csv`** ⭐
   - 3,000 rows
   - 8 Transaction columns + sample_id
   - **INCLUDE revenue_temp column**

**Supporting Files**:
2. `schemas/transaction_schema.yaml`
3. `qa_transaction_report.txt`

---

## 🆘 TROUBLESHOOTING

**Q: Correlation không chính xác?**
→ A: Adjust weights trong formula. Target ranges rộng OK.

**Q: Net cash flow hầu hết âm?**
→ A: Check withdrawal formula. Phải < deposits trung bình.

**Q: Transactions quá ít hoặc quá nhiều?**
→ A: Adjust base Poisson lambda và clip ranges.

**Q: Tại sao cần revenue_temp?**
→ A: Để tạo correlations realistic. Sẽ replace bằng revenue thật sau.

---

## ✅ SUCCESS CRITERIA

- [ ] **3,000 samples** với 8 Transaction features
- [ ] **Correlations** với revenue_temp hợp lý
- [ ] **Logic constraints** (min ≤ avg, etc)
- [ ] **No missing values**
- [ ] **Include revenue_temp**

---

## 🔄 POST-DELIVERY

Sau delivery:

**Người E (Integration Lead) sẽ**:
1. Replace revenue_temp với revenue thật
2. (Optional) Recalculate deposits/withdrawals nếu cần
3. Merge với CIC features

**Bạn done!**

---

**ĐẶC BIỆT LƯU Ý**:
- Làm việc HOÀN TOÀN ĐỘC LẬP
- Song song với Người C (không conflict)
- Revenue_temp enables independence
- Deliver file và xong!
