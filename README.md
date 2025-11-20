# Marine Revenue FY20-FY24 資料處理說明

## 專案概述

本專案從 PDF 文件 (`Marine_Revenue_FY20-FY24.pdf`) 中提取美國海軍陸戰隊各基地的槽機收入數據,並輸出為結構化的 CSV 檔案。

## 資料來源

- **PDF 檔案**: Marine_Revenue_FY20-FY24.pdf
- **下載方式**: 透過 Google Drive (gdown 套件)
- **包含資料**: FY16-FY24 年度各基地槽機收入及 NAFI 分攤金額

## 輸出檔案

1. **Marine_Revenue_FY20-FY24_summary_table.csv** - 匯總表
2. **Marine_Revenue_FY20-FY24_detail.csv** - 詳細月度數據表

---

## 處理流程架構

### 1️⃣ 環境準備 (Preliminary Setup)

#### 安裝依賴套件
```python
pip install pdfplumber tabula-py
```

**核心套件功能:**
- `pdfplumber`: PDF 文字提取,支援字元層級精確定位
- `tabula-py`: PDF 表格解析輔助工具
- `pandas`: 數據處理與清洗
- `numpy`: 數值運算

#### 下載資料
```python
gdown.download(url, 'Marine_Revenue_FY20-FY24.pdf')
```

---

## 📊 Summary Table 提取邏輯

### 目標頁面
從 **5 個特定頁面** 提取槽機收入匯總表:
- 第 1, 35, 73, 115, 157 頁

### 提取欄位
| 欄位 | 說明 |
|------|------|
| Country | 國家 (Japan/Korea) |
| Installation | 基地名稱 |
| FY16-FY19 | 各財政年度收入 |
| FY20 thru SEP | FY20 至 9 月收入 |
| Annualized FY20 | FY20 年化收入 |

### 處理步驟

#### 1. 字元層級提取
```
PDF Characters → Lines (垂直分組) → Words (水平分組)
```

**chars_to_lines()**:
- 按 Y 座標分組字元為行 (`y_tol=3`)
- 合併垂直距離 ≤3 的字元

**line_to_words()**:
- 按 X 座標分組字元為詞
- 合併水平間距 ≤3 的字元

#### 2. 表頭自動檢測
```python
find_header_index(lines_words)
```
尋找包含 "Country", "Installation", "FY" 關鍵字的行

#### 3. 列分割線計算
```python
detect_cuts_from_header(header_words)
```

**處理邏輯:**
- 辨識 "CountryInstallation" 合併標頭 → 按比例分割
- 辨識各 FY 欄位中點 → 計算相鄰欄位分界線
- 若標頭不完整 → 回退使用等寬分割

#### 4. 資料行提取與修復
```python
assign_row(words, cuts)  # 按列分割線分配詞彙
fix_country_installation(row)  # 修復合併欄位
```

**Country/Installation 分離邏輯:**
```
"JapanCampFuji" → ["Japan", "Camp Fuji"]
"KoreaCampMujuk" → ["Korea", "Camp Mujuk"]
```

#### 5. 資料標準化
```python
normalize_country()   # "japan" → "Japan"
normalize_installation()  # "campfuji" → "Camp Fuji"
money_to_float()  # "$(1,234.56)" → -1234.56
```

**金額轉換規則:**
- 移除逗號、美元符號
- 括號表示負數: `(123)` → `-123`

---

## 📈 Detail Table 提取邏輯

### 目標頁面
第 2-202 頁,**排除** 匯總表頁面 (1, 35, 73, 115, 157)

### 提取欄位
| 欄位 | 說明 |
|------|------|
| Page | 來源頁碼 |
| Loc # | 地點編號 |
| Location | 地點名稱 |
| Month | 月份 (格式: Jan-20) |
| Revenue | 月收入 |
| NAFI Amt | NAFI 分攤金額 |
| Annual Revenue | 年度累計收入 |
| Annual NAFI | 年度累計 NAFI |

### 核心處理流程

#### 階段 1: 參數化頁面處理
```python
SPECIAL_PARAMS = {
    range(163, 167): {...},  # 頁面 163-166 特殊參數
    range(172, 178): {...},  # 頁面 172-177 特殊參數
    range(178, 191): {...},  # 頁面 178-190 特殊參數
}
```

**動態參數:**
- `Y_TOL`: 垂直行合併容差 (3.0 - 6.5)
- `X_JOIN`: 水平字元連接容差 (2.6 - 5.0)
- `GAP_RATIO`: 智慧空格插入閾值 (0.52 - 0.8)
- `EDGE_PAD`: 列邊界擴展像素 (8 - 20)
- `LEFT_SHIFT_MONTH/REVENUE`: 特定欄位邊界左移調整

#### 階段 2: 表頭檢測與列邊界計算
```python
cuts_from_header(words, base_cuts, prev_cuts)
```

**三層檢測策略:**
1. **完整標頭行檢測**: 尋找包含 ≥3 個標頭關鍵字的行
2. **散落標頭檢測**: 全頁搜尋各欄位標題的最左位置
3. **回退機制**: 使用前一頁或基準列分割線

**列邊界計算:**
```
相鄰欄位標頭中點 = (左欄X + 右欄X) / 2
應用左移調整 (Month 和 Revenue 欄位)
```

#### 階段 3: 類型感知邊界優化
```python
refine_cuts_typeaware(page, cuts, header_y)
```

**統計方法調整邊界:**
1. 收集所有月份 token 的右邊界 (`month_right`)
2. 收集所有金額 token 的左邊界 (`revenue_left`)
3. 計算安全分界線: `(max(month_right) + min(revenue_left)) / 2`
4. 限制調整幅度在 `CLAMP_DELTA` 範圍內

**處理的邊界:**
- Month ↔ Revenue
- Revenue ↔ NAFI Amt
- NAFI Amt ↔ Annual Revenue
- Annual Revenue ↔ Annual NAFI

#### 階段 4: 資料行提取
```python
rows_from_page(page, cuts, header_y, params...)
```

**流程:**
1. 提取字元 → 分組為行(垂直) → 分組為詞(水平)
2. 過濾標頭以下的正文內容
3. 按列邊界將詞彙分配到各欄位桶
4. 智慧連接詞彙 (`smart_join`)

**smart_join 特殊處理:**
```python
"123" + "," + "456" → "123,456"  # 數字與逗號無空格
"Jan" + "-" + "20" → "Jan-20"    # 月份格式
"Camp" + "Fuji" → "Camp Fuji"    # 一般詞彙加空格
```

#### 階段 5: 資料修復與標準化

##### A. Loc# 與 Location 修復
```python
repair_loc_and_location(df)
```

**處理情境:**
1. **合併分離**: `"123 CampName"` → Loc#=`123`, Location=`CampName`
2. **6位碼附加**: 純數字的 Location 附加到上一行
3. **跨欄溢出**: Loc# 欄的6位碼移到上一行 Location

##### B. Month/Revenue 欄位交換
```python
split_and_swap_month_revenue(df)
```

**情境範例:**
```
Month="123,456" Revenue="Jan-20"
→ Month="Jan-20" Revenue="123456"
```

##### C. 月份格式標準化
```python
normalize_months(df)
```
```
"20-Jan" → "Jan-20"  # 反轉格式
```

##### D. 跨頁前向填充
```python
pagewise_seeded_ffill(df, prev_locnum, prev_loc)
```

**維護連續性:**
- 使用前一頁最後的 Loc# 和 Location 作為種子
- 填充當前頁開頭的空白欄位
- 確保跨頁資料的基地資訊連續

##### E. 月度數據左對齊
```python
monthly_numeric_left_pack(df)
```

**針對月度行:**
- 收集 4 個數值欄位的所有非空值
- 從左到右依序填入: Revenue → NAFI Amt → Annual Revenue → Annual NAFI

#### 階段 6: 最終清理
```python
finalize(df)
```

1. **年份碎片修復**: `Month="Jan" + Revenue="-20"` → `Month="Jan-20"`
2. **全局前向填充**: Loc# 和 Location
3. **金額標準化**:
   - 移除空格、逗號、美元符號
   - 括號轉負號: `(123)` → `-123`
   - 轉換為數值類型

---

## 🔧 技術特點

### 1. 自適應參數系統
針對不同頁面佈局動態調整處理參數,解決 PDF 格式不一致問題。

### 2. 多層次容錯機制
- 標頭檢測失敗 → 使用前一頁參數
- 前一頁無參數 → 使用基準參數
- 基準參數失敗 → 等寬分割

### 3. 類型感知處理
利用資料類型特徵(月份格式、金額格式)進行:
- 列邊界優化
- 欄位錯位修復
- 資料驗證

### 4. 跨頁狀態維護
追蹤前一頁的:
- Loc# / Location (用於前向填充)
- 列邊界 (用於下一頁初始化)

---

## 📤 輸出資料

### Summary Table (97 rows)
各基地歷年槽機收入匯總

### Detail Table (7,309 rows)
月度詳細收入數據,包含:
- 約 200 頁的數據
- 跳過 5 個匯總表頁面
- 每頁平均 35-40 筆記錄

---

## 🚀 使用方式

```python
# 執行整個筆記本
# 1. Preliminary Setup cell
# 2. Summary Table extraction cell
# 3. Detail Table extraction cell
```

輸出檔案位於 PDF 同目錄:
- `Marine_Revenue_FY20-FY24_summary_table.csv`
- `Marine_Revenue_FY20-FY24_detail.csv`

---

## 🛠️ 主要函數說明

### Summary Table
| 函數 | 功能 |
|------|------|
| `chars_to_lines()` | 字元垂直分組為行 |
| `line_to_words()` | 行內字元水平分組為詞 |
| `find_header_index()` | 自動檢測表頭行 |
| `detect_cuts_from_header()` | 計算列分割線 |
| `assign_row()` | 按列分割分配詞彙 |
| `fix_country_installation()` | 修復 Country/Installation 合併 |
| `normalize_country/installation()` | 標準化國家與基地名稱 |
| `money_to_float()` | 金額字串轉浮點數 |

### Detail Table
| 函數 | 功能 |
|------|------|
| `apply_page_params()` | 獲取頁面特定參數 |
| `chars_to_words()` | 提取頁面詞彙 (帶容差參數) |
| `cuts_from_header()` | 多策略表頭檢測與列邊界計算 |
| `refine_cuts_typeaware()` | 基於資料類型優化邊界 |
| `rows_from_page()` | 提取與分組資料行 |
| `repair_loc_and_location()` | 修復地點欄位 |
| `split_and_swap_month_revenue()` | 修復月份/收入錯位 |
| `normalize_months()` | 標準化月份格式 |
| `pagewise_seeded_ffill()` | 跨頁前向填充 |
| `monthly_numeric_left_pack()` | 月度數據左對齊 |
| `finalize()` | 最終清理與類型轉換 |

---

## ⚠️ 注意事項

1. **頁面跳過**: Summary 提取會自動跳過第 1, 35, 73, 115, 157 頁
2. **Detail 提取範圍**: 處理第 2-202 頁,排除 Summary 頁面
3. **參數調整**: 特定頁面範圍使用自訂參數以提高準確性
4. **資料連續性**: 依賴前向填充維護跨頁 Location 資訊

---

## 📊 資料品質

- **Summary Table**: 97 筆記錄涵蓋 Japan/Korea 主要基地
- **Detail Table**: 7,309 筆月度記錄
- **處理成功率**: 約 197 頁成功提取資料 (排除 5 頁跳過)

---

## 開發者
處理邏輯基於 pdfplumber 字元層級提取,針對 Marine Revenue 報表格式優化。
