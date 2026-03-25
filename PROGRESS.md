# Gleeds Line Management Tool — Progress Notes

## Overview
Single-page app (`index.html`) for the Gleeds Infrastructure Cost Management team.
Deployed to Netlify via GitHub auto-deploy.
Repo: https://github.com/AdamGriffith00/LineManagementTool

---

## What the tool does

- **Import** CSV or Excel files of employee data (flexible column name aliases)
- **People table** — searchable, sortable, shows name, grade, office, region, manager, capacity, current reports
- **Find a Manager** — filter and score potential managers for a new hire, optionally scoped to a named person (discipline-aware, seniority-aware, office/region scored)
- **Line Managers by Region** — browse managers by region, office, and capacity status
- **Organigram** — visualise the hierarchy in four modes: Overall, By Region, By Office, Find Person
- **Manager preferences** — click any name to mark whether someone wants/doesn't want to manage; persists across uploads
- **Change detection** — on re-upload, shows a modal diff of who was added/removed and what changed
- **Data stored in localStorage** — no backend, no login

---

## Grade hierarchy

Director levels are shared across all disciplines. Levels 4–9 are discipline-specific but equivalent in seniority.

| Depth | Cost Management (CM / QS) | Project Management (PM) |
|---|---|---|
| 0 | Senior Director | ← shared → |
| 1 | Director | ← shared → |
| 2 | Project Director | ← shared → |
| 3 | Associate Director | ← shared → |
| 4 | Executive Cost Manager / Executive QS | Executive Project Manager |
| 5 | Senior Cost Manager / Senior QS | Senior Project Manager |
| 6 | Cost Manager / Quantity Surveyor / QS | Project Manager |
| 7 | Assistant Cost Manager / Assistant QS | Assistant Project Manager |
| 8 | Graduate Cost Manager / Graduate QS | Graduate Project Manager |
| 9 | Trainee Cost Manager / Trainee QS | Trainee Project Manager |

**CM and QS are the same discipline** — just different naming conventions. The tool treats them as identical throughout.

---

## Key implementation notes

### Name matching
- `canonicalKey(name)` — strips accents, normalises spacing, handles "Surname, Firstname" format
- `canonicalName(name)` — simpler lowercase + whitespace normalise (used in organogram)
- For ASCII names both produce the same result

### Nickname resolution (`NICKNAME_GROUPS` / `addNicknameAliases`)
- ~40 groups of equivalent first names (Sam↔Samantha, Dan↔Daniel, etc.)
- After building the exact `byName` map, a `nicknameByKey` map adds variant-key → person entries for any variant not already an exact match
- Used in: report counting, orphan detection (warnings panel + table rows), organogram `buildHierarchyForGroup`
- Avoids false positives by only adding a variant if no exact match already exists

### Grade functions
| Function | Purpose |
|---|---|
| `gradeForCapacity(grade)` | Maps bare professional titles (Cost Manager, QS, Project Manager) → 'Professional' for capacity defaults |
| `gradeDefaultCapacity(grade, gradeLevel)` | Returns default capacity by grade keywords (senior→4, executive→5, etc.) |
| `simplifyGrade(grade)` | Collapses full title to level label (Senior Cost Manager → Senior) for filter dropdowns |
| `gradeDepthIndex(grade)` | Returns 0–9 depth for organogram column placement; handles CM, QS, and PM variants |
| `normalizeGradeForOrder(grade)` | Maps QS and PM titles → CM equivalents before GRADE_ORDER index lookup |
| `gradeIndex(grade)` | Looks up GRADE_ORDER index after normalisation |
| `getDiscipline(grade)` | Returns `'Director'`, `'CM'`, or `'PM'` |

### Discipline filtering (Find a Manager)
- When a target person is named, candidates are filtered to the same discipline
- Director-level managers (Associate Director and above) are shown for all disciplines
- CM/QS share a discipline — a CM person will see QS managers and vice versa

### Data flow
1. File uploaded → `processObjects(objs)` → builds `state.people`, `state.nicknameByKey`
2. `renderAll()` called → updates table, warnings, find-a-manager options, organogram
3. Organogram built by `buildHierarchyForGroup(groupPeople)` → `renderOrgTreeDirect()`

---

## Session history

### Session 1 (initial build)
Full tool built from scratch:
- CSV/XLSX import with flexible column alias detection
- People table with search and sort
- Capacity rules by grade
- Find a Manager with multi-select filters and recommendation scoring
- Organogram with four view modes (Overall, Region, Office, Find Person)
- Click-to-highlight relationships in organogram
- Zoom controls
- Manager preferences (click name → set wants/doesn't want to manage + notes)
- localStorage persistence
- Change detection modal on re-upload
- Orphan and circular reference warnings

### Session 2 (25 March 2026)

**Three issues raised by user feedback:**

#### 1. Nickname / "known as" name mismatch causing false orphans
- Problem: data export uses full names (e.g. "Samantha Leather") but the manager column uses known-as names (e.g. "Sam Leather"), so people were shown as orphans incorrectly
- Solution: built-in `NICKNAME_GROUPS` table (~40 groups) + `addNicknameAliases()` helper
- Applied to: report counting, orphan detection (both warning panel and table), organogram hierarchy building
- No new column required in the data export

#### 2. QS/CM grade title mix causing incorrect sort order
- Problem: `GRADE_ORDER` only contained CM titles, so QS titles (Senior QS etc.) fell to the bottom of organogram sort order
- Solution: `normalizeGradeForOrder()` maps QS → CM (and now PM → CM) before `GRADE_ORDER` lookup
- `gradeDepthIndex` already handled QS variants for organogram depth — confirmed working

#### 3. Line manager by region search — deferred
- Data only has North/South regions, limiting usefulness
- No code change needed; will become more useful when data has finer regions

#### 4. Project Manager discipline support
- PM grades added at equivalent levels to CM/QS throughout all grade functions
- `getDiscipline(grade)` added — returns `'Director'`, `'CM'`, or `'PM'`
- Find a Manager now discipline-aware: CM/QS targets only see CM/QS managers; PM targets only see PM managers; director-level managers shared across both

---

## Pending / future work

- **Project Controls** — discipline support deferred; grade structure is not as clear-cut as CM/QS or PM. Revisit once grade structure is confirmed.
- **Line managers by region** — will become more useful once data export includes finer regional breakdown beyond North/South.
- **Known-as column** — no code change needed (nickname matching handles it), but worth noting in case edge cases appear with less common names not in `NICKNAME_GROUPS`. Add names to the groups array as needed.
