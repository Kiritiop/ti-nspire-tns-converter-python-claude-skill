---
name: file-to-tns
description: >
  Convert any document file (PDF, TXT, DOCX, MD, CSV, XLSX, HTML, RTF) into a
  TI-Nspire .tns Notes file that can be transferred to a TI-Nspire calculator.
  Use this skill whenever the user wants to put notes, reference material,
  answers, study guides, or any text content onto their TI-Nspire calculator.
  Triggers include: "convert to tns", "put this on my calculator",
  "make a tns file", "TI-Nspire", "nspire", ".tns", or any request to
  transfer a document to a graphing calculator.
---

# File → TI-Nspire .tns Converter

Converts documents of any common format into `.tns` Notes files readable on
a TI-Nspire calculator (OS 3.x and later), using Luna as the backend.

---

## Step 0 — Ensure Luna is available

Luna is the **only** tool that produces `.tns` files compatible with modern
TI-Nspire OS (3.x+). TI uses proprietary binary XML compression + DES
encryption that must be produced by Luna; plain ZIP tricks do not work.

```python
import os, subprocess, shutil, urllib.request, zipfile

LUNA_SEARCH = [
    os.path.expanduser("~/.luna_build/luna"),
    os.path.expanduser("~/Desktop/Coding/Luna/luna"),
    os.path.expanduser("~/Luna/luna"),
    os.path.join(os.path.dirname(os.path.abspath(__file__)), "luna"),
    shutil.which("luna") or "",
]
LUNA_BUILD_DIR = os.path.expanduser("~/.luna_build")
LUNA_BIN       = os.path.join(LUNA_BUILD_DIR, "luna")

def find_or_build_luna():
    # 1. Check known locations
    for p in LUNA_SEARCH:
        if p and os.path.isfile(p) and os.access(p, os.X_OK):
            return p
    # 2. Download + build from source
    os.makedirs(LUNA_BUILD_DIR, exist_ok=True)
    zip_path = os.path.join(LUNA_BUILD_DIR, "luna_src.zip")
    import ssl, certifi
    ctx = ssl.create_default_context(cafile=certifi.where())
    with urllib.request.urlopen(
        "https://github.com/ndless-nspire/Luna/archive/refs/heads/master.zip",
        context=ctx) as resp:
        open(zip_path, "wb").write(resp.read())
    src = os.path.join(LUNA_BUILD_DIR, "src")
    with zipfile.ZipFile(zip_path) as z:
        z.extractall(src)
    luna_src = os.path.join(src, "Luna-master")
    subprocess.run(["make"], cwd=luna_src, check=True)
    shutil.copy2(os.path.join(luna_src, "luna"), LUNA_BIN)
    return LUNA_BIN
```

If build fails on macOS: `brew install zlib` then retry.
On Linux: `sudo apt install zlib1g-dev` then retry.

---

## Step 1 — Extract text from the input file

Detect format by extension and use the appropriate extractor:

```python
def extract_text(path: str) -> str:
    ext = os.path.splitext(path)[1].lower()

    if ext == ".txt" or ext == ".md":
        return open(path, encoding="utf-8", errors="replace").read()

    if ext == ".pdf":
        # Try pdfplumber first (best layout), fall back to pypdf
        try:
            import pdfplumber
            pages = []
            with pdfplumber.open(path) as pdf:
                for i, page in enumerate(pdf.pages):
                    t = page.extract_text()
                    if t and t.strip():
                        pages.append(f"=== Page {i+1} ===\n{t.strip()}")
            if pages:
                return "\n\n".join(pages)
        except ImportError:
            pass
        from pypdf import PdfReader
        reader = PdfReader(path)
        pages = []
        for i, page in enumerate(reader.pages):
            t = page.extract_text()
            if t and t.strip():
                pages.append(f"=== Page {i+1} ===\n{t.strip()}")
        return "\n\n".join(pages)

    if ext == ".docx":
        from docx import Document
        doc = Document(path)
        return "\n".join(p.text for p in doc.paragraphs if p.text.strip())

    if ext in (".xlsx", ".xls"):
        import openpyxl
        wb = openpyxl.load_workbook(path, read_only=True, data_only=True)
        lines = []
        for ws in wb.worksheets:
            lines.append(f"=== Sheet: {ws.title} ===")
            for row in ws.iter_rows(values_only=True):
                cells = [str(c) if c is not None else "" for c in row]
                if any(c.strip() for c in cells):
                    lines.append(" | ".join(cells))
        return "\n".join(lines)

    if ext == ".csv":
        import csv
        lines = []
        with open(path, newline="", encoding="utf-8", errors="replace") as f:
            for row in csv.reader(f):
                lines.append(" | ".join(row))
        return "\n".join(lines)

    if ext in (".html", ".htm"):
        from html.parser import HTMLParser
        class P(HTMLParser):
            def __init__(self):
                super().__init__()
                self.parts = []
            def handle_data(self, d):
                if d.strip():
                    self.parts.append(d.strip())
        p = P()
        p.feed(open(path, encoding="utf-8", errors="replace").read())
        return "\n".join(p.parts)

    if ext == ".rtf":
        # Strip RTF control words with regex
        import re
        raw = open(path, encoding="utf-8", errors="replace").read()
        raw = re.sub(r'\\[a-z]+\d* ?', ' ', raw)
        raw = re.sub(r'[{}\\]', '', raw)
        return re.sub(r'  +', ' ', raw).strip()

    # Fallback: read as plain text
    return open(path, encoding="utf-8", errors="replace").read()
```

---

## Step 2 — Split text into ≤11,500 char chunks

The TI-Nspire OS rejects/hangs on problems larger than ~12 KB uncompressed.
Split on natural boundaries (paragraphs, sections) where possible.

```python
MAX_CHARS = 11500

def split_text(text: str) -> list:
    if len(text) <= MAX_CHARS:
        return [text]
    # Try to split at blank lines
    import re
    paragraphs = re.split(r'\n{2,}', text)
    chunks, current = [], ""
    for para in paragraphs:
        if len(current) + len(para) + 2 <= MAX_CHARS:
            current += ("\n\n" if current else "") + para
        else:
            if current:
                chunks.append(current)
            # Para itself too large — hard split
            if len(para) > MAX_CHARS:
                for i in range(0, len(para), MAX_CHARS):
                    chunks.append(para[i:i+MAX_CHARS])
                current = ""
            else:
                current = para
    if current:
        chunks.append(current)
    return chunks
```

---

## Step 3 — Build Problem XML and call Luna

Each chunk becomes one `ProblemN.xml` file fed to Luna.

```python
import textwrap, tempfile
from xml.sax.saxutils import escape as xml_escape

WRAP = 48   # chars; Nspire CX screen ~53 chars wide

def fmtxt(text: str) -> str:
    paras = []
    for line in text.splitlines():
        if not line.strip():
            paras.append(
                '<node name="1para"><node name="1rtline">'
                '<leaf name="1word"> </leaf></node></node>')
            continue
        for wrapped in textwrap.wrap(line, width=WRAP) or [line]:
            leaves = "".join(
                f'<leaf name="1word">{xml_escape(w)} </leaf>'
                for w in wrapped.split()
            )
            paras.append(
                f'<node name="1para"><node name="1rtline">'
                f'{leaves}</node></node>')
    r2d = ('<r2dtotree><node name="1doc">'
           + "".join(paras)
           + '</node></r2dtotree>')
    return xml_escape(r2d)

def problem_xml(text: str, name: str = "") -> str:
    return (
        '<?xml version="1.0" encoding="UTF-8" ?>\n'
        f'<prob xmlns="urn:TI.Problem" ver="1.0" pbname="{xml_escape(name)}">'
        '<sym></sym>'
        '<card clay="0" h1="10000" h2="10000" w1="10000" w2="10000">'
        '<isDummyCard>0</isDummyCard><flag>0</flag>'
        '<wdgt xmlns:np="urn:TI.Notepad" type="TI.Notepad" ver="2.0">'
        '<np:mFlags>1024</np:mFlags>'
        f'<np:value>{len(text)}</np:value>'
        f'<np:fmtxt>{fmtxt(text)}</np:fmtxt>'
        '</wdgt></card></prob>'
    )

def convert_to_tns(text: str, out_path: str, doc_name: str = "notes"):
    luna = find_or_build_luna()
    chunks = split_text(text)
    with tempfile.TemporaryDirectory() as tmp:
        xml_files = []
        for i, chunk in enumerate(chunks):
            name = f"p{i+1}" if len(chunks) > 1 else doc_name[:20]
            xml_path = os.path.join(tmp, f"Problem{i+1}.xml")
            open(xml_path, "w", encoding="utf-8").write(problem_xml(chunk, name))
            xml_files.append(xml_path)
        result = subprocess.run(
            [luna] + xml_files + [out_path],
            capture_output=True, text=True)
        if result.returncode != 0:
            raise RuntimeError(f"Luna failed:\n{result.stdout}\n{result.stderr}")
    return len(chunks)
```

---

## Step 4 — Wire it together (entry point)

```python
def file_to_tns(input_path: str, output_path: str = None) -> str:
    if not os.path.isfile(input_path):
        raise FileNotFoundError(input_path)
    doc_name = os.path.splitext(os.path.basename(input_path))[0]
    if output_path is None:
        output_path = doc_name + ".tns"

    print(f"[1/3] Extracting text from {input_path} ...")
    text = extract_text(input_path)
    if not text.strip():
        text = "(No extractable text found in this file.)"
    print(f"       {len(text):,} chars extracted")

    print(f"[2/3] Building .tns ...")
    n = convert_to_tns(text, output_path, doc_name)

    size = os.path.getsize(output_path)
    print(f"[3/3] Done → {output_path}  ({size:,} bytes, {n} problem(s))")
    return output_path
```

---

## Supported Formats

| Extension       | Library needed          | Notes                          |
|-----------------|-------------------------|--------------------------------|
| .txt / .md      | none (stdlib)           | Direct read                    |
| .pdf            | pdfplumber or pypdf     | Text PDFs only; scans need OCR |
| .docx           | python-docx             | Paragraphs only (no tables)    |
| .xlsx / .xls    | openpyxl                | All sheets, pipe-delimited     |
| .csv            | csv (stdlib)            | Pipe-delimited rows            |
| .html / .htm    | html.parser (stdlib)    | Strips all tags                |
| .rtf            | re (stdlib)             | Basic RTF stripping            |

Install all at once:
```bash
pip install pdfplumber pypdf python-docx openpyxl
```

---

## Limitations

- **Scanned PDFs**: no text layer → output will be empty. Use OCR first.
- **Multi-page calculator limit**: each problem ≤ ~12 KB uncompressed.
  Large files are automatically split across multiple problems (pages).
- **Math symbols / special chars**: renders as plain text approximations.
- **Images**: not included — TI.Notepad is text-only.
- **OS requirement**: Luna output requires TI-Nspire OS 3.0.2 or later.
