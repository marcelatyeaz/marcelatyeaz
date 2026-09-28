# Meeting Recording → Transcript → Summary → Second Brain

**Status:** Decision proposal (v3, with Yeaz decisions and team size)
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
| Own development | **OutSystems** | Could host a mobile recorder app, but at our size buying is cheaper (see 4.2) |
| Team size | **25 users, of whom about 10 record meetings actively** | Small volume, so running costs are tiny and build effort dominates the total cost. Keep the build minimal |

### Decisions taken (2026-09-28)

| Topic | Decision |
|---|---|
| Summary language | **Always English.** Dutch meetings are summarized in English. The transcript stays in the original language. Dutch product and supplier terms and direct quotes are kept as spoken |
| Repository | **One second-brain repo** for everything. Consequence: confidential meetings (HR, legal, performance) are **not recorded at all**, since the repo has no finer-grained access control |
| Review flow | **One pull request per meeting**, reviewed and merged by the organizer |

---

## 2. Requirements

1. **Capture**
   - Online: Teams meetings, with recording and/or transcription switched on by policy or by the organizer.
   - Offline: one tap on a phone, or one click on a laptop, with no internet needed while recording.
2. **Transcribe**: Dutch, English and mixed. Speaker separation (diarization), timestamps.
3. **Summarize**: Yeaz template covering TL;DR, decisions, action items (owner + due date), open questions and tags. Written in the meeting's main language, or always in one agreed language.
4. **Offload to the second brain**: `Meetings/YYYY/MM/YYYY-MM-DD-<slug>.md` with frontmatter, and `[[wikilinks]]` to existing people and project notes. Opened as **a pull request per meeting**. After the organizer merges it, everyone's local sync picks it up.
5. **Offload action items**: Plane issues created **when the PR is merged**, so the organizer's edits count and rejected summaries create no tickets. Each issue links back to the note, and the Plane IDs are written back into the note.
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
 │  3. Summarize in English with Claude + Yeaz template (JSON out). Context: list of existing │
 │     people/project notes from the repo so it can write correct [[wikilinks]]    │
 │  4. Render Markdown note + frontmatter                                          │
 │  5. Open a PR in the second-brain repo (GitHub App), organizer as reviewer      │
 │  6. Move audio to Blob "raw" container (lifecycle rule: delete after 30–90 days)│
 │  7. Notify organizer in Teams: "Your meeting notes are ready to review" + link  │
 │  ── on PR merged (GitHub webhook) ──                                           │
 │  8. Parse action items from the merged note → Plane Intake issues               │
 │  9. Bot commit writes Plane IDs back into the note's frontmatter                │
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

**Recommendation (with 10 active users):** start with **A** in the pilot (near-zero cost). If people find the drop-folder step too clunky, **buy C for only the few people who record offline** instead of building B:

| | B. Build in OutSystems | C. Buy jamie (about 5 offline recorders) |
|---|---|---|
| One-off | 3–6 dev-weeks ≈ €10–25k | €0, plus a small router adapter (webhook/export) |
| Yearly | Maintenance (OS updates, iOS background audio) | 5 × about €20/month ≈ €1.2k/year (verify pricing) |
| Break-even | — | Roughly 8–20 years, so **buy wins clearly at our size** |

Only build B if the OutSystems team has idle capacity or we want to reuse its components elsewhere.

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

**Make. There's no off-the-shelf product for this.** It's the core of the router.

**PR-per-meeting flow (decided):**

1. The router creates a branch `meeting/2026-10-02-supplier-review-acme` containing the note and `*.transcript.md`.
2. It opens a PR titled `Meeting: Supplier review Acme (2026-10-02)`. The **organizer** is assigned as reviewer. The PR body shows the TL;DR, the proposed action items and a transcription confidence score.
3. The organizer fixes names, action items and wording **directly in the PR** (GitHub web editor, or checking out the branch in Obsidian), then merges it. Merge = `status: reviewed`.
4. **On merge**, a GitHub webhook calls the router. It creates the Plane Intake issues from the *merged* action items and adds a bot commit with the `plane_issues` IDs.
5. **Stale PRs:** a Teams reminder after 3 working days. After 14 days the PR is auto-closed with the label `not-reviewed`, and the branch is kept so it can be reopened.
6. **Closed without merge** means no Plane issues, and the audio is deleted on its normal schedule.

**Setup:**
- A **GitHub App** with `contents:write` and `pull_requests:write` on the second-brain repo only. Its private key lives in Key Vault. Branch protection on `main` requires one review. The App may push its follow-up commit (bypass rule for the App only).
- A **user mapping table** (Entra UPN ↔ GitHub username ↔ Plane member ID) for about 25 people. A YAML file in the repo (`.meta/people.yml`) is enough. If the organizer has no GitHub account, the PR goes to a fallback KB maintainer.
- A `CODEOWNERS` entry isn't needed, because the reviewer is set per PR by the router.
- Obsidian users receive merged notes through their existing sync (obsidian-git plugin or `git pull`).
- **Plane**: `POST /api/v1/workspaces/{slug}/projects/{project_id}/intake-issues/`. The project is chosen from the note's `projects` tag, with a default project as fallback. Owners come from the mapping table. The due date and a backlink to the note go into the issue.

---

## 5. Scorecard (updated)

Scores are 1 (poor) to 5 (great).

| Criterion | Weight | Buy all (Copilot / jamie) | **Make on Azure (Teams-native + router)** | Make + bought offline app |
|---|---:|:-:|:-:|:-:|
| Lands in Git/Obsidian + Plane | 25% | 1 (needs router anyway) | **5** | 5 |
| Dutch/English quality | 15% | 4–5 | 4 (benchmark) | 4–5 |
| Offline capture UX | 15% | 5 | 3 (A) → 4 (B) | 5 |
| AVG / EU / OR | 15% | 3–4 | **5** (stays in our tenant) | 4 |
| Cost (3 yr, 10 active users) | 15% | 3 | 4 (build effort dominates) | 4 |
| Time to value | 10% | 4 | 3 | 3 |
| Maintenance | 5% | 5 | 3 | 3 |
| **Weighted (midpoint)** | | **~3.2** | **~4.2** | **~4.3** |

**Conclusion:** **make the router** on Azure, because nothing we could buy reaches Git or Plane. **Use Teams-native** for online meetings. **Buy** offline capture (jamie) for the few people who need it, if the drop folder isn't good enough. At 10 active users, building our own recorder app doesn't pay off.

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
status: reviewed         # set when the organizer merges the PR
summary_language: en     # always English (decision 2026-09-28)
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

- The **summary is always in English**. The Claude prompt says: translate Dutch content, keep Dutch proper nouns, product names and direct quotes (with an English gloss), and never invent content that's missing from the transcript.
- The **full transcript** goes next to the note as `*.transcript.md`, **in the original language** (Dutch/English), in the same PR. Text is small and git handles it fine.
- **Audio/video: never in git.** Blob with a lifecycle rule. The link expires when the file is deleted.
- The router gives Claude the list of existing `People/` and `Projects/` note names so the wikilinks resolve. Unknown people become plain text rather than new empty notes.

---

## 7. Compliance checklist (Netherlands / EU)

- [ ] **Transparency and consent (AVG):** recording involves personal data. Define the lawful basis (usually legitimate interest, and consent for external parties), announce it in the invite and at the start of the meeting, and give an easy opt-out. The Teams recording banner covers online meetings. For offline recordings, the recorder app or the organizer must say it out loud.
- [ ] **Works council (OR):** a system that can monitor employees' behaviour or performance needs **OR consent under WOR art. 27(1)(l)**. Agree purpose limitation (no performance evaluation), access rules and retention.
- [ ] **DPIA:** very likely required for systematic recording and transcription. Register it in the processing register.
- [ ] **Processors:** DPAs (verwerkersovereenkomsten) with Microsoft (already in place), the LLM provider, and any bought app. EU processing, no training on our data.
- [ ] **Retention:** audio 30–90 days (automatic deletion), transcripts 12–24 months, summaries and decisions per KB policy.
- [ ] **Scope exclusions:** no HR, medical, legal or performance conversations, and 1:1s are off by default. Because we use **one repo**, these meetings are simply **not recorded**. There's no restricted folder.
- [ ] **Access:** everyone with repo access can read every meeting note and transcript. Make this explicit in the OR agreement and the privacy notice. Review repo membership quarterly.
- [ ] **Right to erasure:** git history keeps deleted content. Define a procedure (history rewrite with `git filter-repo` plus a forced re-sync) for the rare erasure request, or keep transcripts out of git and in Blob with a link if we'd rather avoid that.

---

## 8. Build plan and effort

| Phase | Duration | Deliverable |
|---|---|---|
| **0. Decide** | 1 week | Brief the privacy officer and OR, DPIA started, Entra app registration + Graph permissions approved, GitHub App created, Plane API token, `.meta/people.yml` mapping |
| **1. Pilot (Teams via Graph + offline via Inbox)** | 3 weeks | Azure Functions router: Graph transcript fetch, OneDrive Inbox trigger, Azure Speech batch, English Claude summary, PR per meeting, Plane Intake on merge. Transcription benchmark (Azure vs. Whisper). **All 10 active users** |
| **2. Harden** | 1–2 weeks | Stale-PR reminders and auto-close, retention rules, Teams notification card, monitoring (App Insights), retries/dead-letter |
| **3. Offline UX decision** | After 4–6 weeks of use | Keep the drop folder, or buy jamie for the offline recorders (+ webhook adapter, about 2–3 days) |
| **4. Extend** | Ongoing | "Ask the second brain" (Claude over the repo), weekly decision digest, action-item follow-up across meetings |

**Rough effort:** router MVP about **3 engineer-weeks** (Azure/TypeScript or C#), plus 1–2 weeks of hardening. No OutSystems build needed.

**Rough volume and run cost (10 active users):** about 10 × 6–8 recorded meetings/week × about 45 min ≈ **200–250 meeting hours/month**. Shared meetings are counted once, so realistically **about 100–150 unique hours/month**. About 70% of that is Teams, where the transcript is free.

| Item | €/month |
|---|---:|
| Azure AI Speech batch (about 30–50 h offline audio) | about €10–30 |
| Claude summaries (about 150 meetings × €0.05–0.30) | about €10–45 |
| Azure Functions, Blob, Key Vault, App Insights | about €10–25 |
| **Total run cost** | **about €30–100** |
| *Optional:* jamie for about 5 offline recorders | about €100 |
| *Compare:* Copilot for 10 users (still doesn't reach Git or Plane) | about €300 |

**3-year total cost of ownership (indicative):** build about €12–18k (about 4–5 weeks) + run about €1–3.5k + maintenance about 2 days/quarter. The bought alternatives cost about €11k (Copilot, 10 users) and **still need the router**. So the router is a must-build, and the buy decision only concerns offline capture.

**Pilot success metrics**
- ≥90% of recorded meetings produce a note in the repo within 15 minutes (Teams) or 30 minutes (offline)
- Transcription quality is acceptable for Dutch, English and mixed speech (benchmark), with speaker attribution ≥90% correct
- ≥80% of generated action items are accepted in Plane Intake without major edits
- Users rate summaries ≥4/5
- Zero privacy incidents, and OR agreement in place before rollout

---

## 9. Recommendation

1. **Make the Meeting Router** on Azure Functions: Teams-native transcripts via Graph, Azure AI Speech for offline audio, Claude for **English** summaries, **one PR per meeting** in the second-brain repo, and Plane Intake issues **on merge**.
2. **Offline capture:** start with the OneDrive "Meeting Inbox" drop folder. At 10 active users, **buy (jamie) rather than build** a recorder app if the drop folder isn't good enough.
3. **Don't buy** Copilot or a cross-platform notetaker for this. They don't reach Git or Plane.
4. **Next steps this week:** OR and privacy officer briefing, Graph permission request, GitHub App + people mapping, collect 10 sample recordings (Dutch, English, mixed) for the benchmark, scaffold the router repo.

### Open questions
- Do all 10 active users (the organizers) have GitHub accounts and feel comfortable reviewing a PR? If not, who is the fallback KB maintainer?
- Should Teams transcription be on by default for everyone, or only for certain meeting types?
- Router language: TypeScript or C# (whatever the Azure team prefers)?
- Should transcripts stay in git (simple), or go in Blob with a link (easier right-to-erasure)?
