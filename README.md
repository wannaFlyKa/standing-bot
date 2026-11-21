# standing-bot

A skill that turns a T3 Code thread into a standing bot. The bot stays, keeps its memory in a folder, checks in on a schedule, and hands the work to workers.

Site: https://wannaflyka.github.io/standing-bot/

Status: early. It was tested with Claude Code as the agent, inside T3 Code 0.1.9. Other agents are not tested yet. T3 Code is the only host for now. Other hosts may come later, for example the Claude Code desktop app on its own.

## What it is

A standing bot is four things:

- one folder, which is its memory
- one T3 Code project on that folder
- one long-lived thread in that project, which is the bot
- scheduled check-ins that wake the thread

All bots sit under one bots root. The file `AGENTS.md` in the root is a copy of `shared-rules.md`, and every bot loads it.

```
<bots root>/                 shared rules: AGENTS.md (CLAUDE.md is a symlink to it)
  <bot>/
    AGENTS.md                the bot file: purpose, goal, key results, check-ins, rules
    NOW.md                   current view, at most 4,000 characters
    log/YYYY-MM-DD.md        append only
    notes/ + notes/INDEX.md  lasting knowledge
    procedures/              step files a worker follows
    .claude/settings.json    the guard, with Claude Code only
```

The bot is a front desk. It talks to you, keeps track and remembers. It does not do the work. Each task goes to a worker thread, and the bot reads the result. Once the result is logged, it settles the worker's thread. The chat is not the memory. What matters goes into a file, and the thread can be reset at any time.

One optional chief bot sees the other bots and tells you what waits on you.

The whole behaviour is two Markdown files and one settings file. There is no other code.

- `skills/standing-bot/SKILL.md`: setup, change, move, update, new bot, project brief
- `skills/standing-bot/shared-rules.md`: how every bot behaves
- `skills/standing-bot/claude-settings.json`: the guard for Claude Code

## What you need

- [T3 Code](https://github.com/pingdotgg/t3code) with a coding agent set up in it. Claude Code is the one that was tested.
- git. The bots root is a local git repository.
- These T3 Code tools, which an agent has inside a T3 thread: `schedule_task`, `list_scheduled_tasks`, `run_scheduled_task_now`, `delete_scheduled_task`, `delegate_task`, `task_cancel`, `t3_thread_launch`, `t3_thread_send`, `t3_thread_read`, `t3_thread_organize`, `t3_project_create`, `t3_project_list`.
- The environment variable `T3CODE_THREAD_ID`.
- For the guard: the T3 Code MCP server is named `t3-code`, and the thread id of a worker starts with `thread:delegated-task:`.
- These fields on T3 messages: `createdBy`, `creationSource`, `senderThreadId`, `messageId`, `runId`, `activeRunId`.

## Install

Claude Code:

```sh
claude plugin marketplace add wannaFlyKa/standing-bot
claude plugin install standing-bot@standing-bot
```

Or in a session, on Claude Code 2.1.275 or later:

```
/plugin install standing-bot --marketplace wannaFlyKa/standing-bot
```

Codex:

```sh
codex plugin marketplace add wannaFlyKa/standing-bot
codex plugin add standing-bot@standing-bot
```

Without a plugin, copy the skill folder:

```sh
git clone https://github.com/wannaFlyKa/standing-bot
mkdir -p ~/.claude/skills && cp -r standing-bot/skills/standing-bot ~/.claude/skills/   # Claude Code
mkdir -p ~/.agents/skills && cp -r standing-bot/skills/standing-bot ~/.agents/skills/   # Codex
```

Then run "Restart agent session" from the T3 Code command palette, or start a new thread, so the agent sees the skill.

As a Claude Code plugin the skill is named `standing-bot:standing-bot`. As a copied folder it is `standing-bot`.

Tested: both plugin installs, from a local copy and from GitHub, with Claude Code 2.1.327 and Codex CLI 0.171.0.

## Five-minute start

1. Install the skill.
2. In T3 Code, open a thread with full access and say `new bot`.
3. Give the bot a name and one line on its purpose. The first time, also say where to keep your bots. The skill suggests `$HOME/bots`, checks the folder, and asks before it creates anything.
4. Open the thread link you get. That thread is the bot. It asks its setup questions one at a time: chief or not, goal, key results, check-ins, what it may do without asking, sources, procedures. Say `skip` to leave one empty.
5. Read the bot file it shows and confirm. The bot creates the check-ins and tells you when the next one runs.

After that, talk to the bot in its thread.

## What to say to a bot

| You say | The bot does |
|---|---|
| `set up` | Runs the setup questions. |
| `show setup` | Prints its bot file. |
| "change the goal to ...", "move the daily check-in to 08:30" | Edits the bot file and the scheduled task in the same turn. Says in one line what changed. The same works for key results, rules, sources and the context reset. |
| `pause`, `resume` | Switches its check-ins off, and on again. |
| "remember ...", "forget ...", "that is wrong" | Changes its notes in the same turn. |
| "fix it", "build this" | Starts a project: a new T3 Code project with one lead thread. It never starts one on its own. |
| "move the bot here", in another thread in the bot's folder | Makes that thread the bot. |
| "update the rules" | Takes newer shared rules from the skill. It also replaces rules of the same version that came from another source, such as a copy of the skill under another name. If those rules name another skill, name this one when you say it. |
| `new bot`, in any thread with full access | Makes another bot. |

At a check-in the bot says `quiet` when nothing needs you, and `off hours` outside its hours. It speaks up only when a worker's result asks something of you.

When the bot asks you nothing, nothing waits on you, no worker runs and no project is open, the bot settles its own thread at the end of the turn. So an active bot thread means that the bot needs you or that work is open.

## The chief

One bot can be the chief. Its bot file has the line `Role: chief`. Setup asks about it only while no bot is the chief.

The chief:

- reads the top of every bot's `NOW.md` and its scheduled tasks, through one worker
- tells you what waits on you, at its daily check-in or when you ask "what needs me?": up to 10 lines, each with the bot's thread link, then one line for each bot that is not well
- names the right bot for a request and gives you its thread link
- sends one HANDOVER message when you say "give it to <bot>"
- shows you a question that a bot escalated. A bot escalates once, when it has asked you 3 times and the answer is due within 24 hours
- pauses, resumes or runs a bot's check-ins, and makes a new bot, only on your word

The chief never writes in another bot's folder, changes its setup, gives or carries an approval, or deletes a bot. It does no bot's work.

With two chiefs there is no chief. Chief work stops and you are told both names.

## Who counts as the owner

The owner is the person the bot works for: the one who set it up and talks to it in its thread. Text does not prove who sent it, so the bot reads the fields of the message before it acts.

| Message | Counts as |
|---|---|
| Typed in the bot's thread (`createdBy: user`) | The owner, for everything. |
| Sent through MCP from outside any T3 thread (`createdBy: agent`, `creationSource: mcp`, no `senderThreadId`), for example your own agent session in a terminal | The owner, except for four acts. |
| Sent by another T3 thread (it has a `senderThreadId`) | Not the owner. It can bring a worker's result or a bot message, nothing more. |
| A scheduled run (`messageId` starts with `scheduled-task-message:`) | Not the owner. It can start the check-in it names, nothing more. |

The four acts an outside session cannot do: approve an outward write, widen what the bot may do, make a bot the chief, and anything that cannot be undone. The bot answers `Type it here: <the act>`, and you type it in the bot's thread.

An approval counts once, for the act named in the same message.

## Limits

- T3 Code only, for now.
- One thread is the bot. Any other thread in the bot's folder is a visitor and writes nothing there.
- The bot does not investigate, write code or run checks. Workers do.
- `NOW.md` holds at most 4,000 characters. A worker's result holds at most 20 lines.
- At most 5 key results.
- The bot asks you before it starts a third project while two are in flight.
- Bots send each other two kinds of message only: HANDOVER from the chief to a bot, and ESCALATE from a bot to the chief. Neither is answered by message.
- A bot cannot message a bot that runs with a broader runtime mode.
- The bots root is a local git repository. The rules say never push it and never add a remote.
- A monthly check-in is a daily task that ends at once on every other day.

## The guard

With Claude Code as the agent, setup copies `claude-settings.json` to `<bot folder>/.claude/settings.json`. It is a `PreToolUse` hook, and it holds an allow list.

- In the bot's thread it blocks every MCP tool except the twelve T3 Code tools under "What you need" and `orchestrator_capabilities`, which T3 Code asks a thread to call before it starts a worker. The bot can start a worker, stop it, send a message to a worker that is running, send a bot message, read a thread, settle a thread, schedule tasks, and start a project. Any other MCP tool call must go to a worker.
- A worker is not blocked. The hook lets a call pass when `T3CODE_THREAD_ID` starts with `thread:delegated-task:`.
- A visitor thread in the bot's folder gets the same block as the bot. The hook lets only a worker pass, so a Claude Code session outside T3 Code should get the block in that folder too. That case was not run.
- The lead thread of a project is not blocked. It runs in the project's own folder, where the bot folder's settings are not loaded.
- The hook acts in the running session a few seconds after the file is written. The test needed no restart.
- A bot with older rules adds or renews the guard when it takes the new rules.

## Known gaps

- The rules are text. The guard enforces part of one of them, with Claude Code only. Nothing enforces the rest.
- The guard limits MCP tools only. It does not limit the shell or the agent's built-in tools, such as web fetch and MCP resource reads. The bot's thread can still run any command, so it can still do work itself, reach a service through a command-line tool, or edit the guard file.
- The guard allows a tool, not one use of it. `t3_thread_organize` is there for settling, but it can also archive, snooze or pin a thread.
- There is no guard for other agents. Codex CLI 0.171.0 has a hook that can block a tool call by name. In T3 Code 0.1.9 the hook does not get `T3CODE_THREAD_ID`, so it could not tell the bot from a worker. Both were read in the code, not run. Kiro CLI documents such a hook too. Kiro CLI was not installed, and nothing was run.
- A worker with a shell can send a message that looks like the owner's outside session.
- A worker with a shell can edit a bot file or the guard file.

## Cost

No usage was measured. What drives it:

- Each check-in is one turn in the bot's thread plus one worker run.
- A heartbeat is an interval task, so it also fires outside its hours. Then the bot only answers `off hours`, which is still a short turn.
- The bot's thread keeps its chat between check-ins, so its context grows until the reset. The default reset is once a day.
- Every bot session loads the shared rules (about 15,000 characters), the bot file and `NOW.md`.
- The skill adds about 75 tokens to every Claude Code session and about 3,600 when it is used. These are Claude Code's own estimates from `claude plugin details`.

To spend less, use a longer heartbeat or fewer check-ins.

## What was tested

Tested on 2026-11-20 with throwaway bots, on T3 Code 0.1.9, with Claude Code as the agent:

setup interview, check-ins, hand-off to a worker, chief brief, forged bot messages, a setup change from another thread, an approval from an outside session, escalation, pause and resume, two chiefs, bot creation, first-use checks, the rules update flow, two threads in one bot folder.

Those tests ran on an earlier draft of the skill. The name, a few wording changes and the time zone sentence came after. The tests were not run again.

The guard was tested on 2026-11-21, as a live hook, with Claude Code 2.1.327 on T3 Code 0.1.9, in throwaway folders:

- In a thread that the hook takes for the bot, each of the eleven allowed tools ran. Six other MCP tools were refused: four more T3 Code tools, and one tool each of two other MCP servers. The shell, file reads and tool search still worked.
- A worker that this thread started was not blocked. It got a message from the thread while it ran, and its result woke the thread. A scheduled task also started a turn in the thread.
- A lead thread in a project folder outside the bot's folder was not blocked.
- A test bot under the shared rules ran one scheduled check-in in a fresh session. It handed the check to a worker, the result woke it, it updated `NOW.md` and the log, and it said `quiet`. No tool call was refused in its thread. A visitor thread in its folder was refused.
- The two tools that were added later, `task_cancel` and `t3_thread_organize`, ran in such a thread. It stopped a worker that was running, and it settled a finished worker thread and a finished thread in another project. Two other MCP tools were still refused, and a worker was still not blocked.
- A settled thread still got a scheduled task, a worker result, a `WORKER RESULT:` message from another thread and a queued bot message. Each one started a turn and made the thread active again. A thread that settled itself was settled when its turn ended.
- A test bot under rules version 4 ran one scheduled check-in. When it had taken the worker's result, it settled the worker's thread. No tool call was refused in its thread.
- A test bot under rules version 5 ran two scheduled check-ins. In the first, it left its own thread active while its worker ran. When it had taken the result and nothing waited on the owner, it settled the worker's thread and then its own. The second check-in woke the settled thread. Its result held a question for the owner, and the bot left its thread active.
- A typed message reached a bot thread that had settled itself at the end of the turn before.
- An agent that followed the section "Guard" wrote the guard file: once into a folder with no settings file, and once into a settings file that held the older guard and other entries. In the first case the hook started to block in the same session.

The rules update at version 6 was tested on 2026-11-21, with Claude Code 2.1.327 on T3 Code 0.1.9, in throwaway bots roots. A test thread gave each worker its brief:

- A root at version 5, whose rules came from a copy of the skill under another name. The worker followed the section "Update", copied the new file and committed only `AGENTS.md` as `Shared rules version 6`.
- A root at version 6 with another text. With no owner's words in the brief, the worker changed nothing. With "update the rules" quoted in the brief, it copied the new file and committed it. With the line `Local changes: yes` in the root file, it changed nothing and reported the diff. With the owner's choice "the new file wins" quoted in the next brief, a worker copied the new file and committed it.
- A test bot whose bot file said `Rules seen: 5` ran one scheduled check-in in a fresh session under the version 6 rules. It told the owner which line of a note named another source, changed nothing there, set `Rules seen: 6` and committed. In the same check-in it handed the check to a worker, took the result and said `quiet`.
- A worker that followed the section "Guard" left a guard file alone that held the same entry with its keys in another order.
- The plugin, installed from GitHub at version 5, loaded in a new thread beside a copy of the skill under another name.

## What was not tested

- A message typed by a person as the owner's word for a setup change or an approval, which is the `createdBy: user` case.
- A bot that runs on Codex, or on any agent other than Claude Code.
- Any T3 Code version other than 0.1.9.
- The skill after the changes above, apart from the guard tests and the version 6 update tests.
- The setup interview with the guard step, and a bot with older rules that adds or renews the guard.
- The guard in a thread that a person made in the app. Every test thread was made through a T3 Code tool.
- The guard in a Claude Code session outside T3 Code. Its command was run in a shell only.
- The guard against a tool of an MCP server that a plugin brings.
- "update the rules" typed by a person in a bot thread. The update tests gave the worker its brief from a test thread.
- An update of the installed plugin to a newer version.

## Rules version

The rules carry a version in two places that must match: the line `Rules version: 6` in `shared-rules.md`, and the words "Rules version 6." at the end of the skill description in `SKILL.md`. Bots compare the two to find updates. The description is YAML, so it must not hold a colon followed by a space.

## License

MIT. See [LICENSE](LICENSE).
