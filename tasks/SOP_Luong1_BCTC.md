# Hướng Dẫn Chi Tiết: Luồng 1 - Thu Thập BCTC Từ Công Ty Niêm Yết

> **Thời gian**: 3 tuần
> **Người thực hiện**: Thành viên A
> **Output**: `synthetic_financial_10k.csv` + `distribution_params.json`

---

## 📋 TUẦN 1: DOWNLOAD BCTC

### Bước 1.1: Nghiên Cứu Danh Sách Công Ty UPCOM (2 giờ)

**Mục tiêu**: Lấy danh sách 150-200 công ty UPCOM phù hợp với tiêu chí SME

#### Công việc cụ thể:

1. **Truy cập website HNX**
   - URL: https://www.hnx.vn/vi-vn/cong-ty-dai-chung.html
   - Tìm mục "Danh sách công ty đại chúng"
   - Download file Excel danh sách công ty UPCOM

2. **Lọc công ty theo tiêu chí**
   
   Mở Excel, áp dụng filter:
   ```
   Tiêu chí lọc:
   - Doanh thu năm gần nhất < 200 tỷ VNĐ
   - Tổng tài sản < 100 tỷ VNĐ
   - Trụ sở tại TP.HCM (check cột "Địa chỉ")
   - Đang hoạt động (status = "Active")
   ```

3. **Tạo file danh sách cuối**
   
   Tạo file: `upcom_company_list.csv`
   
   Các cột cần có:
   ```csv
   ticker,company_name,address,revenue_2023,total_assets,employees,industry_code
   ABC,Công ty ABC,TP.HCM,150000000000,80000000000,120,46
   ```

4. **Phân nhóm công ty**
   
   Chia thành 4 batch, mỗi batch ~50 công ty:
   - `batch1_tickers.txt` - 50 công ty đầu
   - `batch2_tickers.txt` - 50 công ty tiếp
   - `batch3_tickers.txt` - 50 công ty tiếp
   - `batch4_tickers.txt` - 50 công ty cuối

**Checkpoint**: File `upcom_company_list.csv` với 150-200 dòng

---

### Bước 1.2: Setup Môi Trường & Script Download (4 giờ)

#### 1.2.1: Cài đặt Python Libraries

```bash
# Tạo virtual environment
conda create -n bctc_download python=3.10
conda activate bctc_download

# Cài đặt thư viện
pip install pandas openpyxl requests beautifulsoup4 selenium
pip install webdriver-manager  # Tự động quản lý Chrome driver
```

#### 1.2.2: Tạo Script Download VietStock

**File**: `scripts/download_vietstock.py`

```python
"""
Script download BCTC từ VietStock
"""
import pandas as pd
import requests
import time
import os
from datetime import datetime

class VietStockDownloader:
    def __init__(self, output_dir='data/raw/bctc'):
        self.output_dir = output_dir
        os.makedirs(output_dir, exist_ok=True)
        self.base_url = "https://finance.vietstock.vn"
        
    def download_company_bctc(self, ticker, years=[2020, 2021, 2022, 2023, 2024]):
        """
        Download BCTC cho 1 công ty, nhiều năm
        """
        print(f"Downloading {ticker}...")
        company_dir = f"{self.output_dir}/{ticker}"
        os.makedirs(company_dir, exist_ok=True)
        
        for year in years:
            try:
                # URL format VietStock
                url = f"{self.base_url}/{ticker}/tai-chinh/bao-cao-tai-chinh/?year={year}"
                
                # Gửi request
                response = requests.get(url, timeout=30)
                
                if response.status_code == 200:
                    # Lưu file
                    filename = f"{company_dir}/{ticker}_{year}.html"
                    with open(filename, 'w', encoding='utf-8') as f:
                        f.write(response.text)
                    print(f"  ✓ {ticker} {year}")
                else:
                    print(f"  ✗ {ticker} {year} - Failed")
                
                # Delay để tránh bị ban
                time.sleep(2)
                
            except Exception as e:
                print(f"  ✗ {ticker} {year} - Error: {e}")
                
    def download_batch(self, ticker_file):
        """
        Download batch từ file ticker list
        """
        with open(ticker_file, 'r') as f:
            tickers = [line.strip() for line in f if line.strip()]
        
        for ticker in tickers:
            self.download_company_bctc(ticker)
            time.sleep(3)  # Delay giữa các công ty

# Sử dụng
if __name__ == "__main__":
    downloader = VietStockDownloader(output_dir='data/raw/bctc/batch1')
    downloader.download_batch('batch1_tickers.txt')
```

#### 1.2.3: Script Alternative - CafeF

**File**: `scripts/download_cafef.py`

```python
"""
Backup script download từ CafeF nếu VietStock fail
"""
import requests
import json

def download_cafef(ticker, year):
    """
    Download từ CafeF API
    """
    url = f"https://s.cafef.vn/Ajax/CongTy/BaoCaoTaiChinh.aspx"
    params = {
        'symbol': ticker,
        'year': year,
        'type': 1  # 1: BCTC năm, 2: BCTC quý
    }
    
    response = requests.get(url, params=params)
    if response.status_code == 200:
        data = response.json()
        # Xử lý và lưu data
        return data
    return None
```

---

### Bước 1.3: Download BCTC (3-4 ngày)

#### Chiến lược song song:

**Nếu 1 người**:
- Chạy tuần tự 4 batch, mỗi batch ~1 ngày
- Sáng: Setup + chạy batch
- Chiều: Monitoring + xử lý lỗi

**Nếu 4 người**:
- Mỗi người chạy 1 batch song song
- Hoàn thành trong 1 ngày

#### Checklist thực hiện:

**Batch 1** (50 công ty)
```bash
# Terminal 1
python scripts/download_vietstock.py --batch batch1_tickers.txt --output data/raw/bctc/batch1
```

- [ ] Chạy script
- [ ] Monitor progress (check log file)
- [ ] Kiểm tra số file downloaded: 50 công ty × 5 năm = 250 files
- [ ] Ghi log các ticker failed vào `batch1_failed.txt`
- [ ] Retry failed tickers (nếu có)

**Batch 2** (50 công ty)
```bash
# Terminal 2 (song song)
python scripts/download_vietstock.py --batch batch2_tickers.txt --output data/raw/bctc/batch2
```

- [ ] Tương tự batch 1

**Batch 3 & 4**: Tương tự

#### Xử lý lỗi thường gặp:

| Lỗi | Nguyên nhân | Giải pháp |
|------|-------------|-----------|
| HTTP 429 | Rate limiting | Tăng delay lên 5s, retry sau 1 giờ |
| HTTP 404 | Ticker không tồn tại | Bỏ qua, ghi log |
| Timeout | Mạng chậm | Tăng timeout lên 60s, retry |
| Empty response | Công ty chưa có BCTC năm đó | Bỏ qua, ghi log |

**Checkpoint Tuần 1**:
```
data/raw/bctc/
├── batch1/
│   ├── ABC/
│   │   ├── ABC_2020.html
│   │   ├── ABC_2021.html
│   │   ├── ...
│   ├── XYZ/
│   │   └── ...
├── batch2/
├── batch3/
└── batch4/

Total: ~800-1000 files (200 công ty × 5 năm, chấp nhận 80% completeness)
```

---

## 📋 TUẦN 2: EXTRACT FEATURES

### Bước 2.1: Xây Dựng Parser (1 ngày)

#### 2.1.1: Phân tích cấu trúc HTML VietStock

**Mở 1 file mẫu**: `data/raw/bctc/batch1/ABC/ABC_2023.html`

Quan sát cấu trúc:
```html
<table class="table-balance-sheet">
  <tr>
    <td>Tổng tài sản</td>
    <td class="text-right">1,234,567</td>
  </tr>
  <tr>
    <td>Nợ phải trả</td>
    <td class="text-right">567,890</td>
  </tr>
</table>
```

#### 2.1.2: Viết Parser

**File**: `scripts/parse_bctc.py`

```python
"""
Parser BCTC từ HTML
"""
from bs4 import BeautifulSoup
import pandas as pd
import re

class BCTCParser:
    def __init__(self):
        self.field_mapping = {
            'Tổng tài sản': 'total_assets',
            'Tài sản ngắn hạn': 'current_assets',
            'Nợ phải trả': 'total_liabilities',
            'Nợ ngắn hạn': 'current_liabilities',
            'Vốn chủ sở hữu': 'equity',
            'Doanh thu': 'revenue',
            'Lợi nhuận sau thuế': 'net_profit',
            'Giá vốn hàng bán': 'cogs',
            'Hàng tồn kho': 'inventory',
            'Khoản phải thu': 'receivables'
        }
        
    def clean_number(self, text):
        """
        Chuyển '1,234,567' -> 1234567
        """
        if not text:
            return None
        # Xóa dấu phẩy, đơn vị
        clean = re.sub(r'[,\s]', '', text)
        try:
            return float(clean)
        except:
            return None
    
    def parse_html(self, html_file):
        """
        Parse 1 file HTML
        """
        with open(html_file, 'r', encoding='utf-8') as f:
            soup = BeautifulSoup(f.read(), 'html.parser')
        
        data = {}
        
        # Tìm table balance sheet
        table = soup.find('table', class_='table-balance-sheet')
        if table:
            rows = table.find_all('tr')
            for row in rows:
                cells = row.find_all('td')
                if len(cells) >= 2:
                    label = cells[0].text.strip()
                    value = cells[1].text.strip()
                    
                    if label in self.field_mapping:
                        field_name = self.field_mapping[label]
                        data[field_name] = self.clean_number(value)
        
        return data
    
    def parse_company(self, ticker, company_dir):
        """
        Parse tất cả năm của 1 công ty
        """
        results = []
        
        for year in [2020, 2021, 2022, 2023, 2024]:
            file_path = f"{company_dir}/{ticker}_{year}.html"
            
            if os.path.exists(file_path):
                data = self.parse_html(file_path)
                data['ticker'] = ticker
                data['year'] = year
                results.append(data)
        
        return pd.DataFrame(results)

# Test
parser = BCTCParser()
df = parser.parse_company('ABC', 'data/raw/bctc/batch1/ABC')
print(df)
```

#### 2.1.3: Test Parser

```bash
# Chạy test với 5 công ty mẫu
python scripts/test_parser.py
```

**Expected output**:
```
   ticker  year  total_assets  revenue  net_profit  ...
0  ABC    2020  1000000000   500000000  50000000
1  ABC    2021  1200000000   600000000  60000000
...
```

---

### Bước 2.2: Extract Tất Cả Companies (2-3 ngày)

**File**: `scripts/extract_all.py`

```python
"""
Extract tất cả BCTC thành 1 file CSV
"""
import os
from parse_bctc import BCTCParser
import pandas as pd

def extract_batch(batch_dir, output_file):
    parser = BCTCParser()
    all_data = []
    
    # List tất cả ticker trong batch
    tickers = [d for d in os.listdir(batch_dir) if os.path.isdir(f"{batch_dir}/{d}")]
    
    for ticker in tickers:
        print(f"Processing {ticker}...")
        company_dir = f"{batch_dir}/{ticker}"
        df = parser.parse_company(ticker, company_dir)
        all_data.append(df)
    
    # Combine all
    result = pd.concat(all_data, ignore_index=True)
    
    # Save
    result.to_csv(output_file, index=False)
    print(f"Saved to {output_file}")
    
    return result

# Run cho tất cả batches
for i in range(1, 5):
    extract_batch(
        f"data/raw/bctc/batch{i}",
        f"data/processed/batch{i}_raw.csv"
    )
```

**Chạy**:
```bash
python scripts/extract_all.py
```

**Checkpoint**: 4 files CSV với raw data

---

### Bước 2.3: Tính 12 Chỉ Số Tài Chính (1 ngày)

**File**: `scripts/calculate_ratios.py`

```python
"""
Tính 12 financial ratios
"""
import pandas as pd
import numpy as np

def calculate_financial_ratios(df):
    """
    Input: DataFrame với raw fields
    Output: DataFrame với 12 ratios
    """
    result = df.copy()
    
    # 1. ROA (Return on Assets)
    result['roa'] = result['net_profit'] / result['total_assets']
    
    # 2. ROE (Return on Equity)  
    result['roe'] = result['net_profit'] / result['equity']
    
    # 3. Profit Margin
    result['profit_margin'] = result['net_profit'] / result['revenue']
    
    # 4. Revenue Growth (so với năm trước)
    result = result.sort_values(['ticker', 'year'])
    result['revenue_prev'] = result.groupby('ticker')['revenue'].shift(1)
    result['revenue_growth'] = (result['revenue'] - result['revenue_prev']) / result['revenue_prev']
    
    # 5. Current Ratio
    result['current_ratio'] = result['current_assets'] / result['current_liabilities']
    
    # 6. Quick Ratio
    result['quick_ratio'] = (result['current_assets'] - result['inventory']) / result['current_liabilities']
    
    # 7. Debt to Equity
    result['debt_to_equity'] = result['total_liabilities'] / result['equity']
    
    # 8. Debt to Assets
    result['debt_to_asset'] = result['total_liabilities'] / result['total_assets']
    
    # 9-12: Turnover ratios
    result['asset_turnover'] = result['revenue'] / result['total_assets']
    result['inventory_turnover'] = result['cogs'] / result['inventory']
    result['receivable_turnover'] = result['revenue'] / result['receivables']
    
    # DSCR cần EBITDA - tạm bỏ qua hoặc estimate
    result['dscr'] = np.nan  # Sẽ estimate sau
    
    return result

# Apply
for i in range(1, 5):
    df = pd.read_csv(f"data/processed/batch{i}_raw.csv")
    df_ratios = calculate_financial_ratios(df)
    df_ratios.to_csv(f"data/processed/batch{i}_ratios.csv", index=False)
```

**Output**: `financial_features_real.csv` với 12 features

---

## 📋 TUẦN 3: ANALYZE & GENERATE SYNTHETIC

### Bước 3.1: EDA (1 ngày)

**File**: `notebooks/eda_financial.ipynb`

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

# Load tất cả data
dfs = []
for i in range(1, 5):
    df = pd.read_csv(f"../data/processed/batch{i}_ratios.csv")
    dfs.append(df)

df_all = pd.concat(dfs, ignore_index=True)

# EDA
df_all.describe()

# Visualize distributions
features = ['roa', 'roe', 'current_ratio', 'debt_to_equity']
for feat in features:
    plt.figure()
    df_all[feat].dropna().hist(bins=50)
    plt.title(f'Distribution of {feat}')
    plt.savefig(f'plots/{feat}_dist.png')
```

**Checklist**:
- [ ] Kiểm tra missing values (< 20% cho mỗi feature)
- [ ] Phát hiện outliers (IQR method)
- [ ] Xóa outliers cực đoan (> 5 std)
- [ ] Save cleaned data: `financial_features_clean.csv`

---

### Bước 3.2: Fit Distributions (1 ngày)

```python
from scipy import stats

def fit_distribution(data, feature_name):
    """
    Fit best distribution
    """
    data_clean = data.dropna()
    
    # Test normality
    _, p_value = stats.normaltest(data_clean)
    
    if p_value > 0.05:
        # Normal
        params = stats.norm.fit(data_clean)
        return {'dist': 'normal', 'params': params}
    else:
        # Try lognormal
        params = stats.lognorm.fit(data_clean)
        return {'dist': 'lognorm', 'params': params}

# Fit all features
params_dict = {}
for feat in ['roa', 'roe', 'current_ratio', ...]:
    params_dict[feat] = fit_distribution(df_all[feat], feat)

# Save
import json
with open('data/distribution_params.json', 'w') as f:
    json.dump(params_dict, f, indent=2)
```

---

### Bước 3.3: Generate Synthetic 10k (1 ngày)

```python
import json
import numpy as np
import pandas as pd

# Load params
with open('data/distribution_params.json', 'r') as f:
    params = json.load(f)

# Generate
n_samples = 10000
synthetic = {}

for feat, dist_info in params.items():
    if dist_info['dist'] == 'normal':
        mean, std = dist_info['params']
        synthetic[feat] = np.random.normal(mean, std, n_samples)
    elif dist_info['dist'] == 'lognorm':
        s, loc, scale = dist_info['params']
        synthetic[feat] = stats.lognorm.rvs(s, loc, scale, size=n_samples)

df_synthetic = pd.DataFrame(synthetic)

# Maintain correlations
from sklearn.covariance import LedoitWolf
cov = LedoitWolf().fit(df_all[features].dropna()).covariance_

# Adjust để có correlation
# (Sử dụng Cholesky decomposition)

# Save
df_synthetic.to_csv('data/final/synthetic_financial_10k.csv', index=False)
```

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] Tuần 1: 800-1000 file BCTC downloaded
- [ ] Tuần 2: `financial_features_real.csv` với 12 features
- [ ] Tuần 3: `synthetic_financial_10k.csv` + `distribution_params.json`
- [ ] QA: Kiểm tra phân phối hợp lý
- [ ] Documentation: Ghi log các lỗi, missing data

---

## 🆘 TROUBLESHOOTING

**Q: VietStock block IP?**
A: Chuyển sang CafeF hoặc sử dụng proxy/VPN

**Q: Parser không hoạt động?**
A: Kiểm tra lại cấu trúc HTML, VietStock có thể đã thay đổi format

**Q: Missing data quá nhiều?**
A: Chấp nhận 70-80% completeness, interpolate hoặc bỏ qua công ty đó
