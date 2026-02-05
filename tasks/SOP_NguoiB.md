# SOP - Người B: Dữ Liệu Tham Chiếu (District & Industry)

> **Vai trò**: Reference Data Specialist  
> **Output**: District lookup (24 quận) + Industry risk (3 ngành)  
> **Dependencies**: KHÔNG - Làm việc hoàn toàn độc lập

---

## 🎯 TỔNG QUAN CÔNG VIỆC

Bạn chịu trách nhiệm tạo 2 bảng tra cứu từ dữ liệu công khai:
1. **District Lookup**: 24 quận/huyện TP.HCM
2. **Industry Risk**: 3 ngành (Bán buôn, Sản xuất thực phẩm, F&B)

**Đặc điểm**:
- Dữ liệu từ nguồn công khai
- Công việc nhẹ nhất trong team
- Có thể hoàn thành sớm → hỗ trợ người khác

---

## 📋 PHẦN 1: DỮ LIỆU QUẬN/HUYỆN HCM

### Công Việc 1.1: Download Ni ngám Thống Kê

**Mục tiêu**: Lấy dữ liệu chính thức về 24 quận/huyện

**Nguồn**:
- Website: https://thongkehochiminh.nso.gov.vn/Niengiam/Niengiam
- Tìm năm gần nhất (2024 hoặc 2023)
- Download file PDF (~50-100MB)

**Các bảng cần tìm trong PDF**:
1. Diện tích theo quận/huyện (km²)
2. Số doanh nghiệp đang hoạt động
3. Thu nhập bình quân/đầu người

**Actions**:
- Download PDF
- Đánh dấu số trang của từng bảng cần thiết
- Save: `data/raw/nien_giam_hcm_2024.pdf`

---

### Công Việc 1.2: Trích Xuất Dữ Liệu

**Method**: Copy-paste thủ công hoặc dùng tool (Tabula)

**Output files tạm**:
- `raw_tables/area_by_district.csv` - Diện tích
- `raw_tables/businesses_by_district.csv` - Số DN
- `raw_tables/income_by_district.csv` - Thu nhập

**Clean data**:
- Standardize tên quận: "Quận 1" → "1", "Huyện Củ Chi" → "Củ Chi"
- Remove dấu phẩy trong số: "1,234" → "1234"
- Convert to number type

---

### Công Việc 1.3: Merge và Tính Toán

**Create final table với 7 columns**:

1. `district_code` (1-24)
2. `district` (tên)
3. `area_km2` (diện tích)
4. `num_businesses` (số DN)
5. `business_density` = num_businesses / area_km2
6. `avg_income` (thu nhập TB/tháng)
7. `risk_score` (1-10)

**Logic tính risk_score**:

Base score theo vị trí:
- CBD (Q1, 3, 7, Bình Thạnh): 2-3 (low risk)
- Trung tâm mở rộng (Thủ Đức): 4-5
- Nội thành khác: 5-6
- Ngoại thành (Củ Chi, Hóc Môn, etc): 7-9 (high risk)

Adjust theo thu nhập:
- Thu nhập cao (>12M/tháng): -1 điểm
- Thu nhập thấp (<8M/tháng): +1 điểm

Final: Clip to [1, 10]

**Output**: `district_lookup.csv` (24 rows)

**QA**:
- [ ] Đủ 24 quận/huyện HCM
- [ ] No missing values
- [ ] Risk scores hợp lý (CBD thấp, ngoại thành cao)

---

## 📋 PHẦN 2: RỦI RO NGÀNH

### Công Việc 2.1: Research 3 Ngành

**Mục tiêu**: Thu thập thông tin về rủi ro của 3 ngành

**3 Ngành Target**:
1. **Bán buôn** (VSIC 46)
2. **Sản xuất thực phẩm** (VSIC 10)
3. **F&B / Dịch vụ ăn uống** (VSIC 56)

**Sources**:

**1. Báo cáo NHNN**:
- Website: https://www.sbv.gov.vn/
- Tìm: "Báo cáo Ổn định Tài chính"
- Extract: Tỷ lệ nợ xấu theo ngành

**2. Research reports**:
- SSI Research, VNDirect
- Google Scholar: "Vietnam SME default rate"

**3. Business logic**:
- COVID impact
- Volatility (độ biến động)
- Barriers to entry

---

### Công Việc 2.2: Assign Risk Scores

**Framework đánh giá** (cho MỖI ngành):

| Factor | Weight | Scoring |
|--------|--------|---------|
| Default rate estimate | 40% | <3%=0.5, 3-5%=1.0, 5-7%=1.5, >7%=2.0 |
| Volatility | 30% | Low=0.5, Med=1.0, High=1.5 |
| COVID impact | 20% | Positive=0, Neutral=0.5, Negative=1.0 |
| Barriers to entry | 10% | High=0, Med=0.25, Low=0.5 |

**Ví dụ tính toán**:

**Bán buôn (46)**:
- Default rate: 6% → 1.5 điểm
- Volatility: Medium → 1.0 điểm
- COVID: Medium → 0.5 điểm
- Barriers: Low → 0.5 điểm
- **Total**: 3.5 → **Risk score: 6/10**

**Sản xuất thực phẩm (10)**:
- Default: 4.5% → 1.0 điểm
- Volatility: Low → 0.5 điểm
- COVID: Low → 0 điểm
- Barriers: High → 0 điểm
- **Total**: 1.5 → **Risk score: 4/10**

**F&B (56)**:
- Default: 8% → 2.0 điểm
- Volatility: High → 1.5 điểm
- COVID: Very high → 1.5 điểm
- Barriers: Low → 0.5 điểm
- **Total**: 5.5 → **Risk score: 8/10**

---

### Công Việc 2.3: Create Industry Risk Table

**Output**: `industry_risk.csv` (3 rows)

Columns:
1. `vsic_code` (46, 10, 56)
2. `industry_name` (Bán buôn, Sản xuất thực phẩm, F&B)
3. `sector` (Thương mại, Sản xuất, Dịch vụ)
4. `default_rate_estimate` (decimal: 0.06, 0.045, 0.08)
5. `risk_score` (6, 4, 8)

**Example output**:
```csv
vsic_code,industry_name,sector,default_rate_estimate,risk_score
46,Bán buôn,Thương mại,0.060,6
10,Sản xuất thực phẩm,Sản xuất,0.045,4
56,Dịch vụ ăn uống,Dịch vụ,0.080,8
```

**QA**:
- [ ] Exactly 3 rows
- [ ] Risk scores hợp lý: F&B (8) > Bán buôn (6) > Sản xuất (4)
- [ ] VSIC codes correct (46, 10, 56)
- [ ] Có documentation assumptions

---

## 📦 DELIVERABLES

**Lookup Tables** (MAIN OUTPUTS):
1. **`district_lookup.csv`** - 24 quận HCM ⭐
2. **`industry_risk.csv`** - 3 ngành ⭐

**Supporting Files**:
3. `raw_tables/` - Bảng gốc từ Niên giám
4. `research_notes/` - Ghi chú research về ngành
5. `assumptions_industry_risk.md` - Document rationale cho scores

---

## 🆘 TROUBLESHOOTING

**Q: Không download được Niên giám?**
→ A:
- Dùng data năm trước (2023, 2022)
- Hoặc scrape từ websites BĐS (mật độ dân cư)
- Hoặc estimate dựa trên knowledge chung

**Q: Thiếu data về default rate?**
→ A:
- Dùng international benchmarks
- Adjust cho Vietnam (+1-2%)
- Rely vào qualitative assessment

**Q: Risk scores có hợp lý không?**
→ A:
- Quan trọng là RELATIVE ranking đúng
- F&B PHẢI cao nhất (ngành rủi ro cao)
- Sản xuất thực phẩm PHẢI thấp nhất (ổn định)
- Chênh lệch 2-4 điểm là OK

---

## ✅ SUCCESS CRITERIA

- [ ] District table: **24 rows**, 7 columns
- [ ] Industry table: **3 rows**, 5 columns
- [ ] No missing values
- [ ] Risk scores có rationale rõ ràng
- [ ] Documentation assumptions

---

## 🎯 NEXT STEPS (Sau khi hoàn thành)

Vì công việc của bạn nhẹ nhất, sau khi xong bạn có thể:

1. **QA support**: Giúp người khác review data quality
2. **Research support**: Tìm thêm literature về SME Vietnam
3. **Integration prep**: Chuẩn bị merge scripts

**LƯU Ý ĐẶC BIỆT**:
- Bạn làm HOÀN TOÀN ĐỘC LẬP
- Output của bạn sẽ được dùng trong Integration phase
- Nhưng người khác KHÔNG chờ bạn để bắt đầu công việc của họ
- Optimize để finish early → maximize team efficiency!
