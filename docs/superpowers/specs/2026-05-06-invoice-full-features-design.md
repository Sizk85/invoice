# Invoice/Quotation Generator — Full Feature Set Design

**Date:** 2026-05-06
**Status:** Draft for review
**Scope:** Add 10 features to single-file `index.html` invoice/quotation generator

## Goals

Make the generator usable for real Thai SME workflows: payment info, signatures, document history, auto-numbering, due-date logic, withholding tax, and quotation→invoice conversion. Stay single-file, vanilla JS, LocalStorage-only.

## Confirmed decisions

- **Doc number format:** `INV-2026-0001`, `QUO-2026-0001`. Counter resets each calendar year. Stored per-prefix.
- **Storage strategy:** Saved documents store **data only** — logo/signature/stamp/QR images live in company settings and are pulled in at view/print time. Trade-off: changing company logo updates the look of historical documents. Acceptable; logos rarely change.

## Features (10)

### 1. Payment information
- Multiple bank accounts (add/edit/delete in company settings)
  - Fields: bank name, account name, account number, branch (optional)
- PromptPay: phone number OR national ID OR tax ID (one entry)
- QR code image upload (PromptPay QR or bank QR), shown in preview
- Toggle per-document: which payment methods to display
- Rendered in preview under a "Payment Information" section, only on invoices (hidden on quotations by default but toggleable)

### 2. Signature / Stamp
- Upload signature image (transparent PNG recommended) in company settings
- Upload stamp image in company settings
- Both stored as data URLs alongside logo
- Rendered in preview's signature block, replacing/augmenting the existing line for signature
- Toggle per-document: show signature, show stamp

### 3. Saved documents (LocalStorage history)
- After filling out a document, "Save" button stores it
- Separate panel/modal: list of saved documents with columns (number, type, customer name, date, total, actions)
- Actions per row: Open (load into form), Duplicate (clone + new number), Delete
- Search/filter by number or customer name
- Storage key: `invoiceApp.documents.v1` — array of document data objects (no images)

### 4. Auto-running document number
- Counter stored per prefix and per year: `invoiceApp.counters.v1` = `{ "INV-2026": 12, "QUO-2026": 5 }`
- "New Document" button increments counter and assigns next number
- User can manually override the auto-generated number (free text field)
- On save, if number was auto-generated, counter advances; if manually overridden, no change to counter

### 5. Due date / validity (Net X days)
- Dropdown next to issue date: "Due on receipt", "Net 7", "Net 15", "Net 30", "Net 60", "Custom"
- Selecting Net X auto-calculates due date = issue date + X days
- "Custom" lets user pick any date
- Quotations use same UI but label changes to "Valid until"

### 6. VAT inclusive/exclusive toggle
- Toggle "ราคารวม VAT แล้ว" / "ราคายังไม่รวม VAT" on price entry
- Inclusive mode: unitPrice already contains VAT → subtotal extracted as `unitPrice / (1 + vatRate)`, VAT shown separately for clarity
- Exclusive mode (current behavior): unitPrice is pre-VAT, VAT added on top
- Setting is per-document; default exclusive
- Discount and withholding apply to the pre-VAT amount in both modes

### 7. Withholding tax (หัก ณ ที่จ่าย)
- Toggle: "หัก ณ ที่จ่าย"
- Rate dropdown: 1%, 3%, 5%, custom
- Calculated against subtotal (after discount, before VAT) — Thai standard
- Shown in totals block as a deduction; net payable = total - withholding
- Note line under total: "ยอดสุทธิที่ต้องชำระ"

### 8. Convert Quotation → Invoice
- When viewing a saved quotation, "Convert to Invoice" button
- Creates new invoice document with:
  - All line items copied
  - Customer info copied
  - New auto-generated INV number
  - New issue date (today), recalculated due date
  - Reference field: "Ref: QUO-2026-0001"
- Original quotation remains unchanged in history

### 9. Currency selector
- Per-document currency: THB (฿), USD ($), EUR (€), JPY (¥), GBP (£), CNY (¥)
- Symbol prefix on all money displays (subtotal, line totals, total, payment due)
- Stored as ISO code (THB, USD, ...) in document data
- Default: THB
- No exchange rate / no conversion — purely a display label. Each document is single-currency.
- VAT label adapts: "VAT 7%" stays for THB; for non-THB defaults to "Tax 0%" with editable rate (since 7% VAT is Thai-specific). Simplification: keep the VAT toggle/rate generic — user controls whether tax applies. Withholding tax remains Thai-only (hidden when currency ≠ THB).

### 10. Export / Import JSON
- "Export" button in toolbar: downloads `.json` file with full backup
  - Contents: `{ company, documents, counters, exportedAt, version }`
  - Filename: `invoice-backup-YYYY-MM-DD.json`
- "Import" button: file picker → reads JSON → confirm dialog ("จะแทนที่ข้อมูลปัจจุบันทั้งหมด ดำเนินการต่อ?") → overwrites company/documents/counters
- Validates JSON shape; rejects with alert on malformed file
- Version field for future migration; v1 for now

## Architecture

Stay single-file. Add the following to `index.html`:

### State shape extensions

```js
state = {
  // existing fields...
  company: {
    // existing + new:
    bankAccounts: [{ id, bankName, accountName, accountNumber, branch }],
    promptPay: { type: 'phone'|'id'|'taxId', value: '' },
    qrImage: null,        // dataURL
    signature: null,      // dataURL
    stamp: null,          // dataURL
  },
  // new top-level:
  docNumber: 'INV-2026-0001',  // editable, auto-filled
  docNumberAuto: true,          // tracks whether to advance counter on save
  issueDate: '2026-05-06',
  dueTerm: 'net30',             // 'onreceipt'|'net7'|'net15'|'net30'|'net60'|'custom'
  dueDate: '2026-06-05',
  reference: '',                // for Q→I conversion
  paymentDisplay: { showBanks: true, showPromptPay: true, showQR: true },
  showSignature: true,
  showStamp: false,
  withholding: { enabled: false, rate: 3 },
  vatInclusive: false,           // #6
  currency: 'THB',               // #9 — ISO code
  // existing items, vatEnabled, discountPct, etc.
}
```

### LocalStorage keys

- `invoiceApp.company.v1` — existing, extended with new fields
- `invoiceApp.documents.v1` — new, array of saved documents (data only, no images)
- `invoiceApp.counters.v1` — new, `{ "INV-2026": n, "QUO-2026": n }`

### New modules (logical, all in same file)

1. **Document number module** — `getNextDocNumber(type)`, `advanceCounter(number)`
2. **Due date module** — `computeDueDate(issueDate, term)`, `dueTermLabel(term)`
3. **History module** — `saveDocument()`, `loadDocument(id)`, `deleteDocument(id)`, `duplicateDocument(id)`, `listDocuments()`, `convertQuotationToInvoice(id)`
4. **Payment rendering** — `renderPaymentSection(state)` returns HTML
5. **Signature rendering** — `renderSignatureBlock(state)` returns HTML
6. **Withholding compute** — extends `computeTotals()` to return `{ subtotal, discountAmt, afterDiscount, vat, withholding, total, netPayable }`. Subtotal calc respects `vatInclusive` flag.
7. **Currency module** — `currencySymbol(code)`, `fmtMoney(amount, code)` replaces `fmt()` everywhere money is shown
8. **Backup module** — `exportBackup()` triggers download, `importBackup(file)` reads + validates + writes

### UI changes

**Form panel** — add new sections:
- "Payment Information" — bank accounts list (add/remove), PromptPay, QR upload
- "Signature & Stamp" — upload signature, upload stamp
- "Document Settings" — auto/manual number, due term dropdown, currency selector, VAT inclusive toggle
- "Withholding Tax" — toggle + rate (visible only when currency = THB)
- Toolbar gets: "New", "Save", "History", "Export", "Import" buttons

**History modal** — overlay panel with table of saved docs + actions

**Preview panel** — extend with payment section, signature block (with image), withholding line in totals

### Data flow

```
User edits form → state mutates → renderPreview()
                                ↓
                       computeTotals() includes withholding
                                ↓
                       renders payment + signature blocks
User clicks Save → saveDocument() strips images, advances counter, writes localStorage
User clicks History → renders modal from listDocuments()
User clicks Open → loadDocument() merges saved doc into state, re-render
User clicks Convert (on quotation) → convertQuotationToInvoice() creates new INV
```

### Error handling

- LocalStorage quota exceeded → catch QuotaExceededError, show alert "หน่วยความจำเต็ม กรุณาลบเอกสารเก่า"
- Image upload >2MB → reject with alert
- Invalid date input → fall back to today
- Missing required fields on save → highlight + alert (number, customer name, at least 1 item)

### Testing approach

Single-file static site, no test framework. Manual test checklist in implementation plan covering:
- Each feature's happy path
- Edge cases: counter rollover at year change, save with no items, load deleted doc reference, image too large, quota full
- Print preview with all features enabled

## Out of scope

- Multi-user / cloud sync
- PDF generation library (relies on browser print)
- Server-side anything
- Currency exchange rate / conversion (currency is display label only)
- Auto QR code generation from PromptPay number (user uploads image)

## Open questions

1. **History panel UX:** modal overlay vs separate "tab" in the form panel? (Proposal: modal overlay — keeps form panel uncluttered.)
2. **Counter rollover:** when year changes (2026→2027), should we keep both counters or migrate? (Proposal: keep both — old year stays at last value, new year starts at 1.)
3. **PromptPay QR:** user uploads image only, no auto-generation. (Confirmed.)
