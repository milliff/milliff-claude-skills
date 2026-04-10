# milliff-claude-skills

Custom Claude Code slash commands for political science research workflows.

## Installation

**User-level** (available in all projects):
```bash
cp .claude/commands/replication-package.md ~/.claude/commands/
```

**Project-level** (available only when Claude Code is opened in a cloned copy of this repo):
Just clone the repo — the `.claude/commands/` directory is picked up automatically.

---

## Commands

### `/replication-package [project-directory]`

Guides you through building a submission-ready replication archive following political science journal conventions (APSR/AJPS/IO/Cambridge Elements style).

**What it does:**
- Checks for existing session context before re-auditing anything
- Audits existing code and maps scripts → numbered figures/tables
- Proposes a clean directory structure and gets your approval before touching anything
- Merges, rewrites, and path-fixes scripts into a numbered, ordered set
- Sets up `renv` (R) or `requirements.txt` (Python) for reproducible environments
- Writes a README and LICENSE following journal conventions
- Prepares the archive for GitHub or Harvard Dataverse submission

**Usage:**
```
/replication-package
/replication-package /path/to/project
```

Passing a path skips the "where is your project?" question.

**Designed for:** R-based projects, but covers Python and Stata too. Handles proprietary/licensed data (Gallup, Pew, etc.) with data minimization and pseudocode documentation patterns.
