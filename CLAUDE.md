# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Run CLI processor (reads from facturas/, writes to output/)
python main.py
python main.py --carpeta C:/ruta/facturas --ingresos "2026-04:5000000,2026-03:4500000"

# Run file watcher (auto-processes when new files are added)
python watcher.py
python watcher.py --carpeta C:/ruta/facturas --ingresos "2026-04:5000000"

# Run Streamlit dashboard
streamlit run app.py
```

## Architecture

The system processes Colombian DIAN electronic invoices (PDF/XML) into a structured Excel workbook with three sheets: BASE_DATOS, VALIDACION, PRORRATEO_IVA.

**Data flow:** `facturas/` folder (recursive) → `extractor.py` → `validator.py` → `prorateo.py` → `excel_writer.py` → `output/*.xlsx`

### Module responsibilities

- **`extractor.py`** — Entry point for document parsing. `extract_document()` dispatches to `extract_xml()` or `extract_pdf()`. XML always takes priority over PDF when both exist for the same stem. Returns a flat dict with fixed keys: `archivo, tipo, cufe, folio, fecha, nit_emisor, nombre_emisor, nit_receptor, nombre_receptor, subtotal, iva_19, iva_5, total, fuente`. The `processed` set uses the full path without extension as key (not just stem) to support duplicate detection across subfolders. Falls back to folder name for date when the document has no parseable date.

- **`validator.py`** — Stateless validation over a DataFrame. `validate()` adds `validacion` (OK/ERROR) and `observacion` columns. Checks: CUFE format (96 hex chars), CUFE duplicates, NIT format, subtotal+IVA≈total (tolerance: $1 COP), mandatory empty fields, mandato/peaje documents with IVA. These columns are used internally and in the VALIDACION sheet but are NOT included in BASE_DATOS.

- **`prorateo.py`** — IVA proration per Art. 490 E.T. `calcular_prorateo()` takes dicts `{YYYY-MM: float}` for gravados/excluidos. Mandatos always go to non-deductible. Notas crédito carry negative values so they automatically reduce monthly totals. Use `calcular_prorateo_simple()` when income data is unavailable (defaults to 100% deductible with a warning column).

- **`excel_writer.py`** — Formats and writes the three-sheet workbook. Applies color coding to the VALIDACION sheet (red for ERROR, green for OK). Uses `openpyxl` directly after pandas writes via `ExcelWriter`. BASE_DATOS has no validacion/observacion columns.

- **`main.py`** — CLI via `argparse`. Scans `facturas/` recursively with `rglob("*")`. Maintains a `processed: set[str]` keyed on full path without extension (`str(path.with_suffix("")).lower()`) to avoid double-processing PDF+XML pairs while supporting same filenames across different subfolders. Sets `row["archivo"]` to the relative path from `carpeta` root when the file is in a subfolder.

- **`watcher.py`** — File system watcher using `watchdog`. Monitors `facturas/` recursively for new PDF/XML files. On detection, waits 5 s (to let the OS finish copying) then calls `procesar()`. Debounce of 10 s prevents re-running for multi-file batch drops. Run with `python watcher.py` and leave it open in a terminal.

- **`app.py`** — Streamlit UI. Uploads files to a `tempfile.TemporaryDirectory`, reuses all the same modules, offers download of the generated Excel.

### Subfolder / folder-date support

`facturas/` supports nested subfolders, typically named by date:

```
facturas/
├── 2026-03/         ← date detected from folder name → used as fallback date
│   └── FE-001.pdf
├── 2026-04/
│   └── FE-100.xml
└── FE-999.pdf       ← flat files also supported
```

`_date_from_folder(path)` in `extractor.py` parses the parent folder name and returns a YYYY-MM-DD date. Applied only when the document itself has no parseable date.

### Document type detection

Detected by keywords in text or filename (case-insensitive):
- `"nota cr"` → Nota Crédito (values negated with sign = -1)
- `"mandato"` or `"peaje"` → Mandato/Peaje (IVA non-deductible)
- `"documento soporte"` → Documento Soporte
- Default → Factura Electrónica

### XML parsing

Uses `xml.etree.ElementTree` with UBL 2.1 namespaces (`cac`, `cbc`, `ext`). CUFE is at `cbc:UUID`. Tax percentages are read from `cac:TaxSubtotal/cbc:Percent` to distinguish IVA 19% vs 5%. Supplier name tries `PartyLegalEntity/RegistrationName` first, then falls back to `PartyName/Name`.

### PDF parsing

Uses `pdfplumber`. Extraction uses labeled field patterns matching the standard DIAN graphical representation layout, with generic regex fallbacks:

| Field | Primary pattern | Fallback |
|---|---|---|
| `folio` | `"Número de Factura: FE-001"` | `"No. Factura:"`, `"Nro.:"` |
| `nit_emisor` | `"Nit del Emisor: 123456"` | First NIT found |
| `nombre_emisor` | `"Razón Social: ..."` | Line near emisor NIT |
| `nit_receptor` | `"Número Documento: 9010..."` | Second NIT found |
| `nombre_receptor` | `"Nombre o Razón Social: ..."` | Line near receptor NIT |
| `fecha` | `"Fecha de Emisión: 24/03/2026"` | First date found |

Colombian number format: dots as thousands separator, comma as decimal (`1.234.567,89`). Handled by `_clean_number()`.

### BASE_DATOS columns (ordered)

```
archivo, tipo, cufe, folio, fecha,
nit_emisor, nombre_emisor, nit_receptor, nombre_receptor,
subtotal, iva_19, iva_5, total, fuente
```

`validacion` and `observacion` are NOT in BASE_DATOS — they live only in the VALIDACION sheet.
