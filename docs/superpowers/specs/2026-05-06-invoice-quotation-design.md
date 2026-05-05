# Invoice / Quotation Generator — Design

## Goal
Web-based form to fill in invoice/quotation data and save as PDF. Deployable to Coolify. "Simple" is the guiding principle.

## Stack
- Static HTML + inline CSS + inline JS (single `index.html`)
- Served by `nginx:alpine` via Dockerfile
- No backend, no database, no build step

## File Layout
```
invoice/
├── index.html
├── Dockerfile
├── nginx.conf      (optional, only if needed)
└── README.md
```

## Features

### Document
- Type toggle: **Invoice** / **Quotation**
- Language toggle: **TH** / **EN** (affects all labels in form and preview)
- Document number, date, due/expiry date

### Issuer (our company) — persisted to LocalStorage
- Company name, address, phone, email, tax ID
- Logo upload (stored as data URL in LocalStorage)
- Button: **Save Company Info** writes to LocalStorage
- On load: read LocalStorage and prefill if present

### Customer
- Name, address, phone, tax ID

### Line Items
- Repeating rows: description, quantity, unit price, line total (auto)
- Add row / remove row buttons

### Totals
- Subtotal (sum of line totals)
- Discount (percentage)
- VAT 7% toggle (on/off)
- Grand total
- Notes / payment terms (free text)

### Save PDF
- Button triggers `window.print()`
- CSS `@media print` hides the form panel; only the document preview prints
- User picks "Save as PDF" in browser print dialog

## UI Layout
- Desktop: 2 columns side-by-side
  - Left: form
  - Right: live A4 preview
- Mobile: stacked (form on top, preview below)
- Live preview updates on every input change

## State
Single JS object:
```js
state = {
  docType: 'invoice' | 'quotation',
  lang: 'th' | 'en',
  docNumber, docDate, dueDate,
  company: { name, address, phone, email, taxId, logoDataUrl },
  customer: { name, address, phone, taxId },
  items: [{ description, qty, unitPrice }],
  discountPct: number,
  vatEnabled: boolean,
  notes: string
}
```
- Totals are derived at render time (not stored)
- Re-render preview on any state change

## i18n
- Plain object: `i18n = { th: {...}, en: {...} }`
- Keys: `invoice`, `quotation`, `docNumber`, `date`, `dueDate`, `from`, `to`, `description`, `qty`, `unitPrice`, `amount`, `subtotal`, `discount`, `vat`, `total`, `notes`, `taxId`, `phone`, `email`, etc.
- Toggle changes `state.lang` then re-renders

## Number / Currency
- THB by default, formatted with thousands separator and 2 decimals
- Date format: `YYYY-MM-DD` (universal, works in both languages)

## Print CSS
- `@media print`:
  - Hide `.form-panel`, header buttons, and language/doc toggle
  - `.preview-panel` becomes full page, no shadow/border
  - A4 page size, margins ~15mm
  - Page-break behavior: avoid breaking inside item rows

## Dockerfile
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```
Coolify: point at git repo, choose Dockerfile build pack, deploy.

## Out of Scope (YAGNI)
- Login / multi-user
- Saving past documents
- Database
- Email sending
- Server-side PDF generation
- Multiple currencies
- Custom themes
