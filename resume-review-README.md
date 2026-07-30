# resume-review

A Claude Skill that runs a full resume pipeline for early-career software roles — critique,
JD tailoring, ATS scoring, LaTeX/PDF generation, cover letters, interview prep, version
diffing, and application tracking — with a shared Google Drive layer so it stays in sync
with the companion [`profile-review`](../profile-review) skill.

Built for targeting 20 LPA+ SDE-I roles as a fresher/early-career candidate, but the pipeline
generalizes to any resume review workflow.

---

## What it does

Give it a resume (PDF, DOCX, or LaTeX) with or without a job description, and it runs:

1. **Structured ingest** — parses the resume into a normalized model (header, education,
   experience, projects, skills, positions, open source)
2. **Quick diagnostic** — flags issues immediately: weak bullets, unsupported skills, missing
   links, length problems
3. **JD gap analysis** *(if a JD is provided)* — maps the resume against the JD across 6
   dimensions: keyword match, stack alignment, seniority signal, impact depth, domain fit,
   soft signals
4. **Rewrite engine** — scores and selects top projects, rewrites bullets with a banned-word
   filter to avoid "AI-sounding" phrasing, enforces a strict one-page layout
5. **Multi-persona critique** — scores the resume out of 100 across 5 lenses: ATS bot, HR
   screener, technical reviewer, hiring manager, peer engineer
6. **Interview-prep talking points** — flags every bold or hard-to-verify metric and generates
   an honest, non-fabricated explanation you can actually defend in an interview
7. **LaTeX + PDF output** — compiles against a canonical template, escapes special characters,
   keeps everything to one page
8. **Cover letter generation** *(tailored mode)* — a short, non-generic cover letter built from
   the same gap analysis, never restating the resume verbatim
9. **Google Drive push** — versions the PDF, `.tex` source, and cover letter into a dated
   subfolder
10. **Application tracker** — logs company, role, resume version, and status to a shared
    `applications.csv`, and can report back on it anytime
11. **Version diffing** — compares any two resume versions (e.g. your TCS vs. Eurofins variant)
    and — critically — flags when the *same* claim has drifted to a different number across
    versions, which is the kind of inconsistency an interviewer or reference check would catch

## Modes

| Mode | Trigger | What runs |
|------|---------|-----------|
| **Tailored** | JD provided | Full pipeline, including cover letter and application logging |
| **General** | No JD | Critique + rewrite + interview prep, no gap analysis or cover letter |
| **Version Diff** | "compare this to my last [Company] resume" | Standalone — no full run needed |
| **Application Tracker** | "show my applications" / "log this application" | Standalone — no full run needed |

## Core principles

- **Anti-fabrication** — never invents metrics, skills, or responsibilities. Missing data
  becomes an explicit `[FILL: ...]` placeholder, never a guess.
- **Keyword fidelity** — matches JD terms exactly, no paraphrasing that would break ATS parsing.
- **Interview-prep honesty** — talking points describe what actually happened, including
  "this was an estimate" when that's the truth. The goal is a defensible answer, not a
  confident one.
- **LaTeX integrity** — the structural template is never touched (fonts, spacing, packages)
  unless explicitly asked.

## Output

- `[LastName]_[FirstName]_[Role]_[Company]_[YYYY-MM].tex` + compiled `.pdf`
- `[LastName]_[FirstName]_CoverLetter_[Company]_[YYYY-MM].txt` (tailored mode)
- A resume health score out of 100 with the top 3 highest-ROI fixes
- An interview-prep sheet mapping every bold claim to an honest explanation
- A Drive-synced `applications.csv` tracker

## Data contract with `profile-review`

This skill and `profile-review` share a `ResumeKit/` folder in Google Drive as a lightweight
persistent memory layer:

| File | Written by | Read by |
|------|-----------|---------|
| `profile-snapshot.json` | resume-review | profile-review |
| `profile-review-output.json` | profile-review | resume-review |
| `applications.csv` | resume-review | resume-review |

Each skill runs fully standalone if the other's data isn't present — the cross-referencing is
an enrichment layer, not a hard dependency.

## Reference files

```
resume-review/
├── SKILL.md
├── assets/
│   └── base-template.tex        # canonical LaTeX resume template
└── references/
    ├── fresher-market-2025.md   # market context, salary bands, ATS filters
    ├── rewrite-rules.md         # banned words, bullet formulas, verb bank
    └── critique-personas.md     # scoring rubrics for all 5 reviewer personas
```

## Requirements

- `pdflatex` (texlive-latex-base, texlive-fonts-recommended/extra) for PDF compilation
- Google Drive connector for versioning, the applications tracker, and the profile-review
  data contract (optional — the skill degrades gracefully without it)
