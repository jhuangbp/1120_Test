# Marine Revenue FY20-FY24 data quality notes

This note compares the delivered CSV extracts against the expected PDF contents at a high level and highlights areas to improve before re-running the extractor against the PDF.

## Coverage gaps
- The README describes a 7,309-row detail table and 197 total pages processed, but the current `Marine_Revenue_FY20-FY24_detail.csv` contains only 7,113 rows (including the header), and the FY-only file has 196 data rows. This suggests some pages or records may have been skipped during extraction.

## Field completeness issues (detail file)
- Missing month values: 13 rows lack a `Month` value, which makes those revenue lines hard to place chronologically and impossible to aggregate by period.
- Missing revenue values: 84 rows have empty `Revenue`, and 25 of those still have `NAFI Amt` empty as well. This may indicate OCR or column alignment errors on specific pages.
- Duplicate rows: at least three (Page, Loc #, Location, Month) combinations appear twice—for example, the Hansen TLF location appears with empty months on pages 85, 127, and 171. These may be header fragments that should be filtered out.

## FY-only rows
- The FY-only extract contains only 15 distinct `Location` values, most of which are the FY markers (e.g., `FY16`, `FY17`, ...). If the PDF contains separate subtotal lines for each location, these may need to be merged back into the main detail file or expanded per location.

## Recommendations
- Re-run the PDF extractor on the skipped/short pages to recover the missing ~196 detail rows noted in the README expectations.
- Investigate the 13 rows with blank `Month` values and 84 rows with blank `Revenue`, paying special attention to the Hansen TLF pages where duplicates appear; the PDF may need special-case parameters there.
- Consider merging or properly labeling the FY-only rows so they capture the intended subtotals rather than appearing as orphaned FY markers.
