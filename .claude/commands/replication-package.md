# Replication Package Builder

You are helping create a submission-ready replication archive following political science journal conventions (APSR/AJPS/IO style). The goal is a clean, self-contained directory that reproduces the numbered publication outputs — figures, tables, or both — by running a single master script (or a small set of clearly ordered independent scripts).

## Step 0: Check for existing project context

Before doing *anything* else, check whether a prior session already captured this project:

1. Look for a `CLAUDE.md` at the project root — if it exists, read it and tell the user what it says so they can confirm whether it is current.
2. Look for a `.claude/` directory at the project root for plan files or session notes.
3. Check `~/.claude/plans/` for any plan files that mention the project directory.
4. If any prior context is found, summarize what was already done and ask the user whether to skip the audit and jump to a later step.

If no prior context exists, proceed to Step 1.

---

## Step 1: Get oriented

Ask the user (skip any question for which the answer is already known from Step 0):

1. **Where is the project?** Get the root directory of the existing analysis code.
2. **What are the publication outputs?** Ask for a list of the numbered figures and/or tables that appear in the paper (e.g., "Figure 2-1 through Figure 5-4", "Tables 2–5"). If they aren't sure, offer to explore the code or a PDF of the paper and infer.
3. **Where does existing code write its outputs?** Find out if there's already a `results/` or `figs/` directory, or if outputs go to many different places.
4. **What language/tools?** R, Python, Stata, or mixed? This determines path-handling and environment-locking strategy.
5. **Are there data licensing or privacy restrictions?** Some papers use proprietary or confidential data that cannot be redistributed in full — only the variables needed to reproduce figures. Ask about both privacy/confidentiality concerns and any data use agreements.
6. **What level of detail does the user want in the README?** Some users prefer a minimal README with just reproduction instructions, while others want a detailed overview of the project, data sources, and figure descriptions. Ask for their preference.
7. **What level of detail does the user (or the journal) want in the replication package?** Some users want a minimal package with the code and data needed to reproduce the figures from a cleaned, minimal dataset. Others want a more comprehensive package that includes the full raw data (if possible), along with documentation and scripts to reproduce all analyses and cleaning. Ask for their preference and any journal requirements.

If `$ARGUMENTS` is provided, treat it as the project directory path and skip asking for it.

---

## Step 2: Audit the existing code

Explore the project directory thoroughly. Build a map of the existing code before cleaning it up.

For each script file found, determine:
- **What data does it load?** (`load()`, `read_csv()`, `read_parquet()`, etc.)
- **What numbered publication outputs does it produce?** Look for `ggsave()`, `tmap_save()`, `savefig()`, `interflex(..., file=)`, `modelsummary(..., output="*.tex")`, `kable()` calls that write to `results/` or similar.
- **What working/exploratory outputs does it produce?** Saves to local `code/part-*/figs/` directories, bare variable-name prints, plots built but never saved.
- **What are the dependencies between scripts?** Does script B load an `.RData` object produced by script A?
- **What publication outputs are missing from the code?** If the paper has a figure or table that doesn't appear to be produced by any script, flag it.

Build a simple inventory:

```
Script              → Numbered outputs          → Working/junk outputs
part-3/part-3.R     → fig3-1 through fig3-9    → ~15 intermediate saves to figs/
part-6/part-6.R     → fig2-7, fig2-8           → (none)
```

Identify removable code within scripts:
- **Bare variable-name print expressions** — a variable name alone on a line (e.g., `myplot`) just displays to the interactive console; remove the line, keep the plot object.
- **Unused plot constructions** — a `ggplot()` or `plt.figure()` block whose output is never passed to `ggsave()` / `savefig()` / `tmap_save()` / `interflex(..., file=)`: remove the entire construction, plus any data manipulation that feeds only that construction.
- **Working figure saves** — `ggsave()` calls writing to local `code/part-*/figs/` directories rather than `results/figs/`.
- **Non-publication table outputs** — `gtsave()`, `save_kable()`, `modelsummary(..., output="*.docx")` that require external tools (pandoc, Chrome) and aren't numbered outputs.
- **Package plot calls without `file=`** — these print to the console only; remove along with all upstream code that feeds only them.
- **Data cleaning or analysis code that is not necessary to produce the final figures** — if a block of code produces an intermediate dataset that feeds only a removed plot, remove the entire block. If it feeds a kept plot, keep it.

---

## Step 3: Plan the new structure

Propose a clean directory layout to the user before touching anything. Standard polisci replication structure:

```
replication-material/
├── README.md                        ← overview, requirements, data sources, how to reproduce
├── LICENSE.md                       ← CC-BY 4.0 or journal-required license
├── main.R                           ← master script: sources all code scripts in order
├── .Rprofile                        ← renv auto-activation (created by renv::init())
├── renv.lock                        ← locked package versions
├── code/
│   ├── 01_section-name.R            ← one script per paper section, numbered
│   ├── 02_section-name.R
│   └── __data-creation-pseudocode.R ← documents restricted-data pipeline (no real data)
├── data/                            ← pre-processed data only (see data minimization below)
└── results/
    ├── figs/                        ← all output figures land here
    └── tabs/                        ← all output tables (.tex, .csv) land here
```

Key decisions to surface to the user:
- **Which scripts to merge?** If multiple scripts feed a single paper section, merge them. If a script logically belongs in an earlier section, move it.
- **What to rename?** Scripts should be numbered in paper-section order.
- **What to delete?** Working figure saves, exploratory plots, intermediate outputs that aren't publication figures or tables.
- **What to keep?** Should data-cleaning code that feeds the final figures, even if it doesn't directly produce a figure itself, be kept, or should the cleaned data be pre-saved and the cleaning code removed? If the user wants a more comprehensive package, keep it; if they want a minimal package, remove it and save the cleaned data instead.
- **Independent scripts vs. single master?** If scripts are fully independent (no shared objects), a master `source()` loop is still preferred for reproducibility. If inter-script dependencies exist (script B loads `.RData` from script A), note the ordering requirement explicitly.

Get explicit approval on the plan before implementing.

---

## Step 4: Create the clean archive

Create a new self-contained subdirectory (e.g., `replication-material/`) so the original code is never modified. Work inside that new directory.

### 4a. Directory skeleton

Create the folder structure. Add a `.here` marker file at the project root (R projects) so `here::here()` resolves correctly regardless of working directory. Also ensure `.Rprofile` is present for renv auto-activation.

### 4b. Merge and rewrite scripts

For each new numbered script, add a standard header block:

```r
# =============================================================================
# Script: 01_section-name.R
# Project: [Paper title / repo]
# Description: Produces fig2-1 through fig2-8 (Part 2: Data Descriptions)
# Dependencies: data/foo.RData, data/bar.csv
# Outputs: results/figs/fig2-*.png
# =============================================================================
```

Then:
- Pull in content from the relevant source scripts **in logical order** (data prep first, then figures in paper order)
- **Fix all file paths**: replace relative paths like `../../data/foo.RData` with `here::here("data/foo.RData")`. All paths should resolve from the project root. Alternatively, paths relative to `./` work if `main.R` sets the working directory at the top — but `here::here()` is preferred.
- **Remove working figure saves**: delete `ggsave()` / `savefig()` calls writing to local `code/part-*/figs/` directories
- **Remove bare console prints**: variable-name-only lines (e.g., `myplot` with no assignment or save)
- **Remove unused plot constructions**: entire `ggplot()` / `plt.figure()` blocks whose output is never saved to a numbered figure, including upstream data manipulation that feeds only those constructions
- **Remove non-publication outputs** (e.g. `gtsave()` or `modelsummary(..., output="*.docx")`) that does not produce a numbered figure or table
- **IF USER SPECIFIES: Keep modeling and data manipulation code** even if it appears to feed removed plots; only remove the plot *construction and display* itself. If user wants a minimal package, remove the code and save the cleaned data instead (see next step).

**Cache expensive computations**: if a script runs matching, simulation, or long model fits, save intermediate results to `data/` as `.RData` and print the code block that produces it in the script, but comment it out and add instructions to load the cached version instead. This way users can reproduce the expensive step if they want, but won't have to if they just want to run the whole thing end-to-end:

```r
# Run once and cache:
# match_out <- MatchIt::matchit(formula, data = df, ...)
# save(match_out, file = here::here("data/match_out.RData"))

# Subsequent runs load the cache:
load(here::here("data/match_out.RData"))
```

**Tables**: write publication tables to `results/tabs/` using `modelsummary(..., output = here::here("results/tabs/table1.tex"))` or equivalent.

### 4c. Create `__data-creation-pseudocode.R`

If the project uses restricted/proprietary data that cannot be shipped, create a pseudocode script documenting the pipeline that produces the pre-processed data files:

```r
# __data-creation-pseudocode.R
# -----------------------------------------------------------------------
# This script CANNOT be run without licensed data access.
# It documents the pipeline that produced the data files in data/.
# -----------------------------------------------------------------------

# 1. Load raw Gallup World Poll microdata (not redistributable)
#    Source: Gallup Analytics portal — requires institutional subscription
#    Variables retained: country, year, wave, approval_us, approval_cn, ...

# 2. Filter to India samples, recode approval variables
# 3. Merge with Pew Global Attitudes data
# 4. Save trimmed file:
#    save(df_india, file = "data/india_merged.RData")
```

### 4d. Create `main.R` (or `main.py`)

```r
# Run from project root
# Reproduces all numbered figures and tables in:
# [Paper title, Year]

library(here)

source(here::here("code/01_section.R"))  # fig2-1 through fig2-8
source(here::here("code/02_section.R"))  # fig3-1 through fig3-9
source(here::here("code/03_section.R"))  # fig4-1 through fig4-4
source(here::here("code/04_section.R"))  # fig5-1 through fig5-4

sessionInfo()
```

### 4e. Copy data files

Copy only the data files actually `load()`-ed or `read_*()`-ed by the new scripts. **Do not copy raw data** — only pre-processed files needed for figures.

**Data minimization** (important for proprietary data): If a dataset has many columns but only a few are used for the publication figures, create a trimmed version. For licensed or confidential data:
- Identify which variables appear in the analysis scripts
- Load the full data once, select only those variables, and save the trimmed version
- Replace the full dataset in the archive with the trimmed one
- Pre-compute any summary statistics for data not used in analyses (e.g., DK rates per question) and ship those instead of individual-level columns that aren't needed
- Clear these decisions with the user.

---

## Step 5: Set up the reproducible environment

**R projects:**

```r
# From the project root, in R:
renv::init()           # creates .Rprofile for auto-activation + initializes lockfile
source("main.R")       # run once to install all packages via renv
renv::snapshot(type = "all", prompt = FALSE)
```

Verify `here` is in the lockfile — it is commonly missed if it was loaded from the system library during setup. If absent:

```r
renv::install("here")
renv::snapshot(type = "all", prompt = FALSE)
```

Check that `.Rprofile` contains `source("renv/activate.R")` so the renv environment activates automatically when R starts in this directory.

**Python projects:** Create `requirements.txt` or `pyproject.toml`. Use `pip freeze > requirements.txt` after a successful run in a clean virtual environment.

**Stata:** Note the version and any user-written commands (`ssc install` etc.) in the README.

---

## Step 6: Write the README and LICENSE

### README

```markdown
# Replication Archive: [Paper Title]
## [Journal / Book Series if applicable]

**Authors:** ...
**Contact:** ...
**Last updated:** [year]

---

## Overview
[One paragraph: what this archive contains, how to run it, what it produces]

---

## Requirements
**Software:** R X.X.X (see `renv.lock` for exact package versions)
**Package management:** renv — run `renv::restore()` before sourcing `main.R`

---

## Data
| File | Description | Source | Access |
|------|-------------|--------|--------|
| data/foo.RData | India survey microdata (trimmed) | Gallup World Poll | Institutional subscription required |
| data/bar.csv   | Pre-computed DK rates by question | Derived from Gallup | Included |

Note: raw/full versions of proprietary data files are not redistributed. See
`code/__data-creation-pseudocode.R` for the pipeline that produced the included files.

---

## Reproducing the Figures

**Recommended (with renv):**
```r
renv::restore()
source("main.R")
```

**Without renv:**
```r
source("main.R")
```

To reproduce a subset of figures, source individual scripts:
```r
source(here::here("code/01_data-descriptions.R"))  # fig2-1 through fig2-8
```

---

## Figure Index
| Figure | Script | Description |
|--------|--------|-------------|
| fig2-1 | 01_data-descriptions.R | Survey coverage bar chart |
| fig3-1 | 02_china-analysis.R    | China approval time series |
| ...    | ...    | ... |

---

## Directory Structure
[Tree of the archive]
```

### LICENSE

Add a `LICENSE.md` at the project root. Cambridge University Press and most polisci journals accept CC-BY 4.0 for replication materials:

```markdown
# License

The code and data in this replication archive are released under the
[Creative Commons Attribution 4.0 International License (CC-BY 4.0)](https://creativecommons.org/licenses/by/4.0/).

You are free to share and adapt this material for any purpose, provided
you give appropriate credit to the original authors.
```

If proprietary data are included under a data use agreement, note any restrictions on the data files specifically.

---

## Step 7: Verify end-to-end

Run `main.R` (or equivalent) from a clean working directory and confirm:
1. No errors
2. All expected output files appear in `results/figs/` and `results/tabs/`
3. Figure and table count matches what the README claims

If errors occur, fix them — common issues:
- Trailing commas in function calls after removing an argument
- References to variables defined in a removed code block
- Missing packages not captured by renv (install + re-snapshot)
- Path errors from `here::here()` not resolving (check that `.here` marker file is present at project root)
- `here` package missing from `renv.lock` (run `renv::install("here"); renv::snapshot(type="all", prompt=FALSE)`)

---

## Step 8: Prepare for submission

Ask the user: **"Do you want to submit this to GitHub (as a public repository) or Harvard Dataverse (as a citable dataset deposit)? Or both?"**

Then follow the appropriate path.

---

### Option A: GitHub repository

1. **Initialize git** (if not already a repo):
   ```bash
   git init
   git branch -M main
   ```

2. **Create `.gitignore`** — exclude large raw data files, OS junk, and renv cache:
   ```
   # R
   .Rhistory
   .RData
   .Rproj.user/
   renv/library/
   renv/staging/

   # Data files too large for git (list individually)
   data/gallup_raw.RData

   # OS
   .DS_Store
   Thumbs.db
   ```

3. **Check file sizes** before committing — GitHub hard-limits files at 100 MB; warn if any data file exceeds ~50 MB and suggest Git LFS or Dataverse instead.

4. **Initial commit**:
   ```bash
   git add README.md LICENSE.md main.R renv.lock .gitignore code/ data/ results/
   git commit -m "Initial replication archive"
   ```

5. **Create the remote repo** (via `gh` CLI):
   ```bash
   gh repo create [repo-name] --public --source=. --remote=origin --push
   ```
   Suggested repo name format: `[lastname]-[shorttitle]-replication` (e.g., `milliff-india-opinion-replication`).

6. **Add a citation block** to the README:
   ```markdown
   ## Citation
   If you use this code or data, please cite:
   [Author(s)] ([Year]). [Title]. [Journal/Publisher]. DOI: ...
   Replication archive: https://github.com/[user]/[repo]
   ```

---

### Option B: Harvard Dataverse

1. **Bundle the archive as a zip**:
   ```bash
   zip -r replication-material.zip replication-material/ \
     --exclude "*/renv/library/*" --exclude "*/.git/*" --exclude "*/.DS_Store"
   ```
   Confirm the zip contains: `README.md`, `LICENSE.md`, `main.R`, `renv.lock`, `code/`, `data/`, `results/` (empty or with pre-generated figures).

2. **Prepare Dataverse metadata fields** — draft responses the user can paste into the deposit form:
   - **Title:** "[Paper Title] — Replication Data and Code"
   - **Authors:** [list with affiliations]
   - **Description:** "Replication archive for '[Paper Title],' [Journal/Publisher], [Year]. Contains R code and data to reproduce all [N] numbered figures and tables. Run `main.R` from the project root. See README.md for full instructions."
   - **Keywords:** political science, replication, [country/topic keywords]
   - **Related publication:** DOI of the paper (add once available)
   - **License:** CC-BY 4.0

3. **Restricted files**: if any data files are under a use agreement (Gallup, Pew, etc.), Dataverse supports restricted file access — deposit them as restricted with a note that users must obtain their own license. The `__data-creation-pseudocode.R` and any summary/derived files can remain public.

4. **Deposit URL**: point to [dataverse.harvard.edu](https://dataverse.harvard.edu) → Log in → "Add Data" → "New Dataset". Add the zip and any stand-alone data files separately (Dataverse previews tabular files if uploaded as `.tab` or `.csv`).

5. **Add the Dataverse DOI** to the README and paper once the deposit is published.

---

### Option C: Both

Do GitHub first (steps A1–A6), then Dataverse (steps B1–B5). Add cross-links:
- README on GitHub: "Archival deposit: [Dataverse DOI]"
- Dataverse description: "Live repository: [GitHub URL]"

---

## Conventions to follow throughout

- One script per paper section, numbered in presentation order
- Single master run script at project root — running it reproduces everything
- All outputs go to `results/figs/` or `results/tabs/`; nothing written inside `code/`
- No absolute paths anywhere in the scripts; use `here::here()` throughout
- Standard header block at the top of every script (inputs, outputs, description)
- `__data-creation-pseudocode.R` documents any restricted-data pipeline
- `renv.lock` (or equivalent) committed alongside the code; verify `here` is in it
- `LICENSE.md` at project root
- README at project root with complete reproduction instructions and figure index
