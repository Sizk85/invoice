# Invoice/Quotation Generator — Full 10-Feature Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 10 production features (payment info, signature/stamp, document history, auto-numbering, due-date terms, VAT inclusive toggle, withholding tax, Q→I conversion, currency selector, JSON backup) to the existing single-file invoice/quotation generator.

**Architecture:** Single-file `index.html` (HTML + CSS + vanilla JS, LocalStorage). All new code lives in the same file under clearly labelled sections. State extends the existing `state` object. New LocalStorage keys are versioned (`v1`).

**Tech Stack:** HTML5, CSS, vanilla JavaScript, browser LocalStorage, FileReader API, browser-native print. No build step, no test framework — verification is manual via browser.

**Spec reference:** `docs/superpowers/specs/2026-05-06-invoice-full-features-design.md`

**Verification approach:** No unit test framework. Each task ends with a manual verification checklist (open `index.html` in a browser, follow steps, confirm expected behavior). Commit after each task passes verification.

**File touched:** `index.html` (all changes). Lines referenced are pre-change. After each commit, line numbers shift — use search anchors (e.g. "// ---- LocalStorage ----") rather than absolute lines for subsequent tasks.

---

## File Structure

Single file `index.html`. Logical sections in the `<script>` tag, in this order:

1. `i18n` dictionary (existing) — extend with new keys
2. Storage keys (existing `STORAGE_KEY`, plus new `DOCS_KEY`, `COUNTERS_KEY`)
3. `state` object (existing) — extend with new top-level fields
4. Helpers (existing `$`, `fmt`, `esc`, etc.) — add `currencySymbol`, `fmtMoney`, `uid`
5. LocalStorage I/O (existing) — extend with documents and counters I/O
6. Domain modules (NEW logical groupings):
   - Document number module
   - Due date module
   - Totals module (extend existing)
   - History module
   - Backup module
7. Form rendering (existing `renderItemsForm`, etc.) — add `renderBankAccountsForm`, `renderHistoryList`
8. Preview rendering (existing `renderPreview`) — extend to include payment, signature, withholding
9. Event wiring (existing `setupListeners`) — extend with new buttons
10. Init (existing) — extend to load documents/counters

UI additions in HTML:
- New form sections: Payment, Signature/Stamp, Document Settings, Withholding
- New toolbar buttons: New, Save, History, Export, Import
- New modal: History list

---

## Task 1: Extend state shape and storage keys

**Files:**
- Modify: `index.html` (script block: state + storage keys)

- [ ] **Step 1: Read current state shape**

Search anchor in file: `const STORAGE_KEY = 'invoiceApp.company.v1';`. Read 30 lines around it.

- [ ] **Step 2: Replace storage keys block**

Find:
```js
const STORAGE_KEY = 'invoiceApp.company.v1';
```

Replace with:
```js
const STORAGE_KEY = 'invoiceApp.company.v1';
const DOCS_KEY = 'invoiceApp.documents.v1';
const COUNTERS_KEY = 'invoiceApp.counters.v1';
```

- [ ] **Step 3: Replace `state` object**

Find:
```js
const state = {
  docType: 'invoice',
  lang: 'th',
  docNumber: '',
  docDate: new Date().toISOString().slice(0, 10),
  dueDate: '',
  company: { name: '', address: '', phone: '', email: '', taxId: '', logoDataUrl: '' },
  customer: { name: '', address: '', phone: '', taxId: '' },
  items: [{ description: '', qty: 1, unitPrice: 0 }],
  discountPct: 0,
  vatEnabled: false,
  notes: ''
};
```

Replace with:
```js
const state = {
  docType: 'invoice',
  lang: 'th',
  docNumber: '',
  docNumberAuto: true,
  docDate: new Date().toISOString().slice(0, 10),
  dueTerm: 'net30',
  dueDate: '',
  reference: '',
  currency: 'THB',
  vatInclusive: false,
  company: {
    name: '', address: '', phone: '', email: '', taxId: '',
    logoDataUrl: '',
    signatureDataUrl: '',
    stampDataUrl: '',
    qrDataUrl: '',
    bankAccounts: [],
    promptPay: { type: 'phone', value: '' }
  },
  customer: { name: '', address: '', phone: '', taxId: '' },
  items: [{ description: '', qty: 1, unitPrice: 0 }],
  discountPct: 0,
  vatEnabled: false,
  withholding: { enabled: false, rate: 3 },
  paymentDisplay: { showBanks: true, showPromptPay: true, showQR: true },
  showSignature: true,
  showStamp: false,
  notes: '',
  currentDocId: null
};
```

- [ ] **Step 4: Verify in browser**

Open `index.html`, open DevTools console, run:
```js
console.log(state)
```
Expected: object with all new fields present, no errors. App still renders.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat(state): extend state shape for full feature set"
```

---

## Task 2: Add helper functions (currency, uid, fmtMoney)

**Files:**
- Modify: `index.html` (helpers section)

- [ ] **Step 1: Locate helpers section**

Search anchor: `// ---- helpers ----`. Read 15 lines after.

- [ ] **Step 2: Add new helpers after `nl2br`**

Find:
```js
const nl2br = s => esc(s).replace(/\n/g, '<br>');
```

Add immediately after:
```js
const CURRENCY_SYMBOLS = { THB: '฿', USD: '$', EUR: '€', JPY: '¥', GBP: '£', CNY: '¥' };
const currencySymbol = code => CURRENCY_SYMBOLS[code] || code;
const fmtMoney = (amount, code) => `${currencySymbol(code || state.currency)} ${fmt(amount)}`;
const uid = () => Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
const todayISO = () => new Date().toISOString().slice(0, 10);
const addDaysISO = (iso, days) => {
  const d = new Date(iso);
  d.setDate(d.getDate() + Number(days || 0));
  return d.toISOString().slice(0, 10);
};
```

- [ ] **Step 3: Verify in browser**

Reload page, open DevTools console, run:
```js
fmtMoney(1234.5, 'USD')   // expect "$ 1,234.50"
fmtMoney(1234.5, 'THB')   // expect "฿ 1,234.50"
addDaysISO('2026-05-06', 30) // expect "2026-06-05"
uid()                     // expect non-empty string
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(helpers): add currency, money formatting, uid, date helpers"
```

---

## Task 3: Document number module

**Files:**
- Modify: `index.html` (after `saveCompany` function)

- [ ] **Step 1: Locate insertion point**

Search anchor: `// ---- totals ----`. Insert new module immediately before this anchor.

- [ ] **Step 2: Add document number module**

Insert before `// ---- totals ----`:
```js
// ---- Document numbering ----
function loadCounters() {
  try {
    const raw = localStorage.getItem(COUNTERS_KEY);
    return raw ? JSON.parse(raw) : {};
  } catch (e) { return {}; }
}
function saveCounters(counters) {
  localStorage.setItem(COUNTERS_KEY, JSON.stringify(counters));
}
function nextDocNumber(docType) {
  const prefix = docType === 'invoice' ? 'INV' : 'QUO';
  const year = new Date().getFullYear();
  const key = `${prefix}-${year}`;
  const counters = loadCounters();
  const next = (counters[key] || 0) + 1;
  return `${key}-${String(next).padStart(4, '0')}`;
}
function advanceCounter(docNumber) {
  // docNumber format: PREFIX-YYYY-NNNN
  const m = /^(INV|QUO)-(\d{4})-(\d+)$/.exec(docNumber || '');
  if (!m) return;
  const key = `${m[1]}-${m[2]}`;
  const num = Number(m[3]);
  const counters = loadCounters();
  if ((counters[key] || 0) < num) {
    counters[key] = num;
    saveCounters(counters);
  }
}
```

- [ ] **Step 3: Verify in browser**

Reload, in DevTools console:
```js
nextDocNumber('invoice')  // e.g. "INV-2026-0001"
nextDocNumber('quotation') // e.g. "QUO-2026-0001"
advanceCounter('INV-2026-0001')
loadCounters()  // expect { "INV-2026": 1 }
nextDocNumber('invoice')  // expect "INV-2026-0002"
```

Then clean up:
```js
localStorage.removeItem(COUNTERS_KEY)
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(numbering): add auto-running document number with year-based counter"
```

---

## Task 4: Due date module

**Files:**
- Modify: `index.html` (after document number module)

- [ ] **Step 1: Add due date module**

Insert immediately after `advanceCounter` function, still before `// ---- totals ----`:
```js
// ---- Due date / validity ----
const DUE_TERMS = {
  onreceipt: 0,
  net7: 7,
  net15: 15,
  net30: 30,
  net60: 60,
  custom: null
};
function computeDueDate(issueDate, term) {
  if (term === 'custom' || !DUE_TERMS.hasOwnProperty(term)) return state.dueDate || '';
  return addDaysISO(issueDate || todayISO(), DUE_TERMS[term]);
}
function applyDueTerm() {
  if (state.dueTerm !== 'custom') {
    state.dueDate = computeDueDate(state.docDate, state.dueTerm);
  }
}
```

- [ ] **Step 2: Verify in browser**

Reload, console:
```js
computeDueDate('2026-05-06', 'net30')      // "2026-06-05"
computeDueDate('2026-05-06', 'onreceipt')  // "2026-05-06"
computeDueDate('2026-05-06', 'net7')       // "2026-05-13"
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(due-date): add Net X days computation module"
```

---

## Task 5: Extend totals computation (VAT inclusive + withholding)

**Files:**
- Modify: `index.html` (`computeTotals` function)

- [ ] **Step 1: Locate function**

Search anchor: `function computeTotals()`. Read 12 lines.

- [ ] **Step 2: Replace `computeTotals` entirely**

Find:
```js
function computeTotals() {
  const subtotal = state.items.reduce((s, it) => s + (Number(it.qty) || 0) * (Number(it.unitPrice) || 0), 0);
  const discountAmt = subtotal * (Number(state.discountPct) || 0) / 100;
  const afterDiscount = subtotal - discountAmt;
  const vat = state.vatEnabled ? afterDiscount * 0.07 : 0;
  const total = afterDiscount + vat;
  return { subtotal, discountAmt, vat, total };
}
```

Replace with:
```js
function computeTotals() {
  const VAT_RATE = 0.07;
  const rawLineSum = state.items.reduce((s, it) => s + (Number(it.qty) || 0) * (Number(it.unitPrice) || 0), 0);

  let subtotal, vat;
  if (state.vatInclusive && state.vatEnabled) {
    // unit prices already include VAT — extract pre-VAT amount
    subtotal = rawLineSum / (1 + VAT_RATE);
  } else {
    subtotal = rawLineSum;
  }

  const discountAmt = subtotal * (Number(state.discountPct) || 0) / 100;
  const afterDiscount = subtotal - discountAmt;

  if (state.vatEnabled) {
    vat = afterDiscount * VAT_RATE;
  } else {
    vat = 0;
  }

  const total = afterDiscount + vat;

  const withholdingRate = state.withholding && state.withholding.enabled
    ? (Number(state.withholding.rate) || 0) / 100
    : 0;
  const withholding = afterDiscount * withholdingRate;
  const netPayable = total - withholding;

  return { subtotal, discountAmt, afterDiscount, vat, withholding, total, netPayable };
}
```

- [ ] **Step 3: Verify in browser**

Reload. The preview still renders. Open console:
```js
computeTotals()
```
Expected: object with `subtotal, discountAmt, afterDiscount, vat, withholding, total, netPayable`.

Add 1 item with qty=10, unitPrice=100. Enable VAT. Confirm:
- subtotal = 1000, vat = 70, total = 1070, withholding = 0, netPayable = 1070

Toggle in DevTools `state.withholding = {enabled: true, rate: 3}; renderPreview()` — withholding should be 30 (3% of 1000).

Toggle `state.vatInclusive = true; renderPreview()` — subtotal = 934.58, vat = 65.42, total ≈ 1000.

Reset: `state.withholding={enabled:false,rate:3}; state.vatInclusive=false; renderPreview()`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(totals): support VAT inclusive mode and withholding tax"
```

---

## Task 6: Document history module

**Files:**
- Modify: `index.html` (after due date module)

- [ ] **Step 1: Add history module**

Insert immediately after `applyDueTerm` function, still before `// ---- totals ----`:
```js
// ---- Document history ----
function loadDocuments() {
  try {
    const raw = localStorage.getItem(DOCS_KEY);
    return raw ? JSON.parse(raw) : [];
  } catch (e) { return []; }
}
function saveDocuments(docs) {
  try {
    localStorage.setItem(DOCS_KEY, JSON.stringify(docs));
    return true;
  } catch (e) {
    if (e.name === 'QuotaExceededError' || /quota/i.test(e.message || '')) {
      alert('หน่วยความจำเต็ม กรุณาลบเอกสารเก่าออกบางส่วน\nStorage full — please delete old documents.');
    } else {
      alert('บันทึกไม่สำเร็จ: ' + e.message);
    }
    return false;
  }
}
function snapshotCurrentDoc() {
  // Strip out company-level images; they live in company settings.
  return {
    id: state.currentDocId || uid(),
    docType: state.docType,
    lang: state.lang,
    docNumber: state.docNumber,
    docNumberAuto: state.docNumberAuto,
    docDate: state.docDate,
    dueTerm: state.dueTerm,
    dueDate: state.dueDate,
    reference: state.reference,
    currency: state.currency,
    vatInclusive: state.vatInclusive,
    customer: { ...state.customer },
    items: state.items.map(it => ({ ...it })),
    discountPct: state.discountPct,
    vatEnabled: state.vatEnabled,
    withholding: { ...state.withholding },
    paymentDisplay: { ...state.paymentDisplay },
    showSignature: state.showSignature,
    showStamp: state.showStamp,
    notes: state.notes,
    savedAt: new Date().toISOString()
  };
}
function applyDocSnapshot(doc) {
  state.currentDocId = doc.id;
  state.docType = doc.docType;
  state.lang = doc.lang || state.lang;
  state.docNumber = doc.docNumber;
  state.docNumberAuto = !!doc.docNumberAuto;
  state.docDate = doc.docDate;
  state.dueTerm = doc.dueTerm || 'custom';
  state.dueDate = doc.dueDate || '';
  state.reference = doc.reference || '';
  state.currency = doc.currency || 'THB';
  state.vatInclusive = !!doc.vatInclusive;
  state.customer = { ...{ name:'', address:'', phone:'', taxId:'' }, ...doc.customer };
  state.items = (doc.items && doc.items.length) ? doc.items.map(it => ({ ...it })) : [{ description:'', qty:1, unitPrice:0 }];
  state.discountPct = Number(doc.discountPct) || 0;
  state.vatEnabled = !!doc.vatEnabled;
  state.withholding = { enabled: false, rate: 3, ...(doc.withholding || {}) };
  state.paymentDisplay = { showBanks: true, showPromptPay: true, showQR: true, ...(doc.paymentDisplay || {}) };
  state.showSignature = doc.showSignature !== false;
  state.showStamp = !!doc.showStamp;
  state.notes = doc.notes || '';
}
function persistCurrentDoc() {
  const snap = snapshotCurrentDoc();
  const docs = loadDocuments();
  const idx = docs.findIndex(d => d.id === snap.id);
  if (idx >= 0) docs[idx] = snap;
  else docs.unshift(snap);
  if (!saveDocuments(docs)) return false;
  if (state.docNumberAuto) advanceCounter(state.docNumber);
  state.currentDocId = snap.id;
  return true;
}
function deleteDocument(id) {
  const docs = loadDocuments().filter(d => d.id !== id);
  saveDocuments(docs);
  if (state.currentDocId === id) state.currentDocId = null;
}
function duplicateDocument(id) {
  const docs = loadDocuments();
  const src = docs.find(d => d.id === id);
  if (!src) return null;
  const copy = JSON.parse(JSON.stringify(src));
  copy.id = uid();
  copy.docNumber = nextDocNumber(copy.docType);
  copy.docNumberAuto = true;
  copy.docDate = todayISO();
  copy.dueDate = computeDueDate(copy.docDate, copy.dueTerm || 'net30');
  copy.savedAt = new Date().toISOString();
  return copy;
}
function convertQuotationToInvoice(id) {
  const docs = loadDocuments();
  const src = docs.find(d => d.id === id);
  if (!src || src.docType !== 'quotation') return null;
  const inv = JSON.parse(JSON.stringify(src));
  inv.id = uid();
  inv.docType = 'invoice';
  inv.docNumber = nextDocNumber('invoice');
  inv.docNumberAuto = true;
  inv.docDate = todayISO();
  inv.dueTerm = 'net30';
  inv.dueDate = computeDueDate(inv.docDate, inv.dueTerm);
  inv.reference = src.docNumber;
  inv.savedAt = new Date().toISOString();
  return inv;
}
```

- [ ] **Step 2: Verify in browser**

Reload. Console:
```js
state.docNumber = 'INV-2026-0001'; state.customer.name = 'Test Co';
persistCurrentDoc()  // true
loadDocuments()      // array with 1 doc
const id = loadDocuments()[0].id;
const dup = duplicateDocument(id); console.log(dup.docNumber)  // INV-2026-0002
deleteDocument(id); loadDocuments()  // []
localStorage.removeItem(DOCS_KEY); localStorage.removeItem(COUNTERS_KEY);
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(history): add document save/load/duplicate/delete/convert module"
```

---

## Task 7: JSON backup module

**Files:**
- Modify: `index.html` (after history module)

- [ ] **Step 1: Add backup module**

Insert immediately after `convertQuotationToInvoice`, still before `// ---- totals ----`:
```js
// ---- Backup / Restore ----
function exportBackup() {
  const data = {
    version: 1,
    exportedAt: new Date().toISOString(),
    company: state.company,
    documents: loadDocuments(),
    counters: loadCounters()
  };
  const blob = new Blob([JSON.stringify(data, null, 2)], { type: 'application/json' });
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = `invoice-backup-${todayISO()}.json`;
  document.body.appendChild(a);
  a.click();
  setTimeout(() => { URL.revokeObjectURL(url); a.remove(); }, 0);
}
function importBackup(file) {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const data = JSON.parse(reader.result);
        if (!data || typeof data !== 'object' || !('version' in data)) {
          throw new Error('ไฟล์ backup ไม่ถูกต้อง / invalid backup file');
        }
        if (data.company) {
          Object.assign(state.company, data.company);
          localStorage.setItem(STORAGE_KEY, JSON.stringify(state.company));
        }
        if (Array.isArray(data.documents)) {
          localStorage.setItem(DOCS_KEY, JSON.stringify(data.documents));
        }
        if (data.counters && typeof data.counters === 'object') {
          localStorage.setItem(COUNTERS_KEY, JSON.stringify(data.counters));
        }
        resolve(true);
      } catch (e) { reject(e); }
    };
    reader.onerror = () => reject(reader.error);
    reader.readAsText(file);
  });
}
```

- [ ] **Step 2: Verify in browser**

Reload. Console:
```js
exportBackup()   // file should download as invoice-backup-2026-05-06.json
```
Open the downloaded file in a text editor — confirm it has `version`, `company`, `documents`, `counters` keys.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(backup): add JSON export and import for full data backup"
```

---

## Task 8: Extend i18n dictionary

**Files:**
- Modify: `index.html` (i18n object)

- [ ] **Step 1: Locate i18n**

Search anchor: `const i18n = {`. Read full block (~40 lines).

- [ ] **Step 2: Replace entire i18n block**

Find:
```js
const i18n = {
  th: {
    invoice: 'ใบแจ้งหนี้', quotation: 'ใบเสนอราคา',
```
…through the closing `};` of the `i18n` constant.

Replace with:
```js
const i18n = {
  th: {
    invoice: 'ใบแจ้งหนี้', quotation: 'ใบเสนอราคา',
    docInfo: 'ข้อมูลเอกสาร',
    docNumber: 'เลขที่', date: 'วันที่', dueDate: 'วันครบกำหนด', expiryDate: 'วันหมดอายุ',
    issuer: 'ผู้ออก (บริษัทเรา)', saveCompany: 'บันทึกข้อมูลบริษัท',
    logo: 'โลโก้', companyName: 'ชื่อบริษัท',
    address: 'ที่อยู่', phone: 'เบอร์โทร', email: 'อีเมล', taxId: 'เลขผู้เสียภาษี',
    customer: 'ลูกค้า', customerName: 'ชื่อ',
    items: 'รายการสินค้า/บริการ',
    description: 'รายละเอียด', qty: 'จำนวน', unitPrice: 'ราคา/หน่วย', amount: 'รวม',
    addItem: '+ เพิ่มรายการ',
    summary: 'สรุป', discount: 'ส่วนลด (%)', vatLabel: 'VAT 7%',
    notes: 'หมายเหตุ / เงื่อนไขการชำระเงิน',
    savePdf: 'บันทึก PDF',
    billTo: 'ลูกค้า', from: 'ผู้ออก',
    subtotal: 'ยอดรวม', discountAmt: 'ส่วนลด', vat: 'ภาษีมูลค่าเพิ่ม (7%)', total: 'ยอดสุทธิ',
    currency: '฿', savedMsg: 'บันทึกข้อมูลบริษัทแล้ว',
    payment: 'ข้อมูลการชำระเงิน', bankAccounts: 'บัญชีธนาคาร', bankName: 'ธนาคาร',
    accountName: 'ชื่อบัญชี', accountNumber: 'เลขที่บัญชี', branch: 'สาขา',
    addBank: '+ เพิ่มบัญชี', promptPay: 'พร้อมเพย์', promptPayPhone: 'เบอร์โทร',
    promptPayId: 'บัตรประชาชน', promptPayTax: 'เลขผู้เสียภาษี', qrCode: 'QR Code',
    signature: 'ลายเซ็น', stamp: 'ตราประทับ', signatureStamp: 'ลายเซ็น / ตราประทับ',
    showSignature: 'แสดงลายเซ็น', showStamp: 'แสดงตราประทับ',
    docSettings: 'ตั้งค่าเอกสาร', autoNumber: 'เลขที่อัตโนมัติ',
    dueTerm: 'เงื่อนไขชำระ', onreceipt: 'ชำระทันที', net7: 'Net 7 วัน', net15: 'Net 15 วัน',
    net30: 'Net 30 วัน', net60: 'Net 60 วัน', custom: 'กำหนดเอง',
    validityTerm: 'อายุใบเสนอราคา',
    currencyLabel: 'สกุลเงิน', vatInclusive: 'ราคารวม VAT แล้ว',
    withholdingTax: 'หัก ณ ที่จ่าย', withholdingRate: 'อัตรา (%)',
    netPayable: 'ยอดสุทธิที่ต้องชำระ',
    newDoc: 'เอกสารใหม่', saveDoc: 'บันทึกเอกสาร', history: 'ประวัติ',
    exportData: 'Export', importData: 'Import',
    historyTitle: 'ประวัติเอกสาร', searchPlaceholder: 'ค้นหา (เลขที่ / ลูกค้า)',
    noDocs: 'ยังไม่มีเอกสารที่บันทึก', open: 'เปิด', duplicate: 'คัดลอก', remove: 'ลบ',
    convertToInvoice: 'แปลงเป็นใบแจ้งหนี้', close: 'ปิด',
    confirmDelete: 'ลบเอกสารนี้?', confirmImport: 'จะแทนที่ข้อมูลปัจจุบันทั้งหมด ดำเนินการต่อ?',
    docSavedMsg: 'บันทึกเอกสารแล้ว', importSuccessMsg: 'นำเข้าข้อมูลสำเร็จ',
    importFailMsg: 'ไม่สามารถนำเข้าไฟล์ได้',
    paymentSection: 'ช่องทางการชำระเงิน', referenceLabel: 'อ้างอิง', uploadImage: 'อัพโหลดรูป',
    saveFirst: 'กรุณากรอกข้อมูลขั้นต่ำ (เลขที่ + ชื่อลูกค้า + อย่างน้อย 1 รายการ)'
  },
  en: {
    invoice: 'INVOICE', quotation: 'QUOTATION',
    docInfo: 'Document Info',
    docNumber: 'No.', date: 'Date', dueDate: 'Due Date', expiryDate: 'Valid Until',
    issuer: 'Issuer (Your Company)', saveCompany: 'Save Company Info',
    logo: 'Logo', companyName: 'Company Name',
    address: 'Address', phone: 'Phone', email: 'Email', taxId: 'Tax ID',
    customer: 'Customer', customerName: 'Name',
    items: 'Items',
    description: 'Description', qty: 'Qty', unitPrice: 'Unit Price', amount: 'Amount',
    addItem: '+ Add Item',
    summary: 'Summary', discount: 'Discount (%)', vatLabel: 'VAT 7%',
    notes: 'Notes / Payment Terms',
    savePdf: 'Save PDF',
    billTo: 'Bill To', from: 'From',
    subtotal: 'Subtotal', discountAmt: 'Discount', vat: 'VAT (7%)', total: 'Total',
    currency: '฿', savedMsg: 'Company info saved',
    payment: 'Payment Info', bankAccounts: 'Bank Accounts', bankName: 'Bank',
    accountName: 'Account Name', accountNumber: 'Account No.', branch: 'Branch',
    addBank: '+ Add Bank', promptPay: 'PromptPay', promptPayPhone: 'Phone',
    promptPayId: 'National ID', promptPayTax: 'Tax ID', qrCode: 'QR Code',
    signature: 'Signature', stamp: 'Stamp', signatureStamp: 'Signature / Stamp',
    showSignature: 'Show Signature', showStamp: 'Show Stamp',
    docSettings: 'Document Settings', autoNumber: 'Auto Number',
    dueTerm: 'Payment Terms', onreceipt: 'Due on Receipt', net7: 'Net 7 days', net15: 'Net 15 days',
    net30: 'Net 30 days', net60: 'Net 60 days', custom: 'Custom',
    validityTerm: 'Validity',
    currencyLabel: 'Currency', vatInclusive: 'Prices include VAT',
    withholdingTax: 'Withholding Tax', withholdingRate: 'Rate (%)',
    netPayable: 'Net Payable',
    newDoc: 'New', saveDoc: 'Save', history: 'History',
    exportData: 'Export', importData: 'Import',
    historyTitle: 'Document History', searchPlaceholder: 'Search (number / customer)',
    noDocs: 'No saved documents yet', open: 'Open', duplicate: 'Duplicate', remove: 'Delete',
    convertToInvoice: 'Convert to Invoice', close: 'Close',
    confirmDelete: 'Delete this document?', confirmImport: 'This will replace ALL current data. Continue?',
    docSavedMsg: 'Document saved', importSuccessMsg: 'Backup imported successfully',
    importFailMsg: 'Failed to import file',
    paymentSection: 'Payment Information', referenceLabel: 'Ref.', uploadImage: 'Upload image',
    saveFirst: 'Please fill in minimum info (number + customer name + at least 1 item)'
  }
};
```

- [ ] **Step 3: Verify in browser**

Reload. Switch language EN ↔ TH using existing toggle. Existing labels still translate correctly.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(i18n): add translations for all new UI labels"
```

---

## Task 9: HTML — Toolbar buttons (New, Save, History, Export, Import)

**Files:**
- Modify: `index.html` (toolbar markup)

- [ ] **Step 1: Locate toolbar**

Search anchor: `class="toolbar"`. Read 30 lines.

- [ ] **Step 2: Replace toolbar block**

Find the existing toolbar `<div class="toolbar">...</div>`. The structure has segments and a "Save PDF" button. Replace the toolbar entirely with:

```html
    <div class="toolbar">
      <div class="seg" id="docTypeSeg">
        <button data-val="invoice" class="active" data-i18n="invoice">ใบแจ้งหนี้</button>
        <button data-val="quotation" data-i18n="quotation">ใบเสนอราคา</button>
      </div>
      <div class="seg" id="langSeg">
        <button data-val="th" class="active">ไทย</button>
        <button data-val="en">EN</button>
      </div>
      <div style="flex:1;"></div>
      <button class="btn btn-secondary" id="btnNewDoc" data-i18n="newDoc">เอกสารใหม่</button>
      <button class="btn btn-secondary" id="btnSaveDoc" data-i18n="saveDoc">บันทึกเอกสาร</button>
      <button class="btn btn-secondary" id="btnHistory" data-i18n="history">ประวัติ</button>
      <button class="btn btn-secondary" id="btnExport" data-i18n="exportData">Export</button>
      <button class="btn btn-secondary" id="btnImport" data-i18n="importData">Import</button>
      <input type="file" id="importInput" accept="application/json" style="display:none;">
      <button class="btn" id="btnPrint" data-i18n="savePdf">บันทึก PDF</button>
    </div>
```

- [ ] **Step 3: Verify in browser**

Reload. Toolbar shows: doc type segment, lang segment, then "เอกสารใหม่ บันทึกเอกสาร ประวัติ Export Import บันทึก PDF". No console errors. Existing buttons still work.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(toolbar): add New/Save/History/Export/Import buttons"
```

---

## Task 10: HTML — Document settings form section

**Files:**
- Modify: `index.html` (form panel — after the existing doc number/date inputs)

- [ ] **Step 1: Locate insertion point**

Search anchor: `data-i18n="docInfo"`. The existing block is the document info section with number, date, dueDate inputs. Read 25 lines around to see the block.

- [ ] **Step 2: Replace the entire `docInfo` section**

Find the section starting with `<h2 data-i18n="docInfo">` through the closing `</div>` that ends the document info grid (look for the `data-bind="dueDate"` input — block ends shortly after).

Replace with:
```html
    <h2 data-i18n="docInfo">ข้อมูลเอกสาร</h2>
    <div class="grid-2">
      <div class="field">
        <label data-i18n="docNumber">เลขที่</label>
        <input type="text" data-bind="docNumber" id="docNumberInput">
      </div>
      <div class="field">
        <label>&nbsp;</label>
        <div class="checkbox-row">
          <input type="checkbox" id="autoNumberToggle" data-bind="docNumberAuto">
          <label for="autoNumberToggle" data-i18n="autoNumber">เลขที่อัตโนมัติ</label>
        </div>
      </div>
      <div class="field">
        <label data-i18n="date">วันที่</label>
        <input type="date" data-bind="docDate">
      </div>
      <div class="field">
        <label data-i18n="dueTerm">เงื่อนไขชำระ</label>
        <select data-bind="dueTerm" id="dueTermSelect">
          <option value="onreceipt" data-i18n="onreceipt">ชำระทันที</option>
          <option value="net7" data-i18n="net7">Net 7 วัน</option>
          <option value="net15" data-i18n="net15">Net 15 วัน</option>
          <option value="net30" data-i18n="net30" selected>Net 30 วัน</option>
          <option value="net60" data-i18n="net60">Net 60 วัน</option>
          <option value="custom" data-i18n="custom">กำหนดเอง</option>
        </select>
      </div>
      <div class="field">
        <label data-i18n="dueDate">วันครบกำหนด</label>
        <input type="date" data-bind="dueDate" id="dueDateInput">
      </div>
      <div class="field">
        <label data-i18n="currencyLabel">สกุลเงิน</label>
        <select data-bind="currency">
          <option value="THB">฿ THB</option>
          <option value="USD">$ USD</option>
          <option value="EUR">€ EUR</option>
          <option value="JPY">¥ JPY</option>
          <option value="GBP">£ GBP</option>
          <option value="CNY">¥ CNY</option>
        </select>
      </div>
      <div class="field" style="grid-column: span 2;">
        <label data-i18n="referenceLabel">อ้างอิง</label>
        <input type="text" data-bind="reference" placeholder="QUO-... (optional)">
      </div>
    </div>
```

- [ ] **Step 3: Verify in browser**

Reload. Document info section shows: number, auto checkbox, date, due term dropdown, due date, currency, reference. No console errors. Changing the inputs still updates state (try changing currency — although preview rendering happens later in Task 16, check `state.currency` in console).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(form): replace doc info section with full settings (auto-number, due term, currency, reference)"
```

---

## Task 11: HTML — Payment information form section

**Files:**
- Modify: `index.html` (form panel — after Customer section)

- [ ] **Step 1: Locate insertion point**

Search anchor: `data-i18n="customer"` heading. The customer section is followed by `data-i18n="items"`. Insert payment section between them.

- [ ] **Step 2: Insert payment section**

Just before the line containing `<h2 data-i18n="items">`, insert:
```html
    <h2 data-i18n="payment">ข้อมูลการชำระเงิน</h2>

    <div class="field">
      <label data-i18n="bankAccounts">บัญชีธนาคาร</label>
      <div id="bankAccountsList"></div>
      <button type="button" class="btn btn-secondary" id="btnAddBank" style="margin-top:6px; font-size:0.85rem;" data-i18n="addBank">+ เพิ่มบัญชี</button>
    </div>

    <div class="grid-2">
      <div class="field">
        <label data-i18n="promptPay">พร้อมเพย์</label>
        <select data-bind="company.promptPay.type">
          <option value="phone" data-i18n="promptPayPhone">เบอร์โทร</option>
          <option value="id" data-i18n="promptPayId">บัตรประชาชน</option>
          <option value="taxId" data-i18n="promptPayTax">เลขผู้เสียภาษี</option>
        </select>
      </div>
      <div class="field">
        <label>&nbsp;</label>
        <input type="text" data-bind="company.promptPay.value" placeholder="081-234-5678">
      </div>
    </div>

    <div class="field">
      <label data-i18n="qrCode">QR Code</label>
      <div class="logo-upload">
        <img id="qrPreview" alt="" style="display:none; max-width:120px;">
        <input type="file" id="qrInput" accept="image/*">
      </div>
    </div>

    <div class="grid-2">
      <div class="field">
        <div class="checkbox-row">
          <input type="checkbox" id="showBanksToggle" data-bind="paymentDisplay.showBanks" checked>
          <label for="showBanksToggle">แสดงบัญชีธนาคาร / Show banks</label>
        </div>
      </div>
      <div class="field">
        <div class="checkbox-row">
          <input type="checkbox" id="showQRToggle" data-bind="paymentDisplay.showQR" checked>
          <label for="showQRToggle">แสดง QR / Show QR</label>
        </div>
      </div>
    </div>
```

- [ ] **Step 3: Verify in browser**

Reload. Payment section appears between Customer and Items. Bank accounts list area is empty (rendering done in Task 13). PromptPay type dropdown + value input present. QR upload present.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(form): add Payment Information section (banks, PromptPay, QR)"
```

---

## Task 12: HTML — Signature, Withholding, and additional form sections

**Files:**
- Modify: `index.html` (company section — add signature/stamp uploads; summary section — add VAT inclusive + withholding)

- [ ] **Step 1: Add signature/stamp uploads to company section**

Search anchor: `id="logoInput"`. Find the existing logo upload `<div class="field">` block (logo + companyName + address fields are siblings).

After the `</div>` of the logo upload field, before the `<div class="field"><label data-i18n="companyName">` line, insert:

```html
    <div class="grid-2">
      <div class="field">
        <label data-i18n="signature">ลายเซ็น</label>
        <div class="logo-upload">
          <img id="signaturePreview" alt="" style="display:none; max-width:140px;">
          <input type="file" id="signatureInput" accept="image/*">
        </div>
      </div>
      <div class="field">
        <label data-i18n="stamp">ตราประทับ</label>
        <div class="logo-upload">
          <img id="stampPreview" alt="" style="display:none; max-width:120px;">
          <input type="file" id="stampInput" accept="image/*">
        </div>
      </div>
    </div>
```

- [ ] **Step 2: Extend Summary section with VAT inclusive + withholding + signature toggles**

Search anchor: `data-i18n="summary"`. Find the summary section's closing pattern: the `<textarea data-bind="notes"` block. Just **before** the notes textarea field, insert:

```html
    <div class="grid-2">
      <div class="field">
        <div class="checkbox-row">
          <input type="checkbox" id="vatInclusiveToggle" data-bind="vatInclusive">
          <label for="vatInclusiveToggle" data-i18n="vatInclusive">ราคารวม VAT แล้ว</label>
        </div>
      </div>
      <div class="field">
        <div class="checkbox-row">
          <input type="checkbox" id="whtToggle" data-bind="withholding.enabled">
          <label for="whtToggle" data-i18n="withholdingTax">หัก ณ ที่จ่าย</label>
        </div>
      </div>
    </div>
    <div class="grid-2">
      <div class="field">
        <label data-i18n="withholdingRate">อัตรา (%)</label>
        <select data-bind="withholding.rate">
          <option value="1">1%</option>
          <option value="3" selected>3%</option>
          <option value="5">5%</option>
        </select>
      </div>
      <div class="field"></div>
    </div>
    <div class="grid-2">
      <div class="field">
        <div class="checkbox-row">
          <input type="checkbox" id="showSigToggle" data-bind="showSignature">
          <label for="showSigToggle" data-i18n="showSignature">แสดงลายเซ็น</label>
        </div>
      </div>
      <div class="field">
        <div class="checkbox-row">
          <input type="checkbox" id="showStampToggle" data-bind="showStamp">
          <label for="showStampToggle" data-i18n="showStamp">แสดงตราประทับ</label>
        </div>
      </div>
    </div>
```

- [ ] **Step 3: Verify in browser**

Reload. Company section now has signature + stamp upload fields. Summary section shows VAT inclusive checkbox, withholding checkbox, withholding rate dropdown, signature/stamp display toggles. No console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(form): add signature/stamp uploads, VAT inclusive, withholding, display toggles"
```

---

## Task 13: Bank accounts dynamic list rendering

**Files:**
- Modify: `index.html` (script — form rendering section)

- [ ] **Step 1: Locate insertion point**

Search anchor: `function renderItemsForm()`. Insert new function immediately after this function ends (after its closing `}`).

- [ ] **Step 2: Add bank list rendering**

Insert:
```js
function renderBankAccountsForm() {
  const list = document.getElementById('bankAccountsList');
  if (!list) return;
  const accounts = state.company.bankAccounts || [];
  if (accounts.length === 0) {
    list.innerHTML = '<div style="color:var(--muted); font-size:0.85rem; padding:6px 0;">— ยังไม่มีบัญชี / no accounts —</div>';
    return;
  }
  list.innerHTML = accounts.map((a, i) => `
    <div class="grid-2" style="margin-bottom:6px; padding:8px; background:var(--bg); border-radius:6px;">
      <div class="field"><label>${t('bankName')}</label><input type="text" value="${esc(a.bankName)}" data-bank="${i}" data-bkey="bankName"></div>
      <div class="field"><label>${t('branch')}</label><input type="text" value="${esc(a.branch || '')}" data-bank="${i}" data-bkey="branch"></div>
      <div class="field"><label>${t('accountName')}</label><input type="text" value="${esc(a.accountName)}" data-bank="${i}" data-bkey="accountName"></div>
      <div class="field"><label>${t('accountNumber')}</label><input type="text" value="${esc(a.accountNumber)}" data-bank="${i}" data-bkey="accountNumber"></div>
      <div class="field" style="grid-column: span 2; display:flex; justify-content:flex-end;">
        <button type="button" class="btn btn-danger" data-bank-remove="${i}" style="font-size:0.8rem;">× ${t('remove')}</button>
      </div>
    </div>
  `).join('');
}
```

- [ ] **Step 3: Verify in browser**

Reload. The bank accounts area shows "— ยังไม่มีบัญชี —". (Add button wiring is in Task 17.) Console:
```js
state.company.bankAccounts.push({bankName:'KBank', accountName:'Test', accountNumber:'123-4', branch:'HQ'});
renderBankAccountsForm();
```
Expected: a row with 4 fields appears.

Reset: `state.company.bankAccounts = []; renderBankAccountsForm();`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(form): render bank accounts list dynamically"
```

---

## Task 14: Sync all new images and bank list at startup

**Files:**
- Modify: `index.html` (`syncFormFromState` function)

- [ ] **Step 1: Locate function**

Search anchor: `function syncFormFromState()`. Read full function (~12 lines).

- [ ] **Step 2: Replace function**

Find:
```js
function syncFormFromState() {
  $$('[data-bind]').forEach(el => {
    const path = el.getAttribute('data-bind');
    const v = getNested(state, path);
    if (el.type === 'checkbox') el.checked = !!v;
    else el.value = v ?? '';
  });
  if (state.company.logoDataUrl) {
    $('#logoPreview').src = state.company.logoDataUrl;
    $('#logoPreview').style.display = '';
  }
}
```

Replace with:
```js
function syncFormFromState() {
  $$('[data-bind]').forEach(el => {
    const path = el.getAttribute('data-bind');
    const v = getNested(state, path);
    if (el.type === 'checkbox') el.checked = !!v;
    else el.value = v ?? '';
  });
  syncImagePreview('logoPreview', state.company.logoDataUrl);
  syncImagePreview('signaturePreview', state.company.signatureDataUrl);
  syncImagePreview('stampPreview', state.company.stampDataUrl);
  syncImagePreview('qrPreview', state.company.qrDataUrl);
  renderBankAccountsForm();
}
function syncImagePreview(id, dataUrl) {
  const el = document.getElementById(id);
  if (!el) return;
  if (dataUrl) { el.src = dataUrl; el.style.display = ''; }
  else { el.removeAttribute('src'); el.style.display = 'none'; }
}
```

- [ ] **Step 3: Verify in browser**

Reload. Form populates with no errors. If a logo was previously saved, it still shows. The empty signature/stamp/QR fields show no broken image icons.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(form): sync signature, stamp, QR previews and bank list on load"
```

---

## Task 15: Preview — payment section

**Files:**
- Modify: `index.html` (script — add payment renderer; CSS)

- [ ] **Step 1: Add CSS for payment block**

Search anchor: `.doc-footer {` in the `<style>` block. Insert before this rule:
```css
  .doc-payment {
    margin-top: 1.5rem;
    padding: 1rem 1.25rem;
    border: 1px solid var(--border);
    border-radius: 8px;
    background: rgba(184, 149, 106, 0.06);
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 1rem;
  }
  .doc-payment .label {
    font-family: 'Fraunces', serif;
    font-size: 0.7rem;
    letter-spacing: 0.18em;
    color: var(--muted);
    text-transform: uppercase;
    margin-bottom: 0.5rem;
  }
  .doc-payment .bank-row {
    font-size: 0.85rem;
    margin-bottom: 4px;
    line-height: 1.4;
  }
  .doc-payment .bank-row strong { font-weight: 600; }
  .doc-payment .promptpay { font-size: 0.85rem; margin-top: 6px; }
  .doc-payment .qr img { max-width: 110px; height: auto; display: block; }
  .doc-signature {
    margin-top: 2rem;
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 2rem;
    align-items: end;
  }
  .doc-signature .sig-cell { text-align: center; }
  .doc-signature .sig-cell img { max-height: 80px; max-width: 200px; margin: 0 auto 4px; display: block; }
  .doc-signature .sig-line {
    border-top: 1px solid var(--rule);
    padding-top: 6px;
    font-size: 0.78rem;
    color: var(--muted);
    letter-spacing: 0.08em;
    text-transform: uppercase;
  }
```

- [ ] **Step 2: Add payment + signature renderers**

Search anchor: `function renderAll()`. Insert before this function:
```js
function renderPaymentSection() {
  const c = state.company;
  const pd = state.paymentDisplay || {};
  const banks = (c.bankAccounts || []);
  const ppValue = c.promptPay && c.promptPay.value;
  const qrUrl = c.qrDataUrl;
  const showBanks = pd.showBanks !== false && banks.length > 0;
  const showPP = pd.showPromptPay !== false && ppValue;
  const showQR = pd.showQR !== false && qrUrl;
  if (!showBanks && !showPP && !showQR) return '';

  const banksHtml = showBanks ? `
    <div class="bank-list">
      <div class="label">${t('bankAccounts')}</div>
      ${banks.map(a => `
        <div class="bank-row">
          <strong>${esc(a.bankName)}</strong>${a.branch ? ' · ' + esc(a.branch) : ''}<br>
          ${esc(a.accountName)} · ${esc(a.accountNumber)}
        </div>
      `).join('')}
    </div>
  ` : '';

  const ppLabel = c.promptPay && c.promptPay.type === 'phone' ? t('promptPayPhone')
    : c.promptPay && c.promptPay.type === 'id' ? t('promptPayId')
    : t('promptPayTax');
  const ppHtml = showPP ? `
    <div class="promptpay">
      <span class="label" style="margin:0; display:inline;">${t('promptPay')}</span>
      &nbsp;${esc(ppValue)} <span style="color:var(--muted); font-size:0.75rem;">(${ppLabel})</span>
    </div>
  ` : '';

  const qrHtml = showQR ? `<div class="qr"><div class="label">${t('qrCode')}</div><img src="${esc(qrUrl)}" alt=""></div>` : '<div></div>';

  return `
    <div class="doc-payment">
      <div>
        <div class="label">${t('paymentSection')}</div>
        ${banksHtml}
        ${ppHtml}
      </div>
      ${qrHtml}
    </div>
  `;
}
function renderSignatureBlock() {
  const c = state.company;
  const showSig = state.showSignature && c.signatureDataUrl;
  const showStamp = state.showStamp && c.stampDataUrl;
  if (!showSig && !showStamp) return '';
  return `
    <div class="doc-signature">
      <div class="sig-cell">
        ${showSig ? `<img src="${esc(c.signatureDataUrl)}" alt="">` : '<div style="height:80px;"></div>'}
        <div class="sig-line">${t('signature')}</div>
      </div>
      <div class="sig-cell">
        ${showStamp ? `<img src="${esc(c.stampDataUrl)}" alt="">` : '<div style="height:80px;"></div>'}
        <div class="sig-line">${t('stamp')}</div>
      </div>
    </div>
  `;
}
```

- [ ] **Step 3: Verify in browser**

Reload. No errors. (Wiring into preview happens next task.) Console:
```js
state.company.bankAccounts = [{bankName:'KBank', accountName:'Test', accountNumber:'123-4', branch:'HQ'}];
console.log(renderPaymentSection())  // returns HTML string with bank row
```

Reset: `state.company.bankAccounts = []`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(preview): add payment section and signature block renderers"
```

---

## Task 16: Preview — wire up payment, signature, currency, reference, withholding into renderPreview

**Files:**
- Modify: `index.html` (`renderPreview` function)

- [ ] **Step 1: Locate function**

Search anchor: `function renderPreview()`. Read entire function (~80 lines).

- [ ] **Step 2: Replace `renderPreview`**

Find the entire function. Replace with:
```js
function renderPreview() {
  const totals = computeTotals();
  const dueLabel = state.docType === 'invoice' ? t('dueDate') : t('expiryDate');
  const docTitle = state.docType === 'invoice' ? t('invoice') : t('quotation');
  const eyebrow = state.docType === 'invoice' ? '— BILLING DOCUMENT —' : '— PROPOSAL DOCUMENT —';

  const itemsHtml = state.items.filter(it => it.description || it.qty || it.unitPrice).map(it => `
    <tr>
      <td>${nl2br(it.description) || '&nbsp;'}</td>
      <td>${fmt(it.qty)}</td>
      <td>${fmt(it.unitPrice)}</td>
      <td>${fmt((it.qty || 0) * (it.unitPrice || 0))}</td>
    </tr>
  `).join('') || `<tr><td colspan="4" style="text-align:center; padding:24px; color:var(--ink-faint); font-style:italic;">— no items —</td></tr>`;

  const c = state.company;
  const cu = state.customer;
  const companyMeta = [c.address, c.phone && `T  ${c.phone}`, c.email && `E  ${c.email}`, c.taxId && `${t('taxId')}  ${c.taxId}`]
    .filter(Boolean).join('\n');
  const customerMeta = [cu.address, cu.phone && `T  ${cu.phone}`, cu.taxId && `${t('taxId')}  ${cu.taxId}`]
    .filter(Boolean).join('\n');

  const showWHT = state.withholding && state.withholding.enabled && state.currency === 'THB';
  const referenceHtml = state.reference ? `
    <div class="cell">
      <div class="label">${t('referenceLabel')}</div>
      <div class="value">${esc(state.reference)}</div>
    </div>
  ` : '';
  const paymentHtml = state.docType === 'invoice' ? renderPaymentSection() : '';
  const signatureHtml = renderSignatureBlock();

  $('#doc').innerHTML = `
    <div class="doc-hero">
      <div class="brand">
        ${c.logoDataUrl ? `<img src="${c.logoDataUrl}" alt="">` : ''}
        <div class="company-name">${esc(c.name) || '&nbsp;'}</div>
        <div class="company-meta">${nl2br(companyMeta)}</div>
      </div>
      <div class="stamp">
        <div class="eyebrow">${eyebrow}</div>
        <div class="doc-title">${docTitle}</div>
      </div>
    </div>

    <div class="doc-meta-strip">
      <div class="cell">
        <div class="label">${t('docNumber')}</div>
        <div class="value">${esc(state.docNumber) || '—'}</div>
      </div>
      <div class="cell">
        <div class="label">${t('date')}</div>
        <div class="value">${esc(state.docDate) || '—'}</div>
      </div>
      <div class="cell">
        <div class="label">${dueLabel}</div>
        <div class="value">${esc(state.dueDate) || '—'}</div>
      </div>
      ${referenceHtml}
    </div>

    <div class="billto">
      <div class="label">${t('billTo')}</div>
      <div class="body">
        <div class="name">${esc(cu.name) || '&nbsp;'}</div>
        <div class="meta">${nl2br(customerMeta)}</div>
      </div>
    </div>

    <table class="doc-items">
      <thead>
        <tr>
          <th>${t('description')}</th>
          <th>${t('qty')}</th>
          <th>${t('unitPrice')}</th>
          <th>${t('amount')}</th>
        </tr>
      </thead>
      <tbody>${itemsHtml}</tbody>
    </table>

    <div class="doc-bottom">
      ${state.notes ? `
        <div class="doc-notes-block">
          <div class="label">${t('notes')}</div>
          <div class="text">${nl2br(state.notes)}</div>
        </div>
      ` : '<div></div>'}
      <div class="doc-totals">
        <div class="row"><span>${t('subtotal')}</span><span>${fmtMoney(totals.subtotal)}</span></div>
        ${state.discountPct > 0 ? `<div class="row"><span>${t('discountAmt')} · ${state.discountPct}%</span><span>− ${fmtMoney(totals.discountAmt)}</span></div>` : ''}
        ${state.vatEnabled ? `<div class="row"><span>${t('vat')}${state.vatInclusive ? ' (incl.)' : ''}</span><span>${fmtMoney(totals.vat)}</span></div>` : ''}
        <div class="row grand"><span>${t('total')}</span><span>${fmtMoney(totals.total)}</span></div>
        ${showWHT ? `<div class="row"><span>${t('withholdingTax')} · ${state.withholding.rate}%</span><span>− ${fmtMoney(totals.withholding)}</span></div>` : ''}
        ${showWHT ? `<div class="row grand"><span>${t('netPayable')}</span><span>${fmtMoney(totals.netPayable)}</span></div>` : ''}
      </div>
    </div>

    ${paymentHtml}
    ${signatureHtml}

    <div class="doc-footer">
      <div>${esc(c.name) || '&nbsp;'}</div>
      <div class="ornament">✦ ✦ ✦</div>
      <div>${docTitle} · ${esc(state.docNumber) || '—'}</div>
    </div>
  `;
}
```

- [ ] **Step 3: Verify in browser**

Reload. Preview renders cleanly. Use console:
```js
state.currency = 'USD'; renderPreview();   // totals show "$ X" not "฿ X"
state.currency = 'THB'; renderPreview();
state.withholding = {enabled:true, rate:3}; renderPreview();
state.reference = 'QUO-2026-0099'; renderPreview();  // ref appears in meta strip
```

Add a bank account via console and confirm payment block appears on invoice but not quotation:
```js
state.company.bankAccounts = [{bankName:'KBank', accountName:'A', accountNumber:'1', branch:'HQ'}];
state.docType = 'invoice'; renderPreview();   // payment block visible
state.docType = 'quotation'; renderPreview(); // payment block hidden
```

Reset: `state.company.bankAccounts=[]; state.docType='invoice'; state.reference=''; state.withholding={enabled:false,rate:3}; renderPreview();`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(preview): render currency, reference, withholding, payment, signature"
```

---

## Task 17: Wire image uploads (signature, stamp, QR)

**Files:**
- Modify: `index.html` (`setupListeners` function)

- [ ] **Step 1: Locate listener for logo**

Search anchor: `// Logo upload`. Read 15 lines.

- [ ] **Step 2: Replace logo upload block with generic uploader + 4 wirings**

Find:
```js
  // Logo upload
  $('#logoInput').addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      state.company.logoDataUrl = reader.result;
      $('#logoPreview').src = reader.result;
      $('#logoPreview').style.display = '';
      renderPreview();
    };
    reader.readAsDataURL(file);
  });
```

Replace with:
```js
  // Image uploads (logo, signature, stamp, QR)
  function wireImageUpload(inputId, previewId, stateField) {
    const input = document.getElementById(inputId);
    if (!input) return;
    input.addEventListener('change', (e) => {
      const file = e.target.files[0];
      if (!file) return;
      if (file.size > 2 * 1024 * 1024) {
        alert('ไฟล์ใหญ่เกิน 2MB / file too large (max 2MB)');
        e.target.value = '';
        return;
      }
      const reader = new FileReader();
      reader.onload = () => {
        state.company[stateField] = reader.result;
        const img = document.getElementById(previewId);
        if (img) { img.src = reader.result; img.style.display = ''; }
        renderPreview();
      };
      reader.readAsDataURL(file);
    });
  }
  wireImageUpload('logoInput', 'logoPreview', 'logoDataUrl');
  wireImageUpload('signatureInput', 'signaturePreview', 'signatureDataUrl');
  wireImageUpload('stampInput', 'stampPreview', 'stampDataUrl');
  wireImageUpload('qrInput', 'qrPreview', 'qrDataUrl');
```

- [ ] **Step 3: Verify in browser**

Reload. Upload an image (any small PNG) into each of: logo, signature, stamp, QR. Confirm:
- Each preview appears
- Preview panel shows logo immediately
- Toggle "Show Signature" + ensure preview shows signature block
- Try uploading a 3MB+ file — alert appears, no upload

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(uploads): wire signature, stamp, QR uploads with size limit"
```

---

## Task 18: Wire bank account add/remove/edit

**Files:**
- Modify: `index.html` (`setupListeners`)

- [ ] **Step 1: Locate insertion point**

Search anchor: `// Save company`. Insert new wiring block before this comment.

- [ ] **Step 2: Add bank account wiring**

Insert before `// Save company`:
```js
  // Bank accounts: add
  $('#btnAddBank').addEventListener('click', () => {
    if (!state.company.bankAccounts) state.company.bankAccounts = [];
    state.company.bankAccounts.push({ bankName: '', accountName: '', accountNumber: '', branch: '' });
    renderBankAccountsForm();
    renderPreview();
  });
  // Bank accounts: edit + remove (event delegation)
  document.addEventListener('input', (e) => {
    const el = e.target;
    if (el.matches('[data-bank]')) {
      const i = Number(el.getAttribute('data-bank'));
      const k = el.getAttribute('data-bkey');
      if (state.company.bankAccounts && state.company.bankAccounts[i]) {
        state.company.bankAccounts[i][k] = el.value;
        renderPreview();
      }
    }
  });
  document.addEventListener('click', (e) => {
    const rm = e.target.closest('[data-bank-remove]');
    if (rm) {
      const i = Number(rm.getAttribute('data-bank-remove'));
      state.company.bankAccounts.splice(i, 1);
      renderBankAccountsForm();
      renderPreview();
    }
  });
```

- [ ] **Step 3: Verify in browser**

Reload. Click "+ เพิ่มบัญชี" — empty bank row appears. Type into fields — preview updates with payment section showing the new bank. Click "× ลบ" on the row — it disappears.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(banks): wire add, edit, remove for bank accounts"
```

---

## Task 19: Wire due-term selector to auto-compute due date

**Files:**
- Modify: `index.html` (`setupListeners`)

- [ ] **Step 1: Locate insertion point**

Search anchor: `// Doc type segment`. Insert before this comment.

- [ ] **Step 2: Add due-term + docDate listeners**

Insert before `// Doc type segment`:
```js
  // Due term: when changed, auto-compute due date (unless custom)
  document.addEventListener('change', (e) => {
    if (e.target.id === 'dueTermSelect') {
      state.dueTerm = e.target.value;
      applyDueTerm();
      const dueInput = document.getElementById('dueDateInput');
      if (dueInput) dueInput.value = state.dueDate;
      dueInput.disabled = state.dueTerm !== 'custom';
      renderPreview();
    }
    if (e.target.matches('[data-bind="docDate"]')) {
      // recompute due date if not custom
      applyDueTerm();
      const dueInput = document.getElementById('dueDateInput');
      if (dueInput) dueInput.value = state.dueDate;
      renderPreview();
    }
  });
```

- [ ] **Step 3: Verify in browser**

Reload. Set issue date to today. Select "Net 30" — due date input auto-fills 30 days from today. Select "Custom" — due date input becomes editable, value stays. Select "Net 7" — due date jumps back to 7-day calculation. Change issue date → due date recalculates.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(due-date): wire due term selector to auto-compute due date"
```

---

## Task 20: Wire New / Save document buttons

**Files:**
- Modify: `index.html` (`setupListeners`)

- [ ] **Step 1: Locate insertion point**

Search anchor: `// Print`. Insert before this comment.

- [ ] **Step 2: Add New + Save wiring**

Insert before `// Print`:
```js
  // New document
  $('#btnNewDoc').addEventListener('click', () => {
    state.currentDocId = null;
    state.docType = state.docType || 'invoice';
    state.docNumberAuto = true;
    state.docNumber = nextDocNumber(state.docType);
    state.docDate = todayISO();
    state.dueTerm = state.docType === 'invoice' ? 'net30' : 'net30';
    applyDueTerm();
    state.reference = '';
    state.customer = { name: '', address: '', phone: '', taxId: '' };
    state.items = [{ description: '', qty: 1, unitPrice: 0 }];
    state.discountPct = 0;
    state.notes = '';
    state.withholding = { enabled: false, rate: 3 };
    syncFormFromState();
    renderItemsForm();
    renderPreview();
  });
  // Save document
  $('#btnSaveDoc').addEventListener('click', () => {
    if (!state.docNumber || !state.customer.name || state.items.every(it => !it.description && !it.qty && !it.unitPrice)) {
      alert(t('saveFirst'));
      return;
    }
    if (persistCurrentDoc()) {
      alert(t('docSavedMsg'));
    }
  });
```

- [ ] **Step 3: Verify in browser**

Reload. Click "เอกสารใหม่" — form clears, new doc number appears (e.g. INV-2026-0001), due date auto-set to today+30. Fill in customer name "ABC", item desc "Test", qty 1, price 100. Click "บันทึกเอกสาร" — alert "บันทึกเอกสารแล้ว". Console: `loadDocuments()` returns array with 1 doc. Click "เอกสารใหม่" again — number advances to INV-2026-0002. Save — `loadDocuments()` returns 2 docs.

Cleanup: `localStorage.removeItem(DOCS_KEY); localStorage.removeItem(COUNTERS_KEY); location.reload();`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(history): wire New and Save document buttons"
```

---

## Task 21: History modal — markup and styles

**Files:**
- Modify: `index.html` (HTML body + CSS)

- [ ] **Step 1: Add modal CSS**

In the `<style>` block, before the closing `</style>`, insert:
```css
  .modal-overlay {
    position: fixed; inset: 0;
    background: rgba(0,0,0,0.4);
    display: none;
    align-items: center; justify-content: center;
    z-index: 1000;
  }
  .modal-overlay.open { display: flex; }
  .modal {
    background: var(--panel);
    border-radius: 12px;
    padding: 1.5rem;
    width: min(800px, 92vw);
    max-height: 86vh;
    overflow-y: auto;
    box-shadow: 0 20px 60px rgba(0,0,0,0.25);
  }
  .modal h3 { margin-bottom: 1rem; font-family: 'Fraunces', serif; font-size: 1.4rem; }
  .modal .modal-toolbar { display: flex; gap: 0.5rem; margin-bottom: 1rem; }
  .modal .modal-toolbar input[type="search"] {
    flex: 1; padding: 8px 12px; border-radius: 6px; border: 1px solid var(--border); font: inherit;
  }
  .doc-history-table { width: 100%; border-collapse: collapse; font-size: 0.88rem; }
  .doc-history-table th, .doc-history-table td { padding: 8px 10px; text-align: left; border-bottom: 1px solid var(--border); }
  .doc-history-table th { color: var(--muted); font-weight: 500; }
  .doc-history-table .actions { display: flex; gap: 4px; flex-wrap: wrap; }
  .doc-history-table .actions button { padding: 4px 8px; font-size: 0.78rem; border-radius: 4px; }
  .doc-history-empty { padding: 32px; text-align: center; color: var(--muted); font-style: italic; }
```

- [ ] **Step 2: Add modal markup**

Just before the closing `</body>` tag, insert:
```html
<div class="modal-overlay" id="historyModal">
  <div class="modal">
    <h3 data-i18n="historyTitle">ประวัติเอกสาร</h3>
    <div class="modal-toolbar">
      <input type="search" id="historySearch" placeholder="ค้นหา (เลขที่ / ลูกค้า)" data-i18n-placeholder="searchPlaceholder">
      <button class="btn btn-secondary" id="btnHistoryClose" data-i18n="close">ปิด</button>
    </div>
    <div id="historyListContainer"></div>
  </div>
</div>
```

- [ ] **Step 3: Verify in browser**

Reload. Page renders normally. Console:
```js
document.getElementById('historyModal').classList.add('open')   // modal appears centered
document.getElementById('historyModal').classList.remove('open') // modal hides
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(history-modal): add modal markup and styles"
```

---

## Task 22: History modal — list rendering and actions

**Files:**
- Modify: `index.html` (script — add renderHistoryList; setupListeners)

- [ ] **Step 1: Add list renderer**

Search anchor: `function renderBankAccountsForm()`. Insert immediately after this function (after its closing `}`):
```js
function renderHistoryList(filter) {
  const container = document.getElementById('historyListContainer');
  if (!container) return;
  const docs = loadDocuments();
  const f = (filter || '').trim().toLowerCase();
  const filtered = !f ? docs : docs.filter(d =>
    (d.docNumber || '').toLowerCase().includes(f) ||
    (d.customer && d.customer.name || '').toLowerCase().includes(f)
  );
  if (filtered.length === 0) {
    container.innerHTML = `<div class="doc-history-empty">${t('noDocs')}</div>`;
    return;
  }
  container.innerHTML = `
    <table class="doc-history-table">
      <thead>
        <tr>
          <th>${t('docNumber')}</th>
          <th>${t('customer')}</th>
          <th>${t('date')}</th>
          <th>${t('total')}</th>
          <th></th>
        </tr>
      </thead>
      <tbody>
        ${filtered.map(d => {
          const isQuote = d.docType === 'quotation';
          const grand = (d.items || []).reduce((s, it) => s + (Number(it.qty)||0)*(Number(it.unitPrice)||0), 0);
          const symbol = currencySymbol(d.currency || 'THB');
          return `
            <tr>
              <td>${esc(d.docNumber)}${isQuote ? ' <span style="color:var(--muted);font-size:0.75rem;">QUO</span>' : ''}</td>
              <td>${esc(d.customer && d.customer.name || '')}</td>
              <td>${esc(d.docDate)}</td>
              <td>${symbol} ${fmt(grand)}</td>
              <td class="actions">
                <button class="btn btn-secondary" data-doc-open="${d.id}">${t('open')}</button>
                <button class="btn btn-secondary" data-doc-dup="${d.id}">${t('duplicate')}</button>
                ${isQuote ? `<button class="btn btn-secondary" data-doc-convert="${d.id}">${t('convertToInvoice')}</button>` : ''}
                <button class="btn btn-danger" data-doc-del="${d.id}">${t('remove')}</button>
              </td>
            </tr>
          `;
        }).join('')}
      </tbody>
    </table>
  `;
}
```

- [ ] **Step 2: Wire history button + modal actions**

Search anchor: `// New document` (added in Task 20). Insert immediately after the `btnNewDoc` listener:
```js
  // History modal
  $('#btnHistory').addEventListener('click', () => {
    renderHistoryList(document.getElementById('historySearch').value);
    document.getElementById('historyModal').classList.add('open');
  });
  $('#btnHistoryClose').addEventListener('click', () => {
    document.getElementById('historyModal').classList.remove('open');
  });
  document.getElementById('historyModal').addEventListener('click', (e) => {
    if (e.target.id === 'historyModal') {
      e.target.classList.remove('open');
    }
  });
  $('#historySearch').addEventListener('input', (e) => {
    renderHistoryList(e.target.value);
  });
  // History row actions (delegated)
  document.getElementById('historyListContainer').addEventListener('click', (e) => {
    const open = e.target.closest('[data-doc-open]');
    const dup = e.target.closest('[data-doc-dup]');
    const del = e.target.closest('[data-doc-del]');
    const conv = e.target.closest('[data-doc-convert]');
    if (open) {
      const doc = loadDocuments().find(d => d.id === open.getAttribute('data-doc-open'));
      if (doc) {
        applyDocSnapshot(doc);
        syncFormFromState();
        renderItemsForm();
        renderPreview();
        document.getElementById('historyModal').classList.remove('open');
      }
    } else if (dup) {
      const copy = duplicateDocument(dup.getAttribute('data-doc-dup'));
      if (copy) {
        applyDocSnapshot(copy);
        // duplicate is a NEW doc with new id — currentDocId should be the new one
        state.currentDocId = copy.id;
        syncFormFromState();
        renderItemsForm();
        renderPreview();
        document.getElementById('historyModal').classList.remove('open');
      }
    } else if (conv) {
      const inv = convertQuotationToInvoice(conv.getAttribute('data-doc-convert'));
      if (inv) {
        applyDocSnapshot(inv);
        state.currentDocId = inv.id;
        syncFormFromState();
        renderItemsForm();
        renderPreview();
        document.getElementById('historyModal').classList.remove('open');
      }
    } else if (del) {
      if (confirm(t('confirmDelete'))) {
        deleteDocument(del.getAttribute('data-doc-del'));
        renderHistoryList(document.getElementById('historySearch').value);
      }
    }
  });
```

- [ ] **Step 3: Verify in browser**

Reload. Save 2 documents (use New + Save twice with different customers). Click "ประวัติ" — modal opens listing both. Search by partial customer name — list filters. Click "เปิด" on a row — modal closes, form populates with that doc. Edit and save — same id is updated (loadDocuments still returns 2 docs). Click "คัดลอก" — new doc with next number appears in form. Click "ลบ" with confirm — row disappears.

For Q→I: switch type to quotation, save a quotation, open History — "แปลงเป็นใบแจ้งหนี้" button visible. Click — form switches to invoice type with new INV number, reference field shows the QUO number.

Cleanup: `localStorage.removeItem(DOCS_KEY); localStorage.removeItem(COUNTERS_KEY); location.reload();`

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(history): wire history modal with open/duplicate/convert/delete"
```

---

## Task 23: Wire Export and Import buttons

**Files:**
- Modify: `index.html` (`setupListeners`)

- [ ] **Step 1: Add wiring**

Search anchor: `// New document` (added in Task 20). Insert immediately AFTER the History modal block from Task 22:
```js
  // Export backup
  $('#btnExport').addEventListener('click', exportBackup);
  // Import backup
  $('#btnImport').addEventListener('click', () => $('#importInput').click());
  $('#importInput').addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (!file) return;
    if (!confirm(t('confirmImport'))) { e.target.value = ''; return; }
    importBackup(file).then(() => {
      // refresh state from imported company storage
      loadCompany();
      syncFormFromState();
      renderItemsForm();
      renderPreview();
      alert(t('importSuccessMsg'));
    }).catch(err => {
      alert(t('importFailMsg') + ': ' + err.message);
    }).finally(() => { e.target.value = ''; });
  });
```

- [ ] **Step 2: Verify in browser**

Reload. Save a document. Click "Export" — `.json` file downloads. Open file in editor — confirm shape (`version`, `company`, `documents`, `counters`).

Now clear all data: `localStorage.clear(); location.reload();` Click "Import", pick the downloaded file, confirm dialog → success alert. Open History — saved doc is back. Reload page — data persists.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(backup): wire Export and Import buttons with confirm"
```

---

## Task 24: Init updates — load counters, set initial doc number, sync due date

**Files:**
- Modify: `index.html` (init at end of script)

- [ ] **Step 1: Locate init**

Search anchor: `// ---- Init ----`. Read remaining lines.

- [ ] **Step 2: Replace init block**

Find:
```js
// ---- Init ----
loadCompany();
syncFormFromState();
renderItemsForm();
setupListeners();
renderAll();
```

Replace with:
```js
// ---- Init ----
loadCompany();
// Auto-fill initial document number if none
if (!state.docNumber) {
  state.docNumber = nextDocNumber(state.docType);
}
// Compute initial due date from default term
applyDueTerm();
syncFormFromState();
renderItemsForm();
setupListeners();
// Disable due date input if not custom
const initialDueInput = document.getElementById('dueDateInput');
if (initialDueInput) initialDueInput.disabled = state.dueTerm !== 'custom';
renderAll();
```

- [ ] **Step 3: Verify in browser**

Clear storage and reload: `localStorage.clear(); location.reload();`. Form opens with: `INV-2026-0001` in number field, today's date as docDate, today+30 in dueDate (read-only, since term=net30). Switch term to "Custom" — due date becomes editable.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(init): auto-fill initial doc number and due date on load"
```

---

## Task 25: i18n — apply to placeholder + select option text

**Files:**
- Modify: `index.html` (`applyI18nToForm` function)

- [ ] **Step 1: Locate function**

Search anchor: `function applyI18nToForm()`.

- [ ] **Step 2: Replace function**

Find:
```js
function applyI18nToForm() {
  $$('[data-i18n]').forEach(el => {
    const key = el.getAttribute('data-i18n');
    if (i18n[state.lang][key]) el.textContent = t(key);
  });
}
```

Replace with:
```js
function applyI18nToForm() {
  $$('[data-i18n]').forEach(el => {
    const key = el.getAttribute('data-i18n');
    if (i18n[state.lang][key]) el.textContent = t(key);
  });
  $$('[data-i18n-placeholder]').forEach(el => {
    const key = el.getAttribute('data-i18n-placeholder');
    if (i18n[state.lang][key]) el.placeholder = t(key);
  });
  // Re-translate select options that use data-i18n
  $$('select option[data-i18n]').forEach(opt => {
    const key = opt.getAttribute('data-i18n');
    if (i18n[state.lang][key]) opt.textContent = t(key);
  });
}
```

- [ ] **Step 3: Verify in browser**

Reload. Switch lang TH → EN. The due-term dropdown options now read "Net 30 days" etc. Search input placeholder in History modal updates. Switch back TH — translates back.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(i18n): translate placeholders and select options"
```

---

## Task 26: Final manual verification + cleanup

**Files:**
- Modify: none — verification only

- [ ] **Step 1: Full happy-path test (clean state)**

Run `localStorage.clear(); location.reload();` in DevTools.

Then perform this flow without errors:

1. Fill company: name, address, phone, email, taxId, upload logo, signature, stamp, QR
2. Add 2 bank accounts
3. Set PromptPay phone "081-234-5678"
4. Click "Save Company Info" — alert
5. Reload — company info persists, all images show
6. Default doc shows INV-2026-0001 + today + due 30 days out
7. Fill customer: "ABC Co", address, phone, taxId
8. Add 3 line items with descriptions, qty, prices
9. Set discount 5%, enable VAT, enable withholding 3%
10. Preview shows: subtotal, discount, VAT, total, withholding, net payable, payment block with both banks + PromptPay + QR, signature + stamp block
11. Click "บันทึกเอกสาร" — saved
12. Click "เอกสารใหม่" — number advances to INV-2026-0002
13. Switch type to Quotation — number becomes QUO-2026-0001
14. Save quotation
15. Open History — both docs listed; quotation row has "Convert" button
16. Click Convert — new INV-2026-0003 with `Ref: QUO-2026-0001` appears
17. Save it
18. Switch currency to USD — totals show "$ X" not "฿ X"; withholding section disappears
19. Click Export — JSON file downloads
20. `localStorage.clear(); location.reload();`
21. Click Import, pick file, confirm — data restored, History shows all docs
22. Click "บันทึก PDF" / Print — print preview shows the document including payment + signature blocks

- [ ] **Step 2: Edge cases**

- [ ] Save with empty number → alert "saveFirst"
- [ ] Upload >2MB image → alert; file rejected
- [ ] Toggle "Custom" due term → date input becomes editable
- [ ] Toggle "VAT inclusive" with VAT on → vat amount changes (extracted from prices), totals stay consistent
- [ ] Switch language EN ↔ TH everywhere translates including new sections
- [ ] Currency = USD hides withholding line in totals
- [ ] Quotation type hides payment block in preview (intended)
- [ ] Delete a doc from History → row gone; if it was the currently loaded one, currentDocId clears

- [ ] **Step 3: Commit verification log**

If any issue is found, fix and commit. Otherwise:
```bash
git commit --allow-empty -m "test: verified full feature set passes happy path and edge cases"
```

---

## Spec Coverage Self-Review

Mapping spec features → tasks:

| Spec feature | Task(s) |
|---|---|
| #1 Payment (banks, PromptPay, QR) | 11, 13, 15, 16, 17, 18 |
| #2 Signature/Stamp | 12, 15, 16, 17 |
| #3 Saved documents | 6, 20, 21, 22 |
| #4 Auto-running number | 3, 20, 24 |
| #5 Net X days | 4, 19, 24 |
| #6 VAT inclusive | 5, 12, 16 |
| #7 Withholding tax | 5, 12, 16 |
| #8 Convert Q→I | 6, 22 |
| #9 Currency selector | 2, 10, 16 |
| #10 Export/Import JSON | 7, 23 |
| State shape | 1 |
| i18n | 8, 25 |
| Toolbar | 9 |
| Init | 24 |
| Verification | 26 |

All 10 features and supporting infrastructure covered. No gaps.
