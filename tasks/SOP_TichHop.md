# Hướng Dẫn Chi Tiết: Giai Đoạn Tích Hợp (Integration)

> **Thời gian**: 1-2 tuần
> **Người thực hiện**: Trưởng nhóm / Thành viên D
> **Input**: Outputs từ 4 luồng
> **Output**: `train_7k.csv`, `val_1.5k.csv`, `test_1.5k.csv`, `full_10k.csv`

---

## 📋 TUẦN 1: MERGE DATASETS

### Ngày 1-2: Gộp Tất Cả Features

#### Bước 1.1: Kiểm Tra Input Files

**Checklist trước khi merge**:

```bash
# Verify tất cả files tồn tại
ls -lh data/final/

# Expected files:
# ✓ synthetic_financial_10k.csv (từ Luồng 1)
# ✓ district_lookup.csv (từ Luồng 2)
# ✓ industry_risk.csv (từ Luồng 2)
# ✓ synthetic_cic_transaction_10k.csv (từ Luồng 3)
# ✓ synthetic_demographic_10k.csv (từ Luồng 4)
```

**Kiểm tra số dòng**:

```python
import pandas as pd

files = {
    'Financial': 'data/final/synthetic_financial_10k.csv',
    'CIC-Transaction': 'data/final/synthetic_cic_transaction_10k.csv',
    'Demographic': 'data/final/synthetic_demographic_10k.csv'
}

for name, path in files.items():
    df = pd.read_csv(path)
    print(f"{name}: {len(df)} rows, {len(df.columns)} columns")
    
# Expected: Tất cả 10,000 rows
```

---

#### Bước 1.2: Merge Theo Index

**File**: `scripts/merge_all_features.py`

```python
"""
Gộp tất cả features thành 1 dataset
"""
import pandas as pd
import numpy as np

def load_all_features():
    """Load tất cả feature sets"""
    # Financial (20 features)
    df_financial = pd.read_csv('data/final/synthetic_financial_10k.csv')
    
    # CIC + Transaction (16 features)
    df_cic_trans = pd.read_csv('data/final/synthetic_cic_transaction_10k.csv')
    
    # Demographic (12 features)
    df_demo = pd.read_csv('data/final/synthetic_demographic_10k.csv')
    
    # Verify lengths
    assert len(df_financial) == len(df_cic_trans) == len(df_demo) == 10000
    
    return df_financial, df_cic_trans, df_demo

def merge_features(df_financial, df_cic_trans, df_demo):
    """Merge theo index"""
    # Concat horizontal
    df_merged = pd.concat([df_financial, df_cic_trans, df_demo], axis=1)
    
    # Add sample_id
    df_merged.insert(0, 'sample_id', range(1, 10001))
    
    print(f"Merged dataset: {df_merged.shape}")
    print(f"Total features: {len(df_merged.columns) - 1}")  # -1 for sample_id
    
    return df_merged

# Run
df_fin, df_cic, df_demo = load_all_features()
df_merged = merge_features(df_fin, df_cic, df_demo)

# Save checkpoint
df_merged.to_csv('data/intermediate/merged_raw.csv', index=False)

print("✅ Features merged!")
print(df_merged.head())
```

**Expected output**:
```
Merged dataset: (10000, 49)
Total features: 48

Columns: sample_id, revenue, roa, roe, ..., has_collateral, ltv
```

---

### Ngày 3: Enrich Với Lookup Tables

#### Bước 3.1: Join District Features

**File**: `scripts/enrich_lookups.py`

```python
"""
Làm giàu dữ liệu với lookup tables
"""
import pandas as pd

# Load merged
df = pd.read_csv('data/intermediate/merged_raw.csv')

# Load lookup tables
df_district = pd.read_csv('data/lookup_tables/district_lookup.csv')
df_industry = pd.read_csv('data/lookup_tables/industry_risk.csv')

print(f"Before enrich: {df.shape}")

# Join district features
df = df.merge(
    df_district[['district_code', 'business_density', 'avg_income', 'risk_score']],
    on='district_code',
    how='left',
    suffixes=('', '_district')
)

# Rename for clarity
df.rename(columns={
    'risk_score': 'district_risk_score',
    'business_density': 'district_business_density',
    'avg_income': 'district_avg_income'
}, inplace=True)

# Join industry features
df = df.merge(
    df_industry[['vsic_code', 'industry_name', 'default_rate_estimate', 'risk_score']],
    left_on='industry_code',
    right_on='vsic_code',
    how='left',
    suffixes=('', '_industry')
)

# Rename
df.rename(columns={
    'risk_score': 'industry_risk_score'
}, inplace=True)

# Drop duplicate vsic_code column
df.drop(columns=['vsic_code'], inplace=True)

print(f"After enrich: {df.shape}")
print(f"New columns: {df.columns[-6:].tolist()}")

# Save
df.to_csv('data/intermediate/merged_enriched.csv', index=False)

print("✅ Data enriched with lookups!")
```

**Output**: Dataset với ~58 features (48 original + 10 từ lookups)

---

### Ngày 4-5: Generate Target Variable (Default Label)

#### Bước 4.1: Design Rule-Based Default Logic

**File**: `scripts/generate_target.py`

```python
"""
Sinh target variable (nhãn vỡ nợ)
"""
import pandas as pd
import numpy as np

def calculate_default_probability(row):
    """
    Tính xác suất vỡ nợ dựa trên rules
    
    Factors:
    1. CIC score (40%)
    2. Debt ratios (30%)
    3. Past due history (20%)
    4. Industry/District risk (10%)
    """
    score = 0
    
    # Factor 1: CIC score (inverse)
    if row['cic_score'] < 550:
        score += 0.15
    elif row['cic_score'] < 650:
        score += 0.08
    elif row['cic_score'] < 700:
        score += 0.03
    else:
        score += 0.01
    
    # Factor 2: Debt burden
    if row['debt_burden_ratio'] > 0.8:
        score += 0.12
    elif row['debt_burden_ratio'] > 0.5:
        score += 0.06
    else:
        score += 0.02
    
    # Factor 3: Past due history
    if row['num_past_due_90d'] > 0:
        score += 0.20  # Very high risk
    elif row['num_past_due_30d'] > 2:
        score += 0.10
    elif row['num_past_due_30d'] > 0:
        score += 0.05
    
    # Factor 4: External risks
    score += row['industry_risk_score'] * 0.005  # 0-0.05
    score += row['district_risk_score'] * 0.005   # 0-0.05
    
    # Clip to [0, 1]
    return min(1.0, max(0.0, score))

def generate_default_label(df, target_default_rate=0.05):
    """
    Sinh nhãn default với target rate ~5%
    """
    # Calculate probability cho mỗi row
    df['default_probability'] = df.apply(calculate_default_probability, axis=1)
    
    # Generate binary label
    # Sử dụng probability để sample
    df['default'] = (np.random.random(len(df)) < df['default_probability']).astype(int)
    
    # Check actual rate
    actual_rate = df['default'].mean()
    print(f"Target default rate: {target_default_rate:.2%}")
    print(f"Actual default rate: {actual_rate:.2%}")
    
    # Adjust nếu cần (optional)
    if abs(actual_rate - target_default_rate) > 0.01:
        print("⚠️  Adjusting to target rate...")
        
        # Rebalance
        n_defaults_target = int(len(df) * target_default_rate)
        n_defaults_current = df['default'].sum()
        
        if n_defaults_current < n_defaults_target:
            # Cần thêm defaults
            # Pick from highest probabilities
            non_defaults = df[df['default'] == 0].index
            probs = df.loc[non_defaults, 'default_probability'].values
            probs = probs / probs.sum()  # Normalize
            
            to_flip = np.random.choice(
                non_defaults,
                size=n_defaults_target - n_defaults_current,
                replace=False,
                p=probs
            )
            df.loc[to_flip, 'default'] = 1
        
        else:
            # Cần bớt defaults
            defaults = df[df['default'] == 1].index
            probs = 1 - df.loc[defaults, 'default_probability'].values
            probs = probs / probs.sum()
            
            to_flip = np.random.choice(
                defaults,
                size=n_defaults_current - n_defaults_target,
                replace=False,
                p=probs
            )
            df.loc[to_flip, 'default'] = 0
        
        print(f"Adjusted default rate: {df['default'].mean():.2%}")
    
    return df

# Load
df = pd.read_csv('data/intermediate/merged_enriched.csv')

# Generate target
df = generate_default_label(df, target_default_rate=0.05)

# Save
df.to_csv('data/intermediate/merged_with_target.csv', index=False)

print("✅ Target variable generated!")
print(f"\nDefault distribution:")
print(df['default'].value_counts())
```

---

## 📋 TUẦN 2: QA & FINALIZATION

### Ngày 1-2: Data Quality Checks

**File**: `scripts/qa_final_dataset.py`

```python
"""
Kiểm tra chất lượng dataset cuối cùng
"""
import pandas as pd
import numpy as np

df = pd.read_csv('data/intermediate/merged_with_target.csv')

print("=" * 60)
print("DATA QUALITY REPORT")
print("=" * 60)

# Check 1: Missing values
print("\n1. MISSING VALUES")
missing_pct = (df.isnull().sum() / len(df) * 100).sort_values(ascending=False)
print(missing_pct[missing_pct > 0])

if (missing_pct > 5).any():
    print("⚠️  WARNING: Some features have > 5% missing")
else:
    print("✓ All features < 5% missing")

# Check 2: Outliers (for numeric columns)
print("\n2. OUTLIERS (values > 5 std)")
numeric_cols = df.select_dtypes(include=[np.number]).columns

for col in numeric_cols[:10]:  # Sample 10 columns
    mean = df[col].mean()
    std = df[col].std()
    outliers = df[(df[col] < mean - 5*std) | (df[col] > mean + 5*std)]
    
    if len(outliers) > 0:
        print(f"  {col}: {len(outliers)} outliers ({len(outliers)/len(df)*100:.2f}%)")

# Check 3: Default rate
print("\n3. DEFAULT RATE")
default_rate = df['default'].mean()
print(f"  Overall: {default_rate:.2%}")

# By risk segments
print("\n  By CIC score:")
df['cic_segment'] = pd.cut(df['cic_score'], bins=[0, 600, 650, 700, 850])
print(df.groupby('cic_segment')['default'].mean())

# Check 4: Correlations (sanity check)
print("\n4. KEY CORRELATIONS")
print(f"  cic_score vs default: {df[['cic_score', 'default']].corr().iloc[0, 1]:.3f}")
print(f"  debt_burden_ratio vs default: {df[['debt_burden_ratio', 'default']].corr().iloc[0, 1]:.3f}")

# Check 5: Feature distributions
print("\n5. FEATURE DISTRIBUTIONS")
print(df.describe().T[['mean', 'std', 'min', 'max']].head(10))

print("\n" + "=" * 60)
print("QA COMPLETE")
print("=" * 60)

# Save report
with open('data/final/QA_REPORT.txt', 'w') as f:
    f.write(df.describe().to_string())
```

---

### Ngày 3: Train/Val/Test Split

**File**: `scripts/split_dataset.py`

```python
"""
Chia dataset thành train/val/test
"""
import pandas as pd
from sklearn.model_selection import train_test_split

# Load
df = pd.read_csv('data/intermediate/merged_with_target.csv')

print(f"Total samples: {len(df)}")
print(f"Default rate: {df['default'].mean():.2%}")

# Split ratios: 70% train, 15% val, 15% test
# First split: 70% train, 30% temp
train_df, temp_df = train_test_split(
    df,
    test_size=0.3,
    random_state=42,
    stratify=df['default']  # Maintain default rate
)

# Second split: 50-50 của temp = 15% val, 15% test
val_df, test_df = train_test_split(
    temp_df,
    test_size=0.5,
    random_state=42,
    stratify=temp_df['default']
)

print(f"\nSplit results:")
print(f"  Train: {len(train_df)} ({len(train_df)/len(df)*100:.1f}%)")
print(f"  Val:   {len(val_df)} ({len(val_df)/len(df)*100:.1f}%)")
print(f"  Test:  {len(test_df)} ({len(test_df)/len(df)*100:.1f}%)")

print(f"\nDefault rates:")
print(f"  Train: {train_df['default'].mean():.2%}")
print(f"  Val:   {val_df['default'].mean():.2%}")
print(f"  Test:  {test_df['default'].mean():.2%}")

# Save
train_df.to_csv('data/final/train_7k.csv', index=False)
val_df.to_csv('data/final/val_1.5k.csv', index=False)
test_df.to_csv('data/final/test_1.5k.csv', index=False)
df.to_csv('data/final/full_10k.csv', index=False)

print("\n✅ Datasets saved!")
```

---

### Ngày 4-5: Documentation

**File**: `docs/data_documentation.md`

````markdown
# Data Documentation: SME Credit Scoring Dataset

## Dataset Overview

- **Total Samples**: 10,000
- **Total Features**: 58
- **Target Variable**: `default` (binary)
- **Default Rate**: 5.0%

## Data Split

| Split | Samples | Default Rate |
|-------|---------|--------------|
| Train | 7,000   | 5.0%         |
| Val   | 1,500   | 5.0%         |
| Test  | 1,500   | 5.0%         |

## Feature Groups

### 1. Financial Features (20)
From synthetic data learned from listed companies.

| Feature | Description | Range | Distribution |
|---------|-------------|-------|--------------|
| `revenue` | Doanh thu năm | 10M-5B VNĐ | Lognormal |
| `roa` | Return on Assets | -20% - 30% | Normal |
| ... | ... | ... | ... |

### 2. CIC Features (8)
Synthetic credit bureau data.

| Feature | Description | Range |
|---------|-------------|-------|
| `cic_score` | Credit score | 300-850 |
| `num_active_loans` | Active loans | 0-10 |
| ... | ... | ... |

### 3. Transaction Features (8)
Synthetic banking transaction data.

### 4. Behavioral Features (10)
From public data + synthetic.

### 5. Demographic Features (12)
Fully synthetic.

## Data Generation Process

1. **Financial**: Learned distributions from 200 UPCOM companies
2. **CIC/Transaction**: Rule-based synthetic with realistic correlations
3. **Demographic**: Categorical + numeric synthetic
4. **Target**: Rule-based with 5% default rate

## Assumptions & Limitations

1. **Geographic Scope**: TP.HCM only
2. **Synthetic Data**: 100% synthetic, no real SME data
3. **Correlations**: Designed to be realistic but not validated with actual SME defaults
4. **Time Period**: Simulated as 2024 data

## Quality Metrics

- Missing values: < 2% all features
- Outliers: < 1% extreme outliers
- Default rate balance: Maintained across splits

## Files

```
data/final/
├── train_7k.csv          # Training set
├── val_1.5k.csv          # Validation set
├── test_1.5k.csv         # Test set
├── full_10k.csv          # Complete dataset
├── data_dictionary.csv   # Feature definitions
└── QA_REPORT.txt         # Quality assurance report
```

## Usage

```python
import pandas as pd

# Load training data
train = pd.read_csv('data/final/train_7k.csv')

X_train = train.drop(columns=['sample_id', 'default', 'default_probability'])
y_train = train['default']
```
````

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] `merged_enriched.csv` - Tất cả features gộp
- [ ] `merged_with_target.csv` - Có nhãn default
- [ ] QA Report - Passed (missing < 5%, default ~5%)
- [ ] `train_7k.csv` - Training set
- [ ] `val_1.5k.csv` - Validation set
- [ ] `test_1.5k.csv` - Test set
- [ ] `full_10k.csv` - Full dataset
- [ ] `data_documentation.md` - Complete docs
- [ ] `data_dictionary.csv` - Feature definitions

---

## 🆘 TROUBLESHOOTING

**Q: Default rate không đúng 5%?**
A: Run adjustment logic trong `generate_target.py`

**Q: Train/val/test có distribution khác nhau?**
A: Check `stratify=df['default']` trong train_test_split

**Q: Missing values quá nhiều sau merge?**
A: Verify join keys (district_code, industry_code) match giữa datasets
