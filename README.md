# Inbox Autopilot

**An AI email assistant for businesses. It sorts your Gmail, drafts replies from your own knowledge base, and sends nothing until you approve it on WhatsApp.**

Inbox Autopilot is two connected n8n workflows. The first reads new emails, classifies them and writes a draft. The second waits for your decision on WhatsApp and sends the approved reply in the same Gmail thread. The AI never sends anything on its own.

---

## How it works



### Workflow 1: Intake and Draft
1. **Gmail trigger** polls every minute for unread inbox mail that you didn't send.
2. **Dedupe:** each email's `messageId` is checked against the log table, so the same email is never handled twice.
3. **Classify:** Gemini 2.5 Flash labels the email `important`, `normal` or `spam`. If the result is missing or unclear, it defaults to `important`, so real mail is never dropped as spam by mistake.
4. **Spam** is logged only. No draft and no WhatsApp message.
5. **Draft:** an AI agent searches your Pinecone knowledge base and writes a reply using only those facts. It returns a confidence score (0 to 1) and lists any missing information.
6. **Approval request:** an ID such as `AP-X7K2M9QD` is created with a 24-hour expiry. The full email, the draft, the confidence score and any gaps are sent to your WhatsApp, followed by **Approve** and **Reject** buttons.

No node in this workflow can send an email.

### Workflow 2: Approval and Reply
1. **WhatsApp trigger** receives your message or button tap.
2. **Authorization:** only the number set in the `Config` node is accepted. Messages from any other number are silently dropped.
3. **Commands:**

| Command | What happens |
|---|---|
| `APPROVE AP-XXXX` (or the Approve button) | Sends the AI draft |
| `EDIT AP-XXXX your new reply text` | Sends your text instead of the draft |
| `REJECT AP-XXXX` (or the Reject button) | Discards the draft, nothing is sent |

4. **Lock before send:** the request is flipped from `PENDING` to `APPROVED` before Gmail runs, so a second APPROVE can never send a duplicate reply.
5. **Reply in the same thread**, to the sender only. The workflow then compares the thread ID Gmail returns with the original and warns you on WhatsApp if they differ.
6. **Confirmation** is sent to your WhatsApp after each send.
7. **Expiry:** requests older than 24 hours are rejected when you answer, and an hourly sweep marks stale `PENDING` rows as `EXPIRED`.
8. **Failures:** if Gmail fails, the row is marked `FAILED` and you get a WhatsApp alert.

---

## Features

- AI classification of every incoming email
- Replies grounded only in your knowledge base, so prices, policies and dates are never invented
- Confidence score and "missing information" list on every draft
- Human approval for every reply, with Approve, Reject and Edit
- Owner-only authorization by WhatsApp number
- Duplicate protection by Gmail `messageId`, and a lock step against double-sending
- 24-hour approval expiry with an hourly cleanup
- Same-thread replies with verification
- Full audit log of every email, draft, decision and send result
- Prompt-injection guard: email text is treated as untrusted data, never as instructions

---

## What's in this repo

| File | Purpose |
|---|---|
| `ai-email-assistant-1-intake-and-draft.json` | Workflow 1: Gmail intake, classification, drafting, approval request |
| `ai-email-assistant-2-approval-and-reply.json` | Workflow 2: approval handling, sending, expiry sweep |

Both exports are sanitized. All IDs, phone numbers and credentials are replaced with `YOUR_...` placeholders, and both workflows are set to inactive.

---

## Requirements

- An **n8n** instance (Cloud or self-hosted) with **Data Tables** available
- A **Gmail** account with OAuth2 access
- A **Gemini API key** from Google AI Studio
- A **Pinecone** index containing your knowledge base, embedded with Gemini embeddings
- A **WhatsApp Business Cloud API** setup (Meta): a phone number ID, an access token, and the WhatsApp trigger credential

---

## Setup

### 1. Create the log table
In n8n, create a data table named **`email_assistant_log`** with these columns (all strings except `confidence`, which is a number):

`messageId`, `threadId`, `sender`, `senderEmail`, `subject`, `emailBody`, `classification`, `aiDraft`, `finalReply`, `confidence` (number), `missingInfo`, `approvalId`, `approvalStatus`, `sendStatus`, `requestedAt`, `expiresAtIso`, `decidedAt`, `replyMessageId`, `replyThreadId`

### 2. Prepare your knowledge base
Load your FAQs, pricing, policies and service details into a Pinecone index using Gemini embeddings. The drafting agent will answer only from what it finds there.

### 3. Import the workflows
In n8n, choose **Import from file** and import both JSON files.

### 4. Replace the placeholders

| Placeholder | Where | Replace with |
|---|---|---|
| `YOUR_DATA_TABLE_ID` | Every data table node, both workflows | Your `email_assistant_log` table |
| `YOUR_PINECONE_INDEX` | Knowledge Base Search node (workflow 1) | Your Pinecone index name |
| `YOUR_WHATSAPP_NUMBER` | Workflow 1: send node and buttons node. Workflow 2: `Config` node | Your number with country code, digits only (e.g. `15551234567`) |
| `YOUR_PHONE_NUMBER_ID` | Workflow 1: send node and buttons URL. Workflow 2: `Config` node | Your WhatsApp Business phone number ID |
| `YOUR_CREDENTIAL_ID` | All credential slots | Connect your own credentials in each node |

The easiest way is to open each node and select your credential from the dropdown.

### 5. Connect credentials
Gmail OAuth2, Gemini API, Pinecone, WhatsApp Business Cloud, and WhatsApp Trigger.

### 6. Test
1. Send a test email to your inbox from another address.
2. Run workflow 1 manually, or wait for the next poll.
3. Check that the approval request arrives on WhatsApp.
4. Reply `APPROVE AP-XXXX` (or tap Approve) and check that the reply appears in the original Gmail thread.
5. Try `EDIT` and `REJECT` too.

### 7. Activate
Turn on both workflows.

---

## Customization

- **AI model:** swap the Gemini model nodes for OpenAI, Claude or another provider.
- **Tone and rules:** edit the system messages in the `AI: Classify Email` and `AI: Draft Reply` nodes.
- **Polling speed:** change the Gmail trigger interval.
- **Expiry window:** change the `24` hours in `Prepare Approval Request`.
- **Reply signature:** the draft signs off with "Best regards". Edit the drafting prompt to add yours.
- **Always-important senders:** add rules to the classifier prompt, or a filter before the classify step.

---

## Troubleshooting

| Problem | Likely cause |
|---|---|
| No WhatsApp message arrives | WhatsApp only allows free-form messages within 24 hours of your last message to the business number. Send any message to it first. Also check the access token and phone number ID. |
| Commands are ignored | `ownerWhatsApp` must match your number exactly, digits only, with country code. |
| "No approval request found" | The approval ID is wrong or the row is missing from the log table. |
| Draft has low confidence | The knowledge base doesn't contain the answer. Add the missing content. |
| Reply isn't in the same thread | Check the `SENT_THREAD_MISMATCH` status in the log table. |
| Same email handled twice | Check that the data table ID is set in the dedupe node. |

---

## Security notes

- This repo contains no secrets. Credentials stay in your own n8n instance.
- Anyone who can message your WhatsApp Business number can't trigger sends. Only the configured owner number is accepted.
- The AI never sends email. Sending exists only in workflow 2, after an authorized approval.
- Review your knowledge base before use. The AI can only be as accurate as the content you give it.

---

## Limitations

- Built for Gmail (the Gmail trigger and reply nodes). Other providers need different nodes.
- One inbox per setup.
- Email bodies are trimmed to 6,000 characters before the AI reads them.
- Replies are plain text.
- Requires your own AI, Pinecone and WhatsApp accounts. Their usage costs are billed to you.

---

## Need this built for your business?

I build custom AI email assistants and n8n automations for clients, with setup, knowledge base loading and testing included.

**Chirag Sehgal**

- Website: [flowforge-ai-agents.web.app](https://flowforge-ai-agents.web.app)
- LinkedIn: [linkedin.com/in/chiragsehgal49](https://www.linkedin.com/in/chiragsehgal49/)


---
