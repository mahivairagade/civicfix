# CivicFix

**Report it once. We'll chase it until it's fixed.**

CivicFix is an agentic AI case-worker for civic issue reporting and resolution. A citizen describes a problem in a chat widget — a pothole, a garbage pile-up, a broken streetlight, a water leak — and the agent classifies it, checks whether someone has already reported the same issue nearby, files or merges the report, and can later draft follow-ups on complaints that are still sitting unresolved.

Built for a Lenovo internship project, originally scoped from an SDG/hackathon context (SDG 11: Sustainable Cities & Communities, inspired by SIH Problem Statement 25031).

---

## How it works

1. A citizen opens the chat widget and describes an issue.
2. An AI agent (Groq's `llama-3.3-70b-versatile`) has a short conversation to pin down three things: **category**, **urgency**, and **location**. If the location is missing, it asks for it before continuing.
3. Once the agent confirms it's ready to file, a second LLM call extracts the conversation into structured JSON (`category`, `urgency`, `location`, `description`).
4. The system searches existing open complaints in the same category. If a likely duplicate is found, it increments that record's `report_count` instead of creating a new one. Otherwise, it creates a new record with a generated case ID (`CF-XXXX`).
5. The citizen can return any time and ask for a status update using their case ID, and the agent looks it up directly.
6. A separate, schedulable workflow scans for complaints still marked "Submitted" and drafts a follow-up/escalation note.

---

## Stack

| Layer | Tool |
|---|---|
| Orchestration | [n8n](https://n8n.io/) (self-hosted via Docker, `localhost:5678`) |
| LLM inference | [Groq API](https://groq.com/) — `llama-3.3-70b-versatile` |
| Database | [Airtable](https://airtable.com/) |
| Frontend | Vanilla HTML/CSS/JS |

---

## Architecture

```
Citizen message
      │
      ▼
 Chat Trigger (n8n, publicly available webhook)
      │
      ▼
 Is Status Check? ──true──▶ Extract case ID (regex) ──▶ Search Airtable by case_id
      │false                                                  │
      ▼                                                        ▼
   AI Agent (Groq + memory)                      Found? ──▶ Return case summary
      │                                              │
      ▼                                          Not found ──▶ Return "couldn't find" message
 Ready to file? (checks for "Filing your
 complaint now" in the agent's reply)
      │true                        │false
      ▼                            ▼
 Extract structured JSON      Pass through the clarifying
 (category/urgency/location/  question as-is — skip Airtable
 description) via 2nd LLM     entirely
      │
      ▼
 Search for duplicates (same category, status ≠ Resolved)
      │
   found?───true───▶ Update existing record (report_count + 1)
      │false
      ▼
 Create new record (status = Submitted, report_count = 1)
      │
      ▼
 Merge both branches ──▶ Final reply with case ID
```

A second, independent workflow (**CivicFix – Follow-up Agent**) runs on a schedule (or manually), searches for all complaints still marked `Submitted`, and drafts a follow-up note for each using the same LLM.

---

## Airtable schema

**Base:** `CivicFix` → **Table:** `Complaints`

| Field | Type | Notes |
|---|---|---|
| `Name` | Single line text | Default primary field, unused |
| `description` | Long text | |
| `category` | Single select | Pothole, Garbage, Streetlight, Water, Other |
| `urgency` | Single select | Low, Medium, High |
| `location` | Single line text | |
| `status` | Single select | Submitted, In Progress, Resolved |
| `report_count` | Number | Incremented on duplicate reports |
| `created_at` | Created time | |
| `case_id` | Single line text | Format `CF-XXXX`, citizen-facing reference |

---

## Repo contents

- `civicfix-landing.html` — the main landing page: hero section with an embedded live chat kiosk, problem statement, how-it-works walkthrough, duplicate-detection demo, architecture diagram, impact/SDG section, and a client-side demo auth modal (Sign In / Create Account — **not** backed by real accounts; see below).
- `civicfix-chat.html` — a simpler standalone chat widget (superseded by the landing page, but still functional on its own).
- `workflows/` — exported n8n workflow JSON for both the main intake pipeline and the follow-up agent, so they can be re-imported into any n8n instance.

---

## Running it yourself

### 1. n8n
- Import the workflow JSON from `workflows/` into your own n8n instance.
- Add a Groq API credential and an Airtable Personal Access Token credential.
- Recreate the Airtable base/table using the schema above (or point the Airtable nodes at your own base).
- In the **Chat Trigger** node, toggle **"Make Chat Publicly Available"** on.
- **Publish** the workflow (saving alone does not update the live webhook), then copy the **Production URL**.

### 2. Frontend
- Open `civicfix-landing.html` and paste your n8n production webhook URL into the `WEBHOOK_URL` variable near the top of the `<script>` block.
- Serve the file with a local server — e.g. VS Code's "Go Live," or:
  ```bash
  python -m http.server
  ```
  Do **not** open it via `file://`; that triggers CORS/fetch errors.

### 3. Follow-up agent
- Import and run the separate `CivicFix – Follow-up Agent` workflow on its own schedule trigger (or run it manually for a demo).

---

## Known limitations / future work

- **Authentication is demo-only.** The Sign In / Create Account flow on the landing page stores users in the browser's `localStorage` (`civicfix_users`, `civicfix_session`) purely for demonstration purposes. It is not connected to any backend and should not be treated as real auth. Wiring this up to Airtable-backed accounts is flagged as future work.
- Model availability on Groq's free tier can change; if the agent stops responding, check that the model selected in each **Groq Chat Model** node still exists.

---

## Impact

CivicFix is designed around the scale of India's civic grievance system — CPGRAMS alone has processed **11.2M+** grievances between 2019 and 2024 — with a simple goal: make it as easy as sending a text message to report a problem, and make sure it doesn't get forgotten once it's filed.
