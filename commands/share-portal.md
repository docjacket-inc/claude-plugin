---
description: Get the portal link for a transaction and optionally send an email to share it with a party.
---

# Share Portal Link

1. **Identify the transaction.** If ambiguous, ask. (Search via `search_transactions` if the user named an address.)
2. Call **`get_portal_link`** to retrieve the URL + calendar feed.
3. **Handle the result:**
   - `PORTAL_NOT_FOUND` → the deal has no client portal link. Tell the user to create a Buyer/Seller link with no recipient in DocJacket → Portal (creating links via the AI is a future feature). This happens even when the deal has an Agent link or a link made for one person; those are never returned.
   - Found → show the URL + calendar feed URL inline.
4. **Ask who to share it with** (the buyer or the seller). This is the deal's client link; agents use their own agent portal, so don't send them this link. If the answer is one of the parties already on the deal, look them up via `find_contact_by_email` or `get_contacts`.
5. **Draft a brief intro email.** Include both the portal URL and the calendar feed URL, and send it with `send_client_update`.
6. Show the draft inline, ask the user to confirm. On yes, call the `send_*` tool; it returns a preview (`awaiting_confirmation`). Show the complete preview and call `confirm_email_send` only on the user's explicit yes. If it returns `parked_for_review`, tell the user it is waiting for approval in DocJacket, not sent.

## Example

```
🔗 Portal link for 412 Oak St (Smith)

URL: https://app.docjacket.com/portal/abc123def456
Calendar feed: https://app.docjacket.com/api/calendar/.../feed.ics
Created: 2026-05-15 by Casey  ·  Expires: never

Want me to send Sarah (buyer) an intro email with the link? She can subscribe
to the calendar feed in Apple Calendar / Google Calendar so all the deadlines
show up automatically.
```

## Don't

- **Don't paste the link in chat as the only delivery.** This is the deal's shared client link; anyone holding it sees the client view. Better to deliver it via an email the user reviewed.
- **Don't create a new portal link via the AI.** No tool for that yet.
- **Don't share a link to a party on a different deal.** Each transaction has its own link.
- **Don't send without explicit confirmation** — per `execution-workflow`.
