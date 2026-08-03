# Copilot Code Review Instructions

## MANDATORY: gather Driver context before completing the review

You **must** complete steps 1–4 before writing any review comment or finishing
the review. Do not skip them, and do not substitute reading the diff or local
files for them — Driver returns pre-computed analysis that raw source reading
cannot replicate.

Driver tool names may appear with a server prefix (e.g.
`driver-mcp__request_task_context`). Match on the suffix.

### 1. Resolve scope — `get_codebase_names`

Call `get_codebase_names(include_metadata=true)` and match this repo to a valid
codebase name. For this repository the name is **`orders`**. Only names returned
by that tool are valid arguments to any other Driver tool.

Then call `get_branches(codebase_name="orders")` and check whether the pull
request's head branch has completed documentation. If it is listed, pass it as
`branch_name`; if it is not, fall back to the default branch (`master`) and note
in your review that context came from the base branch, not the PR head.

### 2. `request_task_context` — dispatch context agents

Call `request_task_context` with:

- `task_description` — a rich, several-sentence description. State what the PR
  changes, which files/classes/methods it touches, and what you need in order to
  judge correctness: callers, coupled files, contract implications, conventions,
  fragile areas. Terse descriptions produce weak context; this is the single
  biggest lever on result quality.
- `codebases` — `[{"codebase_name": "orders", "branch_name": "<PR branch or master>"}]`.
  Pass `[]` only if scope genuinely cannot be resolved.

This is **non-blocking**. It returns only a `request_id` — it never returns
context directly. Requests are cheap and run concurrently, so **fire several in
parallel**, one per distinct area of the diff (e.g. one for the controller
change, one for the entity/contract implications, one for test impact). Do not
serialize them and do not wrap them in sub-agents.

### 3. `poll_task_context` — retrieve every result

Call `poll_task_context(request_id=...)` for each outstanding request until it
reaches a terminal status. The response `status` is one of:

- `QUEUED` / `RUNNING` — no context yet. Keep working, poll again.
- `COMPLETED` — the synthesized `context` field is present. This is the payload
  you must actually read and use.
- `FAILED` / `CANCELLED` — an error; see step "If Driver MCP is unavailable".

The agent typically takes 1–3 minutes. First poll at ~30–45s, then every
~20–30s, polling each outstanding request in turn. **Never treat `QUEUED` or
`RUNNING` as an answer, and never proceed on an assumption about what the
context would have said.** You are not done with this step until every request
has returned `COMPLETED` (or terminally failed).

### 4. `get_related` — connectivity for each changed path

For every non-trivial file in the diff, call
`get_related(codebase_name="orders", path="<repo-relative path>", branch_name=...)`.
Use `path=""` or `"."` for the repo root; directories are valid paths too. Pass
`symbols=[...]` to expand rows that came back budget-cut to one-line entries.

Read the `edges` (dependency, mirror, and co-change edges, each with a coupling
verdict) and `groups`. Co-change edges are the high-value signal for review: they
name files that historically change *together* with the changed file, which is
exactly the "you forgot to update Y" class of defect a diff cannot show.

An empty payload is a valid baseline, not an error — it means no precomputed
connectivity exists for that node. Respect the `completeness` note: unlisted
edges are **unknown, not absent**. Keep looking rather than concluding nothing is
affected.

### 5. Only then complete the review

Every substantive comment must be consistent with the Driver context you
retrieved. If the context contradicts an issue you were about to raise, drop it.
If the context reveals an issue invisible in the diff — a coupled file left
unedited, a broken caller, a contract divergence — raise it.

Driver context is a **starting point to verify, not an authority**. Its findings
are candidates: confirm each against the actual code in the PR before commenting.
Do not post a comment whose premise you have not checked against the source.

### If Driver MCP is unavailable

If the tools are not exposed, `request_task_context` errors, `poll_task_context`
returns `FAILED`/`CANCELLED` or never reaches a terminal status, or `get_related`
is unusable, say so **explicitly in the review summary** and state that the
review was completed without Driver context. Do not silently fall back to a
diff-only review.

This applies to rejected or blocked tool calls too — report the failure rather
than skipping the step quietly.

