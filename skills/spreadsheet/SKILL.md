---
name: spreadsheet
description: >
  Work with Excel, CSV, and data tables accurately.
  Use when: analyzing CSV data, converting Excel files, filtering/sorting tabular data.

# === TU-POC Extensions ===
suppliers:
  - source: user_files
    type: external
    description: "CSV/Excel files provided by user"

customers:
  - recipient: report_generator
    type: internal
    receives: "processed data for reports"

with_who:
  executor:
    model: gemini-2.5-flash
    capabilities: [tool_use, reasoning]

risk_profile:
  actions:
    - name: read_data
      rpn: 1
      autonomy: autonomous
    - name: write_data
      rpn: 4
      autonomy: autonomous
    - name: modify_production_data
      rpn: 10
      autonomy: dare

measures:
  - kpi: data_accuracy
    target: "100% correct transformations"

environmental:
  compute_cost: low
  data_retention: 90d

next_review: 2026-08-11
---

# Spreadsheet

Work with Excel, CSV, and tabular data.

## CSV Operations

```bash
# View CSV nicely
column -t -s, data.csv | head -20

# Filter rows
awk -F, '$3 > 1000 {print $1, $3}' data.csv

# Sort by column 3
sort -t, -k3 -n data.csv
```

## Python for Excel/CSV

```bash
# Summarize CSV
python3 -c "
import csv, sys
with open(sys.argv[1]) as f:
    rows = list(csv.DictReader(f))
    print(f'{len(rows)} rows, columns: {list(rows[0].keys()) if rows else \"empty\"}')
" data.csv

# Convert Excel to CSV
python3 -c "
import openpyxl, csv, sys
wb = openpyxl.load_workbook(sys.argv[1])
ws = wb.active
with open('/tmp/output.csv','w') as f:
    writer = csv.writer(f)
    for row in ws.iter_rows(values_only=True):
        writer.writerow(row)
print(f'Converted {ws.max_row} rows')
" data.xlsx
```

## Google Sheets (via gog)

```bash
gog sheets get SHEET_ID "Sheet1!A1:D10" --json
```

## Notes

- For Google Sheets, use `gog` skill.
- For complex analysis, use Python with pandas.
- Always validate data before writing.
