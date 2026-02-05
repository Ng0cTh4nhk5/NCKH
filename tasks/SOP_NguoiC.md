# SOP - Người C: CIC Features (Credit Bureau Data)

> **Vai trò**: Synthetic Data Generator - CIC Features  
> **Output**: 8 CIC features cho 3,000 mẫu  
> **Dependencies**: KHÔNG - Sử dụng revenue TẠM để độc lập

---

## 🎯 TỔNG QUAN CÔNG VIỆC

Tạo 8 features về lịch sử tín dụng từ CIC (Credit Information Center) dưới dạng synthetic.

**8 CIC Features**:
1. cic_score (300-850)
2. num_active_loans (0-10)
3. total_outstanding_debt (VNĐ)
4. max_past_due_days (0-180)
5. num_past_due_30d (count)
6. num_past_due_90d (count)
7. debt_burden_ratio (ratio)
8. credit_history_length (years)

**Đặc điểm**:
- **100% synthetic** (không có real data)
- Dựa trên assumptions từ research
- Làm việc độc lập HOÀN TOÀN

---

## 📋 CĂN CỨ VÀ ASSUMPTIONS

### Tham Khảo SOP Luồng 3

File `SOP_Luong3_CIC_GiaoDich.md` có section:
**"⚠️ QUAN TRỌNG: CĂN CỨ SINH DỮ LIỆU"**

Đọc kỹ section này để hiểu:
- Rationale cho từng parameter
- Sources (FICO score, NHNN reports, banking standards)
- Limitations (synthetic data)

**Key parameters**:
- CIC score: N(650, 50)
- Num loans: Poisson(1.5)
- Past due rate: 5%
- Debt correlation với revenue: 0.6

---

## 📋 GIAI ĐOẠN 1: THIẾT KẾ SCHEMA

### Công Việc 1.1: Create Feature Schema

**Tạo file**: `schemas/cic_schema.yaml` hoặc Excel

Cho MỖI feature, specify:
- **Description**: Ý nghĩa
- **Distribution**: Normal/Lognormal/Poisson/etc
- **Parameters**: mean, std, lambda, etc.
- **Range**: [min, max]
- **Dependencies**: Phụ thuộc features nào
- **Business logic**: Constraints

**Ví dụ**:

```yaml
cic_score:
  description: Điểm tín dụng CIC
  distribution: Normal
  mean: 650
  std: 50
  range: [300, 850]
  dependencies: null
  
num_active_loans:
  distribution: Poisson
  lambda: 1.5
  range: [0, 10]
  dependencies: null

total_outstanding_debt:
  distribution: Lognormal
  mean_log: 18.5  # ~100M VNĐ median
  std: 1.2
  range: [0, 10_000_000_000]
  dependencies: revenue (correlation 0.6)
```

**Output**: File schema đầy đủ cho 8 features

---

## 📋 GIAI ĐOẠN 2: GENERATE INDEPENDENT FEATURES

### Công Việc 2.1: Generate Simple Features

**3 features không phụ thuộc**:
1. cic_score
2. num_active_loans
3. credit_history_length

**Method** (chọn 1):

**Option A: Excel**
```
Column A: sample_id (1-3000)
Column B: =NORM.INV(RAND(), 650, 50)  # cic_score
Column C: Poisson(1.5) - dùng helper or online tool
Column D: =LOGNORM.INV(RAND(), 1.1, 0.6)  # credit_history
```

**Option B: Python**
```python
import numpy as np
n = 3000
cic_score = np.random.normal(650, 50, n)
cic_score = np.clip(cic_score, 300, 850)
```

**Output**: 3 columns × 3000 rows

---

## 📋 GIAI ĐOẠN 3: GENERATE CORRELATED FEATURES

### Công Việc 3.1: Create Temporary Revenue

**VẤN ĐỀ**: `total_outstanding_debt` cần correlate với revenue (từ Người A)

**GIẢI PHÁP**: Generate revenue TẠM

```
revenue_temp = Lognormal(18.5, 1.0)  # Giống params của Người A
```

**Lý do**: 
- Để làm việc ĐỘC LẬP
- Không phải chờ Người A
- Dùng revenue_temp CHỈ để create correlation
- SAU NÀY sẽ replace bằng revenue thật từ Người A

---

### Công Việc 3.2: Generate Debt với Correlation

**Step-by-step**:

1. Generate revenue_temp (3000 samples)
2. Generate debt_base (uncorrelated):
   ```
   debt_base = Lognormal(18.5, 1.2)
   ```
3. Apply correlation 0.6:
   ```
   revenue_factor = revenue_temp / mean(revenue_temp)
   debt_adjusted = debt_base × (0.4 + 0.6 × revenue_factor)
   ```
4. Verify correlation: corr(revenue_temp, debt_adjusted) ≈ 0.5-0.7

**Output**: `total_outstanding_debt` column

---

## 📋 GIAI ĐOẠN 4: CONDITIONAL FEATURES

### Công Việc 4.1: Past Due Features

**Logic dependencies**:
- `num_past_due_90d` ≤ `num_past_due_30d`
- If `max_past_due_days` = 0 → all past_due counts = 0

**Steps**:

1. **Generate max_past_due_days**:
   - 95% samples: 0 (no past due)
   - 5% samples: random [30, 60, 90, 120, 180]

2. **Generate num_past_due_30d**:
   ```
   IF max_past_due_days = 0:
       num_past_due_30d = 0
   ELSE IF max_past_due_days >= 30:
       num_past_due_30d = Poisson(1) + 1
   ```

3. **Generate num_past_due_90d**:
   ```
   IF max_past_due_days < 90:
       num_past_due_90d = 0
   ELSE:
       num_past_due_90d = Binomial(n=num_past_due_30d, p=0.3)
   ```

**Validate**: Check constraints hold 100%

---

## 📋 GIAI ĐOẠN 5: DERIVED FEATURES

### Công Việc 5.1: Calculate Debt Burden Ratio

**Formula**:
```
debt_burden_ratio = total_outstanding_debt / (revenue_temp / 12)
```

**Handle edge cases**:
- If revenue = 0 → set ratio = 0
- If ratio > 3 → clip to 3 (unrealistic)

**Output**: Final 8th feature

---

## 📋 GIAI ĐOẠN 6: QA & VALIDATION

### QA Checklist

**Range checks**:
- [ ] cic_score: all in [300, 850]
- [ ] num_active_loans: all in [0, 10]
- [ ] total_outstanding_debt: all ≥ 0
- [ ] past_due counts: all ≥ 0

**Logic constraints**:
- [ ] num_past_due_90d ≤ num_past_due_30d (100% rows)
- [ ] If max_past_due = 0 → past_due counts = 0 (100%)
- [ ] ~5% samples have past due > 0

**Distributions**:
- [ ] Plot histograms - shapes reasonable?
- [ ] Means close to expected?

**Correlations**:
- [ ] debt vs revenue_temp: 0.5-0.7 ✓
- [ ] Other correlations: not too strong (< 0.8)

---

## 📦 DELIVERABLES

**Main Output**:
1. **`synthetic_cic_3k.csv`** ⭐
   - 3,000 rows
   - 8 CIC columns + sample_id
   - **INCLUDE revenue_temp column** (quan trọng!)

**Supporting Files**:
2. `schemas/cic_schema.yaml` - Feature definitions
3. `qa_cic_report.txt` - QA validation results

---

## 🆘 TROUBLESHOOTING

**Q: Làm sao generate Poisson trong Excel?**
→ A: Dùng online tools hoặc Python helper. Hoặc approximate bằng cách khác.

**Q: Correlation không đúng 0.6?**
→ A: OK nếu trong range [0.5, 0.7]. Adjust weights (0.4, 0.6) nếu cần.

**Q: Logic constraints fail?**
→ A: Re-generate conditional features. Verify formulas.

**Q: Tại sao phải include revenue_temp?**
→ A: Để Integration phase biết correlation structure. Sẽ replace bằng revenue thật sau.

---

## ✅ SUCCESS CRITERIA

- [ ] **3,000 samples** với 8 CIC features
- [ ] **Correlation** với revenue_temp: 0.5-0.7
- [ ] **Logic constraints** satisfied 100%
- [ ] **~5% past due rate**
- [ ] **No missing values**
- [ ] **Include revenue_temp** trong output

---

## 🔄 POST-DELIVERY (Trong Integration)

Sau khi bạn deliver `synthetic_cic_3k.csv`:

**Người E (Integration Lead) sẽ**:
1. Replace `revenue_temp` với revenue thật từ Người A
2. Recalculate `debt_burden_ratio` với revenue thật
3. (Optional) Re-adjust `total_outstanding_debt` để maintain correlation

**Bạn KHÔNG CẦN LÀM GÌ THÊM** - chỉ deliver file và done!

---

**ĐẶC BIỆT LƯU Ý**:
- Làm việc HOÀN TOÀN ĐỘC LẬP
- KHÔNG chờ ai
- Revenue_temp giúp bạn independent
- Deliver và xong task!
