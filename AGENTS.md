# AGENTS.md

Guidance for AI coding agents working in this repository. `CLAUDE.md` imports this file; keep guidance here.

## This repository is public

Everything here is world-readable and permanent: code, comments, commit messages, branch names, PR titles and descriptions, review replies and issue comments. Forks and archives keep copies after a force-push, and GitHub keeps every earlier version of an edited PR description where anyone can read it. So never write any of the following, in the repo or on GitHub:

- names of Elfa's private repositories, or links to their PRs, issues or commits (`org/repo#123`, `repo#123`, URLs)
- internal service names or codenames. Name the public thing instead: "the API", "Auto", "the published spec", "docs.elfa.ai"
- paths into private source code, internal hostnames, staging or preview URLs
- teammates' names, internal threads or decisions ("X confirmed in-thread")
- anything not yet live on docs.elfa.ai: unreleased, flag-gated or staging-only work
- deployment, infrastructure, incident or business detail
- secrets, local machine paths, personal editor or agent config (`.mcp.json`, `.claude/`)

To say where a change came from, cite the public source ("matches the spec published on docs.elfa.ai"), not the internal change behind it. Before pushing, reread the diff, commit messages and PR description as an outsider would. If a sentence would tell a competitor something about how Elfa builds software, cut it.
