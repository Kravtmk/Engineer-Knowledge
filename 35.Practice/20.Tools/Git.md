# List

```
git --version
```

```
vim .gitignore
```

```
vim README.md
```

```
grep -RniE '(password|passwd|token|secret|api[_-]?key|private[_-]?key)' .   --exclude-dir=.obsidian   --exclude=.gitignore
```
  
```
curl -L https://creativecommons.org/licenses/by/4.0/legalcode.txt -o LICENSE
```
 
```
 head -20 LICENSE
```
  
```
ls -lh LICENSE
```
  
```
git init
```
  
```
git branch -m main
```
 
```
 git branch --show-current
```
  
```
git add .gitignore
```
  
```
git status --short
```

```
git status
```
  
```
git add .gitignore
```
  
```
git status --short
```

```
git add README.md
```
  
```
git status --short
```
  
```
git add LICENSE
```
  
```
git status --short
```
  
```
git commit -m "chore: initialize Engineer-Knowledge repository"
```
  
```
git status --short
```
 
 GitHub.com create Repository
  
```
git remote add origin git@github.com:Repository.git
```  
  
```
git remote -v
```
  
```
git status
```
  
```
git show --stat --oneline HEAD
```
  
```
git push -u origin main
```
  
```
git add file.md
```
  
```
git diff --cached
```
  
```
git commit -m "file commit"
```
  
```
git status
```
 
```
 git push
```

  

git init
git config user.name
git config --global user.name
ssh -T git@github.com
git log -1
git ls-tree -r origin/main --name-only | grep .obsidian
git rm -r --cached .obsidian/

# Git Practice

## Kata 1 — Create a safe public repository

### 1. Prepare repository files

```bash
vim .gitignore
vim README.md
```

Check for possible secrets:

```bash
grep -RniE '(password|passwd|token|secret|api[_-]?key|private[_-]?key)' . \
  --exclude-dir=.obsidian \
  --exclude=.gitignore
```

### 2. Initialize Git

```bash
git init
git branch -m main
git branch --show-current
```

### 3. Stage repository files

```bash
git add .gitignore
git add README.md
git add LICENSE
```

Check:

```bash
git status
```

### 4. First commit

```bash
git commit -m "chore: initialize repository"
```

Check:

```bash
git status
git show --stat --oneline HEAD
```

### 5. Connect remote repository

Create an empty repository on GitHub.

```bash
git remote add origin git@github.com:<USER>/<REPOSITORY>.git
git remote -v
```

### 6. First push

```bash
git push -u origin main
```

---

## Kata 2 — Daily workflow

Edit one file.

```bash
git status
git add <file>
git diff --cached
git commit -m "docs: update file"
git status
git push
```

Repeat until this workflow can be done without notes:

```text
edit
 ↓
status
 ↓
add
 ↓
diff
 ↓
commit
 ↓
push
```

## Kata 3 — Undo staging

Stage a test file:

```bash
git add <file>
git status
```

Remove it from staging:

```bash
git restore --staged <file>
git status
```

Verify that the file still exists locally.