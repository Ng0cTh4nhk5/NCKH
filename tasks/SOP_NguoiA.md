# SOP - Người A: Thu Thập & Phân Tích BCTC

> **Vai trò**: Data Collection Lead - Financial Data  
> **Output**: 3,000 mẫu dữ liệu tài chính synthetic (1,000/ngành)  
> **Dependencies**: KHÔNG - Làm việc hoàn toàn độc lập

---

## 🎯 TỔNG QUAN CÔNG VIỆC

Bạn chịu trách nhiệm thu thập Báo cáo Tài chính từ các công ty niêm yết nhỏ và tạo dữ liệu synthetic realistic.

**Scope**:
- Thu thập BCTC từ 100-150 công ty UPCOM (3 ngành)
- Trích xuất 12 chỉ số tài chính
- Phân tích distributions
- Generate 3,000 samples synthetic (1,000/ngành)

---

## 📋 GIAI ĐOẠN 1: THU THẬP BCTC

### Công Việc 1.1: Xác Định Danh Sách Công Ty

**Mục tiêu**: Tạo list 100-150 công ty thuộc 3 ngành mục tiêu

**3 Ngành Target**:
1. Bán buôn (VSIC 46) - Target: 30-50 công ty
2. Sản xuất thực phẩm (VSIC 10) - Target: 30-50 công ty
3. F&B (VSIC 56) - Target: 30-50 công ty

**Nguồn dữ liệu**:
- Website HNX: https://www.hnx.vn/vi-vn/cong-ty-dai-chung.html
- VietStock: https://finance.vietstock.vn/

**Tiêu chí lọc**:
- Sàn: UPCOM
- Mã ngành (VSIC): CHỈ 46, 10, hoặc 56
- Doanh thu < 200 tỷ VNĐ
- Trụ sở: TP.HCM
- Đang hoạt động

**Output**: File Excel `company_list_by_industry.xlsx` với 3 sheets:
- Sheet "Wholesale" (46)
- Sheet "Food_Manufacturing" (10)  
- Sheet "FnB" (56)

---

### Công Việc 1.2: Download BCTC

**Mục tiêu**: Download 400-600 file BCTC (100-150 công ty × 4 năm: 2020-2023)

**Phương pháp**:

**Option 1: Script tự động** (Khuyến nghị)
- Tool: Python + requests/selenium
- Input: Company list by industry
- Output structure:
  ```
  data/raw/bctc/
  ├── wholesale/     # Ngành 46
  ├── food_mfg/      # Ngành 10
  └── fnb/          # Ngành 56
  ```

**Option 2: Manual download**
- Download từ VietStock, CafeF
- Organize theo ngành ngay từ đầu

**Sources**:
- VietStock: https://finance.vietstock.vn/
- CafeF: https://s.cafef.vn/

**QA Check**:
- [ ] Expected: 400-600 files total
- [ ] Mỗi ngành có ít nhất 100 files
- [ ] Accept 75% completeness

---

### Công Việc 1.3: Trích Xuất Features

**Mục tiêu**: Parse BCTC → Extract 10 chỉ tiêu gốc

**10 Chỉ Tiêu Cần Trích Xuất**:

Từ Bảng Cân Đối Kế Toán:
1. Tổng tài sản
2. Tài sản ngắn hạn
3. Nợ phải trả (Tổng nợ)
4. Nợ ngắn hạn
5. Vốn chủ sở hữu
6. Hàng tồn kho
7. Khoản phải thu

Từ BCKQ:
8. Doanh thu thuần
9. Giá vốn hàng bán
10. Lợi nhuận sau thuế

**Method**:
- Dùng BeautifulSoup (HTML) hoặc pandas (Excel)
- Loop qua tất cả files
- Output: `financial_raw.csv` với columns:
  - ticker, year, industry_code, [10 chỉ tiêu]

**Handle missing**:
- Ghi NULL/NaN nếu thiếu data
- Accept < 30% missing rate

---

## 📋 GIAI ĐOẠN 2: TÍNH TOÁN RATIOS

### Công Việc 2.1: Calculate 12 Financial Ratios

**From raw data → 12 ratios**:

1. **ROA** = Lợi nhuận / Tổng tài sản
2. **ROE** = Lợi nhuận / Vốn CSH
3. **Profit Margin** = Lợi nhuận / Doanh thu
4. **Revenue Growth** = (Revenue năm nay - năm trước) / Revenue năm trước
5. **Current Ratio** = Tài sản ngắn hạn / Nợ ngắn hạn
6. **Quick Ratio** = (TSNH - Hàng tồn kho) / Nợ ngắn hạn
7. **Debt-to-Equity** = Nợ / Vốn CSH
8. **Debt-to-Asset** = Nợ / Tài sản
9. **Asset Turnover** = Doanh thu / Tài sản
10. **Inventory Turnover** = Giá vốn / Hàng tồn kho
11. **Receivable Turnover** = Doanh thu / Phải thu
12. **DSCR** = Estimate từ Operating Income

**Output**: `financial_features.csv`
- ~350-450 rows (sau khi filter outliers)
- 12 ratio columns + industry_code

**QA**:
- Remove outliers > mean ± 5 std
- Verify mỗi ngành có ≥100 data points

---

## 📋 GIAI ĐOẠN 3: PHÂN TÍCH & GENERATE SYNTHETIC

### Công Việc 3.1: Phân Tích Distributions (Theo Ngành)

**Mục tiêu**: Hiểu phân phối của 12 ratios cho TỪNG NGÀNH riêng biệt

**Workflow**:

**Cho Mỗi Ngành** (46, 10, 56):

1. Filter data: `df[df.industry_code == 46]`

2. Tính statistics cho 12 ratios:
   - Mean, Median, Std
   - Min, Max, Percentiles

3. Visual check:
   - Histogram mỗi ratio
   - Identify distribution type (Normal, Lognormal, etc.)

4. Fit distributions:
   - Cho mỗi ratio, fit best distribution
   - Save parameters

5. Estimate correlation matrix:
   - 12×12 matrix cho ngành này
   - Save để maintain correlations

**Output PER Industry**:
```
distributions/
├── wholesale_params.json      # Ngành 46
├── food_mfg_params.json       # Ngành 10
└── fnb_params.json            # Ngành 56
```

---

### Công Việc 3.2: Generate Synthetic Data

**Mục tiêu**: Sinh 3,000 samples (1,000/ngành)

**Approach**: Generate RIÊNG cho từng ngành

**Bước 1: Generate Ngành Bán Buôn (46)**

Input:
- `wholesale_params.json` (distributions)
- Correlation matrix cho ngành 46

Process:
- Generate 1,000 samples from multivariate distribution
- Transform về marginal distributions
- Add column `industry_code = 46`

Output: `synthetic_wholesale_1k.csv`

**Bước 2: Generate Ngành Sản Xuất (10)**

Tương tự, output: `synthetic_food_mfg_1k.csv`

**Bước 3: Generate Ngành F&B (56)**

Tương tự, output: `synthetic_fnb_1k.csv`

**Bước 4: Merge 3 Ngành**

Gộp 3 files:
```
pd.concat([
    wholesale_1k,
    food_mfg_1k,
    fnb_1k
])
```

**Final Output**: `synthetic_financial_3k.csv`
- 3,000 rows (1,000 per industry)
- 12 ratio columns + industry_code + sample_id

---

## 📋 GIAI ĐOẠN 4: VALIDATION

### QA Checks

**Distribution Quality**:
- [ ] Visual: Histograms match learned patterns
- [ ] Stats: Means within ±10% of real data
- [ ] Correlations maintained (within ±0.15)

**Data Quality**:
- [ ] Exactly 3,000 samples
- [ ] Each industry has exactly 1,000 samples
- [ ] No missing values
- [ ] industry_code correct (46, 10, 56)
- [ ] All ratios in reasonable ranges

**Industry Balance**:
- [ ] Distributions KHÁC NHAU giữa 3 ngành (verify bằng mắt)
- [ ] VD: Wholesale có Asset Turnover cao hơn Manufacturing

---

## 📦 DELIVERABLES

**Data Files**:
1. `company_list_by_industry.xlsx` - Danh sách công ty
2. `data/raw/bctc/` - 400-600 BCTC files (organized by industry)
3. `financial_raw.csv` - Raw extracted data
4. `financial_features.csv` - 12 ratios from real data
5. **`synthetic_financial_3k.csv`** ⭐ MAIN OUTPUT

**Documentation**:
6. `distributions/` - Distribution parameters per industry
7. `eda_report.pdf` - Exploratory analysis
8. `assumptions_log.md` - Ghi chép assumptions

---

## 🆘 TROUBLESHOOTING

**Q: Không đủ công ty 1 ngành?**
→ A: Giảm target xuống 25 công ty/ngành. Minimum total: 75 công ty.

**Q: Missing data quá nhiều?**
→ A: Accept if < 40% missing. Use mean imputation nếu cần.

**Q: Distribution fit không tốt?**
→ A: OK nếu visual check reasonable. Không cần perfect p-values.

**Q: Synthetic data có outliers?**
→ A: Clip về reasonable ranges. Document trong assumptions.

---

## ✅ SUCCESS CRITERIA

- [ ] **3,000 synthetic samples** (1,000/ngành)
- [ ] **12 financial features** + industry_code
- [ ] **Balanced** across 3 industries
- [ ] **Realistic** distributions per industry
- [ ] **No missing values**
- [ ] **Documented** assumptions và process

---

**ĐẶC BIỆT LƯU Ý**: 
- Bạn làm HOÀN TOÀN ĐỘC LẬP
- KHÔNG cần đợi ai
- KHÔNG ai phụ thuộc vào bạn trong giai đoạn generation
- Chỉ cần deliver `synthetic_financial_3k.csv` khi xong là OK!
