# README - Phân Công Nhóm 5 Người

> **Project**: SME Credit Scoring - Data Acquisition  
> **Team Size**: 5 people  
> **Goal**: Generate 3,000 synthetic samples với 48+ features

---

## 👥 PHÂN CÔNG TỔNG QUAN

| Người | Vai Trò | Output Chính | Độ Khó | SOP File |
|-------|---------|--------------|--------|----------|
| **A** (Bạn) | Financial Data Lead | 3k financial features | ⭐⭐⭐⭐ | `SOP_NguoiA.md` |
| **B** | Reference Data | District + Industry | ⭐⭐ | `SOP_NguoiB.md` |
| **C** | CIC Synthetic | 8 CIC features | ⭐⭐⭐ | `SOP_NguoiC.md` |
| **D** | Transaction Synthetic | 8 Transaction features | ⭐⭐⭐ | `SOP_NguoiD.md` |
| **E** | Demographic + Integration | 12 Demo + Final merge | ⭐⭐⭐⭐ | `SOP_NguoiE.md` |

---

## 🎯 OUTPUTS

### Individual Outputs (Phase 1)

**Người A**:
- `synthetic_financial_3k.csv` (3,000 × 12 features)
- Files: BCTC downloads, distribution params

**Người B**:
- `district_lookup.csv` (24 × 7)
- `industry_risk.csv` (3 × 5)

**Người C**:
- `synthetic_cic_3k.csv` (3,000 × 8 + revenue_temp)

**Người D**:
- `synthetic_transaction_3k.csv` (3,000 × 8 + revenue_temp)

**Người E** (Phase 1):
- `synthetic_demographic_3k.csv` (3,000 × 12)

---

### Team Outputs (Phase 2 - Integration)

**Người E leads, all support**:
- **`train_2.1k.csv`** - 2,100 samples (70%)
- **`val_450.csv`** - 450 samples (15%)
- **`test_450.csv`** - 450 samples (15%)
- `full_3k.csv` - Complete dataset
- `data_dictionary.csv`
- `data_documentation.md`
- `QA_Report.txt`

---

## 📁 FILE STRUCTURE

```
NCKH/
├── main/
│   └── tasks/
│       ├── README_5People.md          # ⭐ This file
│       ├── SOP_NguoiA.md              # Person A instructions
│       ├── SOP_NguoiB.md              # Person B instructions
│       ├── SOP_NguoiC.md              # Person C instructions
│       ├── SOP_NguoiD.md              # Person D instructions
│       ├── SOP_NguoiE.md              # Person E instructions
│       ├── Gantt_Timeline.md          # Timeline visualization
│       └── Dependencies.md            # Dependency analysis
│
├── data/
│   ├── raw/
│   │   └── bctc/                      # A's BCTC files
│   ├── intermediate/
│   │   ├── synthetic_financial_3k.csv      # From A
│   │   ├── district_lookup.csv             # From B
│   │   ├── industry_risk.csv               # From B
│   │   ├── synthetic_cic_3k.csv            # From C
│   │   ├── synthetic_transaction_3k.csv    # From D
│   │   └── synthetic_demographic_3k.csv    # From E
│   └── final/
│       ├── train_2.1k.csv             # ⭐ Final output
│       ├── val_450.csv                # ⭐ Final output
│       ├── test_450.csv               # ⭐ Final output
│       └── full_3k.csv
│
└── docs/
    ├── data_dictionary.csv
    └── data_documentation.md
```

---

## 🚀 GETTING STARTED

### Bước 1: Đọc SOP Của Bạn

Mỗi người:
1. Mở file `SOP_Nguoi[X].md` (X = A/B/C/D/E)
2. Đọc kỹ objectives và deliverables
3. Chuẩn bị tools (Excel/Python/etc)

### Bước 2: Coordination Meeting

**Agenda**:
- [ ] Review phân công
- [ ] Agree on standards (file naming, data types)
- [ ] Verify industry codes (46, 10, 56)
- [ ] Setup shared folder
- [ ] Q&A

**Key Decisions**:
- File naming: `synthetic_[type]_3k.csv`
- Missing values: Use `NaN` (not 0 or null)
- Industry balance: 1,000 samples each
- Sample IDs: 1-3000 (consecutive)

### Bước 3: Phase 1 - Parallel Execution

**Mỗi người làm việc độc lập**:
- A: BCTC collection
- B: Reference data
- C: CIC generation
- D: Transaction generation
- E: Demographic generation

**Communication**:
- Regular updates on Slack/Discord
- Report blockers immediately
- Share learnings and code snippets

### Bước 4: Delivery Phase 1

**Checklist before delivery**:
- [ ] File có đúng 3,000 rows
- [ ] Columns như documented
- [ ] No unexpected missing values
- [ ] File uploaded to shared folder
- [ ] Notify E (Integration Lead)

### Bước 5: Phase 2 - Integration

**E leads, all support**:
- E: Merge all datasets
- A: Verify financial features
- B: Verify lookup joins
- C: Verify CIC features
- D: Verify transaction features

---

## 🔑 KEY SUCCESS FACTORS

### 1. Independence (Phase 1)

**C & D**: Dùng `revenue_temp` để không phụ thuộc A
- Generate: `Lognormal(18.5, 1.0)`
- Include column trong output
- E sẽ replace sau

**Result**: TẤT CẢ 5 người làm song song! 🎯

---

### 2. Standards Consistency

**Industry Codes** (CRITICAL!):
- A sinh financial cho 3 ngành: 46, 10, 56
- E sinh demographic cho 3 ngành: 46, 10, 56
- PHẢI match!

**Sample Count**:
- Tất cả: 3,000 samples
- Mỗi ngành: 1,000 samples
- Không shuffle!

---

### 3. Quality > Speed

**Don't rush**:
- QA kỹ before delivery
- Ask questions early
- Review each other's work

---

## 🆘 SUPPORT & TROUBLESHOOTING

### Common Issues

**1. "Tôi không biết bắt đầu từ đâu?"**
→ Đọc SOP của bạn section-by-section  
→ Bắt đầu từ Giai Đoạn 1, Công Việc 1.1

**2. "Distribution không match expected?"**
→ Visual check OK > perfect parameters  
→ Adjust và regenerate nếu quá khác

**3. "Missing data nhiều?"**
→ Check sources và parsing logic  
→ Accept < 30% missing trong extraction

**4. "Correlation không đúng?"**
→ Adjust weights trong formula  
→ Target ranges, không cần exact

### Getting Help

**Trong Team**:
1. Ask in group chat
2. B có thể hỗ trợ sớm (finish early)
3. Peer review code/data

**External**:
1. Check `SOP_Luong[X]_*.md` (old SOPs) for reference
2. Google/Stack Overflow
3. Ask supervisor

---

## ✅ CHECKLIST TỔNG THỂ

### Phase 1: Individual Outputs

- [ ] **A**: `synthetic_financial_3k.csv` delivered
- [ ] **B**: `district_lookup.csv` + `industry_risk.csv` delivered
- [ ] **C**: `synthetic_cic_3k.csv` delivered (with revenue_temp)
- [ ] **D**: `synthetic_transaction_3k.csv` delivered (with revenue_temp)
- [ ] **E**: `synthetic_demographic_3k.csv` delivered

### Phase 2: Integration

- [ ] **E**: All inputs received and verified
- [ ] **E**: Revenue fix completed (C & D)
- [ ] **E**: Master merge successful
- [ ] **E**: Lookups enriched (B)
- [ ] **E**: Target variable generated (default ~5%)
- [ ] **E**: Train/val/test split (2.1k/450/450)
- [ ] **Team**: QA validation passed
- [ ] **E**: Documentation completed

### Final Delivery

- [ ] 4 CSV files: train, val, test, full
- [ ] Data dictionary
- [ ] Documentation report
- [ ] QA report
- [ ] Ready for model training! 🎉

---

## 📚 DOCUMENT INDEX

| File | Purpose | Audience |
|------|---------|----------|
| `README_5People.md` | Overview & coordination | All team |
| `SOP_NguoiA.md` | A's detailed instructions | Person A |
| `SOP_NguoiB.md` | B's detailed instructions | Person B |
| `SOP_NguoiC.md` | C's detailed instructions | Person C |
| `SOP_NguoiD.md` | D's detailed instructions | Person D |
| `SOP_NguoiE.md` | E's detailed instructions | Person E |
| `Gantt_Timeline.md` | Timeline visualization | All (reference) |
| `Dependencies.md` | Dependency analysis | All (technical) |

---

## 🎯 FINAL NOTES

### Division of Labor Philosophy

**Minimize dependencies** → **Maximize parallelization**

Key innovation: **revenue_temp approach**
- C & D don't wait for A
- E fixes during integration
- Result: 100% parallel Phase 1! ⚡

### Quality Expectations

- **Realistic** > Perfect
- **Documented** > Hidden
- **Consistent** > Optimized

### Team Spirit

- Help each other
- Share code & learnings
- Celebrate milestones
- Ask questions early!

---

**Let's build a great dataset together! 💪**

Questions? Check your individual SOP or ask in team chat.

Good luck! 🍀
