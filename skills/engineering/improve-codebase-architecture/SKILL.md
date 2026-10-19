---
name: improve-codebase-architecture
description: Scan a codebase for deepening opportunities, present them as a visual HTML report, then grill through whichever one you pick.
disable-model-invocation: true
---

# Improve Codebase Architecture

Surface architectural friction and propose **deepening opportunities**: refactors that turn shallow modules into deep ones. The aim is testability and AI-navigability.

This command is _informed_ by the project's domain model and built on a shared design vocabulary:

- Call the Skill tool with "codebase-design" for the architecture vocabulary (**module**, **interface**, **depth**, **seam**, **adapter**, **leverage**, **locality**) and its principles (the deletion test, "the interface is the test surface", "one adapter = hypothetical seam, two = real"). Use these terms exactly in every suggestion, and don't drift into "component," "service," "API," or "boundary."
- The domain language in `GLOSSARY.md` gives names to good seams; ADRs in `docs/adr/` record decisions this command should not re-litigate.

## Process

### 1. Explore

**Scan the code that is actually shipping.** Before anything else, `git fetch` and compare the checkout with the default branch: `git rev-list --count HEAD..origin/<default>` (resolve `<default>` from `git symbolic-ref refs/remotes/origin/HEAD`). If the checkout is behind, explore `origin/<default>`, not the working tree: `git show origin/<default>:<path>`, `git grep <pattern> origin/<default> -- <paths>`, `git ls-tree -r --name-only origin/<default> <dir>`, or a `git archive` extract into a `mktemp -d` directory. Don't switch the user's checkout. Put the ref and the lag into every brief and into the report header. An agent that reads a stale tree reports candidates that were already fixed or rewritten, and the whole zone has to be re-verified.

**Scope before you scan: YAGNI.** Deepening a module pays off by making future changes to it easier, so put extra weight on the parts of the codebase that have recently changed. Decide *where* to look before you look:

- If the user named a direction (a module, a subsystem, a pain point), take it, and skip the inference below.
- Otherwise, walk back a good stretch of the commit history (`git log --oneline`) to find the codebase's hot spots, the files and areas that keep coming up, and let those paths pull your attention first. If the changes are scattered with no clear hot spot, widen the net.

Read the project's domain glossary (`GLOSSARY.md`) and any ADRs in the area you're touching first.

Then spawn sub-agents to walk the codebase, up to 10, sent in a single message so they run concurrently. One agent is right for a repo with a single hot spot. A repo with distinct subsystems gets one agent per subsystem, and each brief names the files that agent owns so two of them don't scan the same tree.

A sub-agent inherits none of this: not the glossary, not `CONTEXT.md`, not the report format. Every brief carries its own copy of:

- **The `/codebase-design` glossary in full**: module, interface, depth, seam, adapter, leverage, locality, plus the deletion test and "one adapter = hypothetical seam, two = real". Skip it and the agents come back talking about components, services and layers, and their findings no longer merge into one report.
- **Read-only discipline**: explore, don't edit, don't build, don't hit the network, don't run the thing being scanned.
- **The ref to read**: `origin/<default>` and how to read it without touching the checkout (see above), whenever the working tree is behind.

**When the user points at data, not only code** ("useless tables in prod", "what bloats the DB"), the storage is part of the scan. Before spawning agents, take one read-only snapshot yourself: table and index sizes, live rows, `seq_scan`/`idx_scan`, `n_tup_ins/upd/del` from `pg_stat_user_tables` and `pg_stat_user_indexes`, matviews, and `stats_reset` so the counters have a known age. Paste that snapshot into the briefs. Give it to every agent whose zone owns tables, and give one dedicated agent the job of mapping every table to its writers and readers in code. That agent gets a narrow exception to "don't hit the network": light read-only queries only (catalog, `pg_stat_*`, `EXPLAIN` without `ANALYZE`, `LIMIT` lookups on indexed columns), no `count(*)` or seq scans on large tables. A write-only table, a matview nobody reads, or a copy of another table's grain is a deletion-test candidate like any shallow module.
- **The files it owns**, plus the neighbouring paths it may read for context but must not report on.
- **The card shape**, per candidate: files with line numbers, problem with a quoted signature or call site as evidence, solution, benefits in locality and leverage terms, a text before/after sketch you can redraw as a diagram, and a strength verdict justified by the deletion test.
- **A closing "Facts" section**: only what the agent measured (copy counts, line counts, call sites, test coverage), no judgement. These numbers are what make the report checkable.
- **Permission to return a negative result.** "These five modules look alike, but the shared module would be shallow" is a finding. It belongs in the report's leave-it-alone section, and it stops the next review from re-proposing it.

Cap each agent at ~4 candidates. Ten agents at ten candidates each is not a report anyone reads.

Don't follow rigid heuristics; explore organically and note where you experience friction:

- Where does understanding one concept require bouncing between many small modules?
- Where are modules **shallow**, with an interface nearly as complex as the implementation?
- Where have pure functions been extracted just for testability, but the real bugs hide in how they're called (no **locality**)?
- Where do tightly-coupled modules leak across their seams?
- Which parts of the codebase are untested, or hard to test through their current interface?

Apply the **deletion test** to anything you suspect is shallow: would deleting it concentrate complexity, or just move it? A "yes, concentrates" is the signal you want.

### 2. Present candidates as an HTML report

**Triage before you write.** Agents at ~4 candidates each hand you more than one report can carry: seven agents come back with something like 26. Rank, then split. The strongest 12–15 get full cards. Everything else goes into one compact table near the end — candidate, the measured fact behind it, strength badge — so the finding survives without costing a diagram. Rank on the deletion test and on the numbers in each agent's Facts section: a candidate with 63 leaking call sites outranks one with two. Don't drop anything to make the split come out even; a cut candidate gets re-proposed by the next review, which is the waste the leave-it-alone section exists to prevent.

Write a self-contained HTML file to the OS temp directory so nothing lands in the repo. Resolve the temp dir from `$TMPDIR`, falling back to `/tmp` (or `%TEMP%` on Windows), and write to `<tmpdir>/architecture-review-<timestamp>.html` so each run gets a fresh file. Open it for the user (`xdg-open <path>` on Linux, `open <path>` on macOS, `start <path>` on Windows) and tell them the absolute path.

The report uses **Tailwind via CDN** for layout and styling, and **Mermaid via CDN** for diagrams where a graph/flow/sequence reliably communicates the structure. Mix Mermaid with hand-crafted CSS/SVG visuals: use Mermaid when relationships are graph-shaped (call graphs, dependencies, sequences), and hand-built divs/SVG when you want something more editorial (mass diagrams, cross-sections, collapse animations). Each candidate gets a **before/after visualisation**. Be visual.

For each candidate, render a card with:

- **Files**: which files/modules are involved
- **Problem**: why the current architecture is causing friction
- **Solution**: plain English description of what would change
- **Benefits**: explained in terms of locality and leverage, and how tests would improve
- **Before / After diagram**: side-by-side, custom-drawn, illustrating the shallowness and the deepening
- **Recommendation strength**: one of `Strong`, `Worth exploring`, `Speculative`, rendered as a badge

End the report with a **Top recommendation** section: which candidate you'd tackle first and why.

**Use GLOSSARY.md vocabulary for the domain, and the `/codebase-design` vocabulary for the architecture.** If `GLOSSARY.md` defines "Order," talk about "the Order intake module," not "the FooBarHandler," and not "the Order service."

**ADR conflicts**: if a candidate contradicts an existing ADR, only surface it when the friction is real enough to warrant revisiting the ADR. Mark it clearly in the card (e.g. a warning callout: _"contradicts ADR-0007, but worth reopening because…"_). Don't list every theoretical refactor an ADR forbids.

See [HTML-REPORT.md](HTML-REPORT.md) for the full HTML scaffold, diagram patterns, and styling guidance.

Do NOT propose interfaces yet. After the file is written, ask the user: "Which of these would you like to explore?"

### 3. Grilling loop

Once the user picks a candidate, call the Skill tool with "grilling" to walk the decision tree with them: constraints, dependencies, the shape of the deepened module, what sits behind the seam, what tests survive.

Side effects happen inline as decisions crystallize; call the Skill tool with "domain-modeling" to keep the domain model current as you go:

- **Naming a deepened module after a concept not in `GLOSSARY.md`?** Add the term to `GLOSSARY.md`. Create the file lazily if it doesn't exist.
- **Sharpening a fuzzy term during the conversation?** Update `GLOSSARY.md` right there.
- **User rejects the candidate with a load-bearing reason?** Offer an ADR, framed as: _"Want me to record this as an ADR so future architecture reviews don't re-suggest it?"_ Only offer when the reason would actually be needed by a future explorer to avoid re-suggesting the same thing; skip ephemeral reasons ("not worth it right now") and self-evident ones.
- **Want to explore alternative interfaces for the deepened module?** Call the Skill tool with "codebase-design" and use its design-it-twice parallel sub-agent pattern.
