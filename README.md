# PRD: Project email visibility via Microsoft Graph
**Status:** Discovery. This document records the direction from the 26 Oct discussion (Amit) and the questions still open. It is not an implementation spec and it does not lock schema, jobs, or UI.
**Decision so far:** Do not use HubSpot (no shared logging mailbox, no BCC, no HubSpot sync) for this feature.
**Related:** Existing outreach still sends through Horizon → SQS → the jobs Lambda → Microsoft Graph `sendMail`. This feature does not replace sending. It adds a **read path**: a background poll stores copies of relevant mail, and Horizon pages read that store.
---
## 1. Problem
Staff already email clients and experts from Outlook and from Horizon. Those threads are not visible next to the project or the expert in Horizon.
Two designs were considered and set aside for the main path:
| Approach | Why it fails the use case |
| --- | --- |
| Subject / reference code per expert or project | A new email that drops the code is invisible. People will not keep a code in every subject. |
| One mailbox per project | Volume is too high (on the order of 50–100 projects a week). |
| HubSpot as the system of record | Extra product, extra mailbox rules, and it does not match how Amit wants mail found (people on the project, not a logging inbox). |
| Call Microsoft Graph when someone opens the panel | Slow, easy to hit Graph limits, and a poor experience. Decrypting a stored row is faster than calling Microsoft 365. |
## 2. Outcome
A person working a project can open Horizon and see the **email conversation that belongs to that project**: messages between the project’s client-service people and that project’s clients, and later the same kind of view for **analyst ↔ expert** on a scope.
The page always reads **our store**. It never calls Graph on click. If the poll is behind, the page shows the last successful sync time and the messages already stored.
## 3. Users
| Who | Need |
| --- | --- |
| Client service (analyst, associate, manager on the project) | See mail with the client for this project without leaving Horizon. |
| Project staff working experts | See outreach and replies for an expert on a given angle (second slice; same mechanism). |
| Admin / SU | Operate the poll, see sync health, and (later) the same threads when supporting a project. |
| Compliance roles | Not in v1 unless explicitly added. Today they are gated away from unrelated internal APIs. |
## 4. Proposed solution (Amit)
One pattern, two audiences. Ship **client ↔ client-service** first. **Expert ↔ analyst** uses the same job shape afterward.
```text
Hourly job
  → separate Azure app, application Mail.Read
  → only mailboxes of client-service users (Exchange RBAC, not the whole tenant)
  → delta: Inbox and Sent Items
  → keep a message only if the other party is a known client (or later, expert)
     on an open project that person is staffed on
  → encrypt subject and body, upsert in euro_db
Project or expert page
  → read the table, decrypt on the server, show the thread
  → show last sync time
```
Sending from the project page is a **later phase**. v1 is read-only in Horizon. People keep sending from Outlook and from existing Horizon outreach. The next poll stores those messages too.
### 4.1 Who is “in scope” for a project
Horizon already has the pair. The job does not invent contacts.
**Client communications (first slice)**
- Horizon side: users on the project’s client-service assignments (analysts, associates, managers).
- Other side: the project client’s known email addresses, plus additional clients on the project.
- A message counts when one side is that CS person and the other side is a client email on a project they are staffed on, and the message time is on or after the project’s `created_at`.
- Direction is `to_client` or `from_client`.
**Expert communications (same pattern, later)**
- Horizon side: the analyst (or sender) mailbox we are allowed to read.
- Other side: active emails on the expert, tied through `angle_experts` on a non-closed project.
- Direction is `to_expert` or `from_expert`.
- Filter the panel by project and angle expert, not “every email this expert ever received.”
### 4.2 How a message is attached to a project
Order, most specific first:
1. **Header we set on mail we send** (`X-IDR-Project-Id`, and for expert mail `X-IDR-Angle-Expert-Id`). Graph accepts custom headers only at send time, and only names starting with `x-` / `X-`. Outlook Reply does **not** copy them onto the reply.
2. **`In-Reply-To` / `References`** pointing at an `internetMessageId` we already stored. That copies `project_id` (and angle expert id) onto the reply.
3. **Address pair.** A brand-new subject with no header and no reply chain still matches if the two people are a known pair on an open project. This is the case the subject-code design cannot cover.
If the same address pair sits on **two open projects**, store the message on **both** and label it shared. Do not guess a single project.
`conversationId` from Graph is for grouping in the UI. It is not required to exist in our database before the message arrives.
Graph delta pages do not include `internetMessageHeaders`. The job selects id, from, to, subject, `internetMessageId`, `conversationId`, and a body preview (or body). Only for a message that already looks like a known client or expert does it fetch headers.
### 4.3 What we do not store
- The rest of the inbox. Delta may return it; the job drops it.
- Mail where **both** addresses are internal (`idr.co` / `idrnetwork.co`) and there is no client or expert on the other side.
- Mail after a project is closed. Rows already stored stay. New mail is not attached. Closed projects are not polled for new pairs.
- Mail to someone who is **not** in the mailbox set we are allowed to read. If a client emails an analyst who is not client-service on that project, it will not appear.
- Mail from an address that is **not** on the client user or expert email record. We only know addresses stored in Horizon.
### 4.4 Microsoft access
- **Separate Azure app** from the one Horizon uses for calendar and `Mail.Send`, so `Mail.Read` can be revoked on its own.
- **Application** permission `Mail.Read`, admin consent.
- **Exchange RBAC for Applications** limited to a mail-enabled group of client-service (and, when expert threads ship, the mailboxes that send expert mail). The app must not be able to open an arbitrary mailbox.
- **Client-credentials** token in the job. A logged-in user’s token cannot run an hourly poll.
- **Delta links** stored per mailbox and folder (`inbox`, `sentitems`). First run is bounded (for example, not older than the oldest open project for that user, with a cap) so the first sync cannot pull years of mail.
- Poll about **once an hour**. Not on the HTTP request that sends outreach, so Graph limits cannot break mail delivery.
- Overlapping runs must not double-write (advisory lock or equivalent).
Graph cannot reliably filter “from any of these clients OR to any of these clients” in the query. Pull the delta page, then filter in the job. That is why the mailbox list has to stay small.
### 4.5 Store
- Database: **euro_db** (same place as projects, experts, and billing).
- Subject and body: **AES-256-GCM** before insert, own secret (same idea as Wise token encryption, not the same key).
- Addresses used for matching may be stored in a form the job can look up; the readable from, to, subject, and body sit in the ciphertext.
- List queries are by `project_id` (and for experts, expert + angle expert), not by scanning ciphertext.
- One Graph message that matches two projects is **two rows**. Uniqueness is Graph message id **per project** (and per angle expert when that dimension exists).
- Delta cursor table: user/mailbox, folder, delta link, last success time.
The UI decrypts on the server for an authorized reader. It does not send ciphertext to the browser as the only copy, and it does not call Graph.
### 4.6 Who can see it (v1)
- SU, Admin, and a user who is on that project’s client-service team (client threads).
- Expert threads: same idea — people who already work that project/scope, plus Admin/SU. Exact role list is an open question below.
- Masking client names (“Client 1 / Client 2”) is **not** v1. The people who can open the tab already know the address.
### 4.7 Later: send from the page
Only after the read path is trusted.
- A client-service member on that project, or an admin, composes in Horizon.
- Send goes through the existing send-email Lambda **as that person’s mailbox**, not as a shared noreply address.
- The next poll stores the sent copy. v1 does not need this to prove the feature.
Optional later: jobs Lambda sets `X-IDR-Project-Id` / `X-IDR-Angle-Expert-Id` on outbound Graph `sendMail` so mail we send matches on header instead of only on address. Replies still use `In-Reply-To` and the address fallback.
---
## 5. User-facing behavior (v1)
**Client slice**
- Project page, lazy-loaded tab (not on first paint of the whole project).
- Thread list grouped in a way people can scan (by conversation or by the other party — exact layout is open).