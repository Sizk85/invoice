# Invoice / Quotation Generator

Simple web app to fill in invoice / quotation data and save as PDF. TH / EN support. Single static HTML file. Deployable to Coolify via Dockerfile.

## Features
- Toggle Invoice / Quotation
- TH / EN language toggle
- Logo upload + company info saved to LocalStorage
- Customer info, line items (add/remove)
- Discount %, VAT 7% toggle
- Notes / payment terms
- Live A4 preview
- Save PDF via browser print dialog

## Run locally
Just open `index.html` in a browser. That's it.

Or with Docker:
```
docker build -t invoice .
docker run -p 8080:80 invoice
```
Then open http://localhost:8080

## Deploy to Coolify
1. Push this repo to GitHub
2. In Coolify: New Resource → Public Repository → paste the repo URL
3. Build pack: **Dockerfile**
4. Port: **80**
5. Deploy

## How to save PDF
Click **Save PDF** → browser print dialog opens → choose **Save as PDF** as destination.
