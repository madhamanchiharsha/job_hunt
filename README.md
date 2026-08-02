# Job Hunt Workspace

This folder is a workspace for the **`linkedin-job-hunt`** Claude Code
plugin. It runs in **two human-separated phases** so saving to LinkedIn is
never a scripted burst (which is what risks bot detection):

1. **Discover** (`/job-hunt`) — searches your target companies, scores each
   job against your resume, and writes a reviewable folder + plain-English
   summary for every matching job. **This phase never touches LinkedIn —
   it only reads and writes files.**
2. **Review** (`/job-hunt phase=review`) — walks you through those jobs one
   at a time. Only when *you* say "save" for a specific job does it tailor
   a resume for it and save it to your LinkedIn (with a small random
   delay). You drive the pace.

You start with:

```
/job-hunt                 # phase 1: discover (read-only)
/job-hunt phase=review    # phase 2: review + save, when you're ready
```

---

## What you maintain (edit these)

These three files are the only inputs you own. Everything else is
generated.

| File | What it is |
|---|---|
| `resume/base.tex` | Your canonical LaTeX resume. Must keep the three `TAILOR` sentinel pairs intact. The plugin never edits this file — it copies it per job and edits only inside the sentinels. |
| `preferences.yaml` | Your filters (location, remote, salary, seniority, avoid-keywords) and scoring thresholds. |
| `target-companies.yaml` | Companies to search, each with optional keywords. |

### The resume sentinels (important)

`base.tex` must contain these three pairs, exactly. The tailor agent
edits only the text between each pair and treats the comment lines as
sacred — do not rename, move, or delete them:

```latex
% <<TAILOR-START: SUMMARY>>   ...   % <<TAILOR-END: SUMMARY>>
% <<TAILOR-START: SKILLS>>    ...   % <<TAILOR-END: SKILLS>>
% <<TAILOR-START: BULLETS>>   ...   % <<TAILOR-END: BULLETS>>
```

Everything outside the sentinels (header, section titles, company names,
dates, education) is frozen and never tailored.

---

## What the plugin generates (don't hand-edit)

Each **discover** run creates the next numbered iteration under
`jobs/itr_N/` (N = 1, 2, 3…), so you can run it repeatedly and compare.

| Path | Phase | Purpose |
|---|---|---|
| `.job-hunt/skills-profile.json` | discover | Structured profile extracted from `base.tex`. Cached by file hash — regenerated only when `base.tex` changes. |
| `jobs/itr_N/jobs.json` | discover | Normalized LinkedIn search results for this iteration. |
| `jobs/itr_N/scored-jobs.json` | discover | Each job scored + a `recommend` / `close-fit` / `skip` decision. |
| `jobs/itr_N/iteration.json` | discover/review | Manifest: per-job status, counts, novelty (new vs seen in earlier iterations), save status. |
| `jobs/itr_N/overview.md` | discover/review | Human-readable iteration report — counts, deltas vs prior runs, and a ranked table of jobs. **Start here.** |
| `jobs/itr_N/<company>/<job_id>/description.md` | discover | The full job description as clean markdown. |
| `jobs/itr_N/<company>/<job_id>/overview.md` | discover | Plain-English summary: what the role is, why it matches you, the gaps. Your decision aid. |
| `jobs/itr_N/<company>/<job_id>/job.json` + `fit.json` | discover | Raw posting + scored entry (used by the tailor/saver later). |
| `…/<job_id>/resume.tex` + `resume.pdf` + `tailor-diff.md` | review | Resume tailored to that job — **only created if you choose to save it.** |
| `…/<job_id>/fit-analysis.md` | review | Why it's a fit + matched/unmet skills. Write-once — safe to add your own notes. |
| `missing-skills.md` | review | Human-readable ledger of skills holding back close-fit jobs you skipped. A learning to-do list. |
| `missing-skills.jsonl` | review | Machine-readable mirror (one JSON record per job) for analysis with DuckDB / `jq`. Stays in lockstep with the `.md`. |

---

## How it runs — two phases

### Phase 1 — discover (`/job-hunt`) — read-only on LinkedIn

1. **Profile** — extract skills/experience from `base.tex` (skips if unchanged).
2. **Search** — fan out LinkedIn searches over your companies + core skills.
3. **Score** — rate each job against your profile and preferences.
4. **Summarize** — for every `recommend` + `close-fit` job, write a folder
   with the JD and a plain-English summary, plus the iteration `overview.md`.

No LinkedIn writes happen here. Review the results in
`jobs/itr_N/overview.md` and the per-job `overview.md` files at your leisure.

### Phase 2 — review (`/job-hunt phase=review`) — you drive the saving

The plugin walks the discovered jobs one at a time (recommend first),
shows you each summary, and asks **save / skip / open / quit**. For each
job you choose to **save**, it tailors a resume PDF for that job and saves
it to LinkedIn after a random 1–30s pause. You can quit anytime and resume
later — already-saved jobs are detected and never double-saved.

**Nothing is ever saved without your per-job "save".**

---

## Quick start

1. **Install prerequisites**
   - [`tectonic`](https://tectonic-typesetting.github.io/) on your PATH
     (compiles the LaTeX resume). Verify: `which tectonic`.
   - LinkedIn MCP authenticated in Claude Code. The plugin checks this
     and will tell you to authenticate if needed.

2. **Fill in your inputs**
   - Replace the placeholder content in `resume/base.tex` with your real
     resume — keeping the sentinels.
   - Edit `preferences.yaml` (locations, salary, seniority, keywords).
   - Edit `target-companies.yaml` (your target companies).
   - Confirm it compiles: `cd resume && tectonic base.tex`.

3. **Discover**
   ```
   /job-hunt
   ```
   Then open `jobs/itr_1/overview.md` and browse the per-job summaries.

4. **Review + save** when you're ready (can be minutes or days later)
   ```
   /job-hunt phase=review
   ```
   Say `save` / `skip` / `open` / `quit` per job. Saved jobs get a tailored
   PDF in their folder and land on your LinkedIn saved-jobs list.

---

## Commands

| Command | What it does |
|---|---|
| `/job-hunt` | Run discovery (the default). Creates the next `jobs/itr_N/`. |
| `/job-hunt phase=discover` | Same as above, explicit. |
| `/job-hunt phase=review` | Review the **latest** iteration and save jobs you approve. |
| `/job-hunt phase=review itr=2` | Review a specific iteration by number. |

Review is resumable: quit partway through and re-run it later — pending
jobs remain pending, saved jobs are skipped (no double-saves).

> `tectonic` is only needed for **review** (it builds the resume PDFs).
> Discovery doesn't need it.

---

## Folder layout

```
job_hunt/
├── README.md                 # this file
├── preferences.yaml          # you edit
├── target-companies.yaml     # you edit
├── resume/
│   └── base.tex              # you edit (keep the sentinels)
├── .job-hunt/                # generated: cached skills profile
├── jobs/                     # generated: one folder per discover run
│   ├── itr_1/
│   │   ├── jobs.json
│   │   ├── scored-jobs.json
│   │   ├── iteration.json    # manifest: status, counts, novelty
│   │   ├── overview.md        # iteration report — start here
│   │   └── <company>/<job_id>/
│   │       ├── description.md # full JD
│   │       ├── overview.md     # why it matches / gaps (decision aid)
│   │       ├── job.json
│   │       ├── fit.json
│   │       ├── resume.tex     # review only, if you save this job
│   │       ├── resume.pdf     # review only
│   │       ├── tailor-diff.md # review only
│   │       └── fit-analysis.md# review only
│   └── itr_2/ ...
├── missing-skills.md        # generated: skill-gap learning ledger (human)
└── missing-skills.jsonl     # generated: same data, queryable (DuckDB/jq)
```

---

## Analyzing your skill gaps

Once a few runs have populated `missing-skills.jsonl`, find the skills
blocking the most close-fit jobs — your highest-leverage things to learn.

With **DuckDB** (recommended — `brew install duckdb`):

```sql
SELECT skill, count(*) AS jobs
FROM (SELECT unnest(missing_required_skills) AS skill
      FROM 'missing-skills.jsonl')
GROUP BY skill ORDER BY jobs DESC;
```

Or with **`jq`** alone (no install if you have it already):

```sh
jq -r '.missing_required_skills[]' missing-skills.jsonl \
  | sort | uniq -c | sort -rn
```
