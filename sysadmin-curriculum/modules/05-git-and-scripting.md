# Module 05: Git and scripting

**Time:** about 5 hours.
**Prerequisites:** module 01.

## By the end you can

- Keep your work in a git repository, and use `status`, `diff`, `add`, `commit`, `log`, branches, and a remote.
- Undo mistakes safely, and explain why secrets in git are a serious problem.
- Write a Bash script that is safe to run: it fails loudly, quotes its variables, and tells you what it's doing.
- Schedule a script with a systemd timer and read its logs.
- Review a script for dangerous patterns.

## Why this matters

Scripts and configs that live only on one server are a liability: they're untested, unreviewed, and lost when the server is. Git gives you history, review, and a way to undo. Scripting lets you do things twice, then never again. Together they are the first step toward building things repeatably (module 11).

## Concepts

### Git

- A **repository** is a project folder plus its history. A **commit** is a snapshot with a message. A **branch** is a line of work. A **remote** (GitHub, GitLab, an internal server) is a shared copy.
- The flow: edit, `git add` (stage), `git commit` (record), `git push` (share).
- **Commit messages** explain why, not just what.
- **Never commit secrets** (passwords, keys, tokens). Once pushed, assume they're compromised, even if you delete the file in the next commit, because the history keeps it. The fix is to rotate the secret. Use a `.gitignore` for files that should never be committed.
- `git revert` makes a new commit that undoes an old one (safe for shared history). `git reset --hard` discards work (dangerous). Learn the difference before you use either.

### Bash safety

- **Start with `set -euo pipefail`:** exit on error, error on unset variables, fail a pipeline if any part fails.
- **Quote your variables:** `"$file"`, not `$file`. Unquoted variables break on spaces and expand globs.
- **Check exit codes.** `$?` is the last command's status; 0 means success. Scripts that monitor things should exit non-zero on failure.
- **Be idempotent:** running the script twice should be safe and give the same result.
- **Offer a dry run** for anything that deletes or changes things.
- **Log what you do**, and send errors to stderr (`>&2`).
- **Guard destructive commands.** `rm -rf "$DIR/"*` with an empty `$DIR` becomes `rm -rf /*`. Use `${DIR:?DIR is not set}`.
- **Prefer `systemd` timers or cron** over a script left running in a terminal.
- Run **`shellcheck`** on everything.

## Lab

Snapshot first.

```bash
sudo dnf install -y git epel-release
sudo dnf install -y ShellCheck
git --version; shellcheck --version
```

### 1. Git basics

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main

mkdir -p ~/notes && cd ~/notes
git init
echo "# Lab notes" > README.md
git status
git add README.md
git commit -m "Add README"
git log --oneline
```

Then make changes and look at them before committing:

```bash
echo "Module 01 done" >> README.md
git diff
git add -p                # stage in pieces; read each change
git commit -m "Record module 01 progress"
```

### 2. Branches and undoing

```bash
git switch -c experiment
echo "try something" >> README.md
git commit -am "Experiment"
git switch main
git merge experiment
git revert HEAD            # make a commit that undoes the last one
git log --oneline
```

Then try: edit a file, run `git restore README.md` to throw the edit away, and explain what you lost. Don't run `git reset --hard` unless you can say what it will discard.

### 3. A remote

Create an empty repository on your hosting service (GitHub, GitLab, or your organization's server), then:

```bash
git remote add origin <url>
git push -u origin main
```

Add a `.gitignore` with `*.key`, `*.pem`, `.env`, and `*.swp`, and commit it.

### 4. Write a safe script

Create `~/notes/disk-report.sh`:

```bash
#!/usr/bin/env bash
# Report filesystems at or above a usage threshold.
# Usage: disk-report.sh [threshold-percent]   (default 85)
# Exit status: 0 if everything is below the threshold, 1 otherwise.
set -euo pipefail

threshold="${1:-85}"
status=0

while read -r pct mount; do
    used="${pct%\%}"
    if [ "$used" -ge "$threshold" ]; then
        echo "WARN: $mount is ${used}% full (threshold ${threshold}%)" >&2
        status=1
    fi
done < <(df --output=pcent,target -x tmpfs -x devtmpfs | tail -n +2)

exit "$status"
```

```bash
chmod +x disk-report.sh
shellcheck disk-report.sh
./disk-report.sh; echo "exit: $?"
./disk-report.sh 1; echo "exit: $?"          # force a warning
```

Explain every line: what does `set -euo pipefail` do, what is `<(...)`, why `"${1:-85}"`?

### 5. Schedule it with a timer

Use the pattern from module 01: a `.service` that runs the script, and a `.timer` that runs it every 15 minutes. Install the script in `/usr/local/bin/`, check `systemctl list-timers`, and read its output with `journalctl -u disk-report.service`. Commit the unit files to your repo.

### 6. Review dangerous patterns (do not run these)

In a scratch directory, with `mktemp -d`, work out what each of these would do before you run anything:

```bash
DIR=""
echo rm -rf $DIR/*                     # echo only! what would this really run?
touch "my file.txt"
for f in $(ls); do echo "got: $f"; done      # what does this print? why is it wrong?
for f in ./*; do echo "got: $f"; done        # why is this better?
```

Then write the safe versions: `rm -rf "${DIR:?DIR is not set}"/*` and a loop that handles spaces.

## Break it

1. **A script that destroys on empty.** Your mentor gives you a script with an unquoted variable and an unguarded `rm`. Run it only in a temp directory, find the bug, fix it, and explain what it could have done on a real server.
2. **A secret in git.** In a throwaway repo, commit a fake API key, delete it in the next commit, and show it's still in `git log -p`. Write down what you would do if it were real.
3. **A merge conflict.** Edit the same line on two branches and merge. Resolve it by reading the conflict markers, not by deleting everything.

## Check your understanding

- What does `set -euo pipefail` do, and why is each part useful?
- Why quote `"$var"`?
- What's the difference between `git revert` and `git reset --hard`?
- You committed a password and pushed. What now?
- What does "idempotent" mean, and why does it matter for scripts?
- Why is a systemd timer often better than a script someone runs by hand?
- What is the risk of `rm -rf "$DIR/"*`?

## Deliverable

A git repository (yours or your team's) containing: your notes so far, `disk-report.sh` with a README line on how to run it, the service and timer files, and a `.gitignore`. Your commits should have meaningful messages.

## Use AI well here

Ask Claude to review `disk-report.sh` for bugs and dangerous patterns. Then **explain every line back to your mentor** and test any change it suggests before you accept it. Ask it to rewrite the script in a way that fails differently, and decide whether the result is better.

## Go deeper

- Pro Git (free online book) for the chapters on branching and undoing things
- `man bash` (sections on quoting and parameter expansion), and the shellcheck wiki
- Google's Shell Style Guide
- A git workflow your team uses: pull requests, branch naming, code review
