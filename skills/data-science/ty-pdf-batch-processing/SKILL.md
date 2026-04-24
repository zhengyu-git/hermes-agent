---
name: ty-pdf-batch-processing
description: 天涯神帖PDF批量处理流程——文件盘点、优先级排序、Node.js后台提取、进度汇报Cron
---

# 天涯神帖PDF批量处理流程

## 使用场景
大批量（500+）PDF文件提取文字，按分类优先级排序，自动后台运行，进度可查。

## 文件盘点（优先级排序）

```python
import os
from collections import defaultdict

src = '/mnt/c/Users/v-zhengyu002/Desktop/PDF/ty社区导入神贴/'
priority_order = [
    'kk大神',
    '缠中说禅',
    '懒兔子',
    '3-【金融财经】【股市楼市】【经济预测】',
    '天涯神贴/天涯全集（更全，但部分需解压）',
    '2-【人在江湖】【煮酒论史】【成长箴言】',
    '1-【莲蓬鬼话】【玄幻灵异】【中医命理】',
    '4-【故事小说】【其他】',
    '天涯合集',
    '天涯绝版',
]

def get_all_files(directory, extensions):
    results = []
    for root, dirs, files in os.walk(directory):
        for f in files:
            if any(f.lower().endswith(ext) for ext in extensions):
                results.append(os.path.join(root, f))
    return results

all_files = get_all_files(src, ['.pdf', '.txt', '.docx'])

# 去重（同名文件只保留一个）
seen = {}
deduped = []
for f in all_files:
    key = os.path.basename(f)
    if key not in seen:
        seen[key] = f
        deduped.append(f)

def get_priority(f):
    rel = f.replace(src, '')
    for i, p in enumerate(priority_order):
        if p in rel:
            return i
    return 99

deduped.sort(key=get_priority)
```

## Node.js处理脚本（关键注意事项）

- **必须先创建输出目录，再写日志**（否则ENOENT）
- pdfjs-dist安装在 `/tmp/node_modules/pdfjs-dist/legacy/build/pdf.mjs`
- 超过200MB的PDF跳过（基本是图片扫描集）
- 超过3000页的PDF跳过
- 每处理一个文件延迟200ms避免IO过载
- 提取结果保存为 `原文件名.txt`

## 进度汇报Cron

```bash
# 30分钟汇报一次
*/30 * * * *
```

prompt需读取 `~/.hermes/ty_pdfs/progress.log`，统计已处理/总数。

## 关键教训

1. Node.js writeFileSync在目录不存在时报ENOENT，先mkdirp(OUT)
2. 天涯神贴同一文件常出现在多个分类目录，必须去重
3. 超大PDF（>200MB）文本提取率极低（~2.5%），直接跳过
4. WSL网络限制：Gitee不通，GitHub畅通
