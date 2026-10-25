# Emergency Triage Bot

An n8n webhook for an HVAC business that reads incoming service messages and sorts them by urgency. A gas leak or fire goes to the technician on Telegram straight away. Problems that need same-day service are sent as urgent. General questions are logged only. An AI model classifies each message, and a keyword check overrides it for safety words, so a gas smell is never treated as routine.

## Architecture

```
Webhook (POST /emergency-triage)
   → Validate Input (name, phone, message)
        ├── Invalid → Log Invalid Request
        └── Valid → Check Duplicate (same phone, last 5 minutes)
                 → Mark Duplicate
                 → Is New Request?
                      ├── No  → Log Duplicate
                      └── Yes → Classify Urgency (OpenAI)
                                → Validate AI Output (safety keywords override)
                                → Route by Category
                                     ├── emergency → Alert Technician Emergency (Telegram)
                                     │                 ├── success → Log Triage Request
                                     │                 └── error   → Backup Emergency Email (Gmail)
                                     ├── urgent    → Alert Technician Urgent (Telegram) → Log Triage Request
                                     └── routine   → Log Triage Request
```

## Features

* Validates name, phone (10+ digits), and message (5+ characters) before any AI call
* Skips repeat messages from the same phone number within five minutes
* AI sorts each message into `emergency`, `urgent`, or `routine`, with a short reason
* Safety keywords (gas, carbon monoxide, smoke, fire, burning, sparks, flood) always mean emergency, whatever the AI says
* If the AI reply cannot be read, the request defaults to `urgent`, so nothing is dropped
* Emergencies go to the technician on Telegram. If Telegram fails, the alert is emailed instead
* Every request is logged to Supabase with its category, reason, and status

## Tech Stack

* n8n 2.x (workflow automation, webhooks)
* OpenAI `gpt-4o-mini` (urgency classification)
* Supabase (triage_requests table)
* Telegram Bot API (technician alerts)
* Gmail API (backup emergency email)

## Setup

1. Create a Supabase table named `triage_requests` with these columns:

```
id | name | phone | message | category | reason | status | created_at
```

   `created_at` should default to the current time. The duplicate check depends on it.

2. Import `emergency_triage_bot.json` into n8n.
3. Add credentials: OpenAI, Supabase, Telegram, and Gmail OAuth2. Assign them to the matching nodes.
4. Replace these placeholders:
   * `YOUR_TELEGRAM_CHAT_ID`: the technician's chat ID (Alert nodes)
   * `YOUR_EMAIL@example.com`: the backup email address (Backup Emergency Email node)
5. Activate the workflow and send a test request.

## Sample Requests

Emergency (the safety keyword overrides the AI):

```
{
  "name": "Ali Raza",
  "phone": "03111222333",
  "message": "I smell gas near the furnace"
}
```

Urgent:

```
{
  "name": "Ali Raza",
  "phone": "03111222333",
  "message": "AC stopped working this morning and it is very hot inside"
}
```

Routine:

```
{
  "name": "Ali Raza",
  "phone": "03111222333",
  "message": "How much does an annual maintenance check cost?"
}
```

Send the same body twice within five minutes to see the duplicate path.

## Project Notes

* Built as part of my AI Automation portfolio.
* The keyword check runs after the AI on purpose. A model can miss a safety risk, but the keyword list cannot.
* Only the OpenAI classification node uses `gpt-4o-mini`. Change its model in that node if you want another model.
