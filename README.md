# LinkedIn Post Approval Workflow (n8n)

## Overview

This workflow receives messages from a Telegram bot, creates a LinkedIn post with an AI Writer Agent, uses a Reviewer Agent tool to check the draft, and sends the result to Gmail for approval. The approval decision is then recorded in Google Sheets, and Telegram sends a confirmation or decline message.

The workflow is designed to keep casual greetings and help questions out of the approval process.

## Workflow at a glance

1. **Telegram Trigger** — receives a Telegram message.
2. **Greeting/help IF (`If1`)** — routes recognised greetings and format/help questions to a Telegram reply.
   - **True:** sends a help response in Telegram.
   - **False:** continues to the Writer Agent.
3. **Writer Agent** — drafts a LinkedIn post based on the incoming Telegram message.
4. **Reviewer Agent** — connected as an AI Agent Tool to review and improve the draft.
5. **Post Approval (Gmail)** — sends the draft for approval and waits for a response.
6. **Approval IF (`If`)** — checks `{{ $json.data.approved }}`.
   - **True:** appends an approved record to Google Sheets, then sends a Telegram confirmation.
   - **False:** appends a declined record to Google Sheets, then sends a Telegram decline message.
7. **IGNORE handling (planned/in progress)** — an additional IF/router is being added before the greeting/help logic to recognise `IGNORE` so it is not treated as a new LinkedIn topic.
8. **EDIT handling (planned/in progress)** — the revision loop is not complete yet. It needs to retrieve the correct declined draft, combine it with the user's Telegram edit instructions, rerun Writer/Reviewer, and request Gmail approval again.

## Telegram message examples

### Start a new post
Send a topic, for example:

`Remote work`

Optional topic and URL format:

`Remote work | https://example.com`

The URL is optional context only. Do not assume the workflow verifies the URL or facts from it.

### Ask for help
Examples that the current greeting/help rule is intended to catch:

- `Hi`
- `Hello`
- `Hey`
- `Good morning`
- `What format should I use?`
- `How does this work?`
- `Help me`

These messages should receive a Telegram response without entering Gmail approval.

### Declined post
When Gmail approval is declined, the workflow appends a row to Google Sheets and sends a Telegram message containing the draft with instructions to reply with editing feedback or `IGNORE`.

Example edit reply (revision handling still needs to be implemented):

`EDIT: Make the hook stronger, shorten it, and end with a question.`

To ignore a declined request:

`IGNORE`

**Current limitation:** `IGNORE` routing and the full `EDIT` revision flow are still being completed. Do not assume these commands work end-to-end until they have been tested in the active workflow.

## Google Sheets history

The spreadsheet used by the workflow is named **LinkedIn - update History**.

The current recorded fields include:

- `Date and Time` — timestamp of the original Telegram message.
- `Subject` — topic text from the Telegram message, before the `|` separator.
- `LinkedIn Content` — Writer Agent output.
- `Approval Status` — should be saved as readable text such as `Approved` or `Declined`.
- `Approval time` — timestamp returned by the Gmail approval node.
- `Telegram Chat ID` — Telegram chat identifier, used to send replies to the same chat.

If additional columns such as `Request Status` and `Revision Instructions` are added, map them in the relevant Google Sheets nodes before relying on them.

## Important n8n expressions

### Approval condition

Use this expression as Value 1 in the approval IF node, with the operation **is true**:

```javascript
{{ $json.data.approved }}
```

### Telegram chat ID

Use this expression in Telegram Send Message nodes that execute after other nodes:

```javascript
{{ $('Telegram Trigger').first().json.message.chat.id.toString() }}
```

### Original Telegram message timestamp

```javascript
{{ DateTime.fromSeconds($('Telegram Trigger').first().json.message.date).toISO() }}
```

### Topic from Telegram

```javascript
{{ $('Telegram Trigger').first().json.message.text.split('|')[0].trim() }}
```

### Approval time

```javascript
{{ $json.data.respondedAt }}
```

## Setup checklist

- [ ] Telegram Trigger is connected to the correct Telegram bot credential.
- [ ] Writer Agent has a configured chat model.
- [ ] Reviewer Agent is connected to Writer Agent through the AI Tool connector and has its own chat model.
- [ ] Gmail `Send and Wait for Response` is configured for approval.
- [ ] Approval IF condition checks `{{ $json.data.approved }}` using **is true**.
- [ ] True branch appends an `Approved` row to Google Sheets and sends a Telegram confirmation.
- [ ] False branch appends a `Declined` row to Google Sheets and sends a Telegram decline message.
- [ ] Telegram chat ID expressions preview as a numeric ID, not `undefined`.
- [ ] Greeting/help messages receive a Telegram reply without Gmail approval.
- [ ] Workflow is saved and activated for automatic operation.
- [ ] `IGNORE` handling has been tested end-to-end.
- [ ] `EDIT` handling has been tested end-to-end, including re-review and a fresh Gmail approval.

## Testing plan

1. Send `Hi` in Telegram. Confirm a help reply is sent and Gmail is not triggered.
2. Send `Remote work`. Confirm a post is generated and a Gmail approval request is sent.
3. Approve the Gmail request. Confirm a row with status `Approved` is appended and Telegram sends a success message.
4. Decline a new Gmail request. Confirm a row with status `Declined` is appended and Telegram sends a decline message.
5. Send `IGNORE`. Confirm the request is not treated as a new topic. Verify the sheet/status update if that part has been configured.
6. Reply with `EDIT: ...`. Confirm the original declined draft is retrieved, revised using the instructions, reviewed, and sent for a new Gmail approval. This test is pending until the revision flow is implemented.

## Troubleshooting

### Telegram error: `Bad Request: chat_id is empty`
- Check the Chat ID expression.
- Prefer `{{ $('Telegram Trigger').first().json.message.chat.id.toString() }}`.
- Check that the expression preview shows a numeric ID rather than `undefined`.
- Test using a real Telegram-triggered execution; manual execution without trigger data may not have the Telegram message context.

### Approval IF branch is incorrect
- Set Value 1 to `{{ $json.data.approved }}` in Expression mode.
- Set operation to **is true**.
- Confirm `approved` is a Boolean, not the literal text `"false"` or an expression entered as fixed text.

### Google Sheets records `true` or `false`
- Set the Approval Status field on the True branch to fixed text `Approved`.
- Set the Approval Status field on the False branch to fixed text `Declined`.

### Telegram confirmation does not run
- Check the latest execution and confirm the correct IF branch was taken.
- Verify the Google Sheets append node succeeded before the Telegram node.
- Verify the Telegram Chat ID is not empty.

## Security and maintenance notes

- Keep Telegram, Gmail, OpenAI, and Google Sheets credentials in n8n's credential manager; do not place secrets in workflow prompts or this README.
- The exported workflow JSON may contain credential references and internal identifiers. Review it before sharing publicly.
- Re-test the workflow after changing prompts, node versions, credentials, or Google Sheets headers.

## Current status

The core draft → review → Gmail approval → Sheets logging → Telegram notification flow is configured. Greeting/help routing is configured. The `IGNORE` route and the full `EDIT`/revision loop are still in progress and must be tested before the workflow can be described as fully complete.
