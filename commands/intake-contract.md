---
description: Intake a contract PDF and build the full transaction in one conversation.
---

# Intake Contract

Walk the TC from "I just got a signed contract" to "the deal is fully set up in DocJacket" without leaving chat. This command invokes the `contract-intake` skill, which handles the full Phase 1-5 flow.

## What the TC provides

- A contract PDF (attached to the conversation)
- Optional context: state ("Florida"), side ("buyer-side" / "seller-side"), or document type ("listing agreement")

## What you do

Per the `contract-intake` skill recipe:

1. **Upload + extract** — five paths, in this order:
   - **A (canonical):** if the doc is already in DocJacket (DocDrop / email / web wizard / prior call), `list_documents({ hasExtraction: false })` to discover it, then `extract_existing_document({ documentId, state?, side?, documentTypeHint? })`. Works in every MCP client; bytes never leave DocJacket.
   - **B (upload link, default for a PDF not in DocJacket):** `create_upload_link`, tell the user to open the link in the browser where they use DocJacket and drop the PDF, then call `get_upload_link_status` (it waits on the server) until `submitted` gives an `extractionJobId`.
   - **C (email):** ask the user to email or forward it from their own address to `get_organization_summary` → `intake_addresses.new_deal.address`, then poll `list_new_deal_intake`. Email from anyone else waits in the Review Queue.
   - **D (code sandbox):** only if the sandbox can reach the `uploadHost` that `request_upload_url` returns. `request_upload_url({ filename })` → HTTP PUT of the bytes to `uploadUrl` from the sandbox → `kick_off_extraction({ uploadId, filename, ... })`. On a network or DNS error, stop and use B or C.
   - **E (last resort):** tiny PDF (<1 MB) attached to chat, no other option. `upload_document_for_extraction({ fileBase64, filename, ... })`. Bytes ride in chat context — compounds with later turns.
2. **Wait** — call `get_extraction_results`; it waits on the server (up to 30 seconds by default) and answers as soon as `status: "complete"`. While still in progress, call it again straight away. Surface a "still working…" update between calls. Cap at 10 minutes total.
3. **Confirm** — paraphrase the extracted fields (property, parties, dates, financials) and ask the TC to confirm or edit. Don't dump JSON.
4. **Apply** — call `apply_extraction` with any user edits as a flat `overrides` map. The response gives you the new `transactionId` — pin every follow-up call to it.
5. **Build the timeline** — propose computed dates (EMD, inspection, appraisal, financing, walkthrough) based on contract terms. On approval, call `add_key_dates_batch`.
6. **Send intros** — `render_email_template` first, present drafts, then `send_client_update` + `send_email_to_agent`. Each returns a preview (`awaiting_confirmation`): show it in full and call `confirm_email_send` only on the TC's explicit yes. A `parked_for_review` intro is waiting for approval in DocJacket, not sent.
7. **Schedule reminders** — `create_reminder` for the deadlines the TC names ("nudge the agent 2 days before EMD" → one call).
8. **Confirm** — call `get_intake_status` and present the final summary with the portal link from `get_portal_link`.

## Failure recovery

If anything fails mid-flow, call `get_intake_status(transactionId)`. The response's `missingRecommendedSteps[]` tells you exactly what to retry — and which tool to retry with. Don't restart from Phase 1.

## Anti-patterns

Per the `contract-intake` skill:
- Don't skip Phase 3 confirmation, even when extraction looks clean.
- Don't pre-populate `overrides` with every field — only the keys the TC actually edited.
- Don't fabricate dates the contract doesn't state.
- Don't call `apply_checklist` after `apply_extraction` — the apply already runs the default state/type/side template.
- Don't wait forever — cap at 10 minutes wall-clock.
