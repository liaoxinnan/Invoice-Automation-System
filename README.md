# Finance Email Processing Agent

An automated pipeline that monitors two Gmail mailboxes for PDF attachments, classifies and extracts data using Claude AI, archives files to Google Drive, and logs structured records into a Google Sheets tracker.

---

## Overview

The agent processes PDFs from two sources:
- **Scanner emails** — scanned documents from the office Apeos C2561 scanner sent to `training@clearsk.com`
- **Vendor emails** — supplier/vendor emails received at `finance.dept@clearsk.com`'ops@clearsk.com'

For each PDF it: classifies the document, extracts finance fields, generates a standardised filename, uploads to Google Drive, appends a tracker row to Google Sheets, and labels the email as `Agent-Processed` to skip it on the next run.

---

## Project structure

```
.
├── app/
│   ├── main.py            # Entry point — orchestrates the full pipeline
│   ├── claude_agent.py    # Builds prompts, calls Claude API, parses responses
│   ├── gmail_reader.py    # Gmail API helpers for both mailboxes
│   ├── drive_uploader.py  # Google Drive upload helpers
│   ├── drive_lister.py    # Utility: list Drive folder contents
│   ├── sheets_writer.py   # Google Sheets append + GL column update
│   └── file_handler.py    # Move and rename local files
├── config/
│   ├── company_mapping.json   # Full legal name → short name for filenames
│   └── gl_mapping.json        # Supplier → GL code/category mapping
├── credentials/
│   ├── credentials.json       # Google OAuth2 client secrets (do not commit)
│   ├── token.json             # Cached token for training@ (auto-generated)
│   └── token_finance.json     # Cached token for finance.dept@ (auto-generated)
├── data/
│   ├── temp_attachments/      # Staging area for downloaded PDFs (auto-cleared)
│   └── archive/               # Local archive organised by category
└── requirements.txt
```

---

## Pipeline

```
Gmail (training@ and finance.dept@)
        ↓
  Fetch emails: has:attachment, newer_than:7d, not labelled Agent-Processed
        ↓
  Download PDF attachment to data/temp_attachments/
        ↓
  Claude (claude-sonnet-4-6) reads PDF + email context and returns JSON:
    • Document category and nature
    • Extracted fields (vendor, invoice #, dates, amounts, GST, etc.)
    • GL account code (matched from gl_mapping.json)
    • 5MPO classification
    • Standardised filename
        ↓
  Rename and move to data/archive/{CATEGORY}/
        ↓
  Upload to Google Drive → Finance Archive/{CATEGORY}/
  (local copy deleted after successful upload)
        ↓
  Append row to Google Sheets (MIT tab) + write GL code to column AA
        ↓
  Apply label Agent-Processed to the Gmail message
```

---

## Document categories and archive folders

| Category          | Archive folder     |
|-------------------|--------------------|
| Stock Invoice     | STOCK INVOICE      |
| Operation Invoice | OPERATION INVOICE  |
| Finance Invoice   | FINANCE INVOICE    |
| SOA               | SOA                |
| Government Letter | GOVERNMENT LETTER  |
| Loan              | LOAN               |
| Rental            | RENTAL             |
| Bank Letter       | BANK LETTER        |
| Receipt           | STOCK INVOICE      |
| Payment Request   | OPERATION INVOICE  |
| Urgent Reply      | OPERATION INVOICE  |
| Other             | OTHERS             |

---

## File naming convention

```
YYMMDD - DOCUMENT_TYPE - SENDER - ADDRESSEE - TOPIC.ext
```

- `YYMMDD` — document date (e.g. `260308` for 2026-03-08); `000000` if not found
- `DOCUMENT_TYPE` — category in uppercase with underscores (e.g. `OPERATION_INVOICE`)
- `SENDER` — vendor name from the document
- `ADDRESSEE` — short name resolved from `config/company_mapping.json`
- `TOPIC` — invoice number or reference; short description if unavailable

Example: `260308 - OPERATION_INVOICE - HIREMOP_PTE_LTD - CSH - INV141309.pdf`

---

## Tracker columns (Google Sheets — MIT tab)

| Column | Field |
|--------|-------|
| A | Receipt Date Stamped |
| B | Date of Input |
| C | Date of Document |
| D | Due Date (If Applicable) |
| E | Sender Name |
| F | Document Nature |
| G | Document Category |
| H | Description |
| I | Invoice # |
| J | Tracking Number |
| K | Invoice Amount |
| L | GST (if any) |
| M | Amount ($) Exc GST |
| N | PIC |
| O | URL LINK |
| P | 5MPO Classification |
| AA | GL Account# Posted |

---

## Setup

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Google Cloud credentials

1. Create a Google Cloud project and enable the Gmail API, Google Sheets API, and Google Drive API.
2. Create an OAuth 2.0 Desktop client and download the client secret file.
3. Place it at `credentials/credentials.json`.

### 3. Anthropic API key

```bash
export ANTHROPIC_API_KEY=sk-...
```

### 4. First run — authenticate both mailboxes

```bash
cd app
python main.py
```

On the first run a browser window opens for `training@clearsk.com`. After labelling finance.dept emails a second window opens for `finance.dept@clearsk.com`. Tokens are saved to `credentials/token.json` and `credentials/token_finance.json` and reused automatically.

### 5. Gmail label

Create a label named exactly `Agent-Processed` in both Gmail accounts. The agent raises an error if the label is missing.

### 6. Google Drive folder structure

Create a `Finance Archive` folder accessible by `training@clearsk.com` with these subfolders:

```
Finance Archive/
├── STOCK INVOICE/
├── OPERATION INVOICE/
├── FINANCE INVOICE/
├── SOA/
├── GOVERNMENT LETTER/
├── LOAN/
├── RENTAL/
├── BANK LETTER/
└── OTHERS/
```

---

## Configuration

### `config/company_mapping.json`

Maps full legal entity names to short codes used in filenames and the tracker.

```json
{
  "CLEARSK HEALTHCARE PTE. LTD.": "CSH",
  "CLEARSK AESTHETICS PTE. LTD.": "CSA"
}
```

### `config/gl_mapping.json`

Maps supplier names to GL account codes. Each entry can be a single object or a list (for suppliers with multi-line invoice logic).

```json
{
  "HIREMOP PTE LTD": {
    "gl_code": "6100",
    "gl_category": "Cleaning Expenses",
    "item_description": "cleaning services"
  }
}
```

---

## Running the agent

```bash
cd app
python main.py
```

Run manually or schedule via Windows Task Scheduler to run daily.

---

## Security notes

- Never commit `credentials/credentials.json`, `credentials/token.json`, or `credentials/token_finance.json`.
- Add the `credentials/` folder to `.gitignore`.
- Uploaded Drive files are shared read-only with `clearsk.com` domain users only.
