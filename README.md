# Agent context

Persistent, git-backed context for agent sessions on this project
(crimson/SeaStore evaluation + Ceph fleet work).

## How to use
1. **Read `MEMORY.md` first.** It opens with the **active assignment** for a new
   agent, the bootstrap facts (which host, which ssh key, the mgr bypass), and the
   two safety rules that have already cost this project three machines. Below that
   it is an index, one line per fact.
2. Open the relevant `memory/*.md` files for detail.
3. Full narrative writeup: `crimson-seastore-vs-bluestore-eval.md`.

Each `memory/<slug>.md` holds **one fact** with frontmatter:

```
---
name: <slug>
type: user | feedback | project | reference
---
<the fact>
```

- `user` — who the user is / preferences.
- `feedback` — how to work (corrections, confirmed approaches); include the why.
- `project` — ongoing work, findings, state not derivable from code/git.
- `reference` — pointers to hosts, repos, artifacts, external resources.

When you learn something durable, add a `memory/*.md` file and a one-line
pointer in `MEMORY.md`. Keep one fact per file; update rather than duplicate.
