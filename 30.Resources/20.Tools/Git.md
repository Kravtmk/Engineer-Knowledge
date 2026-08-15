
# Git is a version control system.

Basic flow:

Working Directory
        ↓
     git add
        ↓
   Staging Area
        ↓
   git commit
        ↓
 Local Repository
        ↓
    git push
        ↓
 Remote Repository

---

## 1. Repository

Create:

```bash
git init
```

Clone existing repository:

```
git clone <URL>
```

Check remote:

```
git remote -v
```

Add remote:

```
git remote add origin <URL>
```

---

## 2. Status

```
git status
```

Shows the current state of the repository.

Common states:

```
??  untracked
M   modified
A   added/staged
```

Always check `git status` before important Git operations.

---

## 3. Staging

Add one file:

```
git add <file>
```

The file is now prepared for the next commit.

Check exactly what is staged:

```
git diff --cached
```

Safe workflow:

```
git status
↓
git add <file>
↓
git diff --cached
↓
git commit
```

For public repositories, prefer adding specific files instead of blindly using:

```
git add .
```

This reduces the risk of accidentally committing secrets or unnecessary files.

---

## 4. Commit

```
git commit -m "message"
```

A commit creates a snapshot in the local Git history.

Example:

```
git commit -m "docs: add Linux system check commands"
```

Commit does NOT automatically send changes to GitHub.

---

## 5. Push

```
git push
```

Sends local commits to the remote repository.

First push:

```
git push -u origin main
```

`-u` connects the local branch with the remote branch.

After that:

```
git push
```

is usually enough.

---

## 6. Pull

```
git pull
```

Gets remote changes and integrates them into the current local branch.

Use when the remote repository may contain newer changes.

---

## 7. History

```
git log --oneline
```

Compact commit history.

Latest commit:

```
git log -1
```

---

## 8. Branches

Show branches:

```
git branch
```

Create and switch:

```
git switch -c feature-name
```

Switch:

```
git switch main
```

Merge into the current branch:

```
git merge feature-name
```

Delete local branch:

```
git branch -d feature-name
```

---

## 9. Undo before commit

Remove a file from staging:

```
git restore --staged <file>
```

The file remains on disk.

Discard local modifications:

```
git restore <file>
```

WARNING: local changes to that file will be lost.

---

## 10. .gitignore

`.gitignore` tells Git which untracked files should not be added.

Example:

```
.env
*.key
*.pem
.obsidian/
```

Important:

`.gitignore` does not automatically remove files that are already tracked by Git.

---

## 11. SSH

Test GitHub SSH authentication:

```
ssh -T git@github.com
```

Check configured remote:

```
git remote -v
```