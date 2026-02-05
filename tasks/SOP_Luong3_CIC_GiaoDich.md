# Hướng Dẫn Chi Tiết: Luồng 3 - Sinh Dữ Liệu Synthetic CIC & Giao Dịch

> **Thời gian**: 1 tuần
> **Người thực hiện**: Thành viên C
> **Output**: `synthetic_cic_10k.csv` + `synthetic_transaction_10k.csv`

---

## 📋 CÔNG VIỆC 3.1: ĐẶC TRƯNG CIC (5 ngày)

### Ngày 1: Thiết Kế Schema

#### Bước 1.1: Định Nghĩa 8 Features CIC

**File**: `schemas/cic_schema.yaml`

```yaml
cic_features:
  - name: cic_score
    description: Điểm tín dụng CIC (300-850)
    distribution: normal
    params:
      mean: 650
      std: 50
    range: [300, 850]
    
  - name: num_active_loans
    description: Số khoản vay đang có
    distribution: poisson
    params:
      lambda: 1.5
    range: [0, 10]
    
  - name: total_outstanding_debt
    description: Tổng dư nợ hiện tại (VNĐ)
    distribution: lognormal
    params:
      mean: 18.5  # log scale: ~100M VNĐ
      std: 1.2
    range: [0, 1e12]
    correlation:
      revenue: 0.6  # Tương quan với doanh thu
    
  - name: max_past_due_days
    description: Số ngày quá hạn tối đa trong 12 tháng
    distribution: conditional
    logic: |
      if random() < 0.95:  # 95% không quá hạn
        return 0
      else:
        return random_choice([30, 60, 90, 120])
    
  - name: num_past_due_30d
    description: Số lần quá hạn > 30 ngày (12 tháng)
    distribution: conditional
    logic: |
      if max_past_due_days == 0:
        return 0
      elif max_past_due_days >= 30:
        return poisson(lambda=1)
    
  - name: num_past_due_90d
    description: Số lần quá hạn > 90 ngày (12 tháng)
    distribution: conditional
    logic: |
      if max_past_due_days < 90:
        return 0
      else:
        return binomial(n=num_past_due_30d, p=0.3)
    
  - name: debt_burden_ratio
    description: Dư nợ / Doanh thu tháng
    distribution: calculated
    formula: total_outstanding_debt / (revenue / 12)
    
  - name: credit_history_length
    description: Số năm có quan hệ tín dụng
    distribution: lognormal
    params:
      mean: 1.1  # log scale: ~3 năm
      std: 0.6
    range: [0, 20]
```

#### Bước 1.2: Validate Logic

**Kiểm tra**:
- [ ] Tương quan hợp lý (debt vs revenue)
- [ ] Conditional logic đúng (past_due phụ thuộc max_past_due)
- [ ] Ranges realistic

---

### Ngày 2-3: Build Generator Script

**File**: `scripts/generate_cic.py`

```python
"""
Sinh dữ liệu CIC synthetic
"""
import numpy as np
import pandas as pd
from scipy import stats

class CICGenerator:
    def __init__(self, n_samples=10000):
        self.n_samples = n_samples
        
    def generate_cic_score(self):
        """Feature 1: CIC Score"""
        scores = np.random.normal(loc=650, scale=50, size=self.n_samples)
        scores = np.clip(scores, 300, 850)  # Giới hạn range
        return scores
    
    def generate_num_active_loans(self):
        """Feature 2: Số khoản vay"""
        loans = np.random.poisson(lam=1.5, size=self.n_samples)
        loans = np.clip(loans, 0, 10)
        return loans
    
    def generate_outstanding_debt(self, revenue):
        """Feature 3: Dư nợ (tương quan với revenue)"""
        # Base: lognormal
        base_debt = np.random.lognormal(mean=18.5, sigma=1.2, size=self.n_samples)
        
        # Adjust để correlation với revenue
        # Công ty revenue cao -> có thể vay nhiều hơn
        correlation_factor = revenue / revenue.mean()
        
        debt = base_debt * (0.4 + 0.6 * correlation_factor)
        return debt
    
    def generate_past_due(self):
        """Features 4-6: Quá hạn"""
        max_past_due = []
        num_past_due_30 = []
        num_past_due_90 = []
        
        for i in range(self.n_samples):
            # 95% không quá hạn
            if np.random.random() < 0.95:
                max_past_due.append(0)
                num_past_due_30.append(0)
                num_past_due_90.append(0)
            else:
                # Có quá hạn
                max_days = np.random.choice([30, 60, 90, 120, 180])
                max_past_due.append(max_days)
                
                # Số lần quá hạn > 30 ngày
                if max_days >= 30:
                    count_30 = np.random.poisson(lam=1)
                    num_past_due_30.append(count_30)
                    
                    # Số lần quá hạn > 90 ngày
                    if max_days >= 90:
                        count_90 = np.random.binomial(n=count_30, p=0.3)
                        num_past_due_90.append(count_90)
                    else:
                        num_past_due_90.append(0)
                else:
                    num_past_due_30.append(0)
                    num_past_due_90.append(0)
        
        return (
            np.array(max_past_due),
            np.array(num_past_due_30),
            np.array(num_past_due_90)
        )
    
    def generate_credit_history(self):
        """Feature 8: Độ dài lịch sử"""
        history = np.random.lognormal(mean=1.1, sigma=0.6, size=self.n_samples)
        history = np.clip(history, 0, 20)
        return history
    
    def generate_all(self, revenue):
        """
        Sinh tất cả features
        
        Args:
            revenue: Array doanh thu từ Luồng 1 (để correlation)
        """
        data = {}
        
        # Generate independent features
        data['cic_score'] = self.generate_cic_score()
        data['num_active_loans'] = self.generate_num_active_loans()
        data['credit_history_length'] = self.generate_credit_history()
        
        # Generate correlated features
        data['total_outstanding_debt'] = self.generate_outstanding_debt(revenue)
        
        # Generate conditional features
        (data['max_past_due_days'],
         data['num_past_due_30d'],
         data['num_past_due_90d']) = self.generate_past_due()
        
        # Calculate derived feature
        data['debt_burden_ratio'] = data['total_outstanding_debt'] / (revenue / 12)
        
        return pd.DataFrame(data)

# Usage
if __name__ == "__main__":
    # Load revenue từ Luồng 1 (hoặc generate tạm)
    revenue = np.random.lognormal(mean=18.5, sigma=1.0, size=10000)
    
    generator = CICGenerator(n_samples=10000)
    df_cic = generator.generate_all(revenue)
    
    print(df_cic.head())
    print(df_cic.describe())
    
    # Save
    df_cic.to_csv('data/synthetic/cic_10k.csv', index=False)
```

---

### Ngày 4: QA & Validation

**File**: `scripts/validate_cic.py`

```python
"""
Kiểm tra chất lượng dữ liệu CIC
"""
import pandas as pd

df = pd.read_csv('data/synthetic/cic_10k.csv')

# Check 1: Ranges
assert df['cic_score'].min() >= 300
assert df['cic_score'].max() <= 850
print("✓ CIC score range OK")

# Check 2: Conditional logic
# Nếu max_past_due = 0 thì num_past_due_30d phải = 0
invalid = df[(df['max_past_due_days'] == 0) & (df['num_past_due_30d'] > 0)]
assert len(invalid) == 0, f"Logic error: {len(invalid)} invalid rows"
print("✓ Past due logic OK")

# Check 3: Default rate estimate
# Khoảng 5% có quá hạn > 90 ngày
default_rate = (df['num_past_due_90d'] > 0).mean()
print(f"Default rate estimate: {default_rate:.2%}")
assert 0.03 < default_rate < 0.07, "Default rate out of range"

# Check 4: Correlations
from scipy.stats import pearsonr

# debt_burden_ratio nên correlation với default
df['is_default'] = (df['num_past_due_90d'] > 0).astype(int)
corr, _ = pearsonr(df['debt_burden_ratio'], df['is_default'])
print(f"Correlation debt_burden vs default: {corr:.3f}")
assert corr > 0.1, "Weak correlation"

print("\n✅ All CIC validations passed!")
```

**Run**:
```bash
python scripts/validate_cic.py
```

---

## 📋 CÔNG VIỆC 3.2: ĐẶC TRƯNG GIAO DỊCH (5 ngày - SONG SONG)

### Ngày 1: Thiết Kế Schema

**File**: `schemas/transaction_schema.yaml`

```yaml
transaction_features:
  - name: avg_daily_balance
    description: Số dư bình quân ngày (3 tháng)
    distribution: lognormal
    correlation:
      revenue: 0.7  # Công ty lớn -> số dư cao
    params:
      base_mean: 16.5  # ~10M VNĐ
      base_std: 1.5
      
  - name: min_balance_3m
    description: Số dư thấp nhất 3 tháng
    distribution: calculated
    formula: avg_daily_balance * random_uniform(0.3, 0.7)
    
  - name: cash_flow_volatility
    description: Độ biến động dòng tiền (std)
    distribution: lognormal
    params:
      mean: 15.0
      std: 1.0
      
  - name: avg_monthly_deposits
    description: Tiền gửi TB/tháng
    distribution: calculated
    formula: revenue * 1.1  # Hơi cao hơn revenue
    
  - name: avg_monthly_withdrawals
    description: Tiền rút TB/tháng
    distribution: calculated
    formula: revenue * random_uniform(0.95, 1.05)
    
  - name: net_cash_flow
    description: Dòng tiền ròng
    distribution: calculated
    formula: avg_monthly_deposits - avg_monthly_withdrawals
    
  - name: num_transactions_3m
    description: Số giao dịch 3 tháng
    distribution: poisson
    params:
      lambda_base: 50
    adjustment: scale by revenue (larger = more transactions)
    
  - name: overdraft_count
    description: Số lần thấu chi
    distribution: binomial
    params:
      n: 90  # 90 ngày
      p: 0.05  # 5% ngày có thấu chi
```

---

### Ngày 2-3: Build Generator

**File**: `scripts/generate_transaction.py`

```python
"""
Sinh dữ liệu Transaction synthetic
"""
import numpy as np
import pandas as pd

class TransactionGenerator:
    def __init__(self, n_samples=10000):
        self.n_samples = n_samples
    
    def generate_balances(self, revenue):
        """Generate số dư"""
        # avg_daily_balance tương quan với revenue
        base_balance = np.random.lognormal(mean=16.5, sigma=1.5, size=self.n_samples)
        
        # Scale theo revenue
        revenue_factor = revenue / revenue.mean()
        avg_balance = base_balance * revenue_factor
        
        # min_balance = 30-70% của avg
        min_balance = avg_balance * np.random.uniform(0.3, 0.7, size=self.n_samples)
        
        return avg_balance, min_balance
    
    def generate_cash_flow(self, revenue):
        """Generate cash flow features"""
        # Volatility
        volatility = np.random.lognormal(mean=15.0, sigma=1.0, size=self.n_samples)
        
        # Monthly deposits (hơi cao hơn revenue)
        deposits = revenue * np.random.uniform(1.05, 1.15, size=self.n_samples)
        
        # Monthly withdrawals (gần bằng revenue)
        withdrawals = revenue * np.random.uniform(0.95, 1.05, size=self.n_samples)
        
        # Net cash flow
        net_flow = deposits - withdrawals
        
        return volatility, deposits, withdrawals, net_flow
    
    def generate_transactions(self, revenue):
        """Generate số lượng giao dịch"""
        # Base: 50 transactions/3 months
        # Scale theo revenue
        revenue_factor = np.sqrt(revenue / revenue.mean())  # Sqrt để không quá extreme
        
        num_trans = np.random.poisson(lam=50 * revenue_factor)
        return num_trans
    
    def generate_overdraft(self):
        """Generate số lần thấu chi"""
        # 5% khả năng thấu chi mỗi ngày
        overdraft_counts = np.random.binomial(n=90, p=0.05, size=self.n_samples)
        return overdraft_counts
    
    def generate_all(self, revenue):
        """Sinh tất cả features"""
        data = {}
        
        # Balances
        (data['avg_daily_balance'],
         data['min_balance_3m']) = self.generate_balances(revenue)
        
        # Cash flows
        (data['cash_flow_volatility'],
         data['avg_monthly_deposits'],
         data['avg_monthly_withdrawals'],
         data['net_cash_flow']) = self.generate_cash_flow(revenue)
        
        # Transactions
        data['num_transactions_3m'] = self.generate_transactions(revenue)
        
        # Overdraft
        data['overdraft_count'] = self.generate_overdraft()
        
        return pd.DataFrame(data)

# Usage
if __name__ == "__main__":
    revenue = np.random.lognormal(mean=18.5, sigma=1.0, size=10000)
    
    generator = TransactionGenerator(n_samples=10000)
    df_trans = generator.generate_all(revenue)
    
    print(df_trans.head())
    print(df_trans.describe())
    
    df_trans.to_csv('data/synthetic/transaction_10k.csv', index=False)
```

---

### Ngày 4-5: Merge & Final QA

**File**: `scripts/merge_cic_transaction.py`

```python
"""
Gộp CIC + Transaction
"""
import pandas as pd

# Load
df_cic = pd.read_csv('data/synthetic/cic_10k.csv')
df_trans = pd.read_csv('data/synthetic/transaction_10k.csv')

# Merge (cùng index = cùng company)
df_merged = pd.concat([df_cic, df_trans], axis=1)

print(f"Merged shape: {df_merged.shape}")
print(f"Features: {df_merged.columns.tolist()}")

# Save
df_merged.to_csv('data/final/synthetic_cic_transaction_10k.csv', index=False)

print("✅ CIC + Transaction merged!")
```

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] `synthetic_cic_10k.csv` - 8 features CIC
- [ ] `synthetic_transaction_10k.csv` - 8 features giao dịch
- [ ] `synthetic_cic_transaction_10k.csv` - 16 features merged
- [ ] Validation passed (logic, correlations, ranges)
- [ ] Documentation: Schema YAML files

---

## 🆘 TROUBLESHOOTING

**Q: Correlation không đúng?**
A: Adjust công thức scaling, thử linear combination thay vì pure multiplication

**Q: Default rate quá cao/thấp?**
A: Điều chỉnh `p` trong conditional logic của past_due

**Q: Dữ liệu có outliers cực đoan?**
A: Thêm `np.clip()` để giới hạn ranges
