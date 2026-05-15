# Engagement Case Sufficiency Analyzer Implementation Plan

**Goal:** For each Salesforce engagement Case with `Status = "New"`, analyze the description for audience / ask / maturity sufficiency, generate clarifying questions when needed, route to a Coded App reviewer with HITL approval, and on approve post a Chatter @mention to the Primary TAM with the initial outreach.
**Source:** None — planned from user request
**Project type:** Solution (Flow + AI Agent + Application)
**Expression language:** N/A (no XAML)
**Approach:** explore-first
**Execution autonomy:** autonomous
**App type:** N/A (no RPA UI automation)
**App state:** N/A
**UI targeting:** N/A
**Solution scope:** SW

## Understanding

The automation periodically lists Salesforce Cases where `Status = "New"`, inspects the Case description for three sufficiency dimensions (**audience type**, **what the customer is looking for**, **customer maturity** on the products mentioned), and uses an LLM-driven Agent to generate clarifying questions when any dimension is underspecified. A reviewer Coded Web App lists each case with the Primary TAM, customer / case links, the description, and the agent-suggested questions (the questions section is hidden when nothing is missing — some cases will have complete info and need no outreach). The human approves, edits, or rejects via an Action Center HITL gate; on approve, the orchestrating Flow posts a Salesforce Chatter comment on the Case @-mentioning the Primary TAM so they are notified before scheduling the engagement start date.

## Decisions & Trade-offs

- **Studio Web solution scope** — user explicitly requested SW; cloud-authored Flow + Agent + Coded App as peer projects under one `.uipx` solution.
- **Multi-skill pattern: Pattern 2 (Flow with local resources)** — all components built fresh in this session as sibling projects; Flow consumes them locally inside the solution (not via tenant identity).
- **Agent: low-code Agent Builder (`agent.json`)** is preferred for this single-prompt structured-output task — simpler than a coded LangGraph agent and sufficient for sufficiency classification + question generation. `uipath-agents` may switch to a coded form if structured-output reliability requires it.
- **HITL via Action Center with a custom Coded App handler** — one shared task type (`engagement-case-review`) authored once and consumed by both producers (Flow) and the reviewer UI (Coded App).
- **Salesforce access via Integration Service** — single tenant SF connection used for both `Query Cases` and `Post Chatter feed comment` (with `MentionSegmentInput` for the @mention).
- **Approach: explore-first / Autonomy: autonomous** — user asked for a plan deliverable. Once the plan is approved, specialists run end-to-end with the stop conditions below.
- **Inferred from "Plan only, no scaffolding":** Step 1 elicitation (generation approach, autonomy, project type, solution scope) is fully resolved from context — no `AskUserQuestion` call was needed.

## Stop conditions

- `uip login` fails or the token cannot be refreshed against the target UiPath Cloud tenant.
- The Salesforce IS connection cannot be created or authorized (OAuth blocked, scopes missing, sandbox vs production mismatch).
- The Salesforce `Case` object's required fields (see §Salesforce Data Contract) are absent or renamed in this org and no equivalent can be inferred.
- The Primary TAM cannot be resolved to a Salesforce `User.Id` from the Case / Account — the Chatter @mention payload is invalid.
- The Action Center task schema fails to register, blocking the HITL gate.
- A pre-existing solution / project with the same slug exists and cannot be reused or overwritten without explicit user confirmation.

## Salesforce Data Contract

### Cases read (filter `Status = 'New'`)
Fields retrieved per Case:
- `Id`
- `CaseNumber`
- `Subject`
- `Description`
- `Status`
- `Account.Id`, `Account.Name`
- `OwnerId`
- `Owner.Id`, `Owner.Name`, `Owner.Type` *(only `User` type can be @-mentioned in Chatter — `Queue` cannot)*
- Custom Primary TAM field, if present in this org (e.g. `Primary_TAM__c` → resolves to a `User.Id`). Resolved at execution time per Open Question 2.
- Custom products-list field, if present (e.g. `Products_Mentioned__c`); otherwise inferred by the Agent from `Description` per Open Question 3.

### Link templates
- Customer (Account) URL: `https://<instance>.lightning.force.com/lightning/r/Account/<Account.Id>/view`
- Case URL: `https://<instance>.lightning.force.com/lightning/r/Case/<Case.Id>/view`

### Chatter @mention payload (post on Case feed)
- Action: Salesforce IS — **Post Chatter feed comment** on parent `Case.Id`.
- Body segments:
  - `MentionSegmentInput` with `id = primary_tam_user_id`
  - `TextSegmentInput` with the rendered outreach message
- Outreach message template (Flow renders at runtime):

```
Hi @{TAM.FirstName}, before scheduling a kickoff for this engagement I'd like
to confirm a few details about the case. Could you clarify the following?

{NumberedListOfFinalQuestions}

Reply on this case and I'll plan the start date from there. — Engagement Bot
```

## Agent I/O Contract

**Input** (single JSON object passed by Flow):
```json
{
  "case_id": "5003X00000ABCD",
  "case_number": "00012345",
  "subject": "string",
  "description": "string (raw case description text)",
  "products_mentioned": ["Studio", "Orchestrator", "Document Understanding"],
  "customer_name": "string"
}
```

**Output** (single JSON object returned to Flow):
```json
{
  "audience_sufficient": true,
  "ask_sufficient": false,
  "maturity_sufficient": false,
  "missing_dimensions": ["ask", "maturity"],
  "needs_questions": true,
  "suggested_questions": [
    "Which business unit is the primary audience for this engagement?",
    "What outcome does the customer want in the first 90 days?",
    "What is the customer's current maturity with Studio and Orchestrator (pilot / production / scaled)?"
  ],
  "reasoning_note": "Description names audience but not the desired outcome or product maturity."
}
```

Rules enforced in the prompt:
- `needs_questions = NOT (audience_sufficient AND ask_sufficient AND maturity_sufficient)`.
- `suggested_questions` is `[]` when `needs_questions = false`.
- Maximum 5 questions: one per missing dimension, plus product-maturity follow-ups (one per product mentioned, capped by the 5-question total).

## HITL Task Schema (Action Center — shared by Flow + Coded App)

**Task type:** `engagement-case-review`

**Input data** (Flow → task):
```json
{
  "case_id": "string",
  "case_number": "string",
  "subject": "string",
  "description": "string",
  "customer_name": "string",
  "customer_link": "string",
  "case_link": "string",
  "primary_tam_name": "string",
  "primary_tam_user_id": "string",
  "products_mentioned": ["string"],
  "needs_questions": true,
  "suggested_questions": ["string"],
  "outreach_preview": "string"
}
```

**Output data** (Coded App → task):
```json
{
  "decision": "approve | edit | reject",
  "final_questions": ["string"],
  "reviewer_comment": "string"
}
```

`edit` lets the reviewer modify questions before approval; Flow uses `final_questions` when posting Chatter (falls back to `suggested_questions` if `final_questions` is empty on `approve`).

## Run Sequence (end-to-end)

```
1. Trigger fires (schedule per Open Question 1).
2. Flow → Salesforce.QueryCases(Status='New', fields per §Salesforce Data Contract).
3. ForEach Case:
   a. Flow resolves primary_tam_user_id + customer_link + case_link.
   b. Flow → Agent.Analyze(case payload) → sufficiency JSON.
   c. Flow → ActionCenter.CreateTask(engagement-case-review, input per §HITL Task Schema).
4. Reviewer opens the Coded App, sees the case queue, reviews fields + suggested questions,
   chooses Approve / Edit & Approve / Reject.
5. Coded App completes the HITL task with the output schema.
6. Flow resumes on task completion:
   - approve / edit → Salesforce.PostChatterFeedComment(case_id,
     mention=primary_tam_user_id, text=rendered template using final_questions).
   - reject → log and continue.
7. Flow records the per-case outcome (trace + run log).
```

## Open questions for the user

1. **Trigger cadence** — schedule (every N hours) or webhook on Case insert / Status change? Default: schedule every 2h during business hours.
2. **Primary TAM field** — is there a custom Salesforce field (e.g. `Primary_TAM__c`) or is `Case.OwnerId` always the TAM? Affects T1 and T8.
3. **Products list source** — custom field listing products, or have the Agent infer from `Description`? Affects T4.
4. **Reject behavior** — on reject, post a different Chatter note or close silently? Default: silent close.
5. **Edit mode** — can the reviewer add free-form questions alongside editing suggested ones? Default: yes.
6. **Outreach message tone** — confirm the template in §Salesforce Data Contract or supply preferred copy.

These are not blockers — the plan ships sensible defaults; T8 (Flow wiring) re-asks any item still unresolved at execution time.

---

## Task T1 — uipath-platform — Auth and Integration Service setup

**Identity:** `platform:engagement-case-analyzer:bootstrap`
**Status:** [ ] pending
**Blocked by:** none
**Skill prompt:**

> Load `uipath-platform`. Run `uip login` against the user's UiPath Cloud tenant. Create a Salesforce Integration Service connection at the tenant level — suggested name `sf-engagement-cases`. Verify the connection can list `Case` records (filter `Status = 'New'`) and call the Chatter `Post feed comment` action. Do not scaffold project files here — that begins in T2.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Run `uip login`; record tenant URL in the plan footer
- [ ] Create Salesforce IS connection `sf-engagement-cases` (interactive OAuth via SW if CLI cannot complete it)
- [ ] Smoke-test: list one `Case` where `Status = 'New'`; confirm schema and Chatter permission
- [ ] **Validate:** `uip is-connection list` shows `sf-engagement-cases` as active

## Task T2 — uipath-maestro-flow — Create SW solution and init Flow project

**Identity:** `flow:engagement-case-analyzer:bootstrap`
**Status:** [ ] pending
**Blocked by:** T1
**Skill prompt:**

> Load `uipath-maestro-flow`. Create a UiPath Studio Web solution at the repo root with slug `engagement-case-analyzer`. Initialize an empty Flow project `engagement-flow` as the orchestrator skeleton — no node wiring yet (T8 owns wiring). Configure the solution to host sibling Agent + Coded App projects (added in T4 and T6). Bind the `sf-engagement-cases` IS connection at the solution level. Do not pack or publish.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Create SW solution `engagement-case-analyzer` (`.uipx`)
- [ ] Init empty Flow project `engagement-flow` inside the solution
- [ ] Bind `sf-engagement-cases` IS connection at the solution level
- [ ] **Validate:** `uip maestro flow list` shows `engagement-flow`; solution structure verified in SW

## Task T3 — uipath-human-in-the-loop — Action Center task schema

**Identity:** `human-in-the-loop:engagement-case-analyzer:engagement-case-review`
**Status:** [ ] pending
**Blocked by:** T2
**Skill prompt:**

> Load `uipath-human-in-the-loop`. Author the Action Center task type `engagement-case-review` and its `action-schema.json` exactly mirroring §HITL Task Schema (input + output data as listed). The form is rendered by a Coded App handler in T6 — write the schema only, mark the task type as app-driven (no inline form HTML). Register the schema so both Flow (T8 producer) and Coded App (T6 consumer) can reference it.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Create `action-schema.json` for task type `engagement-case-review`
- [ ] Input fields per §HITL Task Schema (input)
- [ ] Output fields per §HITL Task Schema (output)
- [ ] Mark schema as app-driven; register against the solution
- [ ] **Validate:** schema parses; new task type visible via `uip tasks` listing or SW UI

## Task T4 — uipath-agents — Build the sufficiency-analysis agent

**Identity:** `agents:engagement-case-analyzer:case-sufficiency-agent`
**Status:** [ ] pending
**Blocked by:** T2
**Skill prompt:**

> Load `uipath-agents`. Build a low-code Agent Builder agent named `case-sufficiency-agent` inside the solution. Input/output contract exactly per §Agent I/O Contract. The prompt must assess audience / ask / maturity sufficiency from the Case description and emit suggested clarifying questions when any dimension is missing. Use structured-output binding (JSON schema) so the agent always returns valid JSON matching the contract. Use the default UiPath LLM Gateway model unless the user has a BYO connection configured. Switch to a coded LangGraph implementation only if structured-output reliability requires it.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Create agent project `case-sufficiency-agent` (`agent.json`)
- [ ] Define input JSON schema per §Agent I/O Contract (input)
- [ ] Define output JSON schema per §Agent I/O Contract (output)
- [ ] Author LLM prompt: assess sufficiency on three dimensions, generate ≤5 questions, include `reasoning_note`
- [ ] **Validate:** `uip agent build` succeeds; one-shot run with a synthetic case payload returns schema-valid JSON

## Task T5 — uipath-agents — Testing (MANDATORY)

**Identity:** `agents:engagement-case-analyzer:testing`
**Status:** [ ] pending
**Blocked by:** T4
**Skill prompt:**

> Load `uipath-agents` and run its testing workflow end-to-end. Always thorough: happy path + edge cases + error scenarios. Build an eval set covering: complete-description case (expect `needs_questions=false`), missing-ask case, missing-maturity case, missing-all case, multi-product case (Studio + Orchestrator + DU), empty-description edge case, non-English description case. See that skill's testing references for commands, eval-set authoring, and best practices. Do not describe the testing procedure here — the specialist owns it.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Run testing workflow per `uipath-agents`'s testing reference
- [ ] **Validate:** all evals pass; record results

## Task T6 — uipath-coded-apps — Engagement Case Reviewer app

**Identity:** `coded-apps:engagement-case-analyzer:engagement-case-reviewer`
**Status:** [ ] pending
**Blocked by:** T3
**Skill prompt:**

> Load `uipath-coded-apps`. Scaffold a coded web app `engagement-case-reviewer` (React + TypeScript) inside the solution. Use `@uipath/uipath-typescript` to subscribe to Action Center tasks of type `engagement-case-review` (schema from T3). UI: list view with one card per pending task showing Primary TAM, customer name (link to `customer_link`), case number + subject (link to `case_link`), full description, products mentioned, and — when `needs_questions = true` — the agent's `suggested_questions` rendered as an editable list. When `needs_questions = false`, hide the questions section and show "No questions needed — Approve to proceed". Each card has Approve / Edit & Approve / Reject buttons that complete the task with the output schema (`decision`, `final_questions`, `reviewer_comment`). Functional UI only — no design polish required.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Scaffold coded app `engagement-case-reviewer` (`.uipath/` + `app.config.json`)
- [ ] Install `@uipath/uipath-typescript`; configure auth via app context
- [ ] Build the task-queue list view bound to `engagement-case-review` tasks
- [ ] Build the per-card detail with editable suggested questions and Approve / Edit / Reject actions
- [ ] Conditional rendering: hide question editor when `needs_questions = false`
- [ ] Wire submit actions to `completeTask` with the output schema from T3
- [ ] **Validate:** `uip codedapp build` succeeds; manual smoke test against a seeded task

## Task T7 — uipath-coded-apps — Testing (MANDATORY)

**Identity:** `coded-apps:engagement-case-analyzer:testing`
**Status:** [ ] pending
**Blocked by:** T6
**Skill prompt:**

> Load `uipath-coded-apps` and run its testing workflow end-to-end. Always thorough: happy path (approve with no edits), edge cases (no questions needed, all questions edited, reject, very long description, zero pending tasks), error scenarios (task already completed by another reviewer, SDK auth failure, network drop mid-submit). See that skill's testing references for commands and best practices. Do not describe the testing procedure here — the specialist owns it.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Run testing workflow per `uipath-coded-apps`'s testing reference
- [ ] **Validate:** all tests pass; record results

## Task T8 — uipath-maestro-flow — Wire the orchestration Flow

**Identity:** `flow:engagement-case-analyzer:engagement-flow-wiring`
**Status:** [ ] pending
**Blocked by:** T3, T5, T7
**Skill prompt:**

> Load `uipath-maestro-flow`. Wire `engagement-flow` (from T2) end-to-end per §Run Sequence. Nodes: (1) Trigger — schedule per Open Question 1 default; (2) Salesforce IS Query Cases — filter `Status = 'New'`, fields per §Salesforce Data Contract; (3) ForEach over results; (4) Script node — build `customer_link`, `case_link`, resolve `primary_tam_user_id`; (5) Agent invocation — call `case-sufficiency-agent` with the per-case payload; (6) Action Center task creation — type `engagement-case-review`, input mapped per §HITL Task Schema; (7) Wait for completion; (8) Branch on `decision`: approve / edit → Salesforce IS Post Chatter feed comment on `Case.Id` with `MentionSegmentInput(primary_tam_user_id)` + `TextSegmentInput` rendered from the template using `final_questions`; reject → log + continue. Finalize per `Solution scope: SW` (publish target = Studio Web).
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Add schedule trigger to `engagement-flow`
- [ ] Salesforce Query Cases node — fields + filter per §Salesforce Data Contract
- [ ] ForEach loop over results
- [ ] Script node: build `customer_link`, `case_link`, resolve `primary_tam_user_id` + `primary_tam_name`
- [ ] Agent invocation node calling `case-sufficiency-agent` with mapped input
- [ ] Action Center task-creation node for `engagement-case-review` with full input payload
- [ ] Wait-for-completion gate
- [ ] Approve / Edit branch: Salesforce Chatter feed-comment node with `MentionSegmentInput` + rendered outreach text using `final_questions`
- [ ] Reject branch: log node + continue loop
- [ ] **Validate:** `uip maestro flow validate engagement-flow` passes with no errors

## Task T9 — uipath-maestro-flow — Testing (MANDATORY)

**Identity:** `flow:engagement-case-analyzer:testing`
**Status:** [ ] pending
**Blocked by:** T8
**Skill prompt:**

> Load `uipath-maestro-flow` and run its testing workflow end-to-end. Always thorough: happy path (case with sufficient info → approve → comment posted), edge cases (case with no description, case with 5+ products, no New cases at all, multiple New cases in one run, owner is a Queue not a User), error scenarios (Salesforce auth expired mid-loop, agent timeout, task abandoned, Chatter post 403). End-to-end pipeline test against a sandbox Salesforce org strongly recommended. See that skill's testing references for `uip maestro flow eval`, eval-set authoring, and best practices. Do not describe the testing procedure here — the specialist owns it.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] Run testing workflow per `uipath-maestro-flow`'s testing reference
- [ ] **Validate:** all evals + e2e pass; record results

## Task T10 — uipath-platform — Publish solution and verify trigger

**Identity:** `platform:engagement-case-analyzer:publish`
**Status:** [ ] pending
**Blocked by:** T9
**Skill prompt:**

> Load `uipath-platform`. Pack and publish the `engagement-case-analyzer` solution to UiPath Studio Web (per `Solution scope: SW`). Confirm the schedule trigger from T8 is registered against the tenant. Verify the coded app and agent are deployed and reachable. Surface the SW URLs of the deployed Solution, Flow, and Coded App to the user.
>
> Use values, mappings, and structure exactly as documented in this plan. Do not infer or guess.

- [ ] `uip solution pack` the workspace
- [ ] `uip solution publish` to the tenant (Studio Web)
- [ ] Verify trigger registration via `uip triggers list`
- [ ] Surface deployed URLs (Solution, Flow, Coded App) to the user
- [ ] **Validate:** dry-run trigger fires `engagement-flow` against current New cases; smoke OK

---

### Notes

- **Live task emission skipped.** This session's `TaskCreate` MCP tool is disconnected. Once it's available again, this plan can be re-loaded and the planner can emit one live task per row above (`addBlockedBy` edges per the `Blocked by` lines).
- **Resume:** to start execution against this plan, ask Claude to "execute the engagement-case-analyzer plan" — it will load the first specialist skill (T1: `uipath-platform`) and walk the task list in order.
