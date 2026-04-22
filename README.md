# claude-orchestrate

A small Claude Code workflow for driving plan execution with parallel sub-agents in isolated git worktrees — without retyping the orchestration prompt every time.

This is my personal take on how a coding agent should execute a plan: a coordinator that doesn't write code itself, sub-agents that do one scoped thing in isolation, parallel work where it's safe, and a strong bias against exiting the loop for ceremony. Take it, fork it, change the rules to match how you work.

## What this is

Two files:

- **`skills/orchestrate/SKILL.md`** — a Claude Code skill, invoked as `/orchestrate`. It turns the main agent into a coordinator that decomposes a plan into waves of work, delegates to sub-agents (parallelizing when safe), stops only at real checkpoints, and reports back tightly.
- **`agents/implementer.md`** — a Claude Code sub-agent definition. Each unit of work the orchestrator delegates is handed to an `implementer`. The implementer's system prompt bakes in the workflow rules you'd otherwise have to repeat in every prompt: test as you go, all tests green before commit, no `--no-verify`, stay in scope, report results in a tight format.

Together they replace the ritual of typing *"You are the orchestrator. Spin up sub-agents in parallel. Don't stop unless you hit a checkpoint. Sub-agents: test before committing, don't push on red…"* every time you implement a plan.

## Why

If you regularly:

1. Brainstorm an idea with Claude until you've both got it.
2. Have Claude write a plan (in a file or just in the conversation).
3. Want Claude to execute the plan, in parallel where possible, without burning down its own context window or constantly stopping for permission.

…then this gives you a single command (`/orchestrate`) that does step 3 with sane defaults.

## How it works

When you run `/orchestrate`:

1. **Locates the plan** — prefers an argument, then the current conversation, then a clearly-referenced plan file. Won't go hunting for stale `PLAN.md` files in random subdirectories.
2. **Shows a delegation plan** — a tight wave/dependency view (what runs in parallel, what runs sequentially, where checkpoints fall). This is *not* a restatement of the plan. One mandatory check-in here, then it goes.
3. **Spawns `implementer` sub-agents in waves** — each in its own git worktree (`isolation: "worktree"`) on its own branch. Parallel within a wave when the work is disjoint; sequential when it isn't.
4. **Keeps moving.** Stops only for: a real fork in the road, a finding that invalidates the plan, an irreversible action you haven't authorized, or a checkpoint marked in the plan. *Not* for progress updates or routine confirmations.
5. **Reports back** — at the end (and at checkpoints), surfaces the worktree paths/branches you need to review and merge, plus any assumptions made.

The `implementer` sub-agents are the lowest-level worker — they don't spawn their own sub-agents. If a unit of work turns out to be too big, they stop and ask the orchestrator to re-decompose.

## Setup

Both files live under `~/.claude/`. From this repo:

```bash
mkdir -p ~/.claude/skills/orchestrate ~/.claude/agents
cp skills/orchestrate/SKILL.md ~/.claude/skills/orchestrate/SKILL.md
cp agents/implementer.md ~/.claude/agents/implementer.md
```

Or symlink if you want updates from `git pull` to apply automatically:

```bash
mkdir -p ~/.claude/skills ~/.claude/agents
ln -s "$(pwd)/skills/orchestrate" ~/.claude/skills/orchestrate
ln -s "$(pwd)/agents/implementer.md" ~/.claude/agents/implementer.md
```

That's it. In your next Claude Code session, type `/orchestrate` to invoke it.

## What a session looks like

Roughly:

```
You:    [pitch an idea, talk it through with Claude]
You:    "Ask me questions until you're 95% sure what to write down for a plan."
Claude: [questions, you answer, repeat until aligned]
Claude: [writes PLAN.md or just holds the plan in context]
You:    /orchestrate

Claude: Delegation plan:
          Wave 1 (parallel):
            - A: add migration for new users.role column
            - B: scaffold the /admin route group
          Wave 2 (sequential, depends on A):
            - C: wire role-based middleware into existing routes
          Checkpoint after Wave 2.
          Wave 3 (parallel):
            - D: admin dashboard UI
            - E: audit log table
        OK to proceed?
You:    yes

Claude: [spawns A and B in parallel, each in its own worktree]
Claude: Wave 1 done. A → worktree at .../wt-a on branch agent/add-role-migration.
        B → worktree at .../wt-b on branch agent/admin-route-scaffold. Starting Wave 2.
Claude: [spawns C]
Claude: Wave 2 done. Hitting the checkpoint — please review before Wave 3.
You:    [review, merge, approve] go
Claude: [spawns D and E in parallel, reports results]
Claude: All waves done. Want me to review the diffs and merge these branches
        into main? (Otherwise I'll just hand you the worktree paths.)
You:    yes
Claude: [reads each diff, judges it against the plan, merges, cleans up worktrees]
Claude: Merged A, B, C, D into main. Held back E — the audit log table is
        missing the indexed_at column the plan called out. Worktree still at
        .../wt-e on branch agent/audit-log if you want to look.
```

## Usage

The typical flow this is built for:

1. Pitch an idea to Claude. Talk it through until you both understand it.
2. Ask Claude to write a plan — either to a file (`PLAN.md`, `docs/plan.md`, whatever) or just in the conversation. Engineer in checkpoints where you want approval gates.
3. Read the plan, tweak as needed.
4. Type `/orchestrate` (optionally with a path: `/orchestrate path/to/plan.md`).
5. Approve the delegation plan.
6. Let it run. Review the worktree branches it produces and merge them back yourself.

### Tips

- **Mark checkpoints explicitly** in your plan — words like "Checkpoint", "stop here", "review", or a `## Checkpoint` heading all work.
- **Default model** for implementers is inherited from the orchestrator. If you want them cheaper, add `model: sonnet` (or `model: haiku`) to the frontmatter of `agents/implementer.md`.
- **Tools** the implementer has are listed in its frontmatter. It deliberately doesn't have access to spawn sub-agents — it's the lowest-level worker. Edit the `tools:` line if you want to scope it down further.

## Review and integration

At the end of each run (and at each checkpoint), the orchestrator asks once:

> "Want me to review the diffs and merge these branches into main?"

- **Yes:** the orchestrator reads each branch's diff, judges it against the plan, merges the ones that look right, and cleans up their worktrees. Anything that looks wrong is held back with a specific reason and left for you to look at.
- **No:** you get the worktree paths and handle merge + cleanup yourself with standard git commands:

  ```bash
  git worktree list
  git worktree remove <path>
  git branch -d <branch>          # -D if not merged and you're sure
  git worktree prune              # bulk: prune worktrees whose dirs were deleted manually
  ```

The model is: the orchestrator is your reviewer of record when you opt in. It has the project context and knows why each branch exists, so it's well-placed to judge whether a diff did what was asked. If you'd rather review yourself, say no and you get the raw worktrees.

## Customizing

Both files are short and meant to be edited. Read them. If a rule doesn't match how you work, change it — that's the point of having them as version-controlled files instead of muscle-memory typing.

## Requirements

- [Claude Code](https://claude.com/claude-code) (skills and sub-agent definitions are Claude Code features).
- A git repository for the worktree isolation to work. (If you run `/orchestrate` outside a git repo, the worktree isolation just no-ops — you'll get sub-agents but no branch separation.)
