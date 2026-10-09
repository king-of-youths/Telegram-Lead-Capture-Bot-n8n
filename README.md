# Telegram Lead Capture & Consultation Booking Bot (V1)

A rule-based Telegram bot built in **n8n** that collects a customer's details, saves the lead to **Google Sheets**, notifies the business owner, and confirms to the customer. No AI agent is used.

```
Telegram -> Collect Information -> Check Existing Lead -> NEW or OLD
        -> Store Lead -> Notify Owner -> Respond to Customer
```

## What it does

1. A customer sends `Hi` or `/start`.
2. The bot asks, one question at a time: **name**, **phone**, **service** (buttons), **preferred date**, **preferred time**.
3. The bot shows a summary with **Confirm** / **Edit** buttons.
4. On Confirm, the workflow:
   - checks the Leads sheet for the phone number and marks the lead **NEW** or **OLD**,
   - generates a unique Lead ID (`LD-001`, `LD-002`, ...),
   - saves the lead to the Leads tab,
   - sends a **NEW LEAD RECEIVED** message to the owner,
   - clears the customer's session,
   - sends a confirmation to the customer.

Services offered: Web Design, AI Automation, Data Analysis, Cybersecurity, Digital Design, Digital Marketing, Web Development.

## Customer commands

| Message | Result |
|---|---|
| `Hi`, `Hello`, `Hey`, `start`, `/start` | Starts (or restarts) the conversation and resets the session |
| `/cancel` or `cancel` | Clears the session and shows the cancellation message |
| Edit button | Restarts the questions from the name |

## Requirements

- An n8n instance reachable over **HTTPS** (Telegram only sends webhooks to HTTPS addresses). Tested on self-hosted n8n 2.x.
- A Telegram bot created with **@BotFather**.
- A Google account and a Google Cloud OAuth client (Client ID and Secret) with the **Google Sheets API** and **Google Drive API** enabled.

## Google Sheet setup

Create one spreadsheet named **Telegram Lead Capture CRM** with two tabs. Names and headers must match exactly (row 1).

**Tab `Leads`** (columns A to L)

| Lead ID | Submitted At | Name | Phone | Telegram User ID | Telegram Username | Service | Preferred Date | Preferred Time | Lead Status | Notes | Last Contacted |
|---|---|---|---|---|---|---|---|---|---|---|---|

**Tab `Sessions`** (columns A to I)

| Telegram User ID | State | Name | Phone | Service | Preferred Date | Preferred Time | Updated At | Last Update ID |
|---|---|---|---|---|---|---|---|---|

**Important:** on both tabs select all cells and choose *Format > Number > Plain text*. Otherwise Google Sheets stores phone numbers as numbers and drops the leading `0`.

## Installation

1. **Create the bot.** In Telegram, open @BotFather, send `/newbot`, and copy the API token. Keep it private.
2. **Create credentials in n8n.**
   - *Telegram API*: paste the bot token.
   - *Google Sheets OAuth2 API*: paste the Client ID and Client Secret, add the n8n redirect URL shown in the credential window to your Google OAuth client, add your Gmail under **Test users**, and click *Sign in with Google*. Publish the consent screen so the login does not expire after 7 days.
3. **Import the workflow.** Workflows > menu > *Import from file*.
4. **Fill in the Config node.**
   - `sheetId`: the spreadsheet ID (the part of the sheet URL between `/d/` and `/edit`).
   - `ownerChatId`: the Telegram chat ID of the person who should receive lead notifications. This must be a **different account** from your test customer if you want to see the owner message separately.
5. **Attach credentials.** Select the Telegram credential on the trigger and every Telegram node. Select the Google Sheets credential on every Google Sheets node.
6. **Check the Telegram trigger.** *Updates* must include both **Message** and **Callback Query** (button taps).
7. **Publish** the workflow. The owner must press **Start** on the bot once, or Telegram will block notifications.

### Finding a chat ID

Have the person send `Hi` to your bot, then open **Executions > latest run > Get Message Data** and copy `chatId`.

## Workflow structure

**A. Conversation engine**

| Node | Type | Purpose |
|---|---|---|
| Telegram Trigger | Telegram Trigger | Receives messages and button taps |
| Config | Set | Holds `sheetId` and `ownerChatId` |
| Get Message Data | Code | Extracts user ID, chat ID, username, text, button-tap flag and Telegram update ID |
| Is Button Tap? / Answer Button Tap | IF / Telegram | Acknowledges button taps so the loading spinner stops |
| Get Session | Google Sheets (Get Row) | Loads the customer's session by Telegram User ID |
| Process Current Input | Code | The state machine: validates input, picks the next state and reply |
| Is Duplicate? | IF | Drops messages Telegram re-delivered (already handled update ID) |
| Is Submit? | IF | Sends confirmed leads to the submit branch |
| Update Session | Google Sheets (Append or Update) | Saves the new state and answers |
| Reply Type? | Switch | Routes to plain text, service buttons or confirm buttons |
| Send Text / Send Service Buttons / Send Confirm Buttons | Telegram | Send the reply (inline keyboards for the buttons) |

**B. Submit lead**

| Node | Type | Purpose |
|---|---|---|
| Check Existing Leads | Google Sheets (Get Row) | Reads the Leads tab |
| Determine NEW or OLD | Code | Matches phone by its last 10 digits; finds the highest Lead ID |
| Generate Lead ID | Set | Builds `LD-001` style ID and the Africa/Lagos timestamp |
| Save Lead | Google Sheets (Append) | Writes the lead row |
| Notify Owner | Telegram | Sends NEW LEAD RECEIVED to the owner (continues on failure) |
| Clear Session | Google Sheets (Append or Update) | Resets the session to COMPLETE |
| Confirm Customer | Telegram | Sends the success message |
| Send Failure Message | Telegram | Sent when any Google Sheets step fails |

## Conversation states

`START` > `WAITING_NAME` > `WAITING_PHONE` > `WAITING_SERVICE` > `WAITING_DATE` > `WAITING_TIME` > `CONFIRMING` > `COMPLETE`

The bot uses the Telegram User ID to find each customer's row in Sessions, so progress is never lost between messages.

## Key behaviours

- **Validation:** name 2 to 60 characters; phone 10 to 15 digits with an optional `+` (`+234...` is converted to `0...`); service must be one of the seven options; date and time must be non-empty and 50 characters or fewer.
- **NEW vs OLD:** the phone number is the identifier. If it already exists in Leads, the status is OLD.
- **Lead ID:** highest existing number + 1, so deleting a row never causes duplicates.
- **Duplicate protection:** the last handled Telegram `update_id` is stored per customer in `Last Update ID`; older or repeated updates are ignored.
- **Markdown safety:** characters `_ * ` [ ]` are stripped from customer-typed text, because Telegram rejects messages with unpaired formatting marks.
- **Timezone:** Africa/Lagos for all timestamps.
- **Errors:** if Google Sheets fails, the customer is told to try again and is never told the lead was saved. If the owner notification fails, the lead is still saved and the customer still gets the confirmation.

## Testing checklist

1. Delete all data rows (not headers) on both tabs.
2. Send `Hi`, then answer: name, phone, tap a service, date (e.g. `October 10`), time (e.g. `2:00 PM`).
3. Check the summary shows your real values, then tap **Confirm**.
4. Verify: a Leads row with `LD-001` and status NEW; the owner received NEW LEAD RECEIVED; the customer received the confirmation; the Sessions row shows `COMPLETE` with empty answers.
5. Repeat with the same phone number. Expect `LD-002` with status OLD.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| "The resource you are requesting could not be found" | Document or Sheet is not set on a Google Sheets node. Set Document to *By ID* with `{{ $('Config').first().json.sheetId }}` and Sheet to *From list*. Grey text in a field means it is empty. |
| Bot ignores changes you made | In n8n 2.x, *Save* only stores a draft. Click **Publish** after every change. |
| Bot replies with old or repeated messages | Another copy of the workflow is published, or Telegram has a backlog. Unpublish old copies, then open `https://api.telegram.org/bot<TOKEN>/deleteWebhook?drop_pending_updates=true` and publish again. |
| Button taps do nothing | The trigger is missing *Callback Query*, or the workflow was not re-published after changing it. |
| "can't parse entities" | Unpaired Markdown characters in a message. Customer input is cleaned in Process Current Input; check any custom text you added. |
| Phone loses its leading zero | The column is not Plain text. Reformat it and clear old rows. |
| Wrong values in the sheet | A Values to Send box in Update Session holds the wrong expression. Each box must use its own field (`$json.name`, `$json.date`, and so on). |
| Owner message goes to the customer | `ownerChatId` equals the customer's own chat ID, or the workflow was not re-published after changing Config. |
| Notify Owner fails silently | The node continues on error. Check *Executions > Notify Owner*. "chat not found" means the owner has not pressed Start on the bot. |

## Operating notes

- Do **not** click *Execute workflow* while the workflow is published. It moves the bot to the temporary test address and the live bot stops receiving messages until you publish again.
- Keep only one published workflow connected to each bot token.
- Never share the bot token. If it leaks, revoke it in @BotFather and update the n8n credential.

## Known limitations and next steps

- Preferred date and time are stored exactly as typed. A later version could parse them into `YYYY-MM-DD` and `HH:mm`.
- Check Existing Leads reads the whole Leads tab. This is fine for hundreds of leads; for thousands, filter by phone and keep a counter row for Lead IDs.
- Two customers confirming in the same second could, in rare cases, receive the same Lead ID.
- Ideas for V2: follow-up reminders using the `Last Contacted` column, owner buttons to mark a lead as contacted, and an AI step for free-text date parsing.
