# standing-bot

A skill that turns a T3 Code thread into a standing bot. The bot stays, keeps its memory in a folder, checks in on a schedule, and hands the work to workers.

Site: https://wannaflyka.github.io/standing-bot/

Status: early. It was tested with Claude Code as the agent, inside T3 Code. Other agents are not tested yet. T3 Code is the only host for now. Other hosts may come later, for example the Claude Code desktop app on its own.

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
```

The bot is a front desk. It talks to you, keeps track and remembers. It does not do the work. Each task goes to a worker thread, and the bot reads the result. The chat is not the memory. What matters goes into a file, and the thread can be reset at any time.

One optional chief bot sees the other bots and tells you what waits on you.

The whole behaviour is two Markdown files. There is no code.

- `skills/standing-bot/SKILL.md`: setup, change, move, update, new bot, project brief
- `skills/standing-bot/shared-rules.md`: how every bot behaves

## What you need

- [T3 Code](https://github.com/pingdotgg/t3code) with a coding agent set up in it. Claude Code is the one that was tested.
- git. The bots root is a local git repository.
- These T3 Code tools, which an agent has inside a T3 thread: `schedule_task`, `list_scheduled_tasks`, `run_scheduled_task_now`, `delete_scheduled_task`, `delegate_task`, `t3_thread_launch`, `t3_thread_send`, `t3_thread_read`, `t3_project_create`, `t3_project_list`.
- The environment variable `T3CODE_THREAD_ID`.
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
| "update the rules" | Takes newer shared rules from the skill. |
| `new bot`, in any thread with full access | Makes another bot. |

At a check-in the bot says `quiet` when nothing needs you, and `off hours` outside its hours. It speaks up only when a worker's result asks something of you.

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

## Known gaps

- The rules are text. Nothing enforces them.
- A worker with a shell can send a message that looks like the owner's outside session.
- A worker with a shell can edit a bot file.

An optional guard for Claude Code is in `examples/claude-settings.json`. Copy it to `<bot folder>/.claude/settings.json` and put your own MCP server names in the matcher. It stops the bot thread from calling those tools itself and allows them in workers, by checking whether `T3CODE_THREAD_ID` starts with `thread:delegated-task:`. It is not part of the skill, and it does not close the gaps above.

## Cost

No usage was measured. What drives it:

- Each check-in is one turn in the bot's thread plus one worker run.
- A heartbeat is an interval task, so it also fires outside its hours. Then the bot only answers `off hours`, which is still a short turn.
- The bot's thread keeps its chat between check-ins, so its context grows until the reset. The default reset is once a day.
- Every bot session loads the shared rules (about 13,000 characters), the bot file and `NOW.md`.
- The skill adds about 75 tokens to every Claude Code session and about 3,600 when it is used. These are Claude Code's own estimates from `claude plugin details`.

To spend less, use a longer heartbeat or fewer check-ins.

## What was tested

Tested on 2026-11-20 with throwaway bots, on a private build of T3 Code, with Claude Code as the agent:

setup interview, check-ins, hand-off to a worker, chief brief, forged bot messages, a setup change from another thread, an approval from an outside session, escalation, pause and resume, two chiefs, bot creation, first-use checks, the rules update flow, two threads in one bot folder.

Those tests ran on the private version of the skill. This copy differs from it by the new name, a few wording changes and the time zone sentence. It was not run again.

## What was not tested

- A message typed by a person, which is the `createdBy: user` case.
- A bot that runs on Codex, or on any agent other than Claude Code.
- A public build of T3 Code.
- This copy after the changes above.
- The example hook with other server names.

## Rules version

The rules carry a version in two places that must match: the line `Rules version: 2` in `shared-rules.md`, and the words "Rules version 2." at the end of the skill description in `SKILL.md`. Bots compare the two to find updates. The description is YAML, so it must not hold a colon followed by a space.

## License

MIT. See [LICENSE](LICENSE).
