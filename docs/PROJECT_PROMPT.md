# PROJECT PROMPT — Nursery Behavior Observation Assistant

> Working name: `nursery-observer`. Read this whole file before doing anything.
> Then follow section 18 ("Your first reply"). Do not write code before I approve your plan.

---

## 1. Who I am and how to work with me

- I am Mohamed, a sports coach and academy director in Egypt. **I am not a programmer.** I will run this project alone with you.
- **Talk to me in simple Egyptian Arabic.** Code, comments, file names and commit messages stay in English.
- Never ask me to edit code by hand. Give me **one command to copy-paste**, or run it yourself.
- When something fails, diagnose it yourself first (read logs, run commands). Only ask me for things you cannot find out.
- Explain every decision in 1–3 short lines: what you chose, why, and what it costs me.
- Work **one milestone at a time**. At the end of each milestone: stop, show me how to test it, wait for my "continue".

## 2. What we are building

A **local, privacy-first tool for nursery teachers** (children aged ~3–5). It analyses video from a fixed room camera and turns it into **objective behavioral observations per child**: how much each child moves, where they spend time, how much time they spend alone or near peers and the teacher, and how this changes over weeks. It compares each child to classmates and to their own baseline, lets the teacher confirm or dismiss what the system noticed, and produces a clear bi-weekly/monthly report in Arabic with the reasons behind each observation and suggested classroom activities.

**The customer** is the nursery. **The user** is the classroom teacher. Reports are for the teacher/nursery, who work with the child first and involve the parent later.

### What it is NOT
- It is **not a diagnostic or medical tool**. It never names a condition, never estimates a severity or probability of a condition.
- It is not a surveillance tool for the owner, and not a teacher-performance tool.
- It does not recognise faces, emotions, or speech.

## 3. Non-negotiable guardrails

Save these into `CLAUDE.md` in the repo so they persist across sessions. If I ever ask for something that breaks one of them, **stop, remind me of the guardrail and the reason, and propose a compliant alternative.**

- **G1 — No diagnosis.** No output (UI, report, log shown to users, variable shown to users) may name or imply a condition. Use neutral observation language only, e.g. "moved more than most classmates during circle time on 6 of 9 days".
- **G2 — Forbidden vocabulary in generated content** (Arabic and English): autism / توحد, ADHD / فرط الحركة / تشتت الانتباه, disorder / اضطراب, diagnosis / تشخيص, symptoms / أعراض, patient / مريض, abnormal / غير طبيعي, severity / degree of condition / درجة الحالة. The only exception is the fixed disclaimer text ("this report is not a diagnosis"). Enforce with an automated test that scans every generated report.
- **G3 — Human in the loop.** A system flag is only a "worth a closer look" prompt for the teacher. Nothing appears in a child's report as an observation until the teacher has confirmed it.
- **G4 — No face recognition, no emotion recognition, no audio analysis.** Child identity comes from manual tagging (section 7).
- **G5 — Local only.** Video, frames and keypoints never leave the machine. No telemetry. No runtime network calls (model weights are downloaded once at setup).
- **G6 — Consent gate.** No per-child data is stored for a child without an active written guardian consent record (section 13).
- **G7 — Data minimisation.** Store numbers, not video. Raw video is deleted after processing. Short event clips are kept locally for a limited time only.
- **G8 — Honest uncertainty.** Every number shown to a user carries its data quality (how long the child was actually tracked, on how many days). If there is not enough data, say so instead of showing a comparison.
- **G9 — Specialist-reviewed content.** All "suggested activities" text comes from a content file that is marked as unreviewed until a child-development specialist signs it off; reports show a visible "draft — pending specialist review" banner until then.
- **G10 — Permissive licences.** Prefer Apache-2.0 / MIT / BSD dependencies. Check the *current* licence of every model and library before using it. Do not use AGPL/GPL components (many popular YOLO implementations are AGPL) without flagging it to me first, because I intend to sell this later.

## 4. Users and daily workflow (target state)

1. Teacher opens the app on the classroom laptop and logs in.
2. Start of session: teacher picks session type (**structured** e.g. circle time / table work, or **free play**) and tags each child on the first frame (section 7).
3. Video is processed (recorded file first; live mode comes much later).
4. End of day: teacher fills a quick per-child form (≤10 minutes for the whole class) and reviews any "worth a closer look" items with their short clips: **confirm / dismiss / keep watching**.
5. Once a month: the teacher runs the same short standard activity ("monthly challenge") so we measure the same thing under the same conditions.
6. Every two weeks or monthly: one report per child + one class overview.

## 5. Architecture (local pipeline)

```
video file (later: RTSP stream)
   -> person detection + pose estimation (keypoints per person per frame)
   -> multi-person tracking (stable track IDs)
   -> identity layer (teacher tags tracks -> pseudonymous child codes; ID-switch repair)
   -> feature extraction (per child, per time window)
   -> SQLite (metrics only)  ->  raw video deleted
   -> comparison engine (vs class, vs own baseline) -> flags (pending teacher review)
   -> event clips (short, local, time-limited)
   -> teacher app (tagging, review, daily form, monthly challenge)
   -> report generator (Arabic, RTL, PDF/HTML)
```

Keep modules small and independent so each can be tested with synthetic data.

## 6. Technology guidance (propose, verify, justify)

- Python. Reproducible environment with pinned dependencies. One-command start scripts for my OS.
- Pose + tracking: choose a **multi-person pose model and tracker with permissive licences** that runs acceptably on my hardware. Check current documentation and licences online before deciding; do not rely on memory for versions. Present 2 options with speed/accuracy/licence trade-offs and recommend one.
- OpenCV for video I/O, SQLite for storage, a simple local web UI (e.g. Streamlit or similar — your call, justify it). The UI must be in Arabic, right-to-left, and usable by a non-technical teacher.
- Reports: HTML -> PDF with correct Arabic shaping and RTL. **Visually verify** that Arabic letters are connected and ordered correctly; this commonly breaks.
- Processing recorded video slower than real time is acceptable for the pilot.

## 7. Child identity without biometrics

- Children are stored under **pseudonymous codes** (e.g. `C07`). The code-to-name mapping lives in a separate local file, never in metrics tables, exports or logs.
- At session start the teacher clicks each child on a still frame and picks their code. Adults are tagged as "adult" with no identity.
- Optional helper: children wear coloured vests/wristbands; use colour only as a hint to keep tracks stable, never the face.
- Detect likely ID switches (track breaks, crossings, long occlusions) and give the teacher a fast screen to fix them ("who is this?" with a thumbnail).
- Always report **tracked time %** per child per session.

## 8. Metrics (first version)

Normalise distances by the child's own body height in pixels to reduce camera-perspective effects. Compute per session and per 1-minute window. All thresholds live in `config.yaml` and are **starting defaults to be tuned with real data and reviewed by a specialist**.

| Metric | Definition (starting point) |
|---|---|
| Activity level | Mean speed of hip-centre, in body-heights per second; plus % of time above a "high movement" threshold |
| Stillness bouts | Number and length of periods with near-zero movement |
| Zone time | % of time in teacher-drawn zones (carpet, tables, play corner, door...) |
| Leaving the activity area | Count of exits from the designated zone during **structured** sessions |
| Transitions | Zone changes per 10 minutes |
| Solitary time | % of time with no peer within ~1.5 body-heights |
| Peer proximity | Number of distinct peers approached; mean duration of proximity bouts |
| Adult proximity | % of time near an adult |
| Repetitive movement (experimental, OFF by default) | Periodicity of wrist/trunk keypoints; always labelled low-confidence |
| Data quality | Tracked time %, mean keypoint confidence, number of repaired ID switches |

Do not add metrics that claim to measure attention, eye contact, emotion or speech.

## 9. Comparison and flag logic

- Compare only **within the same class and the same session type**. Store each child's age band and warn when a class mixes very different ages.
- Use robust statistics (median and MAD z-scores). With fewer than 8 children with good data, show ranks and raw values only and state "not enough classmates for a reliable comparison".
- **Own-baseline trend:** each child's last 2 weeks vs their previous weeks.
- A **flag** is raised only when all are true: |robust z| ≥ 2 on the same metric, on ≥ 3 different days within 10 nursery days, with tracked time ≥ 60% in those sessions.
- Flag names are neutral: "More movement than classmates", "More time alone than classmates", "Less time near peers", "Change from own usual pattern".
- Flag states: `pending` -> `confirmed` / `dismissed` / `watching`. Record who decided and when. Dismissed flags never reach a report.
- If a confirmed pattern persists for a configurable period (default 6 weeks) despite classroom strategies, the report adds a neutral line recommending the nursery suggest the family consult a child-development specialist.

## 10. Event clips

- When a flag condition repeats, save a short clip (10–20 s) of the relevant moment, **locally only**, for the teacher's review.
- Blur everyone except the target child in saved clips.
- Auto-delete clips after a configurable retention period (default 14 days). Show the teacher what will be deleted and when.

## 11. Teacher inputs

- **Daily form:** per child, tap-based 1–5 ratings on the same constructs the camera measures (movement level, joining group activities, playing with others, staying with the activity) plus an optional note. Must take ≤10 minutes for a whole class on a phone or laptop.
- **Monthly challenge:** a short, fixed, repeatable activity protocol (e.g. 5-minute story circle, then 5-minute group building game). Store it as a tagged session so months are comparable.
- The teacher can add a concern the system did not flag. Store these — they measure what the system misses.

## 12. Reports (Arabic, RTL)

Per child, bi-weekly or monthly:
1. Summary in 3–4 plain sentences.
2. Each **teacher-confirmed** observation: what was observed, on how many days, compared to what, with a small chart and the data-quality line.
3. Trend vs the child's own previous period.
4. Teacher's notes.
5. Suggested classroom activities, pulled from `content/strategies.yaml` by observation type (each entry has `reviewed_by_specialist: true/false`).
6. Fixed disclaimer: this is an observation aid, not a diagnosis; only a qualified specialist can assess a child.

Also a one-page class overview for the teacher.

Report text is **template-based by default** and works offline. An optional LLM "polish" step may exist but must be OFF by default, and may only ever receive aggregated numbers and pseudonymous codes — never names, images, video or free-text notes.

## 13. Privacy and compliance features (Egyptian personal data law context)

Children's data is treated as sensitive. Build these as real features, not documentation:
- **Consent registry:** per child code — consent form reference, date, scope, expiry, status (active / withdrawn). The pipeline refuses to store metrics for a child without active consent. Non-consented children are used only transiently in memory as anonymous "peer" positions and are blurred in any clip. Mark this behaviour clearly as **"needs legal review"** in the docs.
- **Erase:** one action deletes everything for a child (metrics, clips, flags, report files) and logs the deletion.
- **Retention policy** in config: raw video (delete after processing), clips (14 days), metrics (one academic year).
- **Access:** app password; teacher role only in v1. Recommend OS full-disk encryption in the setup guide.
- **Audit log:** who viewed/exported/deleted what and when.
- Generate a plain-Arabic **draft** consent form and a one-page "what this system does and does not do" sheet for parents, both clearly marked "draft — must be reviewed by a lawyer".

## 14. Validation built into the product

A hidden "validation" page for me (not the teacher) showing:
- Tracking quality per session (tracked time %, ID switches, and ID accuracy against a small hand-labelled sample — build a tiny labelling tool for this).
- Agreement between each camera metric and the matching teacher rating (Spearman correlation per construct).
- Flag precision (% confirmed by teacher) and misses (teacher concerns the system did not flag).

Pilot success targets: ID consistency ≥ 90% on a 30-minute group video; correlation ≥ 0.6 between camera metrics and teacher ratings; a teacher understands a child report in ≤ 5 minutes.

## 15. Milestones (riskiest first). STOP after each one.

- **M0 — Setup and feasibility.** Detect my OS/CPU/GPU/RAM. Set up the environment. Run pose + tracking on a sample video I provide and produce an annotated video I can watch. Report speed (FPS), a dependency licence table, your model recommendation, and camera placement advice.
  *Exit:* I watch the annotated video and see skeletons following people correctly.
- **M1 — Single-child metrics.** Activity metrics over time, zone-drawing tool, per-session chart.
  *Exit:* I record 3 short videos of one child (calm / active / mixed) and the numbers rank them correctly.
- **M2 — Multi-child tracking and identity.** Tagging UI, ID-switch detection and repair, labelling tool, tracking-quality report.
  *Exit:* ≥ 90% ID consistency on a 30-minute group video — or a clear written explanation of why not, with options (vests, camera height/angle, second camera). **Do not continue past M2 if this fails; we rethink.**
- **M3 — Data model and social metrics.** SQLite schema, session types, all section-8 metrics, unit tests on synthetic trajectories.
- **M4 — Comparison, flags, clips.** Section 9 and 10, flag review screen, retention job.
- **M5 — Teacher inputs.** Daily form and monthly challenge.
- **M6 — Reports.** Arabic RTL child report + class overview, forbidden-vocabulary test, specialist-review banner.
- **M7 — Compliance and handover.** Consent gate, erase, audit log, password, backup, one-click start, Arabic teacher guide, validation page.

For each milestone give me: tasks as a checklist, time estimate, what I must provide (e.g. a video), and exactly how I verify the result.

## 16. Engineering rules

- Git repository from day one; commit at the end of every task with a clear message.
- `pytest` tests for every metric and rule, using **synthetic keypoint trajectories** with known answers.
- No hard-coded thresholds; everything in `config.yaml` with comments.
- Clear logs; friendly Arabic error messages in the UI.
- `README.md` (English, technical) and `دليل_التشغيل.md` (Arabic, step-by-step for me).
- **Test data:** use only videos I provide, plus synthetic data. Never download, scrape or request videos of children from the internet.
- Keep a `DECISIONS.md` log: each significant choice, the alternatives, and why.

## 17. Not now (do not build unless I ask after the pilot)

Live RTSP streaming and event-triggered recording, multiple cameras, cloud sync, parent-facing app, mobile app, owner dashboards, multi-nursery accounts, billing, any ML model trained on our own data.

## 18. Your first reply

1. Confirm in Arabic, in 5 lines or fewer, what you understood we are building and the guardrails.
2. Ask me only the questions you truly need (ideally ≤ 5), and inspect my machine yourself instead of asking about it.
3. Propose the M0 plan as a short checklist with time estimates.
4. Create `CLAUDE.md` with the guardrails from section 3 and the working rules from section 1.
5. Wait for my approval before writing any other code.
