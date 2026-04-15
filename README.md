# ToTNS

A skill for Claude Code and OpenCode that converts any document (PDF, DOCX, TXT, XLSX, CSV, HTML, RTF, MD) into a `.tns` Notes file readable on a TI-Nspire graphing calculator.

## Installation

### Claude Code

Clone directly into Claude Code's skills directory:

```
mkdir -p ~/.claude/skills
git clone https://github.com/Kiritiop/file-to-tns.git ~/.claude/skills/file-to-tns
```

Or copy the skill file manually if you already have this repo cloned:

```
mkdir -p ~/.claude/skills/file-to-tns
cp SKILL.md ~/.claude/skills/file-to-tns/
```

### OpenCode

Clone directly into OpenCode's skills directory:

```
mkdir -p ~/.config/opencode/skills
git clone https://github.com/Kiritiop/file-to-tns.git ~/.config/opencode/skills/file-to-tns
```

> **Note:** OpenCode also scans `~/.claude/skills/` for compatibility, so a single clone into `~/.claude/skills/file-to-tns/` works for both tools.

## Usage

### As a Claude Code / OpenCode skill

Ask Claude to convert a file:

```
Convert notes.pdf to a tns file for my TI-Nspire
```

```
Put this docx on my calculator
```

```
Make a tns from data.csv
```

### As a standalone script

```bash
pip install pdfplumber pypdf python-docx openpyxl

python3 file_to_tns.py notes.pdf
python3 file_to_tns.py chem.docx output.tns
python3 file_to_tns.py data.csv --preview
```

## Supported Formats

| Format | Extension(s) | Library |
|--------|-------------|---------|
| PDF | `.pdf` | pdfplumber / pypdf |
| Word | `.docx` | python-docx |
| Plain text | `.txt` `.md` | stdlib |
| Excel | `.xlsx` `.xls` | openpyxl |
| CSV | `.csv` | stdlib |
| HTML | `.html` `.htm` | stdlib |
| Rich Text | `.rtf` | stdlib (regex) |

## How it works

1. Extracts text from the input file using the appropriate library for each format
2. Splits content into chunks of ≤ 11,500 characters (TI-Nspire's ~12 KB per-page limit)
3. Builds TI-Nspire `TI.Notepad` XML for each chunk
4. Calls [Luna](https://github.com/ndless-nspire/Luna) to produce a valid `.tns` binary with TI's proprietary binary XML compression and DES encryption — required for OS 3.x+

Luna is auto-downloaded and compiled on first run. If the build fails:

- **macOS:** `brew install zlib` then run again
- **Linux:** `sudo apt install zlib1g-dev` then run again
- **Manual:** `git clone https://github.com/ndless-nspire/Luna && cd Luna && make && cp luna ~/.luna_build/luna`

## Limitations

- **Scanned PDFs** have no text layer — extraction will be empty. Run OCR first.
- **Images** are not included. TI.Notepad is text-only.
- **Large files** are automatically split across multiple calculator pages.
- Requires **TI-Nspire OS 3.0.2 or later**.

## References

- [Luna](https://github.com/ndless-nspire/Luna) — the tool that produces valid `.tns` binaries
- [Hackspire: TNS File Format](https://hackspire.org/index.php/TNS_File_Format) — reverse-engineered format documentation

## License

MIT
