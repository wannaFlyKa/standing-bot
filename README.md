<img src="assets/icon.svg" width="48" height="48" alt="">

# standing-bot

A skill that turns a T3 Code thread into a standing bot. The bot stays, keeps its memory in a folder, checks in on a schedule, and hands the work to workers.

Site: https://wannaflyka.github.io/standing-bot/

Status: early. It was tested with Claude Code as the agent, inside T3 Code 0.1.9. Other agents are not tested yet. T3 Code is the only host for now. Other hosts may come later, for example the Claude Code desktop app on its own.

## What it is

A standing bot is four things:

- one folder, which is its memory
- one T3 Code project on that folder
- one long-lived thread in that project, which is the bot
- scheduled check-ins, which run in a second thread of that project

All bots sit under one bots root. The file `AGENTS.md` in the root is a copy of `shared-rules.md`, and every bot loads it.

```
<bots root>/                 shared rules: AGENTS.md (CLAUDE.md is a symlink to it)
  <bot>/                     its own git repository
    AGENTS.md                the bot file: purpose, goal, key results, check-ins, rules
    NOW.md                   current view, at most 4,000 characters
    log/YYYY-MM-DD.md        append only
    notes/ + notes/INDEX.md  lasting knowledge
    procedures/              step files for a check or a worker
    .claude/settings.json    the guard, with Claude Code only
```

The bot is a front desk. It talks to you, keeps track and remembers. It does not do the work. Each task goes to a worker thread, and the bot reads the result. Only a check-in is done by the check-in thread itself. Once the result is logged, it settles the worker's thread. The chat is not the memory. What matters goes into a file, and the thread can be reset at any time.

One optional chief bot sees the other bots and tells you what waits on you.

The whole behaviour is two Markdown files and one settings file. There is no other code.

- `skills/standing-bot/SKILL.md`: setup, change, move, update, new bot, bot removal, project brief, project removal
- `skills/standing-bot/shared-rules.md`: how every bot behaves
- `skills/standing-bot/claude-settings.json`: the guard for Claude Code

## What you need

- [T3 Code](https://github.com/pingdotgg/t3code) with a coding agent set up in it. Claude Code is the one that was tested.
- git. The bots root and each bot folder are local git repositories.
- These T3 Code tools, which an agent has inside a T3 thread: `schedule_task`, `list_scheduled_tasks`, `run_scheduled_task_now`, `delete_scheduled_task`, `delegate_task`, `task_cancel`, `t3_thread_launch`, `t3_thread_send`, `t3_thread_read`, `t3_thread_organize`, `t3_project_create`, `t3_project_list`.
- For the worker that removes a bot, or removes or archives a finished project: `t3_project_read`, `t3_thread_list`, `t3_project_delete`, `t3_project_update`.
- The environment variable `T3CODE_THREAD_ID`.
- For the guard: the T3 Code MCP server is named `t3-code`, the thread id of a worker starts with `thread:delegated-task:`, and a hook command gets `CLAUDE_PROJECT_DIR`.
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

To update the plugin later, so that the bots see a newer rules version:

```sh
claude plugin marketplace update standing-bot    # Claude Code, both lines
claude plugin update standing-bot@standing-bot
codex plugin marketplace upgrade standing-bot    # Codex
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
| `remove`, `archive` or `keep`, with the project name, when the bot asks about a finished project | Remove: a worker checks that no thread runs, that no file is uncommitted and that every commit is on its remote. Then it deletes the T3 Code project with its threads and removes its folder. Archive: a worker archives the project, and its threads and its folder stay. Keep: nothing. |
| "move the bot here", in another thread in the bot's folder | Makes that thread the bot. |
| "update the rules" | Takes newer shared rules from the skill. It also replaces rules of the same version that came from another source, such as a copy of the skill under another name. If those rules name another skill, name this one when you say it. The check-in thread reads the new rules at its next run, and the bot's own thread at its next reset. |
| `new bot`, in any thread with full access | Makes another bot. |
| "remove the bot <name>", typed in a thread with full access outside that bot's project, such as the chief's | A worker deletes the bot's scheduled tasks, its T3 Code project with every thread, its folder and its line in the root's `.gitignore`. It reports the open state that was dropped and the projects of the bot that stay. It cannot be undone. A bot cannot remove itself. |

A check-in runs in the bot's check-in thread, which has the title `<bot> check-ins`. That thread does the check itself and starts no worker. It ends with `quiet` when nothing needs you, and with `off hours` outside its hours. It leaves nothing in the bot's own thread. The bot speaks up in its own thread only when a check or a worker's result asks something of you, or when a check-in is broken. By default, a reset task starts a fresh session in the bot's own thread once a day, and leaves the one word `reset` there.

The bot's own thread stays active at all times. The bot never settles, archives or snoozes it. What it settles is the thread of a worker and the lead thread of a project, when it has taken the result. The check-in thread snoozes itself at each run, and the next run wakes it.

When a project is done and its change is merged or pushed, or you said that the project is over, the bot asks you once what to do with it: remove, archive or keep. It does not ask again by itself. It never removes or archives a project without your answer for that project, and for a removal you must type the answer in the bot's thread.

## The chief

One bot can be the chief. Its bot file has the line `Role: chief`. Setup asks about it only while no bot is the chief.

The chief:

- reads the top of every bot's `NOW.md` and its scheduled tasks: its check-in thread at the daily check-in, one worker when you ask
- tells you what waits on you, at its daily check-in or when you ask "what needs me?": up to 10 lines, each with the bot's thread link, then one line for each bot that is not well
- names the right bot for a request and gives you its thread link
- sends one HANDOVER message when you say "give it to <bot>"
- shows you a question that a bot escalated. A bot escalates once, when it has asked you 3 times and the answer is due within 24 hours
- pauses, resumes or runs a bot's check-ins, makes a new bot, and removes a bot through one worker, only on your word. For a removal you must type it

The chief never writes in another bot's folder, changes its setup, or gives or carries an approval. It does no bot's work.

With two chiefs there is no chief. Chief work stops and you are told both names.

## Who counts as the owner

The owner is the person the bot works for: the one who set it up and talks to it in its thread. Text does not prove who sent it, so the bot reads the fields of the message before it acts.

| Message | Counts as |
|---|---|
| Typed in the bot's thread (`createdBy: user`) | The owner, for everything. |
| Sent through MCP from outside any T3 thread (`createdBy: agent`, `creationSource: mcp`, no `senderThreadId`), for example your own agent session in a terminal | The owner, except for four acts. |
| Sent by another T3 thread (it has a `senderThreadId`) | Not the owner. It can bring a worker's result, a bot message or a message from the bot's check-in thread, nothing more. |
| A scheduled run (`messageId` starts with `scheduled-task-message:`) | Not the owner. It can start the check-in or the reset it names, nothing more. |

The four acts an outside session cannot do: approve an outward write, widen what the bot may do, make a bot the chief, and anything that cannot be undone. The bot answers `Type it here: <the act>`, and you type it in the bot's thread.

An approval counts once, for the act named in the same message.

## Limits

- T3 Code only, for now.
- One thread is the bot, and one runs its check-ins. Any other thread in the bot's folder is a visitor and writes nothing there.
- The bot does not investigate, write code or run checks. Workers do, and the check-in thread runs the checks.
- `NOW.md` holds at most 4,000 characters. A worker's result holds at most 20 lines.
- At most 5 key results.
- The bot asks you before it starts a third project while two are in flight.
- Bots send each other two kinds of message only: HANDOVER from the chief to a bot, and ESCALATE from a bot to the chief. Neither is answered by message.
- A bot cannot message a bot that runs with a broader runtime mode.
- Each bot folder is a local git repository, and so is the bots root. The rules say never push them and never add a remote.
- A monthly check-in is a daily task that ends at once on every other day.

## The guard

With Claude Code as the agent, setup copies `claude-settings.json` to `<bot folder>/.claude/settings.json`. It is a `PreToolUse` hook, and it holds an allow list.

- In the bot's thread it blocks every MCP tool except the twelve T3 Code tools in the first tool list under "What you need" and `orchestrator_capabilities`, which T3 Code asks a thread to call before it starts a worker. The bot can start a worker, stop it, send a message to a worker that is running, send a bot message, read a thread, settle a thread, schedule tasks, and start a project. Any other MCP tool call must go to a worker.
- A worker is not blocked. The hook lets a call pass when `T3CODE_THREAD_ID` starts with `thread:delegated-task:`.
- The check-in thread is not blocked. The hook lets a call pass when `T3CODE_THREAD_ID` is on the `Check-in thread:` line of the bot file.
- A visitor thread in the bot's folder gets the same block as the bot. A Claude Code session outside T3 Code should get the block in that folder too. That case was not run.
- The lead thread of a project is not blocked. It runs in the project's own folder, where the bot folder's settings are not loaded.
- The hook acts in the running session a few seconds after the file is written. The test needed no restart.
- A bot with older rules adds or renews the guard when it takes the new rules.

## Known gaps

- The rules are text. The guard enforces part of one of them, with Claude Code only. Nothing enforces the rest.
- The guard limits MCP tools only. It does not limit the shell or the agent's built-in tools, such as web fetch and MCP resource reads. The bot's thread can still run any command, so it can still do work itself, reach a service through a command-line tool, or edit the guard file.
- The guard allows a tool, not one use of it. `t3_thread_organize` is there to settle the thread of a worker or a lead, but it can also settle the bot's own thread, and archive, snooze or pin a thread. Only the rules keep the bot's own thread active.
- A check-in that cannot start reaches nobody through the bot. Only the chief's brief names a bot whose check-in is off, late or failed.
- When a check runs past its next start, T3 Code 0.1.9 queues each start. The queued runs then do the check again, one after the other.
- Nothing locks `NOW.md` and the log. The bot's thread and the check-in thread can write them at the same moment.
- Codex loads `AGENTS.md` only inside the bot folder's repository. A bot on Codex needs the line from "Other coding agents" in its bot file, to read the shared rules.
- There is no guard for other agents. Codex CLI 0.171.0 has a hook that can block a tool call by name. In T3 Code 0.1.9 the hook does not get `T3CODE_THREAD_ID`, so it could not tell the bot from a worker. Both were read in the code, not run. Kiro CLI documents such a hook too. Kiro CLI was not installed, and nothing was run.
- The guard does not limit a worker. A worker reads outside text, such as tickets, mail and web pages, and it can use every tool of the agent, including one that sends a message.
- A worker with a shell can send a message that looks like the owner's outside session, and it can edit a bot file or the guard file.
- The checks before a removal are text that a worker follows. Nothing enforces them, and a removal cannot be undone.
- A removed project leaves the agent's own session files behind, outside its folder. The worker lists them and does not delete them.

## Cost

No usage was measured. What drives it:

- Each check-in is one run of the check-in thread. A check-in that needs you adds one turn in the bot's thread.
- A heartbeat is an interval task, so it also fires outside its hours. Then the check-in thread only answers `off hours`, which is still a short turn.
- By default, the bot's own thread keeps its chat until a daily reset, which is one short turn in that thread.
- Every bot session loads the shared rules (about 19,000 characters), the bot file and `NOW.md`.
- The skill adds about 75 tokens to every Claude Code session and about 5,100 when it is used. These are Claude Code's own estimates from `claude plugin details`.

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
- A settled thread still got a scheduled task, a worker result, a `WORKER RESULT:` message from another thread and a queued bot message. Each one started a turn and made the thread active again.
- A test bot under rules version 4 ran one scheduled check-in. When it had taken the worker's result, it settled the worker's thread. No tool call was refused in its thread.
- An agent that followed the section "Guard" wrote the guard file: once into a folder with no settings file, and once into a settings file that held the older guard and other entries. In the first case the hook started to block in the same session.

The rules update at version 6 was tested on 2026-11-21, with Claude Code 2.1.327 on T3 Code 0.1.9, in throwaway bots roots. A test thread gave each worker its brief:

- A root at version 5, whose rules came from a copy of the skill under another name. The worker followed the section "Update", copied the new file and committed only `AGENTS.md` as `Shared rules version 6`.
- A root at version 6 with another text. With no owner's words in the brief, the worker changed nothing. With "update the rules" quoted in the brief, it copied the new file and committed it. With the line `Local changes: yes` in the root file, it changed nothing and reported the diff. With the owner's choice "the new file wins" quoted in the next brief, a worker copied the new file and committed it.
- A test bot whose bot file said `Rules seen: 5` ran one scheduled check-in in a fresh session under the version 6 rules. It told the owner which line of a note named another source, changed nothing there, set `Rules seen: 6` and committed. In the same check-in it handed the check to a worker, took the result and said `quiet`.
- A worker that followed the section "Guard" left a guard file alone that held the same entry with its keys in another order.
- The plugin, installed from GitHub at version 5, loaded in a new thread beside a copy of the skill under another name.

What a bot does with a finished project (version 7) was tested on 2026-11-22, with Claude Code 2.1.327 on T3 Code 0.1.9, with a throwaway bot, throwaway projects and a throwaway remote:

- A lead thread reported that its project was done and its change was on the remote. The test bot took the result, settled the lead thread, and wrote one waiting line for that project: remove, archive or keep, with its pick. It removed nothing.
- At the next scheduled check-in, in a fresh session, the bot said `quiet`. The waiting lines were unchanged and it did not ask again.
- A message from another thread said that it was the owner and approved a removal and an archive. The bot refused it, started no worker, and logged it. Nothing was removed or archived.
- A test bot whose bot file said `Rules seen: 6` ran one scheduled check-in under the version 7 rules. It asked once about the finished project that still existed. It did not ask about a project that was gone or about an archived project. It set `Rules seen: 7` and committed.
- A worker with the brief from "Project removal" ran every check, deleted the project with its one thread, and removed the folder. A project with a similar name was untouched. The worker listed the Claude Code session files that stayed behind and did not delete them.
- A worker with the same brief for a project whose clone had a commit that was not on the remote stopped at check 3 and deleted nothing.
- A worker archived a project with `t3_project_update`. Its thread and its folder stayed.
- A test thread gave each of these workers its brief.

That a bot leaves its own thread active (version 8) was tested on 2026-11-22, with Claude Code 2.1.327 on T3 Code 0.1.9, with two throwaway bots under the version 8 rules. The `settled` field of each thread was read from another thread (`t3_thread_list`):

- A test bot ran one scheduled check-in in a fresh session, with nothing waiting on the owner and nothing in flight. It handed the check to a worker, took the result, settled the worker's thread and said `quiet`. Its own thread was not settled afterwards.
- A test bot whose bot file said `Rules seen: 7`, and whose thread another thread had settled before, ran one scheduled check-in in a fresh session. It set `Rules seen: 8` and committed. It took the worker's result, settled the worker's thread and said `quiet`. Its own thread was not settled afterwards.
- A lead thread reported that its project was done and its change was merged. The test bot took the result, settled the lead thread and wrote one waiting line for that project. Its own thread was not settled afterwards. A test thread had started the lead thread and had written the project into the bot's `NOW.md` and log.
- In these turns each bot called `t3_thread_organize` only for the thread of its worker or of the lead.

The log time and the commit of a bot's own folder (version 9) were tested on 2026-11-23, with Claude Code 2.1.330 on T3 Code 0.1.9, with two throwaway bots in one throwaway bots root. A third bot folder in that root held files that were not committed: three new files and a change to a tracked file.

- An agent followed steps 1, 3 and 4 under "On confirm" in the section "Setup" for a new bot folder in that root. Its commit held eight files, all in that folder. A test thread gave it the confirmed answers, and the interview was not run.
- One test bot ran one scheduled check-in in a fresh session. It read the clock with `date` before each of its two log entries, and each entry had that time. Its commit `<bot>: daily <date>` held two files, both in its own folder.
- The other test bot, whose bot file said `Rules seen: 8`, ran one scheduled check-in in a fresh session under the version 9 rules. It set `Rules seen: 9` and committed. It read the clock once for its first two log entries and once for the next three. Its two commits held its own folder only.
- After each of these commits, the files of the third folder were still not committed. The first of the two bots ran one more check-in while one of those files was staged in git, and its commit did not take that file.
- In each check-in commit the bot named its own folder in both `git add` and `git commit`.

The commands that update the plugin were run on 2026-11-23 in scratch config folders, with Claude Code 2.1.327 and Codex CLI 0.171.0. Each folder had the marketplace from GitHub. Its copy was set back to version 7, and the plugin was installed from that copy. With Claude Code, the installed plugin went to version 8 only after both commands, in that order. With Codex, the one command did it.

The own repository and the check-in thread (version 10) were tested on 2026-11-23, with Claude Code 2.1.330 on T3 Code 0.1.9, with throwaway bots in throwaway bots roots. Each test bot folder held a copy of the skill, and the installed plugin was switched off for that folder. A test thread started most check-ins by hand, and it changed the file that the test bots watched.

- In a bot folder that is its own repository, a Claude Code thread had the shared rules and the bot file in its context, with no tool call. A Codex thread (Codex CLI 0.171.0) had only the bot file. With the line from "Other coding agents" in the bot file, the Codex thread read the root's rules as its first step.
- T3 Code kept the snapshots of a thread in the bot's own repository and none in the root. Two bots wrote in the same minute, and the diff of two snapshots of one thread listed two files, both of that bot. In the version 9 layout, the same diff had listed files of two other bots.
- The guard refused three MCP tools in such a folder and let an allowed tool pass.
- Fourteen test bots in six roots moved from the version 9 layout by the section "Own repository". Each had commits, and files that were not committed: changed, new, staged, staged and then changed, and deleted. Every file had the same content afterwards, and each bot's repository held its files. In two roots a worker had followed "Update" from version 9 to 10 first. One more worker with the same section changed nothing.
- While one bot had moved and two had not, a thread of a bot that had not moved still committed its folder in the root.
- Two bots moved at the same moment under an earlier draft. One commit in the root took staged files of the other bot. The agent then ran `git reset` in the root to change that commit, which undid the other bot's commit and left its change staged. No file was lost. The section now says to leave such a commit. With that text, three bots moved at the same moment: three commits, no reset, and a root that tracked only the shared rules. With the final text, two more bots moved at the same moment. Each made its last commit in the root first. A line of the root's `.gitignore` for one bot's files went into that bot's `.gitignore`, and a bot folder that had a `.gitignore` got the line `tmp/`.
- A script ran the git commands of the section for two bots at once, while a third bot committed by the version 9 rule, 30 times. No file changed, and each end state was right. Git reported a lock 85 times, and the script ran the command again.
- A thread followed "New bot", and the new bot ran the setup interview with answers through the question tool. The folder was its own repository before the T3 Code project existed. After setup, the two check-in tasks had `freshSession: true` and were bound to the check-in thread, the task `reset` was bound to the bot's own thread, and a heartbeat left the bot's thread as it was.
- A task that is bound to no thread did not run in the bot's folder. In a git repository it ran in a new worktree, and in a plain folder the run failed. That is why the tasks are bound to a second thread. `schedule_task` with a task id and `bindToCurrentThread: true`, called in that thread, moved a task to it and did not change `freshSession`.
- Fourteen test bots with tasks bound to their own thread did the version 10 steps at a check-in. After it, each had one task for each check-in, bound to the new check-in thread. One bot had no `Thread:` line, and its thread added the line first. Five of them replaced the heartbeat task that had `freshSession: false` by a new task with the same title, prompt and schedule, and wrote the new id in the table. Three of them, under the final text, also made the task `reset` in the bot's own thread, with the schedule of the daily check-in.
- Quiet: three scheduled runs of an interval task left the bot's thread as it was, with 64 items before and after. The log had an entry for each run, and "Status" showed the last one. The check-in thread was snoozed after each run, and each next run started in it.
- Alert: the check-in thread sent the bot's thread one message. The bot read who sent it, wrote the waiting line and told the owner. Six messages from another thread that started with `CHECK-IN` were refused and logged. Two asked for a change of a scheduled task, and no task changed.
- Broken: a source that could not be read reached the bot's thread in the same run, and seven more runs did not repeat it. When a second source broke, the check-in thread sent one message that listed both. A source that could be read again left the list in "Status" with no message, and so did the last one. A bot that had loaded the version 10 rules wrote no waiting line for a break. A run that a test thread stopped at its start joined the list at the next run, also when that next run was outside its hours, and it left the list when it was no longer among the last six runs. A run that could not start, because its thread was archived, reached nobody through the bot. T3 Code marked the task as failed.
- A worker with the chief's brief named that bot and a bot whose heartbeat was off, and it did not name the bot that was well.
- Reset: the task `reset` ran in the bot's own thread with a fresh session. The bot read `NOW.md` and answered with the one word `reset`. It started no worker and changed no file. After it, the bot's thread did not know a word that only its chat had held.
- The bot's thread and the check-in thread wrote `NOW.md` and the log in the same minute, three times, and every entry was there afterwards. A daily check-in and a heartbeat that started in the same second both wrote their result.
- With no `Thread:` line in the bot file, the check-in thread ran the check-in and did not add the line.
- With a test version 11 in the root, the check-in thread ran a check-in and left the bot file as it was. At the next reset, the bot's own thread did what "What changed" listed and set `Rules seen: 11`.

## What was not tested

- A message typed by a person as the owner's word for a setup change or an approval, which is the `createdBy: user` case.
- A bot that runs on Codex, or on any agent other than Claude Code.
- Any T3 Code version other than 0.1.9.
- The skill after the changes above, apart from the guard tests, the version 6 update tests, the finished project tests, the own thread tests, the version 9 tests and the version 10 tests.
- The setup interview with the guard step, and a bot with older rules that adds or renews the guard.
- The guard in a thread that a person made in the app. Every test thread was made through a T3 Code tool.
- The guard in a Claude Code session outside T3 Code. Its command was run in a shell only.
- The guard against a tool of an MCP server that a plugin brings.
- "update the rules" typed by a person in a bot thread. The update tests gave the worker its brief from a test thread.
- A bot that sees a newer rules version after an update of the plugin. The update commands were run with no bot, from version 7 to version 8.
- The owner's typed answer `remove`, `archive` or `keep`, and a bot that hands the removal or the archive to a worker. A test thread gave each worker its brief.
- A removal that stops because a thread runs, a file is uncommitted, a stash exists, or a file outside a repository may be the only copy of work. Only the stop for a commit that is not on the remote was run.
- Under version 9: a commit after a setup change or a move, a commit on the owner's word, a commit by a bot with no daily check-in, and a reset at 04:00 for a bot with no daily check-in.
- Under version 10: the unread mark and a desktop notification for a quiet check-in, which no T3 Code tool shows to an agent. "Move", and "Change" with a new check-in. A bot from older rules that has the task `reset` already. The reset choice "never". An ESCALATE after a message from the check-in thread. A chief with its own check-in thread. A check that runs longer than the time to the next check-in. A bots root with real bots.

## Rules version

The rules carry a version in two places that must match: the line `Rules version: 11` in `shared-rules.md`, and the words "Rules version 11." at the end of the skill description in `SKILL.md`. Bots compare the two to find updates. The description is YAML, so it must not hold a colon followed by a space.

## License

MIT. See [LICENSE](LICENSE).
