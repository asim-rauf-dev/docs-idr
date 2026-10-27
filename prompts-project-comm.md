# Build prompts: project email visibility

One prompt per AI session. Do not paste more than one.

Contract: [prd-project-email-communications.md](prd-project-email-communications.md) and [technical-plan-project-email-communications.md](technical-plan-project-email-communications.md).

Version 1 is client mail on the project page. Expert threads and compose are parked at the bottom.

Before slice 1, these three things must already be true. No prompt covers them.

- Spike notes exist: one real sent message and one reply, including whether `In-Reply-To` was present.
- Microsoft 365 owners have agreed: separate app, named mailbox group, encrypted store.
- Shared address pairs are stored on every matching open project and marked shared.

If the spike showed a reply cannot be tied back to the sent message, do not run slice 1.

---

## Slice 1 — Matcher only

```text
Read docs/prd-project-email-communications.md and docs/technical-plan-project-email-communications.md. Implement only technical plan step 7.

Build a pure function that decides whether one mail message is kept and which projects it belongs to. Cover it with Vitest. No Graph client, no database, no API, no UI, no expert threads, no encryption.

Inputs are plain data: from, to, cc, sent time, optional headers, and the open-project pair list. Outputs are zero or more matches with match_source of header, in_reply_to, or address_pair.

Rules:
- If X-IDR-Project-Id is present and that project is still open, write that one project with match_source header. Do not also fan out by address.
- Else if In-Reply-To or References hits a stored internet_message_id, copy that row's project with match_source in_reply_to. Do not fan out.
- Else if the addresses are a known client-service and client pair on open projects, and sent_at is on or after that project's created_at, write one row per matching project with match_source address_pair. If more than one project matches, mark each shared.
- If both addresses are only @idr.co or @idrnetwork.co, drop the message.
- Normalize @idr.co to @idrnetwork.co before comparing.

Tests must include: header wins one project, reply copies that project, address pair writes two projects with shared true, internal-only mail is dropped, mail before created_at is dropped.

Stop when those tests pass. Do not start step 3, step 6, or the UI.
```

## Slice 2 — Encryption only

```text
Read docs/technical-plan-project-email-communications.md step 3. Implement only that step.

Add AES-256-GCM encrypt and decrypt for a communications payload. Payload shape before encryption: from, to[], cc[], subject, bodyText. Each row gets a random IV, an auth tag, and a numeric key version.

Use a dedicated secret named COMMUNICATIONS_ENCRYPTION_KEY. Do not reuse the Wise encryption secret. Do not log the payload, the subject, or the body.

Vitest must prove: round-trip returns the same payload, a wrong key fails, and a different key version can be selected. No tables, no Graph, no API, no UI.

Stop when those tests pass.
```

## Slice 3 — Tables only

```text
Read docs/technical-plan-project-email-communications.md step 2. Implement only that step.

Add an euro_db migration and the Drizzle definitions for three tables:

- project_communications: id, project_id, graph_message_id, mailbox_upn, direction, angle_expert_id nullable, conversation_id nullable, internet_message_id, sent_at, match_source, shared, encrypted_payload, key_version, synced_at. Unique on (graph_message_id, project_id). Index (project_id, sent_at desc).
- graph_mail_delta: mailbox_upn, folder (inbox or sentitems), delta_link, last_success_at. Primary key (mailbox_upn, folder).
- communications_sync_run: mailbox, started, finished, messages seen, messages stored, skipped, error class. No subject, no body, no recipient list.

v1 directions are to_client and from_client only. Leave angle_expert_id nullable.

Do not write a sync job, a route, or a tab. Stop when the migration and schema types exist.
```

## Slice 4 — Pair index only

```text
Read docs/technical-plan-project-email-communications.md steps 4 and 5. Implement only those steps.

Build the loader that, from euro_db, returns every (client-service email, client email, project id, created_at) triple for projects that are not Closed. Include client-service analysts, associates, and managers, the project client, and additional clients.

Normalize addresses: trim, lowercase, and map @idr.co to @idrnetwork.co the same way modifyEmailToNetwork does. Omit closed projects. Omit pairs where both sides are only @idr.co or @idrnetwork.co.

Tests use fixtures, not a live Graph tenant. Do not call Graph. Do not include expert emails. Do not add a scheduler or a UI.

Stop when the fixture tests pass.
```

## Slice 5 — Delta walker only

```text
Read docs/technical-plan-project-email-communications.md steps 6 and 16. Implement only the delta page walker.

Walk Microsoft Graph delta for one mailbox folder (inbox, then the caller will run sentitems). Use an injected HTTP client. Every test uses a fake HTTP response. No live tenant, no production secrets, no UI.

Behavior:
- If no cursor exists, start a bounded initial delta.
- Follow @odata.nextLink until @odata.deltaLink.
- Cap pages per folder. If the cap hits, do not save a delta link for pages you have not finished.
- Persist the cursor only after the page is successfully handled by the caller.
- HTTP 429: honor Retry-After, then stop the mailbox and keep the last good cursor.
- HTTP 410: drop that folder's cursor and start a bounded resync, not a full mailbox history.
- HTTP 403: record forbidden and do not retry in a loop.
- HTTP 401: refresh the token once, then fail if it repeats.

Tests must cover 200 with a next link, 429, 410, and 403. Stop when those tests pass. Do not wire the matcher or the database in this slice.
```

## Slice 6 — Upsert one matched message

```text
Read docs/technical-plan-project-email-communications.md steps 7 and 8. The matcher, encryption, tables, pair index, and mocked delta walker already exist. Wire them. Do not rebuild them.

For each mocked Graph message: run the matcher, encrypt the payload, upsert project_communications. On unique conflict (graph_message_id, project_id), update the row only if the new match_source is not weaker. Rank is header, then in_reply_to, then address_pair. A second run of the same page must not insert a duplicate. A later address_pair pass must not downgrade a header match.

Save the delta cursor in the same transaction as the upserts for that page.

Tests use the fake HTTP client and a test database or the project’s existing DB test style. One mailbox. No EventBridge. No project tab. No expert matching.

Stop when replay and match-source rank tests pass.
```

## Slice 7 — Hourly job entrypoint

```text
Read docs/technical-plan-project-email-communications.md steps 9 and 16. Matcher, encryption, tables, pair index, delta walker, and upsert already exist. Add only the job entrypoint.

One invocation must:
- Take a Postgres advisory lock. If the lock is held, exit successfully and write nothing.
- Load the pair index.
- Intersect client-service mailboxes with the allowed mailbox list from config. Default that list to one mailbox.
- For each mailbox, delta inbox then sent items, match, upsert, and write one communications_sync_run row with counts and an error class.
- Release the lock.
- Never log subject, body, or the encrypted payload.

Do not add a Horizon tab. Do not add expert matching. Do not call Graph from Next.js.

Stop when a test proves the second overlapping run exits without writing.
```

## Slice 8 — Read API only

```text
Read docs/technical-plan-project-email-communications.md step 10. The tables and ciphertext already exist. Add only the read API.

GET /api/projects/{projectId}/communications

- requireSession. Logged out returns 401.
- Allow SU, Admin, or a user on that project's client-service analyst, associate, or manager list. Everyone else returns 403, including compliance roles.
- Read project_communications for that project_id, decrypt on the server, order by sent_at.
- Return direction, sentAt, subject, bodyText, from, to, conversationId, matchSource, shared, and lastSyncAt from graph_mail_delta for the client-service mailboxes on that project.
- This route must not import or call Microsoft Graph.

Tests: 401 logged out, 403 stranger, 200 client-service member, and a source or import assertion that this handler does not call Graph. No UI in this slice.

Stop when those tests pass.
```

## Slice 9 — Project tab only

```text
Read docs/prd-project-email-communications.md section 5 and docs/technical-plan-project-email-communications.md step 10. The GET API already exists. Add only the tab.

On the project page, add a Communication tab behind a feature flag that defaults off. Lazy-load it so it is not in the first project payload. Use TanStack Query and a query key from lib/querykeys.ts. The tab may only GET /api/projects/{projectId}/communications.

Show the message list, an empty state when there are no rows, and the last sync time always. No compose box. No button that calls Microsoft or runs the sync job.

Stop when the flag-off build does not show the tab and the flag-on tab renders from the API only.
```

## Slice 10 — Human check, no prompt

Turn the job on for one mailbox. For several days, compare Outlook with the tab. Write down misses and messages on the wrong project. Widen the mailbox group only after that list is acceptable. Then enable the flag for that project's client-service team.

---

## Parked — do not run with slices 1–9

### Experts

```text
Read docs/technical-plan-project-email-communications.md step 11. Client communications already work in production. Add expert threads on top. Do not redesign the client path.

Extend the pair index with active expert emails and angle_experts.id on non-closed projects. Extend the matcher so X-IDR-Angle-Expert-Id and In-Reply-To outrank address fan-out, and address match can set angle_expert_id with direction to_expert or from_expert.

Add GET /api/experts/{expertId}/communications?projectId&angleExpertId. The caller must already be allowed to see that project. The expert panel shows the tab only when the expert is opened in a project scope. The tab only reads the API.

Do not change how outreach is sent. Do not add compose. Do not call Graph from the tab.
```

### Headers on send

```text
Read docs/technical-plan-project-email-communications.md step 12, headers on send only.

In the existing jobs Lambda sendMail payload for project mail, add internet message header X-IDR-Project-Id. Where the send is for an expert on a scope, also add X-IDR-Angle-Expert-Id. Do not change the poll, the matcher, the database, or the tab. The send call still does not need to return the stored message. The next poll reads Sent Items.
```

### Compose

```text
Read docs/technical-plan-project-email-communications.md step 12, compose only. The read path is already trusted.

On the project page, a client-service member on that project, or an admin, can compose a message. Submit through the existing email queue so the jobs Lambda sends as that person's mailbox, not as a shared noreply address. Do not add a new Graph send stack in Horizon. Do not write the sent row from the web request. The next poll stores it.
```
