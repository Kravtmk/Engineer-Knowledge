# Git — MIN

## Check

```bash
git --version
git status
git branch --show-current
git remote -v
```

## New repository

```bash
git init
git branch -m main
git remote add origin <URL>
```

## Daily workflow

```bash
git status
git add <file>
git diff --cached
git commit -m "message"
git push
```

## Get repository

```
git clone <URL>
```

## Update local repository

```
git pull
```

## History

```
git log --oneline
git log -1
```

## Undo

```
git restore --staged <file>
git restore <file>
```

## Branches

```
git branch
git switch -c <branch>
git switch <branch>
git merge <branch>
git branch -d <branch>
```

## First push

```
git push -u origin main
```

## GitHub SSH check

```
ssh -T git@github.com
```


