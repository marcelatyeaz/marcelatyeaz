# Meeting Recording → Transcript → Summary → Second Brain

**Status:** Ideation / decision proposal (v2, updated with the Yeaz landscape)
**Date:** 2026-09-28
**Goal:** Every relevant Yeaz meeting, online (Teams) or offline (in person, recorded on a laptop or phone), is transcribed and summarized. The summary lands as a Markdown note in our GitHub-backed Obsidian second brain, and action items land in Plane. Nobody has to copy-paste anything.

---

## 1. The Yeaz landscape (confirmed)

| Area | What we have | What it means for this design |
|---|---|---|
| Knowledge base | `.md` second brain in **GitHub**, synced locally, edited in **Obsidian** | "Offloading to the KB" means **a git commit** of Markdown with YAML frontmatter and `[[wikilinks]]`. No SaaS notetaker writes to Git natively, so we'll build this last step ourselves whichever option we pick |
| Tickets | **Plane** | Action items → Plane issues through the Plane REST API, ideally into **Intake** (triage) so they don't clutter projects |
| Online meetings | **Microsoft Teams** | Teams' own recording and transcription gives capture and transcript for free. We pull them via **Microsoft Graph** |
| Offline meetings | Laptop and mobile | We need a recorder app or a "drop the audio file here" flow, plus our own transcription |
| Languages | **Dutch and English**, often mixed in one meeting | Transcription engine must handle `nl-NL`, `en-US` and switching between them. Benchmark this, it's the biggest quality risk |
| Platform | **Microsoft/Azure** for infra and apps, **no Power Automate** | Use Azure Functions (or Logic Apps / Durable Functions), Blob Storage, Key Vault, Entra ID, Azure AI Speech |
| Own development | **OutSystems** | Good fit for a mobile recorder app and a review/approve UI, if we want those |

---

## 2. Requirements

1. **Capture**
   - Online: Teams meetings, with recording and/or transcription switched on by policy or by the organizer.
   - Offline: one tap on a phone, or one click on a laptop, with no internet needed while recording.
2. **Transcribe**: Dutch, English and mixed. Speaker separation (diarization), timestamps.
3. **Summarize**: Yeaz template covering TL;DR, decisions, action items (owner + due date), open questions and tags. Written in the meeting's main language, or always in one agreed language.
4. **Offload to the second brain**: `Meetings/YYYY/MM/YYYY-MM-DD-<slug>.md` with frontmatter, and `[[wikilinks]]` to existing people and project notes. Committed to GitHub, then picked up by everyone's local sync.
5. **Offload action items**: Plane issues that link back to the note. The Plane ID is written back into the note.
6. **File lifecycle**: audio and video **never go into git**. They go to Blob or OneDrive with automatic deletion after N days. Text lives in git.
7. **Governance**: AVG/GDPR, consent and transparency, works council (OR) consent, EU processing, and exclusion of sensitive meetings.

---

## 3. Target architecture

```
 ONLINE                                  OFFLINE
 ┌───────────────────────┐              ┌──────────────────────────────────────────┐
 │ Teams meeting         │              │ Laptop / phone recorder                  │
 │ recording+transcript  │              │  a) OutSystems mobile app  → Blob upload │
 │ (Teams-native)        │              │  b) any recorder → OneDrive "Meeting     │
 └──────────┬────────────┘              │     Inbox" folder                        │
            │ Graph change notification │  c) bought app (jamie/Plaud) → webhook   │
            │ (transcript available)    └───────────────────┬──────────────────────┘
            ▼                                               ▼
 ┌─────────────────────────────────────────────────────────────────────────────────┐
 │                YEAZ MEETING ROUTER  (Azure Functions, Durable orchestration)    │
 │  1. Fetch transcript (Graph .vtt)       │  1. Azure AI Speech batch transcription│
 │                                         │     (nl-NL/en-US, diarization)         │
 │  2. Normalize → one transcript format (speakers, timestamps, language)          │
 │  3. Summarize with Claude + Yeaz template (JSON out). Context: list of existing │
 │     people/project notes from the repo so it can write correct [[wikilinks]]    │
 │  4. Render Markdown note + frontmatter                                          │
 │  5. Commit to GitHub second-brain repo (GitHub App)  ── or open a PR for review │
 │  6. Create Plane issues (Intake) → write issue IDs back into the note           │
 │  7. Move audio to Blob "raw" container (lifecycle rule: delete after 30–90 days)│
 │  8. Notify organizer in Teams (link to note + Plane issues)                     │
 └─────────────────────────────────────────────────────────────────────────────────┘
            │                                   │
            ▼                                   ▼
   GitHub second brain  ──sync──▶  Obsidian     Plane (Intake → projects)
```

**Main point:** our KB is Git plus Markdown, so the most valuable part of the pipeline (steps 2–8) has to be built. What's left to decide is **capture and transcription**, and that's where the make-or-buy choice sits.

---

## 4. Make or buy, layer by layer

### 4.1 Online capture and transcript (Teams)

| Option | Verdict |
|---|---|
| **Teams-native recording and transcription + Graph API** (make the fetch) | ✅ **Recommended.** Already in our M365 licenses and supports Dutch. Graph `onlineMeeting` → `transcripts` / `recordings` with change notifications. No bot, no new vendor. *Check:* the Graph transcript/recording APIs require specific app permissions and application access policies, and some calls are metered or need licensing. Confirm this for our tenant |
| Microsoft 365 Copilot / Teams Premium (intelligent recap) | ⚠️ Optional nice-to-have for users, not needed for the pipeline. Output stays in Teams and doesn't reach Git. Costs about €30/user/month for Copilot |
| Third-party bot (tl;dv, Fireflies, Otter, Recall.ai) | ❌ Not needed. It duplicates what Teams already does and adds an extra processor under AVG |

### 4.2 Offline capture (laptop and mobile)

| Option | Make/Buy | Pros | Cons |
|---|---|---|---|
| **A. "Meeting Inbox" folder**: record with any app (Windows Sound Recorder, iOS Voice Memos, Android recorder) and save or share to a OneDrive/SharePoint folder. Graph change notification triggers the router | **Make (tiny)** | Days of work, zero new apps, works today | Manual step (share the file). Title and attendees missing unless the user renames the file or picks a calendar event |
| **B. OutSystems mobile and desktop recorder app** | **Make** | Our own stack. Pick the calendar event (Graph) for title and attendees, consent checkbox, offline recording with upload later, tags and project picker at the start. The same app can host a **review/approve screen** before commit | 3–6 weeks of OutSystems work. Needs an audio-capture plugin/Forge component and background-recording handling on iOS |
| **C. Buy a botless recorder app** (e.g. **jamie**: desktop and mobile, EU-hosted, multilingual incl. Dutch) | **Buy** | Polished UX, also works for Teams, good summaries out of the box | Per-seat cost (about €15–30/user/month, verify). Still needs webhook/export → router to reach Git and Plane. Another processor to manage under AVG |
| **D. Buy hardware** (e.g. **Plaud NotePin/Note**) | **Buy** | Great for field and store visits and long in-person sessions, one button | Device plus subscription. Export through their app, integration is weaker. Check where data is stored |

**Recommendation:** start with **A** in the pilot (near-zero cost). Build **B** in OutSystems once we know what people actually need (calendar-linked, consent, tags). Consider **C** only if the UX of A/B isn't good enough.

### 4.3 Transcription engine (offline audio, and fallback for Teams)

| Engine | Notes |
|---|---|
| **Azure AI Speech – batch transcription** | ✅ Default. Azure-native, EU region, `nl-NL` and `en-US`, diarization, language identification. *Risk:* switching between Dutch and English mid-sentence. Test with real Yeaz recordings |
| Whisper (Azure OpenAI or self-hosted `large-v3`) | Strong on mixed-language speech. Use it as a benchmark and fallback. Diarization needs extra work (e.g. pyannote) |
| Deepgram / AssemblyAI / ElevenLabs Scribe | Buy-API alternatives with strong multilingual support. Only if Azure fails the benchmark, and check EU endpoints and the DPA |

Run a **benchmark** in phase 1: 10 real recordings (Dutch, English, mixed, a noisy room). Score word error rate on a few minutes of each, plus speaker accuracy.

### 4.4 Summarization

| Option | Notes |
|---|---|
| **Claude** via **Microsoft Foundry** (Azure) or the Anthropic API | ✅ Long context handles 2h+ transcripts in one pass, strong Dutch/English, reliable structured JSON output. Foundry keeps billing and governance in Azure (check region and model availability). In both cases, contractually no training on our data |
| Azure OpenAI | Also viable in the same Azure setup. Pick based on a quality test on our template |

Cost is roughly €0.05–0.30 per meeting, which is negligible.

### 4.5 Second-brain and Plane integration

**Make. There's no off-the-shelf product for this.** It's the core of the router:
- **GitHub App** with `contents:write` on the second-brain repo only. The token lives in Key Vault.
- Two modes, set per team:
  - **Direct commit** to `main` (fast, `status: draft` in frontmatter until someone reviews it)
  - **Pull request** per meeting (the organizer reviews and merges, which fits a Git KB well)
- Obsidian users receive notes through their existing sync (e.g. the obsidian-git plugin or `git pull`).
- **Plane**: `POST /api/v1/workspaces/{slug}/projects/{project_id}/intake-issues/` (or plain issues). Map owners by email to Plane members, fill in the due date, and put a backlink to the note in the description.

---

## 5. Scorecard (updated)

Scores are 1 (poor) to 5 (great).

| Criterion | Weight | Buy all (Copilot / jamie) | **Make on Azure (Teams-native + router)** | Make + bought offline app |
|---|---:|:-:|:-:|:-:|
| Lands in Git/Obsidian + Plane | 25% | 1 (needs router anyway) | **5** | 5 |
| Dutch/English quality | 15% | 4–5 | 4 (benchmark) | 4–5 |
| Offline capture UX | 15% | 5 | 3 (A) → 4 (B) | 5 |
| AVG / EU / OR | 15% | 3–4 | **5** (stays in our tenant) | 4 |
| Cost (3 yr, ~50 users) | 15% | 2 | **5** | 3 |
| Time to value | 10% | 4 | 3 | 3 |
| Maintenance | 5% | 5 | 3 | 3 |
| **Weighted (midpoint)** | | **~3.1** | **~4.3** | **~4.2** |

**Conclusion:** mostly **make**, on the Azure/OutSystems stack we already run. The only real buy decision left is the offline recorder UX, and we can defer it.

---

## 6. Note format in the second brain

Path: `Meetings/2026/10/2026-10-02-supplier-review-acme.md`

```markdown
---
type: meeting
date: 2026-10-02
start: "10:00"
duration_min: 45
source: teams            # teams | offline-inbox | offline-app
language: nl             # nl | en | mixed
title: Supplier review Acme
attendees: ["[[Jan de Vries]]", "[[Marcel]]", "[[Sara Jansen]]"]
projects: ["[[Project Packaging 2027]]"]
tags: [meeting, supplier, packaging]
status: draft            # draft | reviewed
plane_issues: [YEAZ-412, YEAZ-413]
transcript: "[[2026-10-02-supplier-review-acme.transcript]]"
recording: "https://…blob…/raw/…"   # expires 2026-12-31
---

# Supplier review Acme — 2026-10-02

## TL;DR
- …

## Decisions
- [D1] … (decided by [[Marcel]])

## Action items
- [ ] … — [[Sara Jansen]] — due 2026-10-09 — Plane YEAZ-412

## Open questions / risks
- …

## Key numbers
- …
```

- The **full transcript** goes next to the note as `*.transcript.md`, or in a separate private repo or folder if some meetings are sensitive. Text is small and git handles it fine.
- **Audio/video: never in git.** Blob with a lifecycle rule. The link expires when the file is deleted.
- The router gives Claude the list of existing `People/` and `Projects/` note names so the wikilinks resolve. Unknown people become plain text rather than new empty notes.

---

## 7. Compliance checklist (Netherlands / EU)

- [ ] **Transparency and consent (AVG):** recording involves personal data. Define the lawful basis (usually legitimate interest, and consent for external parties), announce it in the invite and at the start of the meeting, and give an easy opt-out. The Teams recording banner covers online meetings. For offline recordings, the recorder app or the organizer must say it out loud.
- [ ] **Works council (OR):** a system that can monitor employees' behaviour or performance needs **OR consent under WOR art. 27(1)(l)**. Agree purpose limitation (no performance evaluation), access rules and retention.
- [ ] **DPIA:** very likely required for systematic recording and transcription. Register it in the processing register.
- [ ] **Processors:** DPAs (verwerkersovereenkomsten) with Microsoft (already in place), the LLM provider, and any bought app. EU processing, no training on our data.
- [ ] **Retention:** audio 30–90 days (automatic deletion), transcripts 12–24 months, summaries and decisions per KB policy.
- [ ] **Scope exclusions:** no HR, medical, legal or performance conversations. 1:1s are off by default. A `confidential` tag routes the note to a restricted repo or folder.
- [ ] **Access:** second-brain repo permissions are all-or-nothing per repo. Decide whether a single repo is acceptable, or add a `second-brain-restricted` repo for sensitive meetings.

---

## 8. Build plan and effort

| Phase | Duration | Deliverable |
|---|---|---|
| **0. Decide** | 1 week | Brief the privacy officer and OR, DPIA started, Entra app registration + Graph permissions approved, GitHub App created, Plane API token |
| **1. Pilot (offline via Inbox + Teams via Graph)** | 3–4 weeks | Azure Functions router: Graph transcript fetch, OneDrive Inbox trigger, Azure Speech batch, Claude summary, commit to the repo (PR mode), Plane Intake issues. Transcription benchmark (Azure vs. Whisper). 10–15 pilot users |
| **2. Harden** | 2 weeks | Direct-commit mode, retention rules, restricted-repo routing, Teams notification card, monitoring (App Insights), retries/dead-letter |
| **3. OutSystems recorder + review app** | 3–6 weeks (optional, parallel) | Calendar-linked recording, consent capture, offline upload, "approve and publish" screen |
| **4. Extend** | Ongoing | "Ask the second brain" (Claude over the repo), weekly decision digest, cross-meeting action-item follow-up |

**Rough effort:** router MVP about 3–4 engineer-weeks (Azure/TypeScript or C#). OutSystems app 3–6 weeks.
**Rough run cost at about 400 meeting hours/month:** Azure Speech about €150–300, LLM about €20–100, Functions and storage under €50. That's **roughly €250–450/month in total**, versus €750–1,500/month for per-seat SaaS for 50 users, which would still need the router.

**Pilot success metrics**
- ≥90% of recorded meetings produce a note in the repo within 15 minutes (Teams) or 30 minutes (offline)
- Transcription quality is acceptable for Dutch, English and mixed speech (benchmark), with speaker attribution ≥90% correct
- ≥80% of generated action items are accepted in Plane Intake without major edits
- Users rate summaries ≥4/5
- Zero privacy incidents, and OR agreement in place before rollout

---

## 9. Recommendation

1. **Make**, on the stack we already have: Teams-native transcription via Graph, Azure AI Speech for offline audio, Claude for summaries, and an Azure Functions "Meeting Router" that commits Markdown to the second-brain repo and creates Plane Intake issues.
2. **Offline capture:** start with the OneDrive "Meeting Inbox" drop folder, then build a small **OutSystems recorder app** (calendar-linked, consent, offline upload, review screen). Keep **jamie/Plaud** as a buy fallback if the UX falls short.
3. **Don't buy** a cross-platform notetaker or Copilot for this use case. They don't reach Git or Plane, and they add cost and another processor under AVG.
4. **Next steps this week:** OR and privacy officer briefing, Graph permission request, collect 10 sample recordings for the Dutch/English benchmark, scaffold the router repo.

### Open questions
- One second-brain repo for everything, or a separate restricted repo for confidential meetings?
- Summary language: always English, always Dutch, or match the meeting?
- Commit mode: direct to `main` with `status: draft`, or a PR per meeting?
- Should Teams recording or transcription be on by default for everyone, or only for certain meeting types or teams?
- Router language: TypeScript or C# (whatever the Azure team prefers)?
