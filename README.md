# file-to-tns

Convert any document into a TI-Nspire `.tns` Notes file you can transfer to your calculator.

## Supported formats

`.pdf` · `.txt` · `.md` · `.docx` · `.xlsx` · `.csv` · `.html` · `.rtf`

## Requirements

```bash
pip install pdfplumber pypdf python-docx openpyxl
```

Luna is auto-downloaded and built on first run. If it fails:
- **macOS:** `brew install zlib` then run again
- **Linux:** `sudo apt install zlib1g-dev` then run again
- **Manual:** `git clone https://github.com/ndless-nspire/Luna && cd Luna && make && cp luna ~/.luna_build/luna`

## Usage

```bash
python3 file_to_tns.py notes.pdf
python3 file_to_tns.py chem.docx chem_notes.tns
python3 file_to_tns.py data.csv --preview
```

## Notes

- Large files are automatically split across multiple calculator pages (11.5 KB limit per page)
- Scanned/image PDFs have no text layer — plain text extraction won't work for those
- Requires TI-Nspire OS 3.0.2 or later
- Luna (https://github.com/ndless-nspire/Luna) does the actual `.tns` packaging
