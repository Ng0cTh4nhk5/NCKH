# Dependency Matrix - 5 Person Team

> **Mục đích**: Làm rõ ai phụ thuộc vào ai, workflow dependencies

---

## 📊 DEPENDENCY TABLE

| Người | Output | Được dùng bởi | Phase | Notes |
|-------|--------|---------------|-------|-------|
| **A** | `synthetic_financial_3k.csv` | E (Integration) | Phase 2 | Critical for correlation fix |
| **B** | `district_lookup.csv` | E (Integration) | Phase 2 | For enrichment |
| **B** | `industry_risk.csv` | E (Integration) | Phase 2 | For enrichment |
| **C** | `synthetic_cic_3k.csv` | E (Integration) | Phase 2 | Includes revenue_temp |
| **D** | `synthetic_transaction_3k.csv` | E (Integration) | Phase 2 | Includes revenue_temp |
| **E** | `synthetic_demographic_3k.csv` | E (Integration - Self) | Phase 2 | Own phase 1 output |

---

## 🔗 DETAILED DEPENDENCIES

### Người A Dependencies

**Depends ON**: KHÔNG AI
- Làm việc hoàn toàn độc lập
- Không cần output từ người khác

**Depended BY**: C, D (indirect), E (direct)
- **C & D**: Cần revenue cho correlation → dùng revenue_temp để độc lập
- **E**: Cần revenue THẬT để replace revenue_temp

**Phase**: Output cần có trước Phase 2 (Integration)

---

### Người B Dependencies

**Depends ON**: KHÔNG AI
- Làm việc hoàn toàn độc lập
- Dữ liệu từ nguồn công khai

**Depended BY**: E
- E cần 2 lookup tables để enrich

**Phase**: Output có thể sớm → B free to support team

---

### Người C Dependencies

**Depends ON**: KHÔNG AI (trong generation phase)
- Dùng `revenue_temp` để tạo correlation
- Không cần chờ A

**Depended BY**: E
- E sẽ replace revenue_temp bằng revenue thật từ A

**Special**: 
- Output của C CHỨA revenue_temp column
- E phải process để replace

**Phase**: Output cần có trước Phase 2

---

### Người D Dependencies

**Depends ON**: KHÔNG AI (trong generation phase)
- Dùng `revenue_temp` (giống C)
- Không cần chờ A

**Depended BY**: E
- E sẽ replace revenue_temp bằng revenue thật từ A

**Special**:
- Output của D CHỨA revenue_temp column
- E phải process để replace

**Phase**: Output cần có trước Phase 2

---

### Người E Dependencies

**Phase 1 (Demographic)**:
**Depends ON**: KHÔNG AI
- Generate demographic độc lập

**Phase 2 (Integration)**:
**Depends ON**: TẤT CẢ (A, B, C, D, E phase 1)
- Cần financial từ A
- Cần lookups từ B
- Cần CIC từ C
- Cần transaction từ D
- Cần demographic từ chính mình

**Depended BY**: KHÔNG AI
- E tạo final outputs
- Không ai depended

**Phase**: Phase 2 BẮT BUỘC chờ tất cả outputs ready

---

## 🎯 INDEPENDENCE STRATEGY

### Vấn Đề Gốc

CIC và Transaction features cần correlate với revenue:
- `total_outstanding_debt` ∝ revenue (r=0.6)
- `avg_daily_balance` ∝ revenue (r=0.7)
- `avg_monthly_deposits` ≈ revenue/12

**Nếu không xử lý** → C & D phải CHỜ A xong → Sequential!

---

### Giải Pháp: Revenue_Temp

**C & D generate revenue_temp**:
```python
revenue_temp = np.random.lognormal(18.5, 1.0, 3000)
```

**Parameters giống như A sẽ dùng** → Distribution tương tự

**Lợi ích**:
- ✅ C & D làm việc độc lập
- ✅ Correlation structure được tạo ngay
- ✅ Không chờ A

---

### Integration Phase: Fix Revenue

**E sẽ làm** (Phase 2):

```python
# 1. Load real revenue from A
real_revenue = financial_df['revenue']

# 2. Replace trong C's output
cic_df['revenue'] = real_revenue
cic_df = cic_df.drop('revenue_temp', axis=1)

# 3. Recalculate debt_burden_ratio (if needed)
cic_df['debt_burden_ratio'] = cic_df['total_outstanding_debt'] / (real_revenue / 12)

# 4. Replace trong D's output
transaction_df['revenue'] = real_revenue
transaction_df = transaction_df.drop('revenue_temp', axis=1)
```

**Kết quả**:
- Giữ correlation structure (vì distribution tương tự)
- Dùng revenue thật cho consistency
- Fix any recalculated features

---

## 📊 DEPENDENCY GRAPH (Visual)

```
PHASE 1: GENERATION (NO CROSS-DEPENDENCIES)
┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐     ┌─────┐
│  A  │     │  B  │     │  C  │     │  D  │     │  E  │
│BCTC │     │Refs │     │ CIC │     │Trans│     │Demo │
└─────┘     └─────┘     └─────┘     └─────┘     └─────┘
   │           │           │           │           │
   │           │           │           │           │
   └───────────┴───────────┴───────────┴───────────┘
                         │
                         ▼
               ┌─────────────────┐
               │   PHASE 2:      │
               │  INTEGRATION    │
               │   (Người E)     │
               └─────────────────┘
                         │
                         ▼
               ┌─────────────────┐
               │  FINAL OUTPUTS  │
               │ train/val/test  │
               └─────────────────┘
```

---

## ✅ ZERO-DEPENDENCY CHECKLIST

| Requirement | Solution | Status |
|-------------|----------|--------|
| C needs revenue | Use revenue_temp | ✅ Solved |
| D needs revenue | Use revenue_temp | ✅ Solved |
| E needs all inputs | Wait Phase 2 | ✅ Acceptable (unavoidable) |
| Industry must match | A & E both use same 3 industries | ✅ Coordinated |
| Consistent sample count | All generate 3,000 | ✅ Standard |

---

## 🚀 PARALLELIZATION PROOF

**Theorem**: Phase 1 can run 100% parallel

**Proof**:
1. A depends on: {} → Independent ✅
2. B depends on: {} → Independent ✅
3. C depends on: {} (uses revenue_temp) → Independent ✅
4. D depends on: {} (uses revenue_temp) → Independent ✅
5. E (phase 1) depends on: {} → Independent ✅

**Conclusion**: All 5 can work simultaneously! QED. □

---

## ⚠️ COORDINATION POINTS

Although independent, team still needs to coordinate on:

### 1. Standards
- File naming conventions
- Column naming
- Data types (int vs float)
- Missing value encoding (null vs 0)

### 2. Industry Codes (Critical!)
- A & E MUST use same 3 industries
- VSIC codes: 46, 10, 56
- 1,000 samples each
- **Verify before generation!**

### 3. Sample Count
- All generate exactly 3,000
- Rows 1-3000 (ordered)
- No shuffling before delivery

### 4. Revenue Distribution
- C & D use Lognormal(18.5, 1.0) for revenue_temp
- Same as A will use
- **Critical for correlation fix to work!**

---

## 📞 COMMUNICATION PROTOCOL

### Regular Sync
- Progress updates
- Blockers
- Questions

### File Delivery
When completing output:
1. Verify row count = 3,000
2. Verify no missing (except expected)
3. Notify E on Slack/Email
4. Upload to shared folder

### Integration Kickoff (Phase 2)
- E confirms all inputs ready
- Team meeting to review
- Support assignments
- QA plan

---

## ✅ SUCCESS CRITERIA

**Phase 1** (Individual):
- [ ] Each person delivers on time
- [ ] All outputs 3,000 rows
- [ ] No blockers due to dependencies

**Phase 2** (Integration):
- [ ] E receives all inputs
- [ ] Revenue fix successful
- [ ] Merge completes without errors
- [ ] Final datasets validated

---

**KEY TAKEAWAY**:
🎯 **ZERO dependencies in Phase 1 = Maximum efficiency!**  
🔄 **Inevitable dependency in Phase 2 = Mitigated by team support!**
