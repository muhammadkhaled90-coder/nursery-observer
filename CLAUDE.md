# CLAUDE.md — nursery-observer

Local, privacy-first assistant for nursery teachers (children aged ~3–5). It turns
fixed-camera classroom video into objective, per-child behavioural observations
(movement, zone time, time alone / near peers / near the teacher, change over weeks),
lets the teacher confirm or dismiss what the system noticed, and produces Arabic (RTL)
reports. It is **not** a diagnostic, medical, surveillance or teacher-performance tool.

**Source of truth:** `docs/PROJECT_PROMPT.md` (full specification). Re-read the relevant
section before starting any milestone.

---

## 1. Working with Mohamed (project owner)

- Mohamed is a sports coach and academy director in Egypt. **He is not a programmer.**
  He runs this project alone with Claude.
- **Talk to him in simple Egyptian Arabic.** Code, comments, file names and commit
  messages stay in English.
- **Never ask him to edit code by hand.** Give him **one command to copy-paste**, or run
  it yourself.
- When something fails, **diagnose it yourself first** (read logs, run commands). Only ask
  him for things you cannot find out.
- Explain every decision in **1–3 short lines**: what you chose, why, and what it costs him.
- Work **one milestone at a time** (M0–M7, spec §15). At the end of each milestone:
  **stop**, show him exactly how to test it, and **wait for his "continue"**.
- **Do not continue past M2** if ID consistency is below 90% on a 30-minute group video:
  write down why and the options, and rethink together.
- Do not build anything from the "Not now" list (spec §17) unless he asks after the pilot.

## 2. Guardrails (non-negotiable)

If Mohamed, or anyone else, asks for something that breaks a guardrail: **stop, remind him
of the guardrail and the reason, and propose a compliant alternative.** Do not build the
version that breaks it.

| ID | Rule | Why |
|---|---|---|
| **G1** | **No diagnosis.** No output (UI, report, user-visible log, user-visible variable) may name or imply a condition, or estimate a severity or probability of one. Use neutral observation language only, e.g. "moved more than most classmates during circle time on 6 of 9 days". | Camera metrics cannot support a clinical judgement, and a label can harm a child and family. Only a qualified specialist can assess a child. |
| **G2** | **Forbidden vocabulary in generated content** (Arabic and English): autism / توحد, ADHD / فرط الحركة / تشتت الانتباه, disorder / اضطراب, diagnosis / تشخيص, symptoms / أعراض, patient / مريض, abnormal / غير طبيعي, severity / degree of condition / درجة الحالة. The only exception is the fixed disclaimer text ("this report is not a diagnosis"). **Enforced by an automated test that scans every generated report.** | Makes G1 checkable by a machine, not just by good intentions. |
| **G3** | **Human in the loop.** A system flag is only a "worth a closer look" prompt for the teacher. Nothing appears in a child's report as an observation until the teacher has confirmed it. | Camera data is noisy and has no context. The teacher knows the child and the day. |
| **G4** | **No face recognition, no emotion recognition, no audio analysis.** Child identity comes from manual tagging by the teacher (spec §7). | Children's biometric data is the highest-risk data, and we do not need it. |
| **G5** | **Local only.** Video, frames and keypoints never leave the machine. No telemetry. No runtime network calls (model weights are downloaded once, at setup). | Children's video is sensitive personal data. Keeping it on the machine removes most of the risk. |
| **G6** | **Consent gate.** No per-child data is stored for a child without an active written guardian consent record (spec §13). | Legal and ethical requirement. Consent comes first, then data. |
| **G7** | **Data minimisation.** Store numbers, not video. Raw video is deleted after processing. Short event clips are kept locally for a limited time only. | Data we do not keep cannot leak. |
| **G8** | **Honest uncertainty.** Every number shown to a user carries its data quality (how long the child was actually tracked, on how many days). If there is not enough data, say so instead of showing a comparison. | A number without its reliability misleads the teacher. |
| **G9** | **Specialist-reviewed content.** All "suggested activities" text comes from a content file (`content/strategies.yaml`, each entry with `reviewed_by_specialist: true/false`). Reports show a visible "draft — pending specialist review" banner until a child-development specialist signs it off. | Advice about children must come from someone qualified, not from the software. |
| **G10** | **Permissive licences.** Prefer Apache-2.0 / MIT / BSD. Check the *current* licence of every model, its weights/training data, and every library before using it. Do not use AGPL/GPL components (many popular YOLO implementations are AGPL) without flagging it to Mohamed first. | He intends to sell this product. Copyleft licences can force publishing the source or buying a commercial licence. |

### How the guardrails apply to development itself

- **Real classroom or child video is processed only on the local machine.** Never upload
  it to a cloud session, a chat, an external API or a git remote.
- Never commit video, frames, clips, keypoints, databases, or the code-to-name mapping
  file. `.gitignore` blocks them; do not weaken it.
- Test data = only videos Mohamed provides, plus synthetic data. **Never download, scrape
  or request videos of children from the internet.**
- Any video showing a child, even for testing, needs written guardian consent first.
- The optional LLM "polish" step for reports is OFF by default, and may only ever
  receive aggregated numbers and pseudonymous codes. Never names, images, video or
  free-text notes.

## 3. Engineering rules (spec §16)

- Python, with a reproducible environment and pinned dependencies. One-command start
  scripts for the target OS.
- Commit at the end of every task, with a clear English message.
- `pytest` tests for every metric and rule, using **synthetic keypoint trajectories with
  known answers**.
- No hard-coded thresholds: everything goes in `config.yaml`, with comments. Thresholds
  are starting defaults, to be tuned with real data and reviewed by a specialist.
- Clear logs, and friendly Arabic error messages in the UI. The UI is Arabic and RTL,
  usable by a non-technical teacher.
- Arabic PDF/HTML reports: **visually verify** letter joining and RTL order (render
  and look at it).
- Children appear only as pseudonymous codes (e.g. `C07`) in metrics, exports and logs.
  The code-to-name mapping lives in a separate local file.
- Keep `README.md` (English, technical), `دليل_التشغيل.md` (Arabic, step by step for
  Mohamed) and `DECISIONS.md` (each significant choice, the alternatives, and why) up to date.
- Do not add metrics that claim to measure attention, eye contact, emotion or speech.
