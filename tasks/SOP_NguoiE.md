# SOP - Người E: Demographic Features + Integration Lead

> **Vai trò**: Demographic Data Generator + Integration Coordinator  
> **Output Phase 1**: 12 Demographic features (3,000 mẫu)  
> **Output Phase 2**: Final integrated datasets (train/val/test)  
> **Dependencies**: Phase 2 phụ thuộc outputs từ A, B, C, D

---

## 🎯 TỔNG QUAN CÔNG VIỆC

Bạn có 2 phases:

**Phase 1: Generate Demographic** (Độc lập)
- 12 features nhân khẩu học và tài sản đảm bảo
- Làm song song với người khác
- 3,000 samples

**Phase 2: Integration** (Phụ thuộc tất cả)
- Merge tất cả features từ A, B, C, D
- Generate target variable
- Train/Val/Test split
- QA và documentation

---

# PHASE 1: DEMOGRAPHIC FEATURES

## 📋 GIAI ĐOẠN 1.1: CATEGORICAL FEATURES

### Công Việc: Generate 5 Categorical Features

**5 Features**:
1. `business_type` (TNHH/CP/DNTN)
2. `industry_code` (46/10/56)
3. `district_code` (1-24)
4. `owner_gender` (M/F)
5. `has_collateral` (0/1)

**Distributions**:

```python
business_type:
  - TNHH: 60%
  - CP: 30%
  - DNTN: 10%

industry_code:
  - 46 (Bán buôn): 33.3%
  - 10 (Sản xuất): 33.3%
  - 56 (F&B): 33.4%
  # Exactly 1000 each!

district_code:
  - Uniform [1-24]
  # Or weighted by business density

owner_gender:
  - M: 70%
  - F: 30%

has_collateral:
  - 0: 60%
  - 1: 40%
```

**Method** (Excel):
```
Column A: sample_id
Column B: =IF(RAND()<0.6, "TNHH", IF(RAND()<0.75, "CP", "DNTN"))
Column C: Assign 1000 samples to each industry (46, 10, 56)
Column D: =RANDBETWEEN(1, 24)
```

---

## 📋 GIAI ĐOẠN 1.2: NUMERIC FEATURES

### Công Việc: Generate 3 Numeric Features

**3 Features**:
1. `registered_capital` (VNĐ)
2. `owner_age` (years)
3. `owner_experience` (years)

**Distributions**:

```python
registered_capital:
  - Lognormal(19, 1)  # ~150M VNĐ median
  - Range: [10M, 10B]

owner_age:
  - Normal(42, 8)
  - Range: [25, 70]

owner_experience:
  - Poisson(8)
  - Constraint: ≤ owner_age - 20
```

**Logic**: Experience không thể > (age - 20)

```python
# Generate
age = np.random.normal(42, 8, 3000)
age = np.clip(age, 25, 70)

exp_raw = np.random.poisson(8, 3000)
owner_experience = np.minimum(exp_raw, age - 20)
owner_experience = np.maximum(owner_experience, 0)
```

---

## 📋 GIAI ĐOẠN 1.3: COLLATERAL FEATURES

### Công Việc: Generate 4 Collateral Features (Conditional)

**4 Features**:
1. `collateral_type` (null/Real Estate/Vehicle/Other)
2. `collateral_value` (VNĐ)
3. `loan_amount` (VNĐ)
4. `loan_to_value` (ratio)

**Conditional logic**:

```python
IF has_collateral == 0:
    collateral_type = null
    collateral_value = 0
    loan_amount = 0
    loan_to_value = 0
ELSE:
    collateral_type = random("Real Estate": 70%, "Vehicle": 20%, "Other": 10%)
    collateral_value = Lognormal(19.5, 0.8)  # ~200M
    loan_to_value = Uniform(0.5, 0.8)  # 50-80%
    loan_amount = collateral_value * loan_to_value
```

**IMPORTANT**: 60% rows sẽ có nulls/zeros (no collateral)

---

## 📋 GIAI ĐOẠN 1.4: QA DEMOGRAPHIC

### QA Checklist

**Data quality**:
- [ ] Exactly 3,000 samples
- [ ] Industry balanced: 1,000/1,000/1,000
- [ ] owner_experience ≤ owner_age - 20 (100% rows)
- [ ] Collateral features null when has_collateral = 0

**Distributions**:
- [ ] business_type: ~60/30/10% split
- [ ] owner_age: reasonable (không có 18 tuổi)

**Phase 1 Output**: `synthetic_demographic_3k.csv`

---

# PHASE 2: INTEGRATION

> **QUAN TRỌNG**: Phase này chỉ bắt đầu KHI ĐÃ CÓ outputs từ A, B, C, D!

## 📋 GIAI ĐOẠN 2.1: COLLECT INPUTS

### Công Việc: Verify All Inputs Ready

**Checklist inputs**:

| Nguồn | File | Rows | Columns | Status |
|-------|------|------|---------|--------|
| A | `synthetic_financial_3k.csv` | 3,000 | 12 + industry_code | ⏳ |
| B (district) | `district_lookup.csv` | 24 | 7 | ⏳ |
| B (industry) | `industry_risk.csv` | 3 | 5 | ⏳ |
| C | `synthetic_cic_3k.csv` | 3,000 | 8 + revenue_temp | ⏳ |
| D | `synthetic_transaction_3k.csv` | 3,000 | 8 + revenue_temp | ⏳ |
| E | `synthetic_demographic_3k.csv` | 3,000 | 12 | ✅ Ready |

**Actions**:
- Request files nếu chưa có
- Verify row counts = 3,000
- Check column counts match expected

---

## 📋 GIAI ĐOẠN 2.2: MERGE CIC + TRANSACTION

### Công Việc: Fix Revenue Dependency

**ISSUE**: C và D đều có `revenue_temp` columns

**SOLUTION**:

1. **Replace revenue_temp với revenue THẬT từ A**:
```python
# Load
financial = pd.read_csv('synthetic_financial_3k.csv')
cic = pd.read_csv('synthetic_cic_3k.csv')
transaction = pd.read_csv('synthetic_transaction_3k.csv')

# Extract real revenue
real_revenue = financial['revenue']  # or doanh_thu_thuan

# Replace trong CIC
cic['revenue'] = real_revenue
cic = cic.drop('revenue_temp', axis=1)

# Replace trong Transaction
transaction['revenue'] = real_revenue
transaction = transaction.drop('revenue_temp', axis=1)
```

2. **Recalculate debt_burden_ratio** (trong CIC):
```python
cic['debt_burden_ratio'] = cic['total_outstanding_debt'] / (cic['revenue'] / 12)
cic['debt_burden_ratio'] = cic['debt_burden_ratio'].clip(0, 3)
```

3. **Merge CIC + Transaction**:
```python
cic_transaction = pd.concat([cic, transaction], axis=1)
# Remove duplicate columns (sample_id, revenue)
```

**Output**: `merged_cic_transaction_3k.csv` (16 features)

---

## 📋 GIAI ĐOẠN 2.3: MASTER MERGE

### Công Việc: Merge All Feature Sets

**Horizontal merge**:

```python
# All có cùng 3000 rows, theo thứ tự 1-3000
merged_raw = pd.concat([
    financial,           # 12 features
    cic_transaction,     # 16 features
    demographic          # 12 features
], axis=1)

# Remove duplicate columns
# Result: 3000 rows × 40 features (12+16+12)
```

**Verify**:
- 3,000 rows intact
- ~40 columns (depends on duplicates)
- industry_code consistent (từ A và E phải match!)

**Output**: `merged_raw.csv`

---

## 📋 GIAI ĐOẠN 2.4: ENRICH VỚI LOOKUPS

### Công Việc: Join District và Industry Tables

**Join 1: District lookup** (join ON district_code)

```python
district_lookup = pd.read_csv('district_lookup.csv')

enriched = merged_raw.merge(
    district_lookup,
    on='district_code',
    how='left'
)
# Add: business_density, avg_income, district_risk_score
```

**Join 2: Industry lookup** (join ON industry_code)

```python
industry_risk = pd.read_csv('industry_risk.csv')

enriched = enriched.merge(
    industry_risk,
    left_on='industry_code',
    right_on='vsic_code',
    how='left'
)
# Add: industry_name, sector, default_rate_estimate, industry_risk_score
```

**QA**: No missing values after join (verify 100% match)

**Output**: `merged_enriched.csv` (~48 features)

---

## 📋 GIAI ĐOẠN 2.5: GENERATE TARGET VARIABLE

### Công Việc: Create Binary Label `default`

**Objective**: 5% default rate

**Multi-factor approach**:

```python
# Factor 1: CIC score (40% weight)
cic_risk = np.where(cic_score < 550, 0.15,
            np.where(cic_score < 650, 0.08,
            np.where(cic_score < 700, 0.03, 0.01)))

# Factor 2: Debt burden (30% weight)
debt_risk = np.where(debt_burden_ratio > 0.8, 0.12,
             np.where(debt_burden_ratio > 0.5, 0.06, 0.02))

# Factor 3: Past due (20% weight)
past_due_risk = np.where(num_past_due_90d > 0, 0.20,
                 np.where(num_past_due_30d > 2, 0.10,
                 np.where(num_past_due_30d > 0, 0.05, 0)))

# Factor 4: External risks (10% weight)
external_risk = (industry_risk_score / 100) + (district_risk_score / 200)

# Combined
default_probability = (cic_risk * 0.4 + 
                      debt_risk * 0.3 + 
                      past_due_risk * 0.2 + 
                      external_risk * 0.1)

# Generate binary
default = (np.random.rand(3000) < default_probability).astype(int)
```

**Check actual rate**:
```python
actual_rate = default.mean()
# Target: 0.045 - 0.055 (4.5% - 5.5%)
```

**Adjust if needed**: Scale probabilities and regenerate

**Output**: `merged_with_target.csv` (+2 columns: default, default_probability)

---

## 📋 GIAI ĐOẠN 2.6: TRAIN/VAL/TEST SPLIT

### Công Việc: Stratified Split

**Ratios**: 70/15/15

**Stratify by `default`** to maintain 5% rate:

```python
from sklearn.model_selection import train_test_split

# First split: train vs temp (70% vs 30%)
train, temp = train_test_split(
    merged_with_target,
    test_size=0.3,
    stratify=merged_with_target['default'],
    random_seed=42
)

# Second split: val vs test (50% vs 50% of temp)
val, test = train_test_split(
    temp,
    test_size=0.5,
    stratify=temp['default'],
    random_seed=42
)
```

**Verify**:
- train: 2,100 rows (70%)
- val: 450 rows (15%)
- test: 450 rows (15%)
- Default rate ~5% in each split

**Outputs**:
- `train_2.1k.csv`
- `val_450.csv`
- `test_450.csv`
- `full_3k.csv` (backup)

---

## 📋 GIAI ĐOẠN 2.7: QA TỔNG THỂ

### Final QA Checklist

**Completeness**:
- [ ] 3,000 total samples
- [ ] ~48 features + target
- [ ] Splits: 2100/450/450

**Quality**:
- [ ] Missing < 2% (except collateral fields)
- [ ] No NaNs in critical features
- [ ] All ranges reasonable

**Logic**:
- [ ] num_past_due_90d ≤ num_past_due_30d
- [ ] owner_experience ≤ owner_age - 20
- [ ] Collateral nulls when has_collateral = 0

**Target**:
- [ ] Default rate: 4.5-5.5% overall
- [ ] Consistent across train/val/test (±0.5%)
- [ ] High CIC score → low default rate (verify!)

**Industry balance**:
- [ ] Each industry ~1,000 samples
- [ ] Distributed across train/val/test

---

## 📋 GIAI ĐOẠN 2.8: DOCUMENTATION

### Công Việc: Create Documentation

**1. Data Dictionary** (`data_dictionary.csv`):
```csv
feature_name,description,type,range,source
revenue,Doanh thu năm,numeric,10M-5B,Financial
cic_score,Điểm tín dụng,numeric,300-850,CIC
...
```

**2. Data Documentation** (`data_documentation.md`):
- Overview: samples, features, default rate
- Data sources: A/B/C/D/E contributions
- Generation process summary
- Assumptions & limitations

**3. QA Report** (`QA_Report.txt`):
- All checks run
- Pass/fail status
- Issues và resolutions

**4. Process Log** (`process_log.txt`):
- Timeline
- Issues encountered
- Final metrics

---

## 📦 DELIVERABLES

**Phase 1** (Demographic):
1. `synthetic_demographic_3k.csv`

**Phase 2** (Integration):
2. **`train_2.1k.csv`** ⭐ MAIN
3. **`val_450.csv`** ⭐ MAIN
4. **`test_450.csv`** ⭐ MAIN
5. `full_3k.csv`
6. `data_dictionary.csv`
7. `data_documentation.md`
8. `QA_Report.txt`
9. `process_log.txt`

---

## 🆘 TROUBLESHOOTING

**Q: Default rate không đúng 5%?**
→ A: Scale default_probability và regenerate. Accept 4.5-5.5%.

**Q: Missing sau khi join lookups?**
→ A: Check district_code và industry_code có match không. Investigate discrepancies.

**Q: Train/val/test có default rates khác nhau?**
→ A: Verify stratify parameter. Re-split nếu cần.

**Q: Industry không balanced?**
→ A: Check demographic generation. Phải có đúng 1,000/ngành.

---

## ✅ SUCCESS CRITERIA

**Phase 1**:
- [ ] 3,000 demographic samples
- [ ] 12 features
- [ ] Industry balanced

**Phase 2**:
- [ ] 4 dataset files (train/val/test/full)
- [ ] ~48 features + target
- [ ] 5% default rate ± 0.5%
- [ ] Documentation complete
- [ ] All QA passed

---

**VAI TRÒ ĐẶC BIỆT**:
- Bạn là INTEGRATION LEAD
- Coordinate với tất cả members
- Quality gatekeeper
- Deliver final datasets for modeling!
