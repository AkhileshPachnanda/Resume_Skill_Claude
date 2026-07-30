---
name: resume-review
description: >
  Full-pipeline resume review, JD tailoring, ATS optimisation, LaTeX/PDF output, cover letter
  generation, interview-prep talking points, version diffing, application tracking, and
  versioned Google Drive backup. Trigger this skill whenever the user: uploads a resume
  (PDF/DOCX/tex), pastes resume text or LaTeX source, says "review my resume", "tailor for
  this JD", "optimise for ATS", "help me apply for X role", "rewrite my CV", "make my resume
  for [company]", "write a cover letter", "compare this to my last [company] resume",
  "what changed between versions", "prep me for questions about my resume", "log this
  application", "show my applications", "update my application status", or asks anything
  about improving, scoring, saving, or tracking a resume/application. Works both WITH and
  WITHOUT a JD — a JD unlocks full tailoring mode (incl. cover letter), but general critique
  runs fine on the resume alone. Also trigger when the user provides an updated LaTeX
  template and says "use this format". Do NOT skip this skill just because no JD is
  provided — run general critique mode instead.
---

# Resume Review & JD Tailoring Skill

Produces a meticulously reviewed, ATS-clean, multi-perspective-critiqued resume for software
roles targeting **20 LPA+ packages** for a fresher/early-career candidate.

Output is always **LaTeX source + compiled PDF**. Uses the stored base template in
`assets/base-template.tex` unless the user provides a different one.

Pipeline: ingest → critique → [gap analysis if JD present] → rewrite → interview-prep talking
points → LaTeX compile → cover letter (tailored) → Drive push → application tracker → delivery
→ snapshot write

Two on-demand modes run outside this linear pipeline: **Version Diff Mode** and **Application
Tracker Mode** (see their own sections below) — trigger these directly without running the
full pipeline.

---

## Core Principles (read before every run)

- **Anti-fabrication**: Never invent metrics, companies, or responsibilities. Reframe and
  quantify only from stated facts. If a metric is absent, insert a `[FILL: <suggestion>]`
  placeholder — never a fabricated number.
- **Honest skill representation**: Only list skills that appear in the resume or are confirmed
  by the user. Do not add skills just because the JD mentions them.
- **Keyword fidelity**: Match JD keywords exactly (case-matched). Do not paraphrase — use the
  exact string from the JD (e.g. "Node.js" not "NodeJS" if the JD says "Node.js").
- **AI fingerprint avoidance**: Banned words list is in `references/rewrite-rules.md`. Read it.
  Favour concrete active verbs: built, shipped, reduced, implemented, debugged, deployed.
  This applies to cover letters too, not just resume bullets.
- **Freshers compete on projects + real exposure**: For 0–2 YOE, real paid work and deployed
  projects are the primary differentiator. Prioritise both over coursework or club membership.
- **LaTeX integrity**: Preserve the structural style of `assets/base-template.tex` exactly.
  Do not change fonts, margins, section formatting, or spacing unless the user explicitly asks.
  Every output must compile cleanly with pdflatex.
- **Interview-prep honesty**: Never hand the user a talking point that overstates or invents
  methodology. If the real answer is "I estimated this," say that plainly — a defensible
  honest answer beats a confident fabricated one in an interview.
- **Tracker accuracy**: Never log or alter an application record without the user confirming
  the specific fields (date, status). Don't guess a status change from ambiguous phrasing.

---

## Phase 0 — Input Collection & Mode Detection

### Step 1: Collect inputs

| Input | Required | Notes |
|-------|----------|-------|
| Resume (content) | ✅ Always | Accept: uploaded PDF/DOCX, pasted LaTeX source, pasted plain text |
| Job Description | ❌ Optional | If absent → run General Critique Mode |
| Target company / role | ❌ Optional | Infer from JD if present |
| Updated LaTeX template | ❌ Optional | If user provides one, use it instead of `assets/base-template.tex` |

If the resume is not provided, ask for it before proceeding. Do not ask for the JD — proceed
without it if not given.

### Step 2: Detect mode

```
IF JD provided                       → TAILORED MODE   (all phases 0–11)
IF no JD                             → GENERAL MODE    (phases 0–1 → 4 → 5 → 6 → 8 → 10 → 11,
                                                          skip Phase 2, 7, and project scoring)
IF user asks to compare versions     → VERSION DIFF MODE   (standalone, see below)
IF user asks about applications only → APPLICATION TRACKER MODE   (standalone, see below)
```

Announce the mode to the user at the start:
- Tailored: "Running full tailored review for [Role] @ [Company]."
- General: "No JD provided — running general critique and quality pass. To get a JD-tailored
  version later, paste a job description and I'll re-run."
- Diff: "Comparing [Version A] against [Version B]."
- Tracker: "Pulling up your application tracker."

### Step 3: Template selection

- Default: use `assets/base-template.tex` as the structural base.
- If user has provided an updated `.tex` template in this conversation: use that instead.
- Store the active template in memory as `ACTIVE_TEMPLATE` for Phase 6.

### Step 4: Check for profile-review cross-reference data (optional enrichment)

Before running, silently check Google Drive for existing profile data:

```
Google Drive:search_files
  query: name = 'profile-review-output.json' and 'ResumeKit' in parents
```

If found, read it and extract `cross_platform_alerts[]`. Surface any relevant alerts
inline during Phase 1 diagnostics. Example alert to surface:
"⚠️ profile-review found TypeScript listed on resume but 0 TypeScript repos on GitHub —
  flag this for the peer reviewer persona."

If not found: proceed normally, no mention to user.

---

## Phase 1 — Resume Ingest & Structured Extraction

Read the uploaded resume (use PDF skill for .pdf, docx skill for .docx, read directly if .tex).

Parse into this internal model (never show raw JSON to user):

```
{
  header: { name, email, phone, linkedin, github, location },
  education: [{ institution, degree, field, cgpa, year, coursework[] }],
  experience: [{ title, company, type, duration, bullets[] }],
  projects: [{ name, stack, live_url, repo_url, bullets[], has_metric: bool }],
  skills: { languages[], frameworks[], tools[], domains[] },
  positions: [{ title, org, duration, bullets[] }],
  open_source: [{ repo, pr_url, description }],
  certifications: []
}
```

### Immediate Issue Flags (show to user as a quick diagnostic before any phase runs)

Always show this block before proceeding. Be specific — name the exact section/bullet:

```
## Quick Diagnostic

✅ Passing
- [list items that are already strong]

⚠️ Issues Found
- CGPA [X]: [below/at] the 7.5 filter threshold most product companies use
- Bullet "[exact text]": starts with a noun — needs a verb
- Bullet "[exact text]": no metric, no outcome, no scale
- Skill "[X]": listed in skills but unsupported by any bullet
- Resume length: [N pages] — freshers with <2 YOE should be 1 page
- [Other specific issues]

📌 Missing Elements
- No GitHub link in header
- No deployed URL on any project
- [Other missing elements]
```

---

## Phase 2 — JD Gap Analysis (TAILORED MODE ONLY — skip if no JD)

Read `references/fresher-market-2025.md` for current market context before running this phase.

Map the JD against the resume across 6 dimensions:

| Dimension | What to check |
|-----------|--------------|
| **Keyword Match** | Which exact JD keywords are missing from the resume? |
| **Stack Alignment** | Does the resume tech stack match what the JD requires? |
| **Seniority Signal** | Does experience framing match SDE-I / fresher level? |
| **Impact Depth** | Are bullets outcome-oriented or task lists? |
| **Domain Fit** | Industry/domain terms from JD present in resume? |
| **Soft Signals** | Leadership, cross-functional work, ownership — present? |

Show the Gap Report before rewriting:

```
## Gap Report — [Role] @ [Company]

### ✅ Strong Matches
- [specific match with JD reference]

### ⚠️ Partial Matches (present but not keyword-matched)
- Resume says "[X]" — JD says "[Y]" — swap phrasing

### ❌ Missing (JD requires / prefers)
- "[keyword]" — Action: [add to skills / surface in project bullet / leave — can't support truthfully]

### 🔢 Metric Placeholders Needed
- Project "[X]": no impact metric — ask user: GitHub stars, active users, response time, error rate?
- Experience "[Y]": ask user: deployment scale, PR count, team size?
```

Pause here and confirm with user before proceeding to Phase 3.
If user confirms, proceed. If user supplies additional facts/metrics, incorporate them.

**Keep the "Strong Matches" list in memory as `JD_STRONG_MATCHES`** — Phase 7 (Cover Letter)
reuses this directly instead of re-deriving it.

---

## Phase 3 — Rewrite Engine

Read `references/rewrite-rules.md` for full rules, banned words, verb bank, and bullet patterns.

### Section Order (for 0–2 YOE)

1. Header
2. Technical Skills
3. Work Experience (paid roles, internships — ordered by recency)
4. Projects (top 3–4 — see scoring below)
5. Positions of Responsibility
6. Education
7. Relevant Coursework (only if it maps directly to JD or role type)

### Project Scoring

**TAILORED MODE**: Score each project against the JD:
- +2 pts: uses a JD-mentioned technology
- +2 pts: has a deployed/live URL
- +1 pt: has a GitHub repo
- +1 pt: has a measurable metric
- +1 pt: demonstrates system design (caching, APIs, DB, queuing)
- -1 pt: tutorial-following with no original logic

**GENERAL MODE**: Score each project on standalone quality:
- +2 pts: has a live deployed URL
- +2 pts: has a measurable metric (users, latency, uptime, etc.)
- +1 pt: uses a production-grade stack (not purely academic)
- +1 pt: demonstrates a non-trivial technical decision
- +1 pt: has a GitHub repo with visible code

Select top 3–4 by score. Rewrite each selected project's bullets per `references/rewrite-rules.md`.

### Skills Section Rules

- Groups: Languages | Web & Frameworks | Tools & Platforms | Core Areas
- Mirror the style of `assets/base-template.tex` exactly
- Remove any skill with no supporting bullet
- Remove: React Native, MATLAB, AWS, CI/CD, Excel — unless user explicitly confirms experience
- Do not add new skills — only reorder and clean existing ones

### One-Page Enforcement

Resume must fit on one page for 0–2 YOE. Cuts in priority order:
1. Remove "Objective" / "Summary" sections
2. Remove weak positions (one-liner bullets, no impact, < 3 months)
3. Cut projects below top 3–4
4. Reduce each project to 2–3 bullets (cut the weakest)
5. Trim coursework to 4–5 items that directly match the role
6. Tighten margins minimally only as last resort

---

## Phase 4 — Multi-Persona Critique

Read `references/critique-personas.md` for full scoring criteria per persona.

Run all 5 perspectives sequentially. Show results to user:

```
## Resume Health Score

### 1. ATS Bot             [X]/20
   ✅ [strength 1]
   ✅ [strength 2]
   🔧 Fix: [specific actionable fix]
   🔧 Fix: [specific actionable fix]

### 2. HR Screener (30s)   [X]/20
   ...

### 3. Technical Reviewer  [X]/20
   ...

### 4. Hiring Manager      [X]/20
   ...

### 5. Peer Engineer       [X]/20
   ...

─────────────────────────────────
TOTAL:                     [X]/100

### Top 3 High-ROI Fixes (in order of impact)
1. [Exact bullet or section to change, and how]
2. [...]
3. [...]
```

In GENERAL MODE, Persona 1 (ATS) scores against common SDE role keywords from
`references/fresher-market-2025.md` rather than a specific JD.

**Flag every bullet with a bold, hard-to-verify, or estimated metric** (e.g. a claimed
percentage reduction/improvement with no stated methodology) into a running list
`CLAIMS_NEEDING_DEFENSE`. Phase 5 turns this list into interview talking points. A claim
qualifies if: it's a specific number, and a reasonable interviewer would ask "how did you
measure that?"

---

## Phase 5 — Interview-Prep Talking Points

Runs in both modes, always, right after the health score is final and before LaTeX compile —
this is the resume's last chance to change before the numbers on it are locked in, so this
phase works off the final bullet text.

### Step 1: Build the claim list

Take `CLAIMS_NEEDING_DEFENSE` from Phase 4 and add any `[FILL: ...]` placeholder the user
resolved with a number in Phase 2/3. Every specific, quantified, or strong claim on the final
resume should have exactly one entry.

### Step 2: For each claim, generate a talking point — never a fabricated one

For each claim, ask (don't assume) if unclear, then produce:

```
## Interview Prep — Be Ready to Explain These

### "[exact bullet text, e.g. 'Reduced return rate by an estimated 40% via virtual try-on']"
- **What they'll ask**: "Walk me through how you measured that 40%."
- **Your honest answer**: [methodology in plain terms — sample size, before/after comparison,
  proxy metric used, or "this was a projected/estimated figure based on X, not a measured
  production result" if that's the truth]
- **If pushed further**: [one level deeper — what you'd say if they ask for the raw numbers
  or the confidence interval]
- **Fallback if you can't defend it as stated**: [a scaled-back, honest version of the claim
  you could say instead, e.g. "we projected up to 40% based on a pilot with N users" rather
  than a flat "reduced by 40%"]
```

If the user hasn't given you the real methodology, do not invent one — write:
`[NEEDS INPUT: what was this estimate actually based on? A/B test, pilot, industry
benchmark, or projection model?]` and ask them directly before finalizing.

### Step 3: Present as a standalone block

This list is shown to the user in Phase 10 (Delivery Summary) and is **not** put into the
resume or cover letter itself — it's prep material only.

---

## Phase 6 — LaTeX Output & PDF Compilation

### Step 1: Generate LaTeX source

Use `ACTIVE_TEMPLATE` (from Phase 0 Step 3) as the structural skeleton.
**Preserve all preamble, packages, spacing, and formatting exactly.**
Only modify the content sections: header, skills, experience, projects, positions, education,
coursework.

Content rules:
- Escape all special LaTeX characters: `& % $ # _ { } ~ ^ \`
- Hyperlinks: use `\href{URL}{display text}` — verify all URLs preserved from original
- Bold metrics: `\textbf{X}` for all numbers/metrics, matching template style
- GitHub icons on projects: `\href{url}{\faGithub}` — include only if repo URL exists
- Never introduce new LaTeX packages not already in the template preamble

File naming convention:
```
[LastName]_[FirstName]_[Role]_[Company]_[YYYY-MM].tex    ← if tailored
[LastName]_[FirstName]_General_[YYYY-MM].tex              ← if general mode
```

Save `.tex` file to `/mnt/user-data/outputs/`.

### Step 2: Install LaTeX and compile to PDF

```bash
# Install if not present
apt-get install -y texlive-latex-base texlive-fonts-recommended \
  texlive-latex-extra texlive-fonts-extra 2>/dev/null | tail -5

# Install fontawesome if missing
tlmgr install fontawesome 2>/dev/null || true

# Compile (run twice for consistent layout)
pdflatex -interaction=nonstopmode -output-directory /mnt/user-data/outputs/ \
  /mnt/user-data/outputs/[filename].tex

pdflatex -interaction=nonstopmode -output-directory /mnt/user-data/outputs/ \
  /mnt/user-data/outputs/[filename].tex
```

If compilation fails:
1. Show the specific error line from the log
2. Fix the LaTeX source (common issues: unescaped `&`, `%`, `_`; missing `\`)
3. Re-compile
4. If still failing after 2 fix attempts: output `.tex` only with a note to compile locally

Clean up auxiliary files after success:
```bash
rm -f /mnt/user-data/outputs/[filename].aux \
      /mnt/user-data/outputs/[filename].log \
      /mnt/user-data/outputs/[filename].out
```

### Step 3: Present files

Call `present_files` with the `.pdf` and `.tex` (and the cover letter from Phase 7, if generated):
- PDF first (primary deliverable)
- .tex second (for manual editing)
- Cover letter `.txt` third, if generated

---

## Phase 7 — Cover Letter Generation

**TAILORED MODE**: runs automatically (uses the JD + company already on hand).
**GENERAL MODE**: skipped by default — offer it once: "Want a cover letter too? I'll need a
target company/role first." Do not generate a generic, company-less cover letter.

### Step 1: Pull inputs already in memory — don't re-ask

- `JD_STRONG_MATCHES` from Phase 2
- Company name, role title
- Final resume content from Phase 3/6 (for consistent phrasing — never contradict the resume)

### Step 2: Structure (max ~350–400 words, one page)

```
1. Opening hook (1–2 sentences): specific reason for applying to THIS company/role —
   reference something concrete about the company or role, not a generic template line.
2. Body paragraph 1: strongest 1–2 matches from JD_STRONG_MATCHES, told as a short story
   (project or work experience), not a bullet restatement of the resume.
3. Body paragraph 2: one more differentiator — a gap-closing angle if a JD requirement
   was a partial match, framed honestly (e.g. "while I haven't shipped with X in production,
   I've used it in [project] to...").
4. Close: brief, confident, states availability and interest in next steps. No cliché
   ("I am a hard worker", "I am confident I would be a valuable asset").
```

### Step 3: Rules

- Read `references/rewrite-rules.md` banned-words list — applies here too.
- First person, active voice, no restating the resume verbatim — this should read like a
  short case for *why this candidate, why this role*, not a resume in prose form.
- Anti-fabrication rule applies identically: no new claims not already on the resume or
  confirmed by the user.
- One draft only per run — don't offer 3 tone variants unless asked.

### Step 4: Output

Save as plain text:
```
[LastName]_[FirstName]_CoverLetter_[Company]_[YYYY-MM].txt
```
Save to `/mnt/user-data/outputs/`. Show the full text inline to the user as well as the file.

---

## Phase 8 — Google Drive Push

Use the connected Google Drive MCP to version and archive the resume (and cover letter, if
generated).

```
Tool sequence:
1. Google Drive:search_files
   query: name = 'Resumes' and mimeType = 'application/vnd.google-apps.folder'

2. If folder not found:
   Google Drive:create_file  (mimeType: application/vnd.google-apps.folder, name: "Resumes")

3. Google Drive:create_file  (mimeType: application/vnd.google-apps.folder,
   name: "[Company]_[Role]_[YYYY-MM]"  OR  "General_[YYYY-MM]",
   parent: Resumes folder ID)

4. Upload PDF:
   Google Drive:create_file  (file content, mimeType: application/pdf,
   parent: subfolder ID)

5. Upload .tex:
   Google Drive:create_file  (file content, mimeType: text/plain,
   parent: subfolder ID)

6. If cover letter generated, upload it too:
   Google Drive:create_file  (file content, mimeType: text/plain,
   name: "[LastName]_[FirstName]_CoverLetter_[Company]_[YYYY-MM].txt",
   parent: subfolder ID)
```

Confirm: "Saved to Google Drive → Resumes/[subfolder]/ — PDF + .tex source (+ cover letter)"

If Drive push fails (auth/connection error): note it clearly and tell the user to use
`present_files` downloads as the fallback. Do not retry more than once.

---

## Phase 9 — Application Tracker Update

**TAILORED MODE ONLY** — general-mode reviews aren't tied to a specific application.

After a successful Drive push, offer once (don't force it):
"Log this as an application to [Company] – [Role]? I'll add it to your tracker."

If yes, collect (ask only for what's missing, don't re-ask what's inferable):
- `date_applied` — default to today unless user says otherwise
- `company`, `role` — already known from JD context
- `resume_version_file` — the `.tex` filename just generated
- `status` — default `"Applied"`

### Drive read-modify-write sequence (CSV has no partial-update API — always full rewrite)

```
1. Google Drive:search_files
   query: name = 'applications.csv' and 'ResumeKit' in parents

2. If not found:
   Create with header row only:
   date_applied,company,role,resume_version,status,notes
   Google Drive:create_file (mimeType: text/csv, name: 'applications.csv', parent: ResumeKit)

3. If found:
   Google Drive:download_file_content → parse CSV rows
   Append new row (or update existing row if same company+role already logged — ask user
   which they mean if ambiguous)
   Google Drive:create_file to overwrite with the full updated CSV content
```

Confirm: "Logged: [Company] – [Role] · Applied [date] · Status: Applied"

### Standalone tracker commands (Application Tracker Mode — no need to run the full pipeline)

| User says | Action |
|-----------|--------|
| "Show my applications" | Read `applications.csv`, render as a table, sorted by date_applied desc |
| "Update status for [Company] to [Status]" | Read CSV, find row, confirm match, update status field, rewrite CSV |
| "Log this application" (no active resume-review run) | Ask for company/role/resume version manually, append row |

If Drive read/write fails: tell the user plainly and offer to just show them what would have
been logged, so nothing is silently lost.

---

## Phase 10 — Delivery Summary

Show in this order — keep it tight, no filler:

```
## Done — [Role @ Company | General Review] · [YYYY-MM-DD]

Health Score: [BEFORE if general] / [AFTER if rewrite] out of 100

Changes made:
- [Concise list of what was actually changed]

Placeholders to fill in:
- [Project X, Bullet 2]: [FILL: e.g. number of GitHub stars or active users]
- [Skill Y]: confirm you're comfortable being asked about this in interview

Interview prep: [N] claims flagged with talking points ready (see above)

Strategic gaps (not resume fixes — actions to take):
- [e.g. "Add 1 more deployed project — current ratio is 1 live / 2 local"]
- [e.g. "LeetCode signal absent — add count once you hit 100+ solves"]

Cover letter: [generated / not generated — offer if general mode]
Application logged: [Yes → Company/Role/date | No]

Files: PDF + .tex + cover letter (download above) · Google Drive: [link]
```

---

## Phase 11 — Write Profile Snapshot to Drive (always runs silently after Phase 10)

After every completed run, write `profile-snapshot.json` to the shared `ResumeKit/` folder
in Google Drive. This is the data contract that `profile-review` reads.

**Never show this to the user or mention it unless it fails.**

### Snapshot schema

```json
{
  "generated_at": "YYYY-MM-DD",
  "source": "resume-review",
  "version": "[Role]_[Company]_[YYYY-MM] or General_[YYYY-MM]",
  "candidate": {
    "name": "[from header]",
    "email": "[from header]",
    "github_username": "[from header — extract handle only, no URL]",
    "linkedin_handle": "[from header — extract handle only]",
    "location": "[from header]"
  },
  "skills": {
    "languages": [],
    "frameworks": [],
    "tools": [],
    "domains": []
  },
  "experience": [
    { "title": "", "company": "", "type": "paid|internship|part-time", "duration": "" }
  ],
  "projects": [
    {
      "name": "",
      "stack": [],
      "has_live_url": true,
      "has_github_url": true,
      "has_metric": true,
      "score": 0
    }
  ],
  "positions": [
    { "title": "", "org": "", "scale_signal": "" }
  ],
  "health_score": 0,
  "keywords_used": [],
  "placeholders_unfilled": [],
  "claims_needing_defense": [],
  "cover_letter_generated": false,
  "last_target_role": "",
  "last_target_company": "",
  "strategic_gaps": []
}
```

### Drive write sequence

```
1. Google Drive:search_files
   query: name = 'ResumeKit' and mimeType = 'application/vnd.google-apps.folder'
   → If not found: create folder 'ResumeKit'

2. Google Drive:search_files
   query: name = 'profile-snapshot.json' and '{ResumeKit_folder_id}' in parents
   → If found: delete old version (Drive:delete_file)

3. Google Drive:create_file
   name: 'profile-snapshot.json'
   mimeType: 'application/json'
   content: [serialised snapshot JSON]
   parent: ResumeKit folder ID
```

If Drive write fails: silently skip, add a one-line note at the bottom of Phase 10 output:
"Note: Drive snapshot write failed — profile-review cross-reference won't have latest data."

---

## Version Diff Mode (on-demand — does not run the full pipeline)

Trigger phrases: "compare this to my last [Company] resume", "diff my TCS and Eurofins
resumes", "what changed between versions", "did I change the GYF bullet across versions".

### Step 1: Identify the versions to compare

```
Google Drive:search_files
  query: 'Resumes' in parents  (list subfolders, or search by company name in folder name)
```

If the user names two companies/versions, fetch both `.tex` files (or, if available, pull
each version's snippet from historical `profile-snapshot.json` entries if you're keeping
prior snapshots — otherwise fetch the `.tex` source directly and re-parse into the Phase 1
internal model for each).

If the user says only "compare this to my last one," diff the version currently in this
conversation against the most recently modified subfolder in `Resumes/`.

### Step 2: Section-by-section diff

Compare the two parsed resume models field by field:

```
## Version Diff — [Version A] vs [Version B]

### Added
- [Section/bullet present in B, not in A]

### Removed
- [Section/bullet present in A, not in B]

### Changed
- Skills: [X] → [Y]
- Project "[Name]" bullet: "[old]" → "[new]"
- [Any reordering of top projects]

### ⚠️ Consistency Risk
- Bullet about "[project/claim]" states a different number in each version:
  [Version A]: "[X]%" vs [Version B]: "[Y]%" — this is the kind of inconsistency an
  interviewer or reference check could catch. Recommend picking one defensible number
  and using it everywhere.
```

Consistency Risk is the most important output of this mode — flag it even if the user didn't
ask about it directly; a resume drift issue matters more than a stylistic diff.

---

## Application Tracker Mode (on-demand — does not run the full pipeline)

See Phase 9 for the full read-modify-write mechanics. Use this mode when the user only wants
to interact with the tracker, without running a resume review:

| User says | Action |
|-----------|--------|
| "Show my applications" | Fetch and render `applications.csv` as a table |
| "Update status for [Company] to [Status]" | Fetch, find, confirm, update, rewrite |
| "Log this application" | Ask for company/role/resume version, append row |
| "How many places have I applied to" | Fetch CSV, give a count + breakdown by status |

---

## Quick Reference — Trigger Matrix

| User says | Mode | Phases run |
|-----------|------|-----------|
| "Review my resume" | General | 0 → 1 → 4 → 5 → 6 → 8 → 10 → 11 |
| "Review my resume for [JD]" | Tailored | 0 → 11 (all) |
| "ATS check only" | Either | 0 → 1 → ATS persona only → no output |
| "Score my resume" | General | 0 → 1 → 4 → score only |
| "Tailor for this JD" | Tailored | 0 → 11 (all) |
| "Just rewrite bullets" | Either | 0 → 3 → 6 → 8 → 10 |
| "Write me a cover letter" | Tailored | Phase 7 only (uses last generated resume + JD context) |
| "Prep me for questions about my resume" | Either | Phase 5 only (uses last generated resume) |
| "Compare this to my last [Company] resume" | — | Version Diff Mode |
| "Show my applications" | — | Application Tracker Mode |
| "Log this application" | — | Phase 9 / Application Tracker Mode |
| "Push to Drive" | — | Phase 8 only (use last generated file) |
| "Use this template" + .tex | Either | Update ACTIVE_TEMPLATE, re-run Phase 6 only |
| "Fix this LaTeX error" | — | Debug and recompile Phase 6 only |

---

## Template Update Protocol

If the user provides a new LaTeX template (pasted source or uploaded .tex file):
1. Validate it compiles (run pdflatex with a minimal content stub)
2. If it compiles → set as `ACTIVE_TEMPLATE` for all future Phase 6 runs this session
3. Tell user: "Template updated. All future resume outputs will use this format."
4. Do NOT overwrite `assets/base-template.tex` — that is the permanent fallback

---

## Reference Files

- `references/fresher-market-2025.md` — Market context, salary bands, demand signals, ATS tools used in India, what 20LPA+ companies actually filter on
- `references/rewrite-rules.md` — Banned words, 4 bullet formulas, verb bank, metric injection hierarchy, one-page enforcement (also governs cover letter tone)
- `references/critique-personas.md` — Full scoring rubrics for all 5 reviewer personas
- `assets/base-template.tex` — Akhilesh's canonical LaTeX resume template (structural base for all outputs)

## Drive Interface Contract

**Reads** (Phase 0, optional): `ResumeKit/profile-review-output.json`
**Writes** (Phase 11, always): `ResumeKit/profile-snapshot.json`
**Reads/Writes** (Phase 9 & Application Tracker Mode): `ResumeKit/applications.csv`
**Reads** (Version Diff Mode): `Resumes/[subfolders]/*.tex` — historical versions

The `profile-review` skill reads the snapshot and writes output to the same folder.
Both skills operate independently but share this Drive folder as a persistent memory layer.
