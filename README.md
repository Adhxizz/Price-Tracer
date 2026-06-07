# WhatsApp Price Tracker — n8n Automation

Get a WhatsApp message the moment a product price drops. This workflow scrapes any product URL on a schedule, compares the price against the last stored value in Google Sheets, and fires a WhatsApp alert when it finds a drop.

---

## How it works

```
Schedule Trigger (every 6h)
      ↓
Fetch Product Page  →  HTTP GET with browser-like headers
      ↓
Extract Price       →  Regex parser (Amazon IN, Flipkart, generic ₹)
      ↓
Read Last Price     →  Google Sheets — PriceLog tab
      ↓
Compare Prices      →  Calculates drop amount + % savings
      ↓
Price Dropped? (IF)
    ↙        ↘
  YES          NO → stop, wait for next run
    ↓
Update Sheets + Send WhatsApp Alert (parallel)
```

---

## Prerequisites

| What | Where to get it |
|------|-----------------|
| n8n instance | [n8n.io](https://n8n.io) — cloud or self-hosted |
| Google account | For Google Sheets storage |
| Meta Business account | [business.facebook.com](https://business.facebook.com) |
| WhatsApp Business API access | Via Meta Developer Portal |

> **Fastest way to test WhatsApp without API approval:** Use the **Twilio WhatsApp sandbox** (see the Twilio alternative section below).

---

## Setup — step by step

### Step 1 — Import the workflow

1. Open your n8n instance.
2. Click **Workflows → Import from file**.
3. Select `price-tracker-workflow.json`.
4. The workflow opens with 8 nodes. Do not activate it yet.

---

### Step 2 — Set up Google Sheets

1. Create a new Google Sheet and name it anything (e.g. `Price Tracker`).
2. Create a tab named exactly **`PriceLog`** (case-sensitive).
3. Add these headers in row 1:

```
| price | product_name | url | checked_at |
```

4. Copy the **Sheet ID** from the URL:
   `https://docs.google.com/spreadsheets/d/186MUfN-2ESoY5PTIe135RNvR9r-AOuT6xrm1HZGIuhc/edit`

---

### Step 3 — Connect Google Sheets credential

1. Click the **Read Last Price (Sheets)** node.
2. Under **Credential**, click **Create new** → choose **Google Sheets OAuth2**.
3. Follow the OAuth flow to connect your Google account.
4. Apply the same credential to the **Update Price in Sheets** node.

---

### Step 4 — Connect WhatsApp Business API

#### Option A — Meta WhatsApp Business API (recommended for production)

1. Go to [developers.facebook.com](https://developers.facebook.com) → **My Apps → Create App**.
2. Add the **WhatsApp** product to your app.
3. Under **WhatsApp → API Setup**, copy:
   - **Phone Number ID**
   - **Access Token** (generate a permanent token in Business Settings)
4. In n8n, click the **Send WhatsApp Alert** node.
5. Under **Credential**, click **Create new → WhatsApp Business Cloud API**.
6. Paste your **Access Token**.

#### Option B — Twilio WhatsApp Sandbox (fastest to test)

If you don't have Meta API access yet, use Twilio:

1. Sign up at [twilio.com](https://twilio.com) and activate the **WhatsApp Sandbox** in the console.
2. Send `join <your-sandbox-word>` from your personal WhatsApp to Twilio's sandbox number.
3. Delete the **Send WhatsApp Alert** node (the n8n WhatsApp node).
4. Add an **HTTP Request** node in its place with:
   - **Method:** POST
   - **URL:** `https://api.twilio.com/2010-04-01/Accounts/{{ ACCOUNT_SID }}/Messages.json`
   - **Authentication:** Basic Auth (Account SID as username, Auth Token as password)
   - **Body (Form-Data):**
     - `From`: `whatsapp:+14155238886` (Twilio sandbox number)
     - `To`: `whatsapp:+91XXXXXXXXXX` (your number with country code)
     - `Body`: your message template

---

### Step 5 — Configure variables

In n8n, go to **Settings → Variables** and create these:

| Variable name | Example value | Description |
|---|---|---|
| `PRODUCT_URL` | `https://www.amazon.in/dp/B09XS7JWHH` | Full URL of the product page |
| `SHEET_ID` | `1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms` | Google Sheet ID from the URL |
| `WHATSAPP_PHONE_NUMBER_ID` | `123456789012345` | From Meta Developer Portal |
| `RECIPIENT_PHONE` | `+919876543210` | Your WhatsApp number with country code |

> Variables keep your credentials out of the workflow JSON. Do not hardcode them inside nodes.

---

### Step 6 — Test before activating

1. Click **Execute workflow** (manual run button) to test.
2. Check each node output by clicking the node after execution.
3. Verify the **Extract Price** node returns a clean number.
4. On the first run, the sheet is empty so `lastPrice` will be `null` — no alert is sent (this is expected behaviour).
5. Manually edit the sheet: set `price` to a value higher than the current product price to simulate a drop.
6. Run again — you should receive a WhatsApp message.

---

### Step 7 — Activate

Once tests pass, toggle **Active** in the top-right corner. The workflow will now run automatically every 6 hours.

---

## Tracking multiple products

The current workflow tracks one URL via the `PRODUCT_URL` variable. To track multiple products:

1. Add all products as rows in the `PriceLog` sheet (one per row, with the URL filled in, price left blank initially).
2. Replace the **Schedule Trigger → HTTP Request** chain with:
   - A **Google Sheets** node set to read all rows.
   - A **Split In Batches** node (batch size = 1) to loop through each row.
   - Feed each row's `url` into the **Fetch Product Page** node.
3. The compare and alert logic stays the same.

---

## Customising the WhatsApp message

The alert message in the **Send WhatsApp Alert** node currently reads:

```
*Price Drop Alert!* 

*{{ productName }}*

 Old price: ₹{{ lastPrice }}
 New price: ₹{{ currentPrice }}
 You save: ₹{{ dropAmount }} ({{ dropPercent }}% off)

Buy now: {{ url }}

_Tracked by your n8n Price Bot_
```

Edit the `textBody` field in the node to change the format. WhatsApp supports basic markdown: `*bold*`, `_italic_`, `` `code` ``.

> **Note:** If using the Meta API with a Business account (not sandbox), message templates must be pre-approved by Meta before sending to users who haven't messaged you first. For sending to yourself, free-form messages work fine.

---

## Supported sites

The **Extract Price** node contains regex patterns for:

| Site | Status |
|---|---|
| Amazon India | Supported |
| Flipkart | Supported |
| Any site showing ₹ price | Generic fallback |

To add support for another site, open the **Extract Price** (Code node) and add a new pattern array following the existing examples.

---

## Troubleshooting

**"Could not extract price" error**
The page structure may have changed, or the site uses JavaScript to render the price (common with SPAs). Try fetching the URL in your browser's DevTools Network tab to see the actual HTML served. You may need to use a scraping API like Browserless or ScrapingBee instead of a plain HTTP request.

**WhatsApp message not delivered**
- Confirm the recipient number includes the country code (e.g. `+91` for India).
- If using Meta API, ensure your phone number is verified and the access token has `whatsapp_business_messaging` permission.
- If using Twilio sandbox, the recipient must have joined the sandbox first.

**Sheet not updating**
- Confirm the `PriceLog` tab name matches exactly (case-sensitive).
- Make sure the Google Sheets credential has edit access to the file.
- The `url` column is used as the matching key — ensure it contains the full URL on first insert.

**Schedule not running**
- The workflow must be set to **Active**.
- Self-hosted n8n requires the process to be running continuously (use PM2 or Docker).

---

## File structure

```
price-tracker-n8n/
├── price-tracker-workflow.json   ← import this into n8n
└── README.md                     ← this file
```

---

## Tech stack

- **n8n** — workflow automation engine
- **Google Sheets** — price storage and history log
- **WhatsApp Business Cloud API** — alert delivery
- **HTTP Request + Code node** — web scraping and price parsing

---

*Built with n8n. For questions or improvements, fork the workflow and adapt it to your needs.*
