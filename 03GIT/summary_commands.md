# Git - Version Control System

Complete guide to Git version control, from basics to production workflows.

## Table of Contents
- [Git Basics](#git-basics)
- [Core Commands](#core-commands)
- [Branching and Merging](#branching-and-merging)
- [Remote Repositories](#remote-repositories)
- [Collaboration Workflows](#collaboration-workflows)
- [Advanced Operations](#advanced-operations)
- [Production Best Practices](#production-best-practices)
- [Troubleshooting](#troubleshooting)

---

## Git Basics

### Initialize Repository

```bash
# Create new Git repository
git init

# Creates .git/ directory with configs
```

### Check Status

```bash
# View file states
git status

# Short format
git status -s
```

### Add Files to Staging

```bash
# Add specific file
git add filename.txt

# Add all files
git add .

# Add all files of type
git add *.js

# Add directory
git add src/
```

### Commit Changes

```bash
# Commit with message
git commit -m "Add user authentication"

# Add and commit in one step
git commit -am "Fix login bug"

# Amend last commit
git commit --amend -m "Updated message"
```

### View History

```bash
# View commit history
git log

# One line per commit
git log --oneline

# Graph view
git log --graph --oneline --all

# Last 5 commits
git log -5

# Specific file history
git log -- filename.txt
```

---

## Core Commands

### Unstaging Files

```bash
# Remove from staging area
git restore --staged filename.txt

# Remove all from staging
git restore --staged .
```

### View Differences

```bash
# Changes not staged
git diff

# Changes staged
git diff --staged

# Between commits
git diff <commit1> <commit2>

# Specific file
git diff filename.txt

# Between branches
git diff main feature
```

### Undo Changes

```bash
# Discard local changes
git restore filename.txt

# Reset to specific commit (keep changes)
git reset <commit-hash>

# Reset and discard changes (dangerous!)
git reset --hard <commit-hash>

# Reset last commit (keep changes)
git reset HEAD~1

# Reset last 3 commits
git reset HEAD~3
```

### Stash Changes

```bash
# Save work in progress
git stash

# Stash with message
git stash save "WIP: feature implementation"

# List stashes
git stash list

# Apply last stash
git stash apply

# Apply and remove
git stash pop

# Apply specific stash
git stash apply stash@{2}

# Delete stash
git stash drop stash@{0}

# Clear all stashes
git stash clear
```

---

## Branching and Merging

### Branch Operations

```bash
# List branches
git branch

# List all (local + remote)
git branch -a

# Create branch
git branch feature-login

# Create and switch
git checkout -b feature-login
# Or (newer)
git switch -c feature-login

# Switch branch
git checkout main
# Or
git switch main

# Rename branch
git branch -m old-name new-name

# Delete branch
git branch -d feature-login

# Force delete (unmerged)
git branch -D feature-login
```

### Merging

```bash
# Merge feature into main
git switch main
git merge feature-login

# Merge with commit message
git merge feature-login -m "Merge feature-login"

# Abort merge
git merge --abort

# Continue after resolving conflicts
git merge --continue
```

### Merge Conflicts

**When conflicts occur:**
```bash
# 1. Git shows conflict
git merge feature-login
# CONFLICT (content): Merge conflict in file.txt

# 2. View conflicts
git status

# 3. Edit files manually
# <<<<<<< HEAD
# Current code
# =======
# Incoming code
# >>>>>>> feature-login

# 4. Mark as resolved
git add file.txt

# 5. Complete merge
git commit -m "Resolve merge conflict"
```

### Rebase

```bash
# Rebase feature onto main
git switch feature-login
git rebase main

# Interactive rebase (edit history)
git rebase -i HEAD~3

# Abort rebase
git rebase --abort

# Continue after resolving
git rebase --continue
```

---

## Remote Repositories

### SSH Key Setup

```bash
# Generate SSH key
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"

# Start SSH agent
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/id_rsa

# Copy public key
cat ~/.ssh/id_rsa.pub
# Add to GitHub: Settings → SSH Keys
```

### Remote Operations

```bash
# Add remote
git remote add origin git@github.com:user/repo.git

# View remotes
git remote -v

# Change remote URL
git remote set-url origin git@github.com:user/new-repo.git

# Remove remote
git remote remove origin

# Rename remote
git remote rename origin upstream
```

### Push and Pull

```bash
# Push to remote
git push origin main

# Push new branch
git push -u origin feature-login

# Push all branches
git push --all

# Force push (dangerous!)
git push --force

# Safer force push
git push --force-with-lease

# Pull changes
git pull origin main

# Pull with rebase
git pull --rebase origin main

# Fetch without merge
git fetch origin
```

### Clone Repository

```bash
# Clone repository
git clone git@github.com:user/repo.git

# Clone specific branch
git clone -b develop git@github.com:user/repo.git

# Clone with depth (shallow)
git clone --depth 1 git@github.com:user/repo.git

# Clone private repo (SSH)
git clone git@github.com:user/private-repo.git
```

---

## Collaboration Workflows

### Feature Branch Workflow

```bash
# 1. Create feature branch
git checkout -b feature/user-auth

# 2. Make changes and commit
git add .
git commit -m "Add user authentication"

# 3. Push to remote
git push -u origin feature/user-auth

# 4. Create Pull Request on GitHub

# 5. After review, merge to main
git switch main
git pull origin main
git merge feature/user-auth
git push origin main

# 6. Delete branch
git branch -d feature/user-auth
git push origin --delete feature/user-auth
```

### Gitflow Workflow

```bash
# Main branches
# - main (production)
# - develop (integration)

# Feature development
git checkout -b feature/new-feature develop
# ... work ...
git checkout develop
git merge feature/new-feature

# Release
git checkout -b release/1.0.0 develop
# ... final fixes ...
git checkout main
git merge release/1.0.0
git tag -a v1.0.0 -m "Version 1.0.0"

# Hotfix
git checkout -b hotfix/critical-bug main
# ... fix ...
git checkout main
git merge hotfix/critical-bug
git checkout develop
git merge hotfix/critical-bug
```

### Pull Requests

**Creating PR:**
1. Push branch to GitHub
2. Click "New Pull Request"
3. Select base (main) and compare (feature)
4. Review changes
5. Add description
6. Create pull request

**Reviewing PR:**
```bash
# Fetch PR locally
git fetch origin pull/123/head:pr-123
git checkout pr-123

# Test changes
# ... review code ...

# Add review comments on GitHub
# Approve or request changes
```

---

## Advanced Operations

### Tags

```bash
# List tags
git tag

# Create lightweight tag
git tag v1.0.0

# Create annotated tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# Tag specific commit
git tag v1.0.0 <commit-hash>

# Push tags
git push origin v1.0.0

# Push all tags
git push origin --tags

# Delete tag
git tag -d v1.0.0
git push origin --delete v1.0.0
```

### Cherry Pick

```bash
# Apply specific commit to current branch
git cherry-pick <commit-hash>

# Cherry pick multiple commits
git cherry-pick <hash1> <hash2>

# Cherry pick without committing
git cherry-pick -n <commit-hash>
```

### Reflog

```bash
# View reference logs
git reflog

# Recover lost commit
git reflog
git checkout <commit-hash>
git branch recovery-branch
```

### Bisect (Find Bug)

```bash
# Start bisect
git bisect start

# Mark current as bad
git bisect bad

# Mark old commit as good
git bisect good <commit-hash>

# Git will checkout middle commit
# Test and mark:
git bisect good  # or
git bisect bad

# Repeat until found
# Reset when done
git bisect reset
```

### Submodules

```bash
# Add submodule
git submodule add https://github.com/user/repo.git libs/repo

# Clone with submodules
git clone --recursive <repo-url>

# Update submodules
git submodule update --init --recursive

# Pull submodule changes
git submodule update --remote
```

---

## Production Best Practices

### Commit Messages

**Good format:**
```
<type>: <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Formatting
- `refactor`: Code restructuring
- `test`: Adding tests
- `chore`: Maintenance

**Examples:**
```bash
git commit -m "feat: add user login endpoint"
git commit -m "fix: resolve race condition in auth"
git commit -m "docs: update API documentation"
```

### .gitignore

```bash
# .gitignore
node_modules/
.env
*.log
dist/
build/
.DS_Store
*.swp
```

**Global ignore:**
```bash
git config --global core.excludesfile ~/.gitignore_global
```

### Aliases

```bash
# Useful aliases
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual 'log --graph --oneline --all'
```

### Protected Branches

**GitHub settings:**
- Settings → Branches → Add rule
- Require pull request reviews
- Require status checks
- Enforce admins
- Restrict pushes

### Code Review Checklist

- [ ] Code follows style guidelines
- [ ] Tests pass
- [ ] Documentation updated
- [ ] No merge conflicts
- [ ] Commit messages clear
- [ ] Breaking changes documented

---

## Troubleshooting

### Undo Last Commit

```bash
# Keep changes
git reset --soft HEAD~1

# Discard changes
git reset --hard HEAD~1
```

### Fix Wrong Branch

```bash
# Committed to main instead of feature
git branch feature-branch
git reset --hard HEAD~1
git checkout feature-branch
```

### Recover Deleted Branch

```bash
# Find commit hash
git reflog

# Recreate branch
git checkout -b recovered-branch <commit-hash>
```

### Large File Issues

```bash
# Remove from history
git filter-branch --tree-filter 'rm -f large-file.zip' HEAD

# Or use BFG
java -jar bfg.jar --delete-files large-file.zip
```

### Detached HEAD

```bash
# Create branch from detached state
git checkout -b new-branch

# Or go back
git checkout main
```

---

## Quick Reference

### Daily Commands

```bash
git status
git add .
git commit -m "message"
git pull
git push
```

### Branch Workflow

```bash
git checkout -b feature
# work
git add .
git commit -m "message"
git push -u origin feature
# create PR
git checkout main
git pull
```

### Undo Operations

```bash
git restore --staged file    # Unstage
git restore file             # Discard changes
git reset HEAD~1             # Undo commit
git revert <commit>          # Safe undo
```

---

This guide covers essential Git operations for development teams and production workflows.