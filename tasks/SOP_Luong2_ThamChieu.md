# Hướng Dẫn Chi Tiết: Luồng 2 - Dữ Liệu Tham Chiếu Công Khai

> **Thời gian**: 1 tuần
> **Người thực hiện**: Thành viên B
> **Output**: `district_lookup.csv` + `industry_risk.csv`

---

## 📋 CÔNG VIỆC 2.1: DỮ LIỆU QUẬN (3 ngày)

### Ngày 1: Download Niên Giám Thống Kê

#### Bước 1.1: Truy cập Website Cục Thống Kê TP.HCM

1. **Mở trình duyệt**
   - URL: http://www.pso.hochiminhcity.gov.vn/

2. **Tìm Niên giám**
   - Menu: "Ấn phẩm" → "Niên giám thống kê"
   - Hoặc search: "Niên giám thống kê TP.HCM 2024"

3. **Download PDF**
   - Click vào "Niên giám thống kê năm 2024"
   - Save file: `data/raw/nien_giam_hcm_2024.pdf`
   - Size: ~50-100 MB

**Checkpoint**: File PDF đã download ✅

---

#### Bước 1.2: Xác Định Bảng Cần Trích Xuất

**Mở PDF**, tìm các bảng sau:

| Bảng | Nội dung | Trang (ước lượng) |
|------|----------|-------------------|
| Bảng 2.1 | Diện tích theo quận/huyện | ~trang 50-60 |
| Bảng 3.5 | Số doanh nghiệp đang hoạt động theo quận | ~trang 100-120 |
| Bảng 5.2 | Thu nhập bình quân theo quận | ~trang 200-220 |
| Bảng 6.1 | GDP theo quận (nếu có) | ~trang 250+ |

**Ghi chú**: Số trang có thể khác tùy năm, dùng chức năng Find (Ctrl+F) để tìm nhanh

---

### Ngày 2: Trích Xuất Dữ Liệu

#### Bước 2.1: Cài Đặt Tools

```bash
# Cài tabula (Java-based PDF extractor)
pip install tabula-py

# Hoặc pdfplumber (Python-native)
pip install pdfplumber

# Recommendation: Dùng pdfplumber vì dễ hơn
```

#### Bước 2.2: Extract Bảng Thủ Công (Cách Đơn Giản)

**Nếu bảng đơn giản**: Copy-paste trực tiếp

1. Mở PDF bằng Adobe Reader
2. Chọn bảng "Số DN theo quận"
3. Copy (Ctrl+C)
4. Paste vào Excel
5. Save as CSV: `raw_tables/businesses_by_district.csv`

**Lặp lại** với các bảng khác

---

#### Bước 2.3: Extract Bằng Code (Cách Tự Động)

**File**: `scripts/extract_nien_giam.py`

```python
"""
Trích xuất bảng từ Niên giám PDF
"""
import pdfplumber
import pandas as pd

def extract_table_from_pdf(pdf_path, page_num):
    """
    Extract 1 bảng từ 1 trang
    """
    with pdfplumber.open(pdf_path) as pdf:
        page = pdf.pages[page_num]
        table = page.extract_table()
        
        if table:
            # Convert to DataFrame
            df = pd.DataFrame(table[1:], columns=table[0])
            return df
    return None

# Extract bảng số DN (giả sử trang 115)
df_businesses = extract_table_from_pdf(
    'data/raw/nien_giam_hcm_2024.pdf',
    page_num=115
)

print(df_businesses)

# Save
df_businesses.to_csv('raw_tables/businesses_by_district.csv', index=False)
```

**Test**:
```bash
python scripts/extract_nien_giam.py
```

**Expected output**:
```
Quận/Huyện       Số DN
Quận 1           12,450
Quận 3           8,320
Bình Thạnh       6,789
...
```

---

### Ngày 3: Xây Dựng Bảng Tra Cứu

#### Bước 3.1: Merge Các Bảng

**File**: `scripts/build_district_lookup.py`

```python
"""
Gộp các bảng thành 1 bảng tra cứu cuối
"""
import pandas as pd

# Load các bảng raw
df_area = pd.read_csv('raw_tables/area_by_district.csv')
df_businesses = pd.read_csv('raw_tables/businesses_by_district.csv')
df_income = pd.read_csv('raw_tables/income_by_district.csv')

# Clean tên quận (loại bỏ "Quận ", "Huyện ")
def clean_district_name(name):
    name = str(name).strip()
    name = name.replace('Quận ', '').replace('Huyện ', '')
    return name

df_area['district'] = df_area['Quận/Huyện'].apply(clean_district_name)
df_businesses['district'] = df_businesses['Quận/Huyện'].apply(clean_district_name)
df_income['district'] = df_income['Quận/Huyện'].apply(clean_district_name)

# Merge
df_merged = df_area[['district', 'Diện tích (km²)']].copy()
df_merged = df_merged.merge(
    df_businesses[['district', 'Số DN']], 
    on='district', 
    how='left'
)
df_merged = df_merged.merge(
    df_income[['district', 'Thu nhập TB (VNĐ/tháng)']], 
    on='district', 
    how='left'
)

# Rename columns
df_merged.columns = ['district', 'area_km2', 'num_businesses', 'avg_income']

# Add district_code (1-24)
district_mapping = {
    '1': 1, '3': 3, '4': 4, '5': 5, '6': 6, '7': 7, '8': 8,
    '10': 10, '11': 11, '12': 12, 'Tân Bình': 13, 'Tân Phú': 14,
    'Bình Thạnh': 15, 'Phú Nhuận': 16, 'Gò Vấp': 17,
    'Bình Tân': 18, 'Thủ Đức': 2, 'Củ Chi': 19, 'Hóc Môn': 20,
    'Bình Chánh': 21, 'Nhà Bè': 22, 'Cần Giờ': 23
}

df_merged['district_code'] = df_merged['district'].map(district_mapping)

# Calculate business density
df_merged['business_density'] = df_merged['num_businesses'] / df_merged['area_km2']

print(df_merged)
```

**Output mẫu**:
```
  district  district_code  area_km2  num_businesses  avg_income  business_density
0 1         1              7.73      12450           15000000    1610.09
1 3         3              4.90      8320            14000000    1697.96
...
```

---

#### Bước 3.2: Tính District Risk Score

**Công thức** (có thể adjust):

```python
def calculate_district_risk(row):
    """
    Risk score: 1 (thấp nhất) đến 10 (cao nhất)
    
    Logic:
    - CBD (Q1, 3, 7, Bình Thạnh): Low risk (1-3)
    - Khu công nghiệp (Thủ Đức, Q9, Q2): Medium-low (3-5)
    - Ngoại thành (Củ Chi, Hóc Môn): High (7-9)
    """
    district = row['district']
    income = row['avg_income']
    density = row['business_density']
    
    # Baseline từ location
    if district in ['1', '3', '7', 'Bình Thạnh']:
        base_score = 2
    elif district in ['Thủ Đức', '2', '9']:
        base_score = 4
    elif district in ['4', '5', '6', '8', '10', '11', 'Tân Bình', 'Phú Nhuận']:
        base_score = 5
    elif district in ['Tân Phú', 'Gò Vấp', 'Bình Tân']:
        base_score = 6
    else:  # Ngoại thành
        base_score = 8
    
    # Adjust dựa trên income (income cao -> risk thấp)
    income_factor = 0
    if income > 12000000:
        income_factor = -1
    elif income < 8000000:
        income_factor = +1
    
    # Final score
    risk_score = max(1, min(10, base_score + income_factor))
    return risk_score

# Apply
df_merged['risk_score'] = df_merged.apply(calculate_district_risk, axis=1)
```

---

#### Bước 3.3: Save Final Table

```python
# Reorder columns
columns_order = [
    'district_code', 'district', 'area_km2', 'num_businesses',
    'business_density', 'avg_income', 'risk_score'
]

df_final = df_merged[columns_order]

# Save
df_final.to_csv('data/lookup_tables/district_lookup.csv', index=False)

print("✅ District lookup table created!")
print(df_final)
```

**Checkpoint**: File `district_lookup.csv` với 24 dòng (24 quận/huyện HCM) ✅

---

## 📋 CÔNG VIỆC 2.2: RỦI RO NGÀNH (4 ngày)

### Ngày 1-2: Nghiên Cứu Literature

#### Bước 1: Thu Thập Tài Liệu

**Nguồn 1: Báo cáo NHNN**
- Website: https://www.sbv.gov.vn/
- Tìm: "Báo cáo Ổn định Tài chính" (Financial Stability Report)
- Download PDF năm gần nhất

**Nguồn 2: Báo cáo Nghiên cứu**
- SSI Research: https://www.ssi.com.vn/
- VNDirect Research: https://www.vndirect.com.vn/
- Tìm: "SME Outlook", "Industry Report"

**Nguồn 3: Academic Papers**
- Google Scholar: Search "SME default rate Vietnam by industry"
- Tìm papers 2020-2024

#### Bước 2: Extract Thông Tin

Tạo file: `research_notes/industry_research.md`

```markdown
# Industry Risk Research

## Findings from NHNN Report 2023

- Retail/Wholesale: Default rate ~6%
- Manufacturing: Default rate ~4-5%
- F&B/Services: Default rate ~8% (post-COVID)
- Real Estate: Default rate ~7%
- Technology: Default rate ~3%

## Impact Factors

1. **COVID Impact** (2020-2022):
   - Tourism/F&B: High impact
   - Tech/E-commerce: Low impact or positive

2. **Volatility**:
   - Construction: High volatility
   - Retail: Medium volatility
   - Tech: Low volatility

3. **Barriers to Entry**:
   - Manufacturing: High barriers (capital intensive)
   - Retail: Low barriers
```

---

### Ngày 3: Build Industry Mapping

#### Bước 1: Lấy Danh Sách VSIC Codes

**File**: `data/vsic_codes.csv` (tạo thủ công hoặc search online)

```csv
vsic_code,industry_name_vi,industry_name_en,sector
46,Bán buôn,Wholesale,Thương mại
47,Bán lẻ,Retail,Thương mại
10,Sản xuất thực phẩm,Food Manufacturing,Sản xuất
56,Dịch vụ ăn uống,Food & Beverage Services,Dịch vụ
62,Lập trình máy tính,Computer Programming,Công nghệ
...
```

**Lọc 10-15 ngành phổ biến nhất** cho SME:

```python
priority_industries = [
    '46',  # Bán buôn
    '47',  # Bán lẻ  
    '10',  # Sản xuất thực phẩm
    '56',  # F&B
    '62',  # IT
    '41',  # Xây dựng
    '49',  # Vận tải
    '68',  # Bất động sản
    '71',  # Kiến trúc
    '82',  # Hỗ trợ văn phòng
]
```

---

### Ngày 4: Assign Risk Scores

**File**: `scripts/build_industry_risk.py`

```python
"""
Xây dựng bảng industry risk score
"""
import pandas as pd

# Manual scoring dựa trên research
industry_risk_data = [
    {
        'vsic_code': '46',
        'industry_name': 'Bán buôn',
        'sector': 'Thương mại',
        'default_rate_estimate': 0.06,  # 6%
        'volatility': 'Medium',
        'covid_impact': 'Medium',
        'barriers': 'Low',
        'risk_score': 6
    },
    {
        'vsic_code': '47',
        'industry_name': 'Bán lẻ',
        'sector': 'Thương mại',
        'default_rate_estimate': 0.065,
        'volatility': 'Medium',
        'covid_impact': 'High',
        'barriers': 'Low',
        'risk_score': 7
    },
    {
        'vsic_code': '10',
        'industry_name': 'Sản xuất thực phẩm',
        'sector': 'Sản xuất',
        'default_rate_estimate': 0.045,
        'volatility': 'Low',
        'covid_impact': 'Low',
        'barriers': 'High',
        'risk_score': 4
    },
    {
        'vsic_code': '56',
        'industry_name': 'Dịch vụ ăn uống',
        'sector': 'Dịch vụ',
        'default_rate_estimate': 0.08,
        'volatility': 'High',
        'covid_impact': 'Very High',
        'barriers': 'Low',
        'risk_score': 8
    },
    {
        'vsic_code': '62',
        'industry_name': 'Lập trình máy tính',
        'sector': 'Công nghệ',
        'default_rate_estimate': 0.03,
        'volatility': 'Low',
        'covid_impact': 'Negative (beneficial)',
        'barriers': 'Medium',
        'risk_score': 3
    },
    # ... Thêm 5-10 ngành nữa
]

df_industry = pd.DataFrame(industry_risk_data)

# Save
df_industry.to_csv('data/lookup_tables/industry_risk.csv', index=False)

print("✅ Industry risk table created!")
print(df_industry)
```

**Output**:
```
  vsic_code industry_name          sector  default_rate  risk_score
0 46        Bán buôn               Thương mại  0.060      6
1 47        Bán lẻ                 Thương mại  0.065      7
2 10        Sản xuất thực phẩm     Sản xuất    0.045      4
3 56        Dịch vụ ăn uống        Dịch vụ     0.080      8
4 62        Lập trình máy tính     Công nghệ   0.030      3
```

---

## ✅ CHECKLIST HOÀN THÀNH

- [ ] `district_lookup.csv` - 24 quận với 7 cột
- [ ] `industry_risk.csv` - 10-15 ngành với risk scores
- [ ] `research_notes/` - Tài liệu ghi chép nguồn
- [ ] QA: Cross-check risk scores với literature
- [ ] Documentation: Ghi rõ assumptions và nguồn

---

## 🆘 TROUBLESHOOTING

**Q: PDF không extract được?**
A: Thử copy-paste thủ công, hoặc dùng OCR (pytesseract)

**Q: Không tìm thấy báo cáo NHNN?**
A: Dùng industry benchmarks từ research papers quốc tế

**Q: Risk score assign như thế nào?**
A: Dựa trên 3 factors: default rate (40%), volatility (30%), COVID impact (30%)
