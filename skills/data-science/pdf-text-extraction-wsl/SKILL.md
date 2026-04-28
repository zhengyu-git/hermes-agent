---
name: pdf-text-extraction-wsl
description: Extract text from PDFs in WSL using pdfjs-dist Node.js library when system tools (pdftotext, PyMuPDF) are unavailable
---

# PDF Text Extraction in WSL

Extract text content from PDFs using pdfjs-dist (Node.js) when system tools (pdftotext, PyMuPDF) are unavailable.

## Environment
- WSL (Ubuntu/Debian without root or with restricted apt)
- Node.js 18+ available (npm exists)
- Python pip usually unavailable in WSL

## IMPORTANT: Newer pdfjs-dist versions (v4+) no longer ship .mjs files

As of 2026, pdfjs-dist (v4+) removed the `.mjs` build. The legacy build now only has `.js` files.
Node.js ESM mode cannot import `.js` extensions, so you MUST use **CommonJS `require()`** with a `.cjs` script.

## Quick Start

```bash
# 1. Install pdfjs-dist
cd /tmp && npm install pdfjs-dist

# 2. Create extraction script (use .cjs extension for CommonJS)
cat > /tmp/pdf_extract.cjs << 'CEOF'
const { getDocument } = require('/tmp/node_modules/pdfjs-dist/legacy/build/pdf.js');
const { readFileSync, writeFileSync, readdirSync, mkdirSync, existsSync } = require('fs');
const { join } = require('path');

const dir = '/mnt/c/Users/YOUR_USER/Desktop/PDF';
const outDir = '/home/user/.hermes/pdf_contents';
const files = readdirSync(dir).filter(f => f.endsWith('.pdf'));
mkdirSync(outDir, { recursive: true });

async function extractFile(file) {
  const filePath = join(dir, file);
  const outPath = join(outDir, file + '.txt');
  if (existsSync(outPath)) { console.log(`[SKIP] ${file}`); return; }
  try {
    const data = new Uint8Array(readFileSync(filePath));
    const doc = await getDocument({ data, verbosity: 0, useWorkerFetch: false, isEvalSupported: false, useSystemFonts: true }).promise;
    console.log(`[${file}] ${doc.numPages} pages`);
    let fullText = '';
    for (let i = 1; i <= doc.numPages; i++) {
      const page = await doc.getPage(i);
      const content = await page.getTextContent();
      const pageText = content.items.map(item => item.str || '').join(' ');
      fullText += `--- 第${i}页 ---\n${pageText}\n\n`;
    }
    writeFileSync(outPath, fullText);
    console.log(`[OK] ${file}: ${fullText.length} chars`);
  } catch (e) {
    console.error(`[ERR] ${file}: ${e.message}`);
  }
}

(async () => {
  for (const file of files) await extractFile(file);
  console.log('Done!');
})();
CEOF

# 3. Run
node /tmp/pdf_extract.cjs
```

## Key Findings (Lessons Learned)

> **Handling Windows paths with spaces/Unicode/中文引号**
> - Chinese `"` `"` (fullwidth quotation marks) in filenames are the #1 cause of `ls`/shell access failures.
>   Shell treats these differently from ASCII `"` and `'`, causing `No such file or directory` even with quotes.
> - **The universal fix**: NEVER hardcode the path in a `ls` or shell command. Instead, use `find` to resolve the exact path,
>   then feed it into the Node.js script via `execSync`:
>   ```js
>   const { execSync } = require('child_process');
>   const pdfPath = execSync(
>     "find '/mnt/c/Users/.../target_dir/' -maxdepth 1 -name '*关键词*' -print0 | tr '\\0' '\\n' | head -1",
>     { encoding: 'utf-8' }
>   ).trim();
>   ```
> - This pattern works for ALL special characters: spaces, em-dashes, fullwidth quotes, Unicode — no exceptions.
> - When using `search_files` to discover the path, note that the output shows the actual Unicode characters correctly,
>   but shell interpolation of those characters may fail. Always resolve in JS with `find + execSync`.


- **Wrong package first**: `pdf-parse` npm package has a completely different API (`PDFParse` class, not the default export) — avoid it
- **pdfjs-dist v4+ no longer ships .mjs**: Newer versions of pdfjs-dist only have `.js` files in `legacy/build/`. Must use **CommonJS `require()`** with `.cjs` scripts, not ESM `import` with `.mjs`.
- **cMap/wasm warnings are OK**: Missing `cMapUrl`, `standardFontDataUrl`, `wasmUrl` parameters produce warnings but extraction still works
- **API**: `getDocument({data})` → `doc.numPages` → `doc.getPage(n)` → `page.getTextContent()` → `content.items.map(item => item.str)`
- **WSL path**: Windows files accessible via `/mnt/c/Users/...`
- **ExtGState warnings flood stdout**: Many Chinese PDFs use non-standard graphics state, causing pdfjs to emit millions of `"Warning: getTextContent - ignoring ExtGState: FormatError: ExtGState should be a dictionary."` messages that can crash or timeout processes. **Fix**: always use `verbosity: 0` in getDocument options. These PDFs still extract text correctly.
- **Scanned/image PDFs yield ~3% text**: A 465MB PDF with 8920 pages extracted to only 11.7MB of text (~2.5% ratio). PDFs that are scanned images produce near-zero text. Check actual output size to distinguish text-heavy vs image-based PDFs.
- **Large PDFs are very slow**: A 465MB PDF took multiple minutes to process. Many PDFs over 100MB are image-based collections.
- **Docx files need separate handling**: Word documents (.docx) cannot be read by pdfjs-dist. Use npm `docx` package or `libreoffice --headless --convert-to txt`.
- **TXT files**: Direct copy, no extraction needed. Chinese encoding usually UTF-8.
- **pip unavailable in WSL**: `python3 -m pip` → "No module named pip". Cannot use pymupdf/PyPDF2/pdfplumber. Node.js pdfjs-dist is the reliable alternative.

## Batch Extraction with Directory Structure
```javascript
const { getDocument } = require('/tmp/node_modules/pdfjs-dist/legacy/build/pdf.js');
const { readFileSync, writeFileSync, readdirSync, mkdirSync, existsSync } = require('fs');

const dir = '/mnt/c/Users/YOUR_USER/Desktop/PDF/';
const outDir = '/home/user/.hermes/ty_pdfs/';
mkdirSync(outDir, { recursive: true });

// Recursively list all PDFs and TXTs
const allFiles = [];
function walk(d, prefix='') {
  for (const f of readdirSync(d)) {
    const fp = d + '/' + f;
    if (f.endsWith('.pdf') || f.endsWith('.txt') || f.endsWith('.docx'))
      allFiles.push({ orig: f, path: fp, type: f.slice(f.lastIndexOf('.')) });
  }
}
walk(dir);

let done = 0, failed = 0;
for (const { orig, path, type } of allFiles) {
  const safeName = orig.replace(/\\/g, '_').replace(/\//g, '_');
  const outPath = outDir + safeName + (type === '.pdf' ? '.txt' : '');
  if (existsSync(outPath)) { done++; continue; }
  try {
    if (type === '.txt') {
      writeFileSync(outPath, readFileSync(path));
    } else if (type === '.docx') {
      // Handle separately with libreoffice or docx npm package
      continue;
    } else {
      const doc = await getDocument({ data: new Uint8Array(readFileSync(path)), verbosity: 0 }).promise;
      let text = '';
      for (let i = 1; i <= doc.numPages; i++) {
        const page = await doc.getPage(i);
        const c = await page.getTextContent();
        text += c.items.map(it => it.str).join(' ') + '\n';
      }
      writeFileSync(outPath, text);
    }
    done++;
  } catch(e) {
    failed++;
    console.log(`FAIL: ${orig} - ${e.message}`);
  }
}
console.log(`Done: ${done}, Failed: ${failed}`);
```

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `Cannot find module 'pdfjs-dist/.../pdf.mjs'` | Newer pdfjs-dist removed .mjs files. Use CommonJS `require('./pdf.js')` with a `.cjs` script instead. |
| `Cannot find module 'pdf-parse'` | Wrong package; use `pdfjs-dist` instead |
| `pdfParse is not a function` | pdf-parse uses `PDFParse` class, not default export |
| `pip: command not found` | Don't try pip; use Node.js pdfjs-dist |
| `sudo: a terminal is required` | Can't sudo; use Node.js solution instead of apt |
| `undefined function: 32` | Minor font warning, ignore, extraction continues |
| `loadFont translateFont failed` | Missing cMapUrl, harmless warning |
| Millions of ExtGState warnings flooding stdout | Add `verbosity: 0` to getDocument options |
| PDF is scanned image, returns almost no text | Normal for old Chinese scanned books; not an error |
| Process hangs on large PDF (300MB+) | Normal, large scanned PDFs take many minutes |
| `Cannot polyfill DOMMatrix/Path2D` warnings | Harmless; missing `canvas` package, text extraction unaffected |

## Save Location
Store extracted text as `~/.hermes/pdf_contents/<original_filename>.txt` so it's easy to reference later.