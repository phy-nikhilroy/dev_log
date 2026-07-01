# Command-Line Verification Techniques, From Building `claude-usage` and `agyusage`

**Date:** 2026-07-01
**Context:** Built while writing the `claude-usage` and `agyusage` polybar modules in `~/dotfiles/scripts`

---

## Background

While building these two polybar modules, I learned a habit that's since become my default: **don't assume, check.** It's easy to reason your way to a conclusion about what a script *should* be doing — but if there's a command that proves or disproves it in a few seconds, run that command instead of reasoning about it in my head.

This log is the set of concrete techniques I actually used during that session — reading process state, diffing files before/after, driving a TUI app headlessly with `tmux`, and checking claims against real shipped docs instead of guessing. Every example below is a real command I ran.

---

## 1. Run It Directly and Read the Output

The simplest verification there is. Before trusting a script, just run it and look:

```bash
/home/nikhil/dotfiles/scripts/claude-usage
```

If I need to see *exactly* what's in the output — trailing spaces, hidden characters, tabs — piping through `cat -A` shows spaces normally but marks line endings with `$` and tabs with `^I`:

```bash
/home/nikhil/dotfiles/scripts/claude-usage | cat -A
# 049 012$
```

This mattered when I needed to check whether the output was space-padded or zero-padded — impossible to tell by eye in a terminal, but `cat -A` shows it immediately.

## 2. Time It

`time` prefixes any command and reports wall-clock, user CPU, and system CPU time. Useful for deciding polling intervals — I didn't want a polybar module that takes 6 seconds to run polling every 30 seconds.

```bash
time /home/nikhil/dotfiles/scripts/agyusage
# real  0m6.466s
# user  0m0.150s
# sys   0m0.096s
```

`real` is wall-clock time (what I actually wait). `user`+`sys` is CPU time consumed. A big gap between `real` and `user+sys` usually means the command was waiting on something external (network, another process, a sleep) rather than burning CPU.

## 3. Check Whether a Process Is Actually Running

`pgrep` searches running processes by name and prints matching PIDs (`-a` prints the full command line too):

```bash
pgrep -a polybar
```

Nothing printed = nothing running. This is the fastest way to answer "is X actually up?" instead of guessing from symptoms.

To inspect a specific PID in more detail — parent process, elapsed time, full command:

```bash
ps -p 50490 -o pid,ppid,etime,cmd
```

To find *children* of a process:

```bash
ps -o pid,ppid,cmd --ppid <pid>
```

This is how I caught a leaked background process: a script was supposed to clean up after itself, but `pgrep -a '^agy$|/agy$'` kept finding a stray process after the script exited — proof the cleanup wasn't actually working, regardless of what the code looked like it should do.

## 4. Diff File/Directory State Before and After

A very general and powerful technique: snapshot something, run the thing I'm testing, snapshot again, compare.

```bash
find ~/.claude/projects -name "*.jsonl" | wc -l   # before
/home/nikhil/dotfiles/scripts/claude-usage         # run it
find ~/.claude/projects -name "*.jsonl" | wc -l   # after
```

If the count changed, the script created a file somewhere — even if nothing in its own logic obviously suggested that. This is how I discovered that `claude -p "/usage"` silently creates a new session transcript file on every call, which the script then had to clean up.

Finding files modified recently (last N minutes) is the other half of this pattern:

```bash
find ~/.claude/projects -name "*.jsonl" -mmin -20 -exec ls -la {} \;
```

(`-mmin -20` = modified in the last 20 minutes. Some `find` variants don't support `-newermt "-20 minutes"` — `-mmin` is more portable.)

`stat` gives a single file's exact modification time as a raw timestamp, useful for comparing against another timestamp (like "did this file change before or after that process started?"):

```bash
stat -c '%Y %n' /home/nikhil/dotfiles/scripts/agyusage
```

## 5. Read the Logs

If something has a log file, read the end of it before guessing what went wrong:

```bash
tail -n 40 /tmp/polybar1.log
```

Look for the actual failure line, not just "it's not working." In this session, polybar's log showed `Termination signal received, shutting down...` as the last line — which immediately explained why the bar had disappeared: it wasn't a crash, it was just quit and never relaunched.

## 6. Driving an Interactive TUI App From a Script (`tmux`)

Some tools (like the `agy` CLI) only expose a feature inside their interactive terminal UI — no flag, no non-interactive mode. `tmux` lets me script an interactive program as if a human were typing into it, entirely in the background:

```bash
# Start a detached session running a program
tmux new-session -d -s mysession -x 220 -y 50 "some-interactive-program"

# Send it keystrokes, as if typed
tmux send-keys -t mysession "/usage" Enter

# Grab exactly what's currently rendered on screen, as plain text
tmux capture-pane -p -t mysession

# Tear it down when done
tmux kill-session -t mysession
```

Since the program takes a moment to respond, I don't just `sleep` a guessed amount — I poll until the expected text shows up, with a timeout so it can't hang forever:

```bash
deadline=$(( $(date +%s) + 15 ))
while [ "$(date +%s)" -lt "$deadline" ]; do
    tmux capture-pane -p -t mysession | grep -q "ready marker" && break
    sleep 0.3
done
```

**Gotcha:** `tmux kill-session` only tears down the tmux side. If the program underneath doesn't exit cleanly on its own (some apps hold a socket or file descriptor open and never notice their terminal is gone), it leaks a process that keeps running invisibly. Check for this explicitly:

```bash
pgrep -af 'the-program-name'
```

If something's still there after `kill-session`, kill its whole process group directly instead of trusting the session teardown:

```bash
pane_pid=$(tmux list-panes -t mysession -F '#{pane_pid}')
kill -9 -"$(ps -o pgid= -p "$pane_pid" | tr -d ' ')"   # note the leading "-"
```

(The leading `-` before the PID tells `kill` to signal the whole process group, not just one PID.)

## 7. Inspecting Data Formats Directly Instead of Guessing

When a tool stores data in some binary/DB format, I open it directly rather than reading someone else's writeup about it.

For a SQLite database:

```bash
sqlite3 somefile.db ".tables"      # list tables
sqlite3 somefile.db ".schema"      # full schema of every table
sqlite3 somefile.db "SELECT * FROM sometable LIMIT 5;"
```

For a raw binary blob (this project involved a hand-decoded protobuf field inside a SQLite column), I write a tiny throwaway Python script that decodes it and prints the structure, then eyeball the output for something recognizable, like a Unix timestamp or a familiar string:

```python
import datetime
print(datetime.datetime.fromtimestamp(1782905548))
# 2026-07-01 17:02:28   <- sanity-check: does this match "now"?
```

That sanity-check step matters — a number that *could* be a timestamp isn't confirmed to *be* one until I convert it and check it lands somewhere plausible.

## 8. Checking Real Docs Instead of Guessing Syntax

Man pages aren't always installed for every tool, and even when they are, they don't always cover the thing I need (module-specific config syntax, for example, often isn't in the general man page). Before guessing:

```bash
man polybar                     # general man page, if it exists
man -k polybar                  # search all man pages mentioning "polybar"
pacman -Ql polybar | grep -i doc   # what files did this package actually install?
```

That last one is often the most useful — it shows exactly what a package put on disk, including example configs, which frequently demonstrate the exact syntax I'm unsure about better than prose documentation does:

```bash
cat /usr/share/doc/polybar/examples/config.ini
```

This is how I confirmed the fixed-width numeric formatting trick (`%percentage:2%%`) — not by guessing that it might work, but by finding it used in the package's own shipped example file.

## 9. Verifying Git State Before and After

Never assume `git commit` did what I think — check status before staging, review the actual diff before committing, and check status again after:

```bash
git status                          # what's changed?
git diff path/to/file               # exactly what changed, line by line
git add path/to/file
git commit -m "..."
git status                          # confirm: "nothing to commit, working tree clean"
git push
```

Reading the diff before committing catches accidental changes I didn't mean to make — it's cheap insurance.

## 10. Testing for Reliability, Not Just Correctness

Running something once and seeing the right output only proves it works *that one time*. For anything with side effects (spawning processes, writing files), I run it two or three times in a row and check that nothing accumulates:

```bash
for i in 1 2; do
  /path/to/script
  pgrep -af 'thing-that-should-not-linger' || echo "clean"
done
```

If run #1 leaves something behind that run #2 doesn't clean up, it piles up across iterations — a single run often looks perfectly fine and hides exactly this kind of bug.

---

## Summary Checklist

When I'm not sure whether something works, I ask which of these answers it fastest — then go run that command instead of reasoning about it in my head:

- Did it print what I expected? → run it, read the output (`cat -A` if whitespace matters)
- Is it still running / did it leave something behind? → `pgrep -a`, `ps -p <pid>`
- Did it write/change a file somewhere I didn't expect? → count/list files before and after
- What actually went wrong? → read the log
- How do I script something that's normally interactive? → `tmux` (`new-session -d`, `send-keys`, `capture-pane`)
- What's the real syntax for this? → check the package's own shipped docs/examples before guessing (`pacman -Ql <pkg>`, man pages)
- Did my change actually get committed correctly? → `git status` / `git diff` before *and* after
- Does it work reliably, not just once? → run it 2-3 times in a row and check nothing piles up
