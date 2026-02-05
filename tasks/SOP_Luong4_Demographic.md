# Hướng Dẫn Chi Tiết: Luồng 4 - Sinh Dữ Liệu Demographic

> **Thời gian**: 1 tuần
> **Người thực hiện**: Thành viên D (hoặc A nếu 3 người)
> **Output**: `synthetic_demographic_10k.csv`

---

## 📋 CÔNG VIỆC 4.1: ĐẶC TRƯNG PHÂN LOẠI (3 ngày)

### Ngày 1: Thiết Kế Schema

**File**: `schemas/demographic_schema.yaml`

```yaml
demographic_features:
  categorical:
    - name: business_type
      description: Loại hình doanh nghiệp
      distribution: categorical
      probabilities:
        TNHH: 0.60
        CP: 0.30
        Tư nhân: 0.10
      
    - name: industry_code
      description: Mã ngành VSIC
      distribution: categorical
      source: industry_risk.csv  # Từ Luồng 2
      
    - name: district_code
      description: Mã quận/huyện
      distribution: categorical
      probabilities: uniform  # Hoặc weighted theo population
      range: [1, 24]
      
    - name: business_zone
      description: Khu vực kinh doanh
      distribution: calculated
      logic: |
        if district_code in [1, 3, 4, 5, 10, 11]:
          return "CBD"
        else:
          return "Ngoại thành"
      
    - name: owner_education
      description: Trình độ học vấn chủ DN
      distribution: categorical
      probabilities:
        Đại học: 0.50
        Cao đẳng: 0.30
        Trung học phổ thông: 0.15
        Sau đại học: 0.05
  
  numeric:
    - name: registered_capital
      description: Vốn điều lệ (VNĐ)
      distribution: lognormal
      params:
        mean: 19.0  # ~50M-500M VNĐ
        std: 1.0
      range: [10000000, 10000000000]  # 10M-10B
      
    - name: owner_age
      description: Tuổi chủ doanh nghiệp
      distribution: normal
      params:
        mean: 42
        std: 8
      range: [25, 70]
      
    - name: owner_experience
      description: Số năm kinh nghiệm
      distribution: lognormal
      params:
        mean: 2.5  # ~12 năm
        std: 0.7
      range: [0, 40]
```

---

### Ngày 2: Generate Categorical Features

**File**: `scripts/generate_demographic_categorical.py`

```python
"""
Sinh categorical features
"""
import numpy as np
import pandas as pd

class DemographicCategorical:
    def __init__(self, n_samples=10000):
        self.n_samples = n_samples
    
    def generate_business_type(self):
        """Loại hình DN"""
        types = ['TNHH', 'CP', 'Tư nhân']
        probs = [0.60, 0.30, 0.10]
        
        return np.random.choice(types, size=self.n_samples, p=probs)
    
    def generate_industry_code(self):
        """Mã ngành - Load từ Luồng 2"""
        # Load industry_risk.csv
        df_industry = pd.read_csv('data/lookup_tables/industry_risk.csv')
        
        # Lấy list codes
        codes = df_industry['vsic_code'].tolist()
        
        # Random choice (có thể weight theo popularity sau)
        return np.random.choice(codes, size=self.n_samples)
    
    def generate_district_code(self):
        """Mã quận - Uniform hoặc weighted"""
        # Option 1: Uniform
        districts = list(range(1, 25))
        return np.random.choice(districts, size=self.n_samples)
        
        # Option 2: Weighted theo population (nếu có data)
        # weights = [0.08, 0.12, ...]  # 24 weights
        # return np.random.choice(districts, size=self.n_samples, p=weights)
    
    def generate_business_zone(self, district_codes):
        """Khu vực KD - Calculated từ district"""
        cbd_districts = [1, 3, 4, 5, 10, 11]
        
        zones = []
        for district in district_codes:
            if district in cbd_districts:
                zones.append('CBD')
            else:
                zones.append('Ngoại thành')
        
        return zones
    
    def generate_owner_education(self):
        """Trình độ học vấn"""
        levels = ['Đại học', 'Cao đẳng', 'Trung học phổ thông', 'Sau đại học']
        probs = [0.50, 0.30, 0.15, 0.05]
        
        return np.random.choice(levels, size=self.n_samples, p=probs)
    
    def generate_all(self):
        """Sinh tất cả categorical"""
        data = {}
        
        data['business_type'] = self.generate_business_type()
        data['industry_code'] = self.generate_industry_code()
        data['district_code'] = self.generate_district_code()
        data['business_zone'] = self.generate_business_zone(data['district_code'])
        data['owner_education'] = self.generate_owner_education()
        
        return pd.DataFrame(data)

# Test
generator = DemographicCategorical(n_samples=10000)
df_cat = generator.generate_all()

print(df_cat.head())
print("\nDistributions:")
print(df_cat['business_type'].value_counts(normalize=True))
print(df_cat['business_zone'].value_counts(normalize=True))

# Save
df_cat.to_csv('data/temp/demographic_categorical.csv', index=False)
```

---

### Ngày 3: Generate Numeric Features

**File**: `scripts/generate_demographic_numeric.py`

```python
"""
Sinh numeric features
"""
import numpy as np
import pandas as pd

class DemographicNumeric:
    def __init__(self, n_samples=10000):
        self.n_samples = n_samples
    
    def generate_registered_capital(self):
        """Vốn điều lệ"""
        capital = np.random.lognormal(mean=19.0, sigma=1.0, size=self.n_samples)
        
        # Clip range
        capital = np.clip(capital, 10_000_000, 10_000_000_000)
        
        return capital
    
    def generate_owner_age(self):
        """Tuổi chủ DN"""
        age = np.random.normal(loc=42, scale=8, size=self.n_samples)
        age = np.clip(age, 25, 70)
        age = np.round(age).astype(int)
        
        return age
    
    def generate_owner_experience(self):
        """Số năm kinh nghiệm"""
        exp = np.random.lognormal(mean=2.5, sigma=0.7, size=self.n_samples)
        exp = np.clip(exp, 0, 40)
        exp = np.round(exp).astype(int)
        
        return exp
    
    def generate_all(self):
        """Sinh tất cả numeric"""
        data = {}
        
        data['registered_capital'] = self.generate_registered_capital()
        data['owner_age'] = self.generate_owner_age()
        data['owner_experience'] = self.generate_owner_experience()
        
        return pd.DataFrame(data)

# Test
generator = DemographicNumeric(n_samples=10000)
df_num = generator.generate_all()

print(df_num.describe())

# Save
df_num.to_csv('data/temp/demographic_numeric.csv', index=False)
```

---

## 📋 CÔNG VIỆC 4.2: TÀI SẢN ĐẢM BẢO (3 ngày)

### Ngày 1-2: Generate Collateral Features

**File**: `scripts/generate_collateral.py`

```python
"""
Sinh features tài sản đảm bảo
"""
import numpy as np
import pandas as pd

class CollateralGenerator:
    def __init__(self, n_samples=10000):
        self.n_samples = n_samples
    
    def generate_has_collateral(self):
        """Binary: Có TSĐB hay không"""
        # 60% có TSĐB
        has = np.random.binomial(n=1, p=0.6, size=self.n_samples)
        return has.astype(bool)
    
    def generate_collateral_value(self, has_collateral):
        """Giá trị TSĐB (conditional)"""
        values = np.zeros(self.n_samples)
        
        # Chỉ sinh giá trị cho những DN có TSĐB
        num_with_collateral = has_collateral.sum()
        
        if num_with_collateral > 0:
            # Lognormal: ~500M - 5B VNĐ
            values_positive = np.random.lognormal(
                mean=20.5,
                sigma=1.0,
                size=num_with_collateral
            )
            
            # Clip
            values_positive = np.clip(values_positive, 100_000_000, 50_000_000_000)
            
            # Gán vào vị trí có collateral
            values[has_collateral] = values_positive
        
        return values
    
    def generate_collateral_location(self, district_codes, has_collateral):
        """Vị trí TSĐB (quận/huyện)"""
        locations = []
        
        for i, has in enumerate(has_collateral):
            if not has:
                locations.append(None)
            else:
                # 70% cùng quận, 30% quận láng giềng
                if np.random.random() < 0.7:
                    locations.append(district_codes[i])
                else:
                    # Random quận khác
                    other_districts = [d for d in range(1, 25) if d != district_codes[i]]
                    locations.append(np.random.choice(other_districts))
        
        return locations
    
    def generate_loan_to_value(self, has_collateral):
        """LTV - Loan to Value ratio"""
        ltv = np.zeros(self.n_samples)
        
        # Chỉ có LTV nếu có TSĐB
        num_with_collateral = has_collateral.sum()
        
        if num_with_collateral > 0:
            # Uniform 50-80%
            ltv_values = np.random.uniform(0.5, 0.8, size=num_with_collateral)
            ltv[has_collateral] = ltv_values
        
        return ltv
    
    def generate_all(self, district_codes):
        """Sinh tất cả collateral features"""
        data = {}
        
        # Binary
        data['has_collateral'] = self.generate_has_collateral()
        
        # Conditional features
        data['collateral_value'] = self.generate_collateral_value(data['has_collateral'])
        data['collateral_location'] = self.generate_collateral_location(
            district_codes, 
            data['has_collateral']
        )
        data['loan_to_value'] = self.generate_loan_to_value(data['has_collateral'])
        
        return pd.DataFrame(data)

# Test
# Cần district_codes từ categorical
df_cat = pd.read_csv('data/temp/demographic_categorical.csv')

generator = CollateralGenerator(n_samples=10000)
df_collateral = generator.generate_all(df_cat['district_code'].values)

print(df_collateral.head(20))
print(f"\nHas collateral: {df_collateral['has_collateral'].mean():.2%}")

# Save
df_collateral.to_csv('data/temp/demographic_collateral.csv', index=False)
```

---

### Ngày 3: Merge All Demographic

**File**: `scripts/merge_demographic.py`

```python
"""
Gộp tất cả demographic features
"""
import pandas as pd

# Load các phần
df_cat = pd.read_csv('data/temp/demographic_categorical.csv')
df_num = pd.read_csv('data/temp/demographic_numeric.csv')
df_collateral = pd.read_csv('data/temp/demographic_collateral.csv')

# Merge (cùng index)
df_demographic = pd.concat([df_cat, df_num, df_collateral], axis=1)

print(f"Final shape: {df_demographic.shape}")
print(f"Columns: {df_demographic.columns.tolist()}")

# QA
print("\n=== QA Checks ===")
print(f"Missing values:\n{df_demographic.isnull().sum()}")
print(f"\nBusiness type distribution:\n{df_demographic['business_type'].value_counts(normalize=True)}")

# Save final
df_demographic.to_csv('data/final/synthetic_demographic_10k.csv', index=False)

print("\n✅ Demographic data generated!")
```

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] `demographic_categorical.csv` - 5 features phân loại
- [ ] `demographic_numeric.csv` - 3 features số
- [ ] `demographic_collateral.csv` - 4 features TSĐB
- [ ] `synthetic_demographic_10k.csv` - 12 features merged
- [ ] QA: Phân phối hợp lý, no missing (except collateral_location)

---

## 🆘 TROUBLESHOOTING

**Q: Null values ở collateral_location?**
A: OK, đây là expected (DN không có TSĐB thì không có location)

**Q: Owner experience > owner age?**
A: Add validation check, clip experience = min(experience, age - 20)

**Q: District codes không match với Luồng 2?**
A: Verify `industry_risk.csv` và `district_lookup.csv` đã tồn tại và đúng format
