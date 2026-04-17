<<<<<<< HEAD
# team-mode notepad — learnings

## Reference Projects (MANDATORY for all implementers)
- `/Users/yeongyu/local-workspaces/free-code` — read-only reference; look up Claude Code Agent Teams native patterns.
- `/Users/yeongyu/local-workspaces/opencode` — read-only reference; opencode plugin API, server types, session API.

## Worktree Setup
- Worktree: `/Users/yeongyu/local-workspaces/omo-wt/team-mode` (branch: `feat/team-mode` off `dev`)
- `.sisyphus/` is gitignored except `rules/`. Plan copy at `.sisyphus/plans/team-mode.md`. Evidence at `.sisyphus/evidence/team-mode/`.
- Base branch for PR: `dev`.

## Key Architectural Decisions (Momus-approved iteration 7)
1. **Spawn path**: `BackgroundManager.launch()` ONLY. Do NOT call `session.create()` separately — it is done internally.
2. **Ack lifecycle**: `poll.ts` NEVER acks messages. Records `pendingInjectedMessageIds` in durable `RuntimeState`. `session.idle` hook (Task 21) calls `ackMessages()` after turn completes — at-least-once per D-15.
3. **Tool registration**: team_* tools register GLOBALLY via plugin `ToolRegistry` when `team_mode.enabled=true`. Skill has NO `mcpConfig`. Access gated by `teamToolGating` hook (role-based: lead/member/target-member/neither).
4. **Team spec storage**: `~/.omo/teams/{name}/config.json` (directory structure, not single file).
5. **Task storage**: `~/.omo/runtime/{teamRunId}/tasks/{id}.json` (individual JSON files, NOT JSONL).
6. **Mailbox**: `~/.omo/runtime/{teamRunId}/inboxes/{memberName}/{messageUuid}.json` (per-recipient dir, immutable files).
7. **Lock file format** (§III.7): EXACTLY 3 lines — `<ownerMemberName>\n<ownerPid>\n<acquiredAtEpochMs>`
8. **worktreePath**: filesystem path only (`./`, `../`, `/` start). Bare branch names REJECTED.
9. **team_create gating**: ONLY `neither` (non-participating sessions) can call `team_create`. Lead/member rejected.
10. **Approve/reject shutdown**: target member OR lead can call. Other members rejected.
11. **Skill mcpConfig**: NEVER add. Permanent decision.

## 2026-04-18 Task 4: lock utilities
- Advisory locking uses mkdir-based exclusive lock directory with `owner` file inside the lock dir.
- Canonical owner format is exactly 3 lines: owner tag, pid, acquired timestamp.
- `atomicWrite()` uses tmp file + fsync + rename and cleans tmp on failure.

## Key Constants
- `max_members: 8`, `max_parallel_members: 4` (D-25)
- `message_payload_max_bytes: 32768` (32KB, D-06)
- `recipient_unread_max_bytes: 262144` (256KB, D-06b)
- `mailbox_poll_interval_ms: 3000`
- `member_delegate_task_budget: 0` (D-13)

## File Target Paths
```
src/config/schema/team-mode.ts
src/features/team-mode/
  types.ts
  index.ts
  team-registry/{paths,loader,validator}.ts
  team-state-store/{store,locks,resume}.ts
  team-mailbox/{inbox,send,poll,ack}.ts
  team-tasklist/{store,claim,update,dependencies,get,list}.ts
  team-worktree/{manager,cleanup}.ts
  team-layout-tmux/layout.ts
  team-runtime/{create,shutdown,status,resolve-member}.ts
  tools/{lifecycle,messaging,tasks,query}.ts
  deps.ts
  integration.test.ts
src/features/builtin-skills/skills/team-mode.ts
src/hooks/team-mailbox-injector/hook.ts
src/hooks/team-tool-gating/hook.ts
src/hooks/team-session-events/
  team-lead-orphan-handler.ts
  team-member-error-handler.ts
  team-idle-wake-hint.ts
src/cli/doctor/checks/team-mode.ts
```

## Existing Files to Reference
- `src/config/schema/openclaw.ts` — closest analog pattern for team_mode config.
- `src/config/schema/oh-my-opencode-config.ts` — root schema (add team_mode field).
- `src/plugin-config.ts` — deepMerge (add team_mode entry).
- `src/plugin-handlers/tool-config-handler.ts` — Task 5 D-36 fix (hephaestus teammate:"allow").
- `src/features/builtin-skills/skills/playwright.ts` — gating pattern (Task 7 model; NO mcpConfig mirror).
- `src/features/builtin-skills/skills.ts` — `createBuiltinSkills()` (add teamModeEnabled param).
- `src/features/builtin-skills/skills/index.ts` — barrel.
- `src/config/schema/agent-names.ts` — BuiltinSkillNameSchema ("team-mode").
- `src/plugin/skill-context.ts` — skill context builder (thread teamModeEnabled).
- `src/tools/delegate-task/category-resolver.ts` — `resolveCategoryExecution` (Task 13 reuse).
- `src/tools/delegate-task/subagent-resolver.ts` — `resolveSubagentExecution` (Task 13 reuse).
- `src/tools/delegate-task/prompt-builder.ts` — `buildSystemContent` (D-44 mandatory reuse).
- `src/features/background-agent/manager.ts` — `BackgroundManager.launch()` (Task 16 ONLY spawn path).
- `src/features/background-agent/spawner.ts` — spawn pattern.
- `src/features/tmux-subagent/manager.ts` — `TmuxSessionManager` (Task 14 reuse).
- `src/features/context-injector/injector.ts` — `createContextInjectorMessagesTransformHook` (Task 19 pattern).
- `src/openclaw/session-registry.ts` — durable JSON + advisory locking (Tasks 4, 9, 11 ref).
- `src/openclaw/reply-listener-paths.ts` — path resolution pattern (Task 3 ref).
- `src/plugin/messages-transform.ts` — transform hook registration order (Task 19).
- `src/plugin/tool-execute-before.ts` — tool.execute.before (Task 20).
- `src/plugin/event.ts` — event handler registration (Task 21).
- `src/index.ts` — plugin init (Task 27).
- `src/tools/delegate-task/tools.ts:28-260` — tool factory pattern (Tasks 22-25).

## Conventions
- Bun runtime ONLY. `bun test`, `bun run build`, `bun run typecheck`.
- Tests: `bun:test`, co-located `*.test.ts`, given/when/then inline comments.
- Factory pattern: `createXXX()`.
- Relative imports within module; barrel imports cross-module.
- NO `as any`, NO `@ts-ignore`, NO `@ts-expect-error`.
- NO emojis in source/comments.
- NO catch-all files (`utils.ts`, `helpers.ts`).
- 200 LOC soft limit per file.
- index.ts barrels only — never dump business logic there.
## Task 5
- Hephaestus tool permissions live in the same cluster as Atlas, Sisyphus, Prometheus, and Sisyphus-Junior.
- The surgical assertion belongs in the existing tool-config-handler test file, using the existing agentResult helper.

## 2026-04-18 Task 2: types module

- `MemberSchema` needs `.strict()` on the base shape so the discriminatedUnion rejects members that mix `category` and `subagent_type`.
- `backendType` and `isActive` defaults are part of the schema contract, so tests should use `toMatchObject` instead of exact object equality.
- The eligibility registry must preserve the plan strings verbatim, especially the hard-reject messages for Momus verification.
## Task 12 learnings

- `git worktree remove` can leave prunable entries behind, so pruning after removal keeps the repo index tidy.
- For testability, a tiny git command runner hook made git-unavailable coverage simpler than mocking Bun directly.
- Detached worktrees need unique temp paths in tests to avoid cross-run collisions.
