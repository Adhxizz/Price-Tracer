# Telegram Price Tracker — n8n Automation

Get a Telegram message the moment a product price drops. This workflow scrapes any product URL on a schedule, compares it against the last price stored in Google Sheets, and fires a Telegram alert on a drop.

> ✅ **100% free-tier compatible.** No `$vars`, no paid n8n plan needed. All config lives in a single **Config (Set) node** at the top of the workflow.

---

## How it works

```
Schedule Trigger (every 6h)
        ↓
Config (Set node)    →  PRODUCT_URL, SHEET_ID, TELEGRAM_CHAT_ID
        ↓
Fetch Product Page   →  HTTP GET with browser-like headers
        ↓
Extract Price        →  Regex parser (Amazon IN, Flipkart, generic ₹)
        ↓
Read Last Price      →  Google Sheets — PriceLog tab
        ↓
Compare Prices       →  Calculates drop amount + % savings
        ↓
Price Dropped? (IF)
    ↙          ↘
  YES            NO → stop, wait for next run
    ↓
Update Sheets  +  Send Telegram Alert  (parallel)
```

---

## What you'll receive on Telegram

```
🔔 Price Drop Alert!

📦 Sony WH-1000XM5 Wireless Headphones
💸 Old price: ₹24990
✅ New price: ₹19999
🎉 You save: ₹4991 (19.9% off)

🛒 Buy now → [product link]

Tracked by your n8n Price Bot
```

---

## Prerequisites

| What | Where to get it |
|---|---|
| n8n instance (free) | [n8n.io](https://n8n.io) — cloud or self-hosted |
| Google account | For Google Sheets storage |
| Telegram account | [telegram.org](https://telegram.org) |
| Telegram Bot Token | Free — created via @BotFather in 2 minutes |

---

## Setup — step by step

### Step 1 — Import the workflow

1. Open your n8n instance.
2. Click **Workflows → Import from file**.
3. Select `price-tracker-telegram-workflow.json`.
4. You'll see **9 nodes** — do not activate yet.

---

### Step 2 — Fill in the Config node

This is the **only place** you need to edit values. No variables settings page required.

1. Click the **Config** node (second node, labelled with ✏️).
2. Update the three fields:

| Field | What to put |
|---|---|
| `PRODUCT_URL` | Full URL of the product page (Amazon, Flipkart, etc.) |
| `SHEET_ID` | Your Google Sheet ID (see Step 4 below) |
| `TELEGRAM_CHAT_ID` | Your Telegram Chat ID (see Step 3 below) |

That's it — all three values flow through the rest of the workflow automatically via `$('Config').first().json`.

---

### Step 3 — Create your Telegram Bot (2 minutes)

1. Open Telegram → search **@BotFather** → send `/newbot`.
2. Pick a display name and a username ending in `bot`.
3. BotFather gives you a **Bot Token** — looks like:
   ```
   7123456789:AAFxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```
4. **Get your Chat ID:**
   - Send any message to your new bot (e.g. `/start`).
   - Open this URL in a browser (swap in your token):
     ```
     https://api.telegram.org/bot8976660596:AAGGwmtuqHFgq1lgXKbKPVv85SEYUqKL7Tg/getUpdates
     ```
   - Find `"chat":{"id":` — that number is your **Chat ID**.
5. Paste the Chat ID into the **Config node** → `TELEGRAM_CHAT_ID` field.

> **Sending to a group or channel?** Add the bot as Admin, then use the group/channel Chat ID (starts with `-100`).

---

### Step 4 — Connect Telegram credential in n8n

1. Click the **Send Telegram Alert** node.
2. Under **Credential** → **Create new → Telegram API**.
3. Paste your **Bot Token** → Save.

---

### Step 5 — Set up Google Sheets

1. Create a new Google Sheet (name it anything).
2. Add a tab named exactly **`PriceLog`** (case-sensitive).
3. Put these headers in row 1:

   | price | product_name | url | checked_at |
   |---|---|---|---|

4. Copy the **Sheet ID** from the URL bar:
   `https://docs.google.com/spreadsheets/d/1rCmmN0h3jTTeREeVG2GfgAYcmEJinIH2Y5Gxgsjh8x4/edit`

5. Paste it into the **Config node** → `SHEET_ID` field.

---

### Step 6 — Connect Google Sheets credential

1. Click **Read Last Price (Sheets)** → **Credential → Create new → Google Sheets OAuth2**.
2. Complete the OAuth flow.
3. Apply the **same credential** to the **Update Price in Sheets** node.

---

### Step 7 — Test before activating

1. Click **Execute workflow** (manual run button).
2. Click through each node to inspect outputs.
3. Confirm **Extract Price** returns a clean number.
4. First run: sheet is empty → `lastPrice` is `null` → no alert fires. This is expected.
5. **Simulate a drop:** Manually set the `price` cell in your sheet to a value higher than the real current price. Run again — you should receive a Telegram message.

---

### Step 8 — Activate

Toggle **Active** (top-right). The workflow now runs every 6 hours automatically.

---

## Adjusting the check frequency

Open the **Schedule Trigger** node:

| Frequency | Setting |
|---|---|
| Every 6 hours (default) | Field: `Hours` → value: `6` |
| Every 1 hour | Field: `Hours` → value: `1` |
| Daily at 9 AM | Switch to `Cron` → `0 9 * * *` |
| Twice daily | Switch to `Cron` → `0 9,21 * * *` |

---

## Tracking multiple products

The Config node holds one URL. To track several products:

1. Add all products as rows in `PriceLog` (fill the `url` column, leave `price` blank).
2. Replace **Schedule Trigger → Config → Fetch Product Page** with:
   - A **Google Sheets** node reading all rows.
   - A **Split In Batches** node (batch size 1) to loop through each.
   - Feed each row's URL into **Fetch Product Page**.
3. The Config node is no longer needed for the URL — just remove `PRODUCT_URL` from it (keep `SHEET_ID` and `TELEGRAM_CHAT_ID`).

---

## Customising the Telegram message

Edit the `text` field in **Send Telegram Alert**. Telegram uses **MarkdownV2**:

```
🔔 *Price Drop Alert\!*

📦 *{{ productName }}*

💸 Old price: ₹{{ lastPrice }}
✅ New price: ₹{{ currentPrice }}
🎉 You save: ₹{{ dropAmount }} \({{ dropPercent }}% off\)

🛒 [Buy now]({{ url }})

_Tracked by your n8n Price Bot_
```

**MarkdownV2 rules:**
- Special chars `! ( ) - .` must be escaped: `\!` `\(` `\)` etc.
- Bold: `*text*` — Italic: `_text_` — Link: `[label](url)`
- The Code node already escapes special characters in the product name automatically.

---

## Supported sites

| Site | Status |
|---|---|
| Amazon India | ✅ Supported |
| Flipkart | ✅ Supported |
| Any site showing ₹ | ✅ Generic fallback |

To add another site, open the **Extract Price** Code node and add a pattern array following the existing examples.

---

## Troubleshooting

**"Could not extract price" error**
The site likely renders prices via JavaScript. The HTTP Request node only fetches static HTML. Fix: replace the HTTP Request node with a scraping service like [Browserless](https://browserless.io) or [ScrapingBee](https://scrapingbee.com).

**Telegram message not delivered**
- Check the Bot Token in the n8n Telegram credential.
- Make sure you sent at least one message to the bot before fetching the Chat ID.
- For channels/groups, the bot must have Admin + posting rights.
- Re-visit `https://api.telegram.org/botYOUR_TOKEN/getUpdates` to verify the Chat ID.

**Sheet not updating**
- Tab must be named `PriceLog` exactly (case-sensitive).
- Google Sheets credential needs edit access.
- The `url` column is the match key — it must be populated on first run.

**Workflow not running on schedule**
- Toggle must be set to **Active**.
- Self-hosted n8n must stay running — use Docker or PM2.

---

## Node-by-node reference

| # | Node | Type | Purpose |
|---|---|---|---|
| 1 | Schedule Trigger | scheduleTrigger | Fires every 6 hours |
| 2 | Config | set (Edit Fields) | Holds PRODUCT_URL, SHEET_ID, TELEGRAM_CHAT_ID |
| 3 | Fetch Product Page | httpRequest | GETs the product HTML |
| 4 | Extract Price | code | Parses price + product name from HTML |
| 5 | Read Last Price | googleSheets | Reads stored price from PriceLog |
| 6 | Compare Prices | code | Calculates drop, amount, percent |
| 7 | Price Dropped? | if | Branches on drop = true/false |
| 8 | Update Price in Sheets | googleSheets | Saves new price + timestamp |
| 9 | Send Telegram Alert | telegram | Sends formatted Telegram message |

---

## File structure

```
price-tracker-telegram/
├── price-tracker-telegram-workflow.json   ← import into n8n
└── README.md                              ← this file
```

---

## Tech stack

- **n8n** — workflow automation (free tier)
- **Google Sheets** — price history storage
- **Telegram Bot API** — instant alert delivery
- **HTTP Request + Code node** — scraping and price parsing

---

*Built with n8n free tier. No paid features used.*
