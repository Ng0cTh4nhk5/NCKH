# Index: Tất Cả Hướng Dẫn SOP Thu Thập Dữ Liệu

> **Mục đích**: Tổng hợp tất cả file hướng dẫn chi tiết cho 4 luồng + tích hợp

---

## 📚 Danh Sách SOPs

### 1. Luồng 1: BCTC Công Ty Niêm Yết
**File**: `SOP_Luong1_BCTC.md`

**Nội dung**:
- Tuần 1: Download 200 BCTC từ UPCOM
- Tuần 2: Extract 12 financial ratios
- Tuần 3: Analyze distributions & generate synthetic 10k

**Output**: `synthetic_financial_10k.csv` + `distribution_params.json`

**Công sức**: 3 tuần | **Độ khó**: ⭐⭐⭐⭐

---

### 2. Luồng 2: Dữ Liệu Tham Chiếu
**File**: `SOP_Luong2_ThamChieu.md`

**Nội dung**:
- Ngày 1-3: Download & extract Niên giám HCM → District data
- Ngày 4-7: Research & build Industry risk table

**Output**: `district_lookup.csv` + `industry_risk.csv`

**Công sức**: 1 tuần | **Độ khó**: ⭐⭐

---

### 3. Luồng 3: CIC & Giao Dịch Synthetic
**File**: `SOP_Luong3_CIC_GiaoDich.md`

**Nội dung**:
- Ngày 1-3: Design schemas & generate CIC (8 features)
- Ngày 1-3: Generate Transaction (8 features) - SONG SONG
- Ngày 4-5: Merge & validate

**Output**: `synthetic_cic_transaction_10k.csv`

**Công sức**: 1 tuần | **Độ khó**: ⭐⭐⭐

---

### 4. Luồng 4: Demographic Synthetic
**File**: `SOP_Luong4_Demographic.md`

**Nội dung**:
- Ngày 1-3: Generate categorical + numeric (7 features)
- Ngày 4-6: Generate collateral features (5 features)
- Ngày 7: Merge all

**Output**: `synthetic_demographic_10k.csv`

**Công sức**: 1 tuần | **Độ khó**: ⭐⭐

---

### 5. Tích Hợp (Integration)
**File**: `SOP_TichHop.md`

**Nội dung**:
- Tuần 1: Merge tất cả features (58 features)
- Tuần 1: Enrich với lookup tables
- Tuần 1: Generate target variable (default)
- Tuần 2: QA + Train/Val/Test split + Documentation

**Output**: `train_7k.csv`, `val_1.5k.csv`, `test_1.5k.csv`, `full_10k.csv`

**Công sức**: 1-2 tuần | **Độ khó**: ⭐⭐⭐

---

## 🗺️ Lộ Trình Thực Hiện

### Timeline Tổng Thể: 6 Tuần

```
Tuần 1:
├─ Luồng 1: Download BCTC (Batch 1-4)
├─ Luồng 2: Download Niên giám + Extract tables
├─ Luồng 3: Design schemas + Start generating
└─ Luồng 4: Design schemas + Start generating

Tuần 2:
├─ Luồng 1: Extract features từ BCTC
├─ Luồng 2: Build industry risk table ✓ DONE
├─ Luồng 3: Complete CIC + Transaction ✓ DONE
└─ Luồng 4: Complete Demographic ✓ DONE

Tuần 3:
├─ Luồng 1: Analyze distributions + Generate synthetic ✓ DONE
└─ Wait for Luồng 1...

Tuần 4:
└─ (Buffer nếu Luồng 1 trễ)

Tuần 5:
└─ Integration: Merge + Enrich + Generate target

Tuần 6:
└─ Integration: QA + Split + Documentation ✓ DONE
```

---

## 👥 Phân Công Theo Số Người

### 4 Người (OPTIMAL)

| Người | Tuần 1-2 | Tuần 3 | Tuần 4-6 |
|-------|----------|--------|----------|
| **A** | Luồng 1 | Luồng 1 | Hỗ trợ Integration |
| **B** | Luồng 2 | Hỗ trợ Luồng 1 | Lead Integration |
| **C** | Luồng 3 | Hỗ trợ Luồng 1 | Hỗ trợ Integration |
| **D** | Luồng 4 | QA preliminary | Integration + Docs |

### 3 Người

| Người | Công việc |
|-------|-----------|
| **A** | Luồng 1 (3 tuần) + Luồng 4 (tuần 2) |
| **B** | Luồng 2 (tuần 1) + Integration (tuần 2-3) |
| **C** | Luồng 3 (tuần 1) + Hỗ trợ Luồng 1 (tuần 2) + Integration (tuần 3) |

### 1 Người (Sequential)

1. Luồng 2 (1 tuần) - Dễ nhất
2. Luồng 3 (1 tuần)
3. Luồng 4 (1 tuần)
4. Luồng 1 (3 tuần) - Khó nhất, để cuối
5. Integration (2 tuần)

**Total**: 8 tuần

---

## 📦 Deliverables Cuối Cùng

### Dữ Liệu
```
data/
├── final/
│   ├── train_7k.csv               # Training set (70%)
│   ├── val_1.5k.csv               # Validation set (15%)
│   ├── test_1.5k.csv              # Test set (15%)
│   ├── full_10k.csv               # Full dataset
│   └── QA_REPORT.txt              # Quality report
├── lookup_tables/
│   ├── district_lookup.csv        # 24 quận HCM
│   └── industry_risk.csv          # 10-15 ngành
└── metadata/
    ├── distribution_params.json   # Financial distributions
    └── data_dictionary.csv        # Feature definitions
```

### Tài Liệu
```
docs/
├── data_documentation.md          # Overview tổng thể
├── data_acquisition_report.pdf    # Báo cáo quá trình
└── assumptions_log.md             # Ghi chép assumptions
```

### Code
```
scripts/
├── download_vietstock.py          # Luồng 1
├── parse_bctc.py
├── calculate_ratios.py
├── extract_nien_giam.py           # Luồng 2
├── build_industry_risk.py
├── generate_cic.py                # Luồng 3
├── generate_transaction.py
├── generate_demographic.py        # Luồng 4
└── merge_all_features.py          # Integration
```

---

## ✅ Checklist Tổng Thể

### Luồng 1: Financial ✓
- [ ] 200 BCTC files downloaded
- [ ] 12 financial ratios extracted
- [ ] Distribution analysis complete
- [ ] 10k synthetic samples generated

### Luồng 2: Reference Data ✓
- [ ] District lookup table (24 quận)
- [ ] Industry risk table (10-15 ngành)
- [ ] Research notes documented

### Luồng 3: CIC & Transaction ✓
- [ ] CIC 8 features generated
- [ ] Transaction 8 features generated
- [ ] Validation passed

### Luồng 4: Demographic ✓
- [ ] Categorical 5 features
- [ ] Numeric 3 features
- [ ] Collateral 4 features
- [ ] Total 12 features

### Integration ✓
- [ ] All features merged (58 features)
- [ ] Lookup tables enriched
- [ ] Target variable generated (5% default)
- [ ] QA passed (missing < 5%)
- [ ] Train/Val/Test split
- [ ] Documentation complete

---

## 🎯 Success Criteria

- [ ] **Completeness**: 10,000 samples với ≥ 95% có đủ 58 features
- [ ] **Quality**: Missing values < 5% mỗi feature
- [ ] **Realism**: Financial distributions match learned patterns
- [ ] **Balance**: Default rate = 5% ± 1% across all splits
- [ ] **Documentation**: Complete data dictionary + assumptions
- [ ] **Timeline**: Hoàn thành trong 6 tuần (nếu 4 người) hoặc 8 tuần (nếu 1 người)

---

## 📞 Support & Questions

**Nếu gặp vấn đề kỹ thuật**:
1. Check troubleshooting section trong mỗi SOP
2. Review validation scripts
3. Check data types và ranges

**Nếu thiếu dữ liệu**:
1. Chấp nhận 80% completeness cho BCTC
2. Sử dụng literature values cho industry risk
3. Interpolate missing values hợp lý

**Nếu timeline trễ**:
1. Giảm số công ty xuống 100 (từ 200)
2. Giảm số năm từ 5 xuống 3 năm
3. Sử dụng pure synthetic cho Financial (bỏ qua BCTC download)

---

Bắt đầu từ SOP nào? Khuyến nghị: **Luồng 2** (dễ nhất, 1 tuần)
