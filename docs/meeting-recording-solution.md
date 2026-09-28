# Meeting Recording → Transcript → Summary → KB

**Status:** Ideation / decision proposal
**Date:** 2026-09-28
**Goal:** Every relevant Yeaz meeting is recorded (with consent), transcribed, and summarized, and the summary lands in our knowledge base (KB) with no manual copy-paste. Raw recordings are offloaded and retained according to policy.

---

## 1. Assumptions to confirm

These drive the make/buy decision. Please correct any that are wrong.

| # | Assumption | Why it matters |
|---|-----------|----------------|
| A1 | Our main meeting platform is **Microsoft Teams** (plus some Google Meet/Zoom with externals) | Native options (Teams Premium / Copilot) may already cover 70% of the need |
| A2 | The KB is **one of** Notion, Confluence, or SharePoint/OneDrive | Sets the integration target. Some vendors push natively to Notion/Confluence, others don't |
| A3 | We're a German/EU company, and meetings are in **German and English** | GDPR, §201 StGB (recording the spoken word), works council (BetrVG §87(1) Nr. 6), and German-language transcription quality |
| A4 | Roughly 30–100 employees record meetings | Per-seat SaaS cost vs. usage-based build cost |
| A5 | We don't record HR, legal, or performance conversations | Scope and risk |

---

## 2. What the solution has to do

1. **Capture**: record internal and external calls (Teams, Meet, Zoom) and ideally in-person meetings too (mobile or desktop audio).
2. **Transcribe**: accurate German/English transcription, speaker names (diarization), timestamps.
3. **Summarize**: in a Yeaz template: TL;DR, decisions, action items (owner + due date), open questions, and tags (project, team, customer/partner).
4. **Offload to the KB**: the summary and a transcript link are filed automatically in the right KB space/page, searchable and permissioned.
5. **File lifecycle**: audio/video goes to cheap storage (SharePoint/Blob/S3) and is deleted after N days. Transcript and summary are kept.
6. **Governance**: consent notice, opt-out, EU data residency, DPA (AVV), retention, access control that mirrors meeting attendees.
7. **Nice to have**: action items pushed to our task tool, "ask the KB" Q&A across all meetings, a CRM/Klaviyo link for partner calls.

---

## 3. Target architecture (applies to make and buy)

```
 ┌───────────────┐   ┌───────────────┐   ┌──────────────────┐   ┌──────────────┐
 │   CAPTURE     │──▶│  TRANSCRIBE   │──▶│ SUMMARIZE/ENRICH │──▶│   OFFLOAD    │
 │ Teams/Meet/   │   │ STT + speaker │   │ LLM + Yeaz       │   │ KB page      │
 │ Zoom, bot or  │   │ diarization   │   │ template, tags,  │   │ + raw file → │
 │ botless app   │   │ (DE/EN)       │   │ action items     │   │ storage/TTL  │
 └───────────────┘   └───────────────┘   └──────────────────┘   └──────┬───────┘
                                                                       │
                                     ┌─────────────────────────────────┴───┐
                                     │ Tasks tool · Search/Q&A over KB     │
                                     └─────────────────────────────────────┘
```

The pipeline has four layers. **Capture and transcription are commodities.** **Summarization and KB offloading are Yeaz-specific** because they depend on our templates, taxonomy, and permissions. That split is the core of the recommendation below.

---

## 4. Options

### Option B1: Buy native (the platform we already pay for)

| Product | What you get | Gaps |
|---|---|---|
| **Microsoft Teams Premium / Microsoft 365 Copilot** (Intelligent Recap) | Recording and transcript stored in OneDrive/SharePoint automatically, AI recap with tasks, EU data boundary, no new vendor | Teams meetings only. Recap lives in Teams, not in our KB structure. Copilot costs about €30/user/month. Limited template control |
| **Google Meet + Gemini "Take notes for me"** | Notes Doc in Drive, auto-shared with attendees | Meet only. Docs land in Drive, not the KB |
| **Zoom AI Companion** | Included with paid Zoom. Summary and next steps | Zoom only. Export to KB is manual or via Zapier |

**Good fit if** A1 holds and the KB is SharePoint. The "offload to KB" step is then mostly done already.

### Option B2: Buy a dedicated AI notetaker (cross-platform)

| Vendor | Notes relevant to Yeaz |
|---|---|
| **jamie** (German, botless) | Records locally on desktop and mobile (works in person too), no bot joins the call, EU-hosted, strong German. Integrations: Notion, Confluence, others via Zapier/webhooks |
| **tl;dv** (EU/Germany) | Bot-based, EU hosting, strong on Meet/Zoom/Teams, native Notion/HubSpot/Slack integrations, templates |
| **Fireflies.ai / Otter / Fathom / Read.ai** | Mature, many integrations, and APIs/webhooks (Fireflies has the richest). US-hosted by default, so check EU residency and the DPA |
| **Granola** | Botless desktop notetaker. The user edits the notes and AI enhances them. Good UX, but lighter on org-wide governance |
| **Notion AI Meeting Notes** | Only if the KB is Notion. Notes are created in the KB directly, which is the shortest path to "offloaded" |

Typical cost is about €10–30 per user per month for business tiers with SSO, admin controls, and integrations (indicative, verify current pricing).

**Good fit if** we use several meeting platforms or need in-person capture, and the vendor has a native connector to our KB.

### Option M1: Make it end to end

Build our own pipeline.

- **Capture:** Microsoft Graph API (Teams `callRecording` / `callTranscript` change notifications), Zoom cloud-recording webhooks, Google Meet REST API. Or buy only the capture layer from **Recall.ai**, a meeting-bot API for all platforms. That's the only really hard part to build.
- **Transcribe:** Azure AI Speech (EU region), Deepgram, AssemblyAI (EU endpoint), or self-hosted Whisper large-v3 + pyannote for diarization. Or use the platform's own transcript for free.
- **Summarize:** Claude (Anthropic API) with a Yeaz prompt template and structured JSON output (decisions, action items, tags). Long-context models handle 2h+ transcripts in a single pass.
- **Offload:** a small service (Azure Function / AWS Lambda / n8n) that writes to the Notion, Confluence, or Graph API, uploads the raw file to Blob/S3 with a lifecycle rule, and creates tasks.
- **Q&A:** connect the KB to Claude (via a connector/MCP) so people can ask "what did we decide with supplier X in Q3?"

Indicative running cost at about 400 meeting hours/month: STT ~€0.25–0.50/h, LLM ~€0.05–0.30 per meeting, Recall.ai ~$0.50–1/h if used. That's **low hundreds of euros per month**, far below per-seat SaaS. The real cost is **build (4–8 engineer-weeks) plus ongoing ownership**: API changes, bot reliability, security reviews.

### Option H1: Hybrid (buy capture, make the KB layer) ⭐ recommended

```
Teams Premium / jamie / tl;dv / Fireflies   ──webhook/API──▶   "Yeaz Meeting Router"   ──▶  KB
   (capture + transcript + basic summary)                     (Claude + Yeaz template,     (+ storage TTL,
                                                               tagging, routing, ACLs)       tasks, Q&A)
```

- **Buy** capture and transcription. Choose one vendor based on A1/A3: Teams Premium if we're Teams-only, jamie if we're German-first with in-person meetings, tl;dv/Fireflies if we're multi-platform and bot-OK.
- **Make** a thin router (about 1–3 engineer-weeks, or a low-code n8n/Power Automate flow) that:
  1. receives the "transcript ready" webhook,
  2. re-summarizes with **our** template and taxonomy (project/team/customer tags, decision log format),
  3. files the page in the **right KB space**, with permissions matching the attendees,
  4. moves the raw recording to our storage and deletes it at the vendor after X days,
  5. pushes action items to the task tool.
- **Why:** we avoid building the fragile part (bots and recording) and own the part that makes the KB valuable (structure, consistency, portability). If we switch notetaker vendors later, only the input adapter changes. The KB format and history stay the same.

---

## 5. Make vs. buy scorecard

Scores are 1 (poor) to 5 (great), and weights add to 100%.

| Criterion | Weight | B1 Native | B2 Notetaker | M1 Make | **H1 Hybrid** |
|---|---:|:-:|:-:|:-:|:-:|
| Time to value | 20% | 5 | 5 | 2 | 4 |
| KB integration and structure fit | 20% | 2–4* | 3 | 5 | **5** |
| GDPR / EU residency / works council | 15% | 5 | 3–5** | 5 | 4–5 |
| German transcription quality | 10% | 4 | 4–5 | 4 | 4–5 |
| Cross-platform + in-person | 10% | 2 | 4–5 | 3 | 4–5 |
| Total cost (3 yr, ~50 users) | 15% | 2 | 3 | 4 | 3–4 |
| Maintenance burden | 10% | 5 | 5 | 1 | 4 |
| **Weighted (midpoint)** | | **~3.6** | **~3.9** | **~3.5** | **~4.3** |

\* 4 if the KB is SharePoint, 2 if Notion/Confluence.
\** 5 for EU vendors (jamie, tl;dv), 3 for US-default vendors without an EU region.

---

## 6. Compliance checklist (Germany/EU): do this before any pilot

- [ ] **Consent:** under §201 StGB, recording the non-public spoken word without consent is a criminal offense. Use an automatic recording notice in the invite, an in-call banner, and an easy opt-out. External participants must be informed as well.
- [ ] **Works council:** recording and AI tools that could monitor behavior or performance need works council co-determination (BetrVG §87(1) Nr. 6). Agree a Betriebsvereinbarung covering purpose limitation, no performance evaluation, and access rules.
- [ ] **GDPR:** a DPA (AVV) with every vendor, EU data residency, sub-processor list, an entry in the processing register (VVT), and a DPIA (DSFA). A DPIA is likely required for systematic recording.
- [ ] **No training on our data:** contractually excluded for both the notetaker vendor and the LLM provider.
- [ ] **Retention:** raw audio/video 30–90 days, transcripts 12–24 months, summaries/decisions per KB policy. Enforce with automatic deletion.
- [ ] **Scope exclusions:** no recording of HR, medical, legal, or performance meetings. Default to OFF for 1:1s.
- [ ] **Access:** a KB page inherits the attendee list, and sensitive tags restrict visibility.

---

## 7. Rollout plan

| Phase | Duration | Content |
|---|---|---|
| **0. Decide** | 1 week | Confirm A1–A5. Brief DPO and works council. Shortlist two vendors |
| **1. Pilot** | 3–4 weeks | 10–15 users across 2 teams. Two vendors head to head, e.g. Teams Premium vs. jamie or tl;dv. Manual KB filing to validate the template |
| **2. Router MVP** | 2–3 weeks (parallel) | Webhook → Claude summary with the Yeaz template → KB page + storage TTL. Low-code first (n8n / Power Automate), code if needed |
| **3. Rollout** | 4 weeks | Company-wide. Betriebsvereinbarung signed. Training and a "how we record" one-pager |
| **4. Extend** | Ongoing | Action items → task tool, KB Q&A via Claude, partner calls → CRM |

**Pilot success metrics**
- ≥80% of recorded meetings reach the KB automatically within 15 minutes
- German transcript word error rate is acceptable (spot-check 10 meetings) and speaker attribution is ≥90% correct
- Users rate summary usefulness ≥4/5 and save ≥10 minutes per meeting
- Zero consent or compliance incidents

---

## 8. Yeaz summary template (draft, used by the router)

```markdown
# {Meeting title} — {date}
**Attendees:** … | **Type:** internal / partner / customer | **Tags:** #project #team
## TL;DR (3 bullets)
## Decisions
- [D1] … (decided by …)
## Action items
- [ ] … — @owner — due YYYY-MM-DD
## Open questions / risks
## Key numbers mentioned
## Links
- Transcript · Recording (expires YYYY-MM-DD)
```

---

## 9. Recommendation and next steps

1. **Go with H1 (Hybrid):** buy capture and transcription, and build the thin KB router ourselves.
2. **Vendor choice depends on A1:**
   - Teams-only and SharePoint KB: **Teams Premium/Copilot** + router (router may be optional)
   - German-first, many in-person meetings: **jamie** + router
   - Multi-platform, Notion KB: **tl;dv** or **Notion AI Meeting Notes** + router
3. **This week:** confirm A1–A5, send the DPO/works council brief, book a pilot with the top two vendors, and scaffold the router (webhook → Claude → KB).

### Open questions for Marcel
- Which KB (Notion, Confluence, SharePoint, other) and which meeting platform(s)?
- Headcount recording meetings, and share of external/partner calls?
- Is there engineering capacity for about 2 weeks of router work, or should it be low-code only?
