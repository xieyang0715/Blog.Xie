---
title: 处理excel表格
tags:
  - python常用模块
photos:
  - http://myapp.img.mykernel.cn/123123zzzzzzzzzz.png
date: 2021-11-14 11:13:23
---

# 前言

处理excel的常用库:  `csv`，`pandas`, `xlwt/xlrd`, `openpyxl`

pandas处理excel: https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.to_json.html

openpyxl处理: https://stackoverflow.com/questions/42102690/valueerror-row-or-column-values-must-be-at-least-1-when-using-openpyxl

以下是我处理的excel的2个小示例

<!--more-->



# openpyxl

openpyxl处理: https://stackoverflow.com/questions/42102690/valueerror-row-or-column-values-must-be-at-least-1-when-using-openpyxl

```python
from openpyxl import load_workbook

wb = load_workbook(name)
ws = wb.active
ws.insert_rows(0)
ws.cell(row=1, column=1).value = 'src_text'
ws.cell(row=1, column=2).value = 'tgt_text'
```

# pandas

pandas处理excel: https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.to_json.html

```python
   excel_data_df = pandas.read_excel(name, sheet_name='Sheet1')

    result = excel_data_df.to_json(orient='records')
    parsed = json.loads(result)

    with open(f"{name}.json", 'w', encoding='utf-8') as f:
        for line in parsed:
            f.write(json.dumps(line,ensure_ascii=False)+'\n')
        # json.dump(parsed, f, ensure_ascii=False, indent=1)
```

