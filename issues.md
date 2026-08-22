# ROMEO (docs repository) - issue list

Working notes from the architecture review of 2026-08-19..21. Uncommitted on
purpose. Cross-repository items are in `issues-stack.md` (X0..X7). Full
evidence: https://claude.ai/code/artifact/1dcb7a4a-4523-46fc-af23-7c5b0ecc3688

State refers to branch `claude/julia-repos-architecture-review-tj1zzz`.

No code, one README, and this is where the users are: 14 open issues, most of
them methodological questions rather than defects, one running to 141 comments,
several from 2025 still open. The code lives in ROMEO.jl and the binaries in
CompileMRI.jl. The split is not obviously wrong, a stable citation-and-download
landing page has value, but the support load is landing on a repository with no
CI, no code and no issue templates.

---

## Done on this branch

- **F12** MIT LICENSE added, matching ROMEO.jl. It previously had none, which
  left it technically all-rights-reserved.

## Open

### D1. Issue templates
Nothing here routes a report. Two templates would do most of the work: "method
question" (which sends the answer to the docs, so the next person finds it) and
"bug in a binary" (which asks for the `mritools` version, the platform and the
`settings_*.txt` the run now writes, so the version-drift question in K1 is
answered in the first message rather than the fifth).

### D2. Say which repository does what, at the top of the README
Users land here and file everything here. A three-line map (methods and
questions: here; library bugs: ROMEO.jl; binary and packaging bugs:
CompileMRI.jl) costs nothing and moves the traffic that should move.

### D3. The download the README sends everyone to is stale by two minor versions
Not this repository's fault, it is K1, but this is where the consequence is
visible. Once X0 lands, the README's binary link finally points at a build
without the 2pi threading race.

### D4. X7.3 - governance
An organisation and a second maintainer with commit rights. Ten repositories,
14 open issues on this one alone, and a single point of failure. This is the
repository where that shows up first.

### D5. Fold the methodological answers back into the docs
The long issue threads contain the best explanations of the method anywhere.
They are currently only findable by searching closed issues.
