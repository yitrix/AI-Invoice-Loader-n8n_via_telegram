# AI Invoice Loader — n8n Workflow

This workflow accepts an invoice/receipt through Telegram, extracts invoice text, converts it into structured expense data with an LLM, checks Google Sheets for possible duplicates, and then either auto-approves the expense or sends it for manual Telegram approval.

## Files

- `invoice_loader_reviewed.json` — reviewed and improved n8n workflow.
- `README_invoice_loader.md` — setup, logic, changes, and testing guide.

## Workflow architecture

```text
Telegram Trigger
      |
      v
Is it a photo?
   /       \
 Yes       No / document
  |             |
OCR.Space     Telegram Get File
  |             |
  |       Universal File Extractor
   \           /
      invoice text
           |
           v
       AI Agent
           |
 Structured Output Parser
           |
           v
 Google Sheets duplicate lookup
 (vendor + date + amount)
           |
      Duplicate?
       /      \
     Yes       No
      |         |
 Manual     Amount > 0 and < 250?
 Review       /           \
             Yes           No
              |             |
        Auto-approved    Manual Review
              |             |
              +------> Google Sheets
```

## What the workflow extracts

The AI output schema contains:

- `vendor_name`
- `invoice_date` in `YYYY-MM-DD`
- `total_amount`
- `currency`
- `category`
- `line_items`

Allowed categories are:

- office supplies
- utilities
- travel
- food
- software
- other

## Improvements made

### 1. Safer AI extraction prompt

The extraction prompt now tells the model to treat invoice text as untrusted data and not follow instructions embedded inside an invoice. It also tells the model not to invent missing values and to prefer the final payable/grand total rather than subtotal or tax values.

### 2. Better duplicate detection

The original workflow looked up only:

```text
vendor_name + invoice_date
```

The reviewed workflow checks:

```text
vendor_name + invoice_date + total_amount
```

This reduces false duplicate warnings when several purchases are made from the same vendor on the same day.

### 3. Cleaner amount routing

The workflow now treats a positive amount below `250` as eligible for automatic approval when no duplicate is found and the vendor exists.

Anything else goes to manual review, including:

- possible duplicates
- missing vendor name
- missing/invalid amount
- amount `>= 250`

You can change the `250` threshold in the `If1` node.

### 4. Useful manual-review reason

Before the Telegram approval step, the workflow now creates a `review_reason`, for example:

- `Possible duplicate: same vendor, date and amount already exists`
- `Vendor name missing`
- `Invalid or missing total amount`
- `Amount exceeds auto-approval limit of 250`

The reason is shown to the approver.

### 5. Approval typo fixed

The original Telegram form had:

```text
Rejecet
```

It is now:

```text
Reject
```

### 6. Google Sheets status improved

Auto-approved rows now explicitly save:

```text
status = Approved - Auto
```

Manual-review rows save the approver's `Approve` or `Reject` response.

### 7. Line items stored safely

`line_items` is converted with `JSON.stringify(...)` before being written to Google Sheets. This prevents arrays/objects from turning into unusable values such as `[object Object]`.

Example cell value:

```json
[{"description":"Notebook","amount":80},{"description":"Pen","amount":20}]
```

### 8. Safer Telegram expressions

Photo and document expressions use optional chaining so missing Telegram fields are less likely to cause expression errors.

### 9. Secrets removed from the shareable JSON

The OCR.Space API key and fixed Telegram approver Chat ID were replaced with placeholders in the reviewed file.

Do **not** publish real API keys in an exported n8n JSON or GitHub repository.

## Required services / credentials

You need:

1. Telegram Bot credential
2. Mistral Cloud credential
3. Google Sheets OAuth2 credential
4. OCR.Space API key for image OCR
5. `n8n-nodes-word-extractor` community node used by `Universal File Extractor`

If you import the workflow into the same n8n instance, some existing credential mappings may reconnect automatically. On another n8n instance, select credentials again manually.

## Required setup after import

### A. OCR.Space API key

Open the node:

```text
recived file
```

Find the HTTP header:

```text
apikey
```

Replace:

```text
REPLACE_WITH_OCR_SPACE_API_KEY
```

with your OCR.Space API key.

For a production workflow, using an n8n credential or another secret-management method is preferable to storing an API key directly in the workflow JSON.

### B. Telegram approval Chat ID

Open:

```text
Send message and wait for response
```

Replace:

```text
REPLACE_WITH_APPROVER_TELEGRAM_CHAT_ID
```

with the Telegram Chat ID of the person who should approve flagged invoices.

### C. Google Sheet

The workflow currently points to the spreadsheet and worksheet configured in your original workflow. If you use another sheet, reselect it in both append nodes and in `Get row(s) in sheet`.

The worksheet should contain these headers exactly:

```text
vendor_name
invoice_date
total_amount
currency
category
line_items
submitted_by
status
flag_reason
```

## Main node logic

### Telegram Trigger

Listens for incoming Telegram messages and downloads attachments.

### If

Checks whether the Telegram message contains a photo.

- **true** → OCR.Space image OCR
- **false** → Telegram file download → Universal File Extractor

This workflow is intended for **invoice photos or documents**, not ordinary Telegram text messages.

### recived file

Sends the Telegram image binary to OCR.Space.

### Edit Fields

Reads OCR.Space output:

```javascript
$json.ParsedResults[0].ParsedText
```

and stores it as `invoice`.

### Universal File Extractor

Extracts text from the uploaded document and places it in `invoice`.

### AI Agent + Structured Output Parser

Transforms unstructured invoice text into the required JSON structure.

The Structured Output Parser is valuable here because downstream nodes can reference predictable fields such as:

```javascript
$('AI Agent').item.json.output.total_amount
```

rather than parsing free-form model text.

### Get row(s) in sheet

Searches for a previous row matching:

```text
vendor_name
invoice_date
total_amount
```

### If3

Determines whether a matching row already exists.

A match is treated as a possible duplicate and sent for manual review.

### If1

Checks the auto-approval amount rule:

```text
amount > 0
AND
amount < 250
```

### If2

Checks that `vendor_name` exists before automatic insertion.

### Append row in sheet

Stores a normal auto-approved invoice with:

```text
status = Approved - Auto
```

### Edit Fields1

Builds the manual-review context and reason.

### Send message and wait for response

Sends invoice information to the approver and waits for either:

```text
Approve
Reject
```

The approver can optionally enter a rejection reason.

### Append row in sheet1

Logs the manually reviewed invoice and stores the approval status/reason.

## Recommended test cases

Run each of these before using the workflow for real expenses:

| Test | Expected result |
|---|---|
| Clear invoice photo, amount 100 | Auto-approved |
| PDF invoice, amount 150 | Auto-approved |
| Invoice amount 300 | Manual approval |
| Same vendor/date/amount submitted again | Manual duplicate review |
| Missing vendor | Manual review |
| OCR cannot determine amount | Manual review |
| Approver chooses Reject | Row recorded as `Reject` with reason if supplied |

## Important limitations

### OCR quality

Poor photos, rotated receipts, handwriting, very small fonts, or multi-page scans can produce incomplete OCR. The LLM cannot reliably recover information that the OCR never extracted.

### Duplicate detection

`vendor + date + amount` is a practical heuristic, not a true invoice ID. A stronger production version should also extract and store `invoice_number` and use it as the primary duplicate key when available.

### Currency and threshold

The `250` approval threshold currently does not convert currencies. For example, `250 INR` and `250 USD` are treated numerically the same. If invoices can use different currencies, add currency normalization before approval logic.

### Rejected records

Rejected invoices are intentionally written to the sheet so there is an audit trail. If you want rejected invoices stored in a separate worksheet instead, split the approval result with an IF node before the final append.

## Production upgrades I would recommend next

1. Add `invoice_number` to the structured schema and duplicate check.
2. Add a Telegram confirmation to the original submitter after approval/rejection.
3. Add an Error Trigger workflow for OCR/API/Google Sheets failures.
4. Store the raw Telegram `file_id`, Telegram user ID, and submission timestamp for auditability.
5. Add currency-aware approval thresholds.
6. Replace the AI Agent with a simpler LLM chain if you do not plan to use tools; an Agent is more flexible than required for pure structured extraction.
7. Move secrets to credentials/environment-backed configuration before publishing the workflow.

## Import

In n8n:

```text
Workflows → Import from File → invoice_loader_reviewed.json
```

Then reconnect/verify credentials, enter the OCR key and approver Telegram Chat ID, and execute the workflow manually with a sample invoice before activating it.

## Summary

The reviewed version preserves your original concept while making duplicate detection, structured storage, approval reasons, status tracking, prompt safety, and secret handling cleaner. It is suitable as a stronger learning/portfolio workflow, and the upgrades listed above are the next steps for making it closer to production-grade expense automation.
