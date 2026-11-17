# Tickets-manager

A VS Code custom agent (**Ticket Calendar**) that reads a train ticket (PDF or image), builds a standardized tag from its details, and adds the journey to Google Calendar after you confirm.

Agent definition: [.github/agents/ticket-calendar.agent.md](.github/agents/ticket-calendar.agent.md)

## Scope
- Train tickets only (IRCTC e-ticket PDFs or images of tickets).
- Extracts: source and destination station codes, train number and name, class, journey date, departure and arrival times, PNR, passenger names, status, coach and seat.
- Builds a tag used as the calendar event title.
- Creates one Google Calendar event per ticket (account `personal`, primary calendar, timezone Asia/Kolkata).

## Limitations
- Train tickets only. Flights, buses, movies and other tickets are declined.
- Google Calendar access is limited to listing calendars, listing events and creating events. The agent cannot edit or delete events.
- Only the `personal` Google account is connected. The OAuth app is in Testing mode, so only listed test users can sign in.
- The account name (used in the tag) is not on the ticket, so the agent asks for it every time.
- The "first 3 letters" rules are applied to what is printed on the ticket; unusual train or passenger names may need your correction at the confirmation step.
- Printed departure and arrival times can change; the agent uses what the ticket shows.
- Not tested with multi-passenger tickets or tickets with confirmed coach and seat yet.

## Commands
| Command | Effect |
|---|---|
| `start` | Begin a new flow at the Upload phase |
| `restart` | Clear all state (ticket, details, account name, tag, progress) and start over |
| `help` | List commands and the current step; changes nothing |
| `retry` | Resume from the step that failed or was interrupted, without repeating finished steps or creating duplicate events |

## Workflow
1. **Upload**: attach the ticket PDF or image.
2. **Account**: the agent asks which account the ticket is booked under, then confirms it back.
3. **Parsing**: the agent extracts the ticket details and asks about anything it cannot read.
4. **Format generation**: the agent shows a details table and the tag. You confirm or request changes; it re-shows until you confirm.
5. **Calendar**: after confirmation, the agent checks for an existing event with the same tag, then creates the event and reports the result.

## Tag rules
Format (lowercase, `-` separated):

```
[source]-[destination]-[acc3]-[train3]-[coach]-[pax1_3]-[seat1]-[pax2_3]-[seat2]...
```

- `acc3`: first 3 letters of the account name.
- `train3`: first 3 letters of the train name, except **MAS PGT EXPRESS (22651)**, which always uses `pgt`.
- Each passenger adds the first 3 letters of the name, then the seat number, in ticket order.
- If coach and seat are not allocated (RAC/waitlist), the coach slot is `rac<n>` and the seat is `<n>` (for `RAC/7`: `rac7` and `7`).

Examples:
- `ms-dg-abi-tej-s9-abi-23-hen-24`
- `mas-dg-abi-pgt-rac7-abi-7`

## Calendar event rules
- Title: the tag.
- Start and end: departure and arrival date-times, timezone Asia/Kolkata.
- Location: source station.
- Description: train, class, PNR, passengers with status/coach/seat, account name.
- Reminders: popup 1 day and 2 hours before. No notification emails.
- Never created before you confirm, and never duplicated.

## How to use
1. Open the Chat view in VS Code and select **Ticket Calendar** in the agent picker.
2. Type `start`.
3. Attach the ticket file when asked.
4. Answer the account name question.
5. Review the extracted details and tag; reply with changes or `confirm`.
6. The agent creates the event and returns the details. If anything fails, type `retry`.

## Setup (done so far)
1. Created the agent file `.github/agents/ticket-calendar.agent.md`.
2. In Google Cloud: created a project, enabled the Google Calendar API, configured the OAuth consent screen (Testing, own account added as test user), and created a **Desktop app** OAuth client.
3. Saved the downloaded credentials as `.secrets/gcp-oauth.keys.json`. The folder is in `.gitignore`; never commit it.
4. Added the MCP server in `.vscode/mcp.json` using `@cocal/google-calendar-mcp` with `GOOGLE_OAUTH_CREDENTIALS` pointing at the credentials file.
5. Started the server from **MCP: List Servers** and signed in with the nickname `personal`. Verified by listing calendars.

To connect another account or re-authenticate, use the server's account management (add with a nickname) and sign in through the browser.

## Security notes
- `.secrets/` is gitignored. The folder sits inside OneDrive, so the credentials file may still sync there; consider moving the project or file outside synced folders.
- Calendar scope grants full calendar access; the agent is restricted to list and create tools.
