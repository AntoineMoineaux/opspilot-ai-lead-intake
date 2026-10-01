# OpsPilot: AI Lead Intake, Qualification and Routing

An n8n system that takes an inbound form submission, qualifies it with an LLM, writes it to the CRM, and asks a human to approve before anything leaves the building.

Built and deployed for a client. The client's identity and industry are not disclosed. The workflows in this repository are the production workflows with every credential, identifier and real data sample removed. See [What was removed](#what-was-removed-before-publishing).

## Why it exists

A small team was handling inbound demo requests by hand: read the form, judge the intent, look the person up in the CRM, create or update the contact, write a note, decide who follows up, draft a reply. Around ten minutes per request, done inconsistently, and the CRM slowly filled with duplicates.

## What it does

```
Form submission
      |
      v
  Normalize and validate  ------------- rejected -----> logged for review
      |
      v
  HubSpot contact search (dedup)
      |
      v
  Claude classification  --------------- failed -----> safe fallback record
      |                                                 (Needs Review, low confidence)
      v
  Strict output validation
      |
      v
  Create or update contact -> deal -> note -> task
      |
      v
  Airtable record (audit trail)
      |
      v
  Telegram alert to a human  ---------> approve / reject ---> status written back
```

Four workflows:

| Workflow | Nodes | Role |
|---|---|---|
| `OpsPilot - Main Intake` | 21 | The pipeline above, from webhook to human alert |
| `OpsPilot - Telegram Callback Handler` | 4 | Receives the human decision and writes the status back |
| `OpsPilot - Error Handler` | 2 | Catches any failure in any workflow and alerts on a separate channel |
| `OpsPilot - Daily Report` | 4 | Scheduled digest of the day's volume and outcomes |

Five external services, orchestrated by n8n: the form provider, the Anthropic API, HubSpot CRM v3, Airtable, Telegram.

## Results

Measured on the deployment:

| | |
|---|---|
| Requests processed in production | 45+ |
| Handling time before | about 10 minutes, manual |
| Handling time after | under 2 minutes |
| Reduction | about 80% |
| Operator approval rate | about 70% of AI drafts approved without an edit |
| Replies sent without human approval | zero, by design |

**Honest limitation.** End-to-end QA was run on controlled scenarios, not at high production volume. The 45+ figure is real traffic, not a load test.

## The design decisions worth reading

These are the parts that took thinking, and the parts I would defend in a review.

**A human approves before anything is sent.** Every record carries `human_review_required: true`, written in the code, not in a setting someone can flip by accident. The AI drafts a reply. It never sends one. The operator sees a Telegram summary, opens the draft in Airtable, sends from their own mail client, and confirms. The approval rate then becomes a real quality signal instead of a vanity metric.

**Deduplication happens before the CRM write, not after.** The system searches HubSpot first, attaches the result as explicit context (`hubspot_contact_exists`, `hs_contact_id`, `duplicate_risk`), and only then decides between create and update. Cleaning duplicates afterwards is a losing game.

**The model's output is validated against a whitelist, not trusted.** Categories, priorities and flags are checked against fixed lists. An invalid category throws. A truncated response (`stop_reason: max_tokens`) throws. A missing JSON object throws.

**Every failure has a defined resting place.** When classification fails for any reason, the record is still written, with category `Needs Review`, priority `Needs Review`, confidence `Low`, a plain-language note saying classification failed, and the error message stored in `processing_error`. Nothing is silently dropped.

**Rules encoded in the prompt, not left to taste.** For example: a message under fifteen words with no company or role is `Incomplete`; if priority is `Needs Review` then confidence must be `Low`; never infer customer status from the CRM alone. The prompt also forbids inventing anything not present in the input.

**A stable identity per lead.** When the form provides no submission id, the system derives one from a hash of submission id, email and message, so a retry does not create a second record.

**Errors alert on a separate channel from business events.** A failed workflow and a hot lead are two different problems for two different people. They do not share an inbox.

**Ownership is assigned by rule.** Urgent and Hot go to an AE, Incomplete goes to an SDR, everything else goes to Ops. Written once, applied every time.

## Running it yourself

1. Import the four JSON files into n8n.
2. Create the credentials the nodes expect: form webhook, Anthropic API, HubSpot private app token, Airtable OAuth2, Telegram bot. Each node points at `REPLACE_WITH_YOUR_CREDENTIAL_ID`.
3. Replace the placeholders: `appYOURBASEID0000` and `tblYOURTABLEID000` with your Airtable base and table, `REPLACE_WITH_YOUR_CHAT_ID` with your Telegram chat, and the three HubSpot deal stage ids.
4. Create the HubSpot custom properties used by the contact write: `opspilot_ai_category`, `opspilot_ai_priority`, `opspilot_ai_confidence`.
5. Set `OpsPilot - Error Handler` as the error workflow on the other three.

## What was removed before publishing

Nothing about the logic changed. What was stripped: all credential ids and names, webhook ids, the n8n instance id, Airtable base, table and field ids with their cached URLs, HubSpot deal stage ids, the Telegram chat id, and the pinned sample data, which contained a real person's email address, message and submission URL.

## License

MIT.
