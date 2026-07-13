# Git Branching and Merging - Visual Guide

Complete guide to Git branches, merging, and conflict resolution with detailed diagrams.

## Table of Contents
- [Branch Operations](#branch-operations)
- [Switching Branches](#switching-branches)
- [Merging](#merging)
- [Merge Conflicts](#merge-conflicts)
- [Rebase](#rebase)

---

## Branch Operations

### git branch (list branches)

**Lists all local branches.**

```bash
git branch
```

**Visual:**
```
Current Repository:
  main        ← Current branch (*)
  feature
  bugfix

Output:
  feature
  bugfix
* main        ← * indicates current branch
```

### git branch -a (all branches)

**Lists local and remote branches.**

```bash
git branch -a
```

**Visual:**
```
Local Branches:
  * main
    feature

Remote Branches:
  remotes/origin/main
  remotes/origin/develop
  remotes/origin/feature-login

Branch Tree:
Local                Remote (origin)
┌─────────┐         ┌─────────────────┐
│ * main  │ ←────→  │ origin/main     │
│ feature │ ←────→  │ origin/feature  │
└─────────┘         │ origin/develop  │
                    └─────────────────┘
```

### git branch feature-login (create)

**Creates new branch.**

```bash
git branch feature-login
```

**Visual:**
```
Before:
main
 ↓
[A] ── [B] ── [C]
              ↑
             HEAD

After git branch feature-login:
main
 ↓                    feature-login
[A] ── [B] ── [C] ←──────┘
              ↑
             HEAD (still on main)

Branch Pointers:
main           → [C]
feature-login  → [C]
HEAD           → main
```

### git checkout -b feature-login

**Create and switch to new branch.**

```bash
git checkout -b feature-login
# Or newer syntax:
git switch -c feature-login
```

**Visual:**
```
Before:
main
 ↓
[A] ── [B] ── [C]
              ↑
             HEAD

After:
main          feature-login
 ↓                 ↓
[A] ── [B] ── [C]
              ↑
             HEAD

Branch Pointers:
main           → [C]
feature-login  → [C]
HEAD           → feature-login
```

### git branch -m old-name new-name

**Rename branch.**

```bash
git branch -m old-feature new-feature
```

**Visual:**
```
Before:
main          old-feature
 ↓                 ↓
[A] ── [B] ── [C] ── [D]

After git branch -m old-feature new-feature:
main          new-feature
 ↓                 ↓
[A] ── [B] ── [C] ── [D]

⚠️ Only the label changed, commits unchanged
```

### git branch -d feature-login

**Delete branch (safe).**

```bash
git branch -d feature-login
```

**Visual:**
```
Before:
main          feature-login
 ↓                 ↓
[A] ── [B] ── [C] ── [D]
       ↑
      HEAD

After git branch -d feature-login:
main
 ↓
[A] ── [B] ── [C] ── [D]
       ↑            ↑
      HEAD     (commits remain)

✅ Safe: Only deletes if merged
❌ Error if unmerged changes exist
```

### git branch -D feature-login

**Force delete branch.**

```bash
git branch -D feature-login
```

**Visual:**
```
Before:
main          feature-login (unmerged)
 ↓                 ↓
[A] ── [B]    [X] ── [Y]
       ↑
      HEAD

After git branch -D feature-login:
main
 ↓
[A] ── [B]    [X] ── [Y]
       ↑            ↑
      HEAD    (orphaned commits)

⚠️ Force delete: Unmerged changes lost!
```

---

## Switching Branches

### git checkout main

**Switch to existing branch.**

```bash
git checkout main
# Or newer:
git switch main
```

**Visual:**
```
Before (on feature):
main          feature
 ↓                ↓
[A] ── [B] ── [C] ── [D]
                     ↑
                    HEAD

After git switch main:
main          feature
 ↓                ↓
[A] ── [B] ── [C] ── [D]
              ↑
             HEAD

Working Directory:
Updated to match commit [C]

HEAD Movement:
feature → main
```

### Switching with uncommitted changes

**Visual:**
```
Scenario: Uncommitted changes on feature

Current State:
feature
   ↓
[C] ── Working Directory (modified file1.txt)
       ↑
      HEAD

Attempt: git switch main

Case 1: No conflicts
✅ Success: Changes carried to main
main
 ↓
[C] ── Working Directory (still modified)
       ↑
      HEAD

Case 2: Conflicts with target branch
❌ Error: "Your local changes would be overwritten"
Solution: Commit or stash first
```

---

## Merging

### git merge feature-login (fast-forward)

**Merge when no divergent commits.**

```bash
git switch main
git merge feature-login
```

**Visual:**
```
Before Merge:
main          feature-login
 ↓                 ↓
[A] ── [B] ── [C] ── [D] ── [E]
              ↑
             HEAD (on main)

After Merge (Fast-Forward):
main
feature-login
     ↓
[A] ── [B] ── [C] ── [D] ── [E]
                            ↑
                           HEAD

Fast-Forward Result:
- No merge commit created
- main pointer moved forward
- Linear history maintained
```

### git merge feature-login (3-way merge)

**Merge with divergent branches.**

```bash
git switch main
git merge feature-login
```

**Visual:**
```
Before Merge:
       feature-login
            ↓
       [D] ── [E]
      /
[A] ── [B] ── [C] ── [F]
                     ↑
                    main
                    HEAD

After Merge:
       feature-login
            ↓
       [D] ── [E]
      /           \
[A] ── [B] ── [C] ── [F] ── [M]
                            ↑
                           main
                           HEAD

Merge Commit [M]:
- Two parents: [F] and [E]
- Combines both branches
- Message: "Merge branch 'feature-login'"

Three-Way Merge Uses:
- Common ancestor: [B]
- Main branch: [F]
- Feature branch: [E]
```

### git merge --abort

**Cancel merge in progress.**

```bash
git merge --abort
```

**Visual:**
```
During Merge (conflicts):
       feature
            ↓
       [D] ── [E]
      /           \
[A] ── [B] ── [C] ── [F] ── [M?]
                            ↑
                           main
                           HEAD
                    MERGING state

After git merge --abort:
       feature
            ↓
       [D] ── [E]
      /
[A] ── [B] ── [C] ── [F]
                     ↑
                    main
                    HEAD

Result:
- Merge cancelled
- Working directory restored
- No merge commit created
```

---

## Merge Conflicts

### Understanding Conflicts

**When conflicts occur:**

```bash
git merge feature-login
# CONFLICT (content): Merge conflict in file.txt
```

**Visual:**
```
Scenario: Both branches modified same file

main branch [F]:
file.txt:
  line 1: Hello
  line 2: World (main version)
  line 3: End

feature branch [E]:
file.txt:
  line 1: Hello
  line 2: World (feature version)
  line 3: End

Conflict in file.txt:
<<<<<<< HEAD (main)
  line 2: World (main version)
=======
  line 2: World (feature version)
>>>>>>> feature-login

Git Status During Conflict:
       feature
            ↓
       [D] ── [E]
      /           \
[A] ── [B] ── [C] ── [F] ── [M?]
                            ↑
                           main
                           HEAD
                    MERGING (conflict)
```

### Resolving Conflicts

**Step-by-step resolution:**

```bash
# 1. View conflicts
git status

# 2. Edit conflicted files
# Remove markers, keep desired content

# 3. Mark as resolved
git add file.txt

# 4. Complete merge
git commit -m "Resolve merge conflict"
```

**Visual:**
```
Step 1: Identify conflicts
┌────────────────────────────┐
│ Unmerged paths:            │
│   both modified: file.txt  │
└────────────────────────────┘

Step 2: Edit file.txt
Before (conflict markers):
<<<<<<< HEAD
  line 2: World (main version)
=======
  line 2: World (feature version)
>>>>>>> feature-login

After (resolved):
  line 2: World (combined version)

Step 3: Stage resolution
Working Directory    Staging Area
┌──────────────┐   ┌──────────────┐
│ file.txt ✓   │ → │ file.txt ✓   │
└──────────────┘   └──────────────┘

Step 4: Complete merge
       feature
            ↓
       [D] ── [E]
      /           \
[A] ── [B] ── [C] ── [F] ── [M]
                            ↑
                           main
                           HEAD
Merge commit [M] created with resolution
```

---

## Rebase

### git rebase main (linear history)

**Reapply commits on top of another branch.**

```bash
git switch feature
git rebase main
```

**Visual:**
```
Before Rebase:
       feature
            ↓
       [D] ── [E]
      /
[A] ── [B] ── [C] ── [F]
                     ↑
                    main

After Rebase:
main                    feature
 ↓                          ↓
[A] ── [B] ── [C] ── [F] ── [D'] ── [E']
                                     ↑
                                    HEAD

Changes:
- [D] and [E] rewritten as [D'] and [E']
- New commit hashes (history rewritten!)
- Linear history (no merge commit)
- Base changed from [B] to [F]

⚠️ Never rebase commits pushed to shared branches!
```

### Rebase vs Merge

**Comparison:**

```
Merge (3-way):
       feature
            ↓
       [D] ── [E]
      /           \
[A] ── [B] ── [C] ── [F] ── [M]
                            ↑
                           main

Result:
- Non-linear history
- Merge commit [M] created
- Original commits preserved
- Safe for shared branches

Rebase:
main                    feature
 ↓                          ↓
[A] ── [B] ── [C] ── [F] ── [D'] ── [E']
                                     ↑
                                    HEAD

Result:
- Linear history
- No merge commit
- History rewritten (new hashes)
- Cleaner log
- Dangerous for shared branches
```

### git rebase -i HEAD~3 (interactive)

**Edit commit history interactively.**

```bash
git rebase -i HEAD~3
```

**Visual:**
```
Current History:
feature
   ↓
[A] ── [B] ── [C] ── [D] ── [E]
                            ↑
                           HEAD

Interactive Rebase Options:
pick [C] Third commit
pick [D] Fourth commit
pick [E] Fifth commit

Commands:
- pick   = use commit
- reword = change message
- edit   = amend commit
- squash = combine with previous
- drop   = remove commit

Example: Squash [D] into [C]
pick [C] Third commit
squash [D] Fourth commit
pick [E] Fifth commit

After Rebase:
feature
   ↓
[A] ── [B] ── [C'] ── [E']
                     ↑
                    HEAD

Changes:
- [C] and [D] combined into [C']
- [E] rewritten as [E']
- Cleaner history
```

### git rebase --abort

**Cancel rebase in progress.**

```bash
git rebase --abort
```

**Visual:**
```
During Rebase (conflict):
main              feature (rebasing)
 ↓                      ↓
[A] ── [B] ── [F] ── [D'?]
                    ↑
                   HEAD
            REBASE in progress

After git rebase --abort:
       feature
            ↓
       [D] ── [E]
      /
[A] ── [B] ── [C] ── [F]
                     ↑
                    main

Result:
- Rebase cancelled
- Original commits restored
- Branch state unchanged
```

### git rebase --continue

**Continue after resolving conflicts.**

```bash
git rebase --continue
```

**Visual:**
```
Rebase Conflict:
main              feature (rebasing)
 ↓                      ↓
[A] ── [B] ── [F] ── [D'?] (conflict)
                    ↑
                   HEAD

Steps:
1. Resolve conflicts in files
2. git add <resolved-files>
3. git rebase --continue

After Continue:
main                    feature
 ↓                          ↓
[A] ── [B] ── [F] ── [D'] ── [E']
                             ↑
                            HEAD

Rebase completed successfully
```

---

## Branch Strategy Visual Summary

### Feature Branch Workflow

```
Timeline:

1. Create Feature Branch
main          feature
 ↓                ↓
[A] ── [B] ── [C]

2. Work on Feature
main          feature
 ↓                ↓
[A] ── [B] ── [C] ── [D] ── [E]

3. Meanwhile, main progresses
main                  feature
 ↓                        ↓
[A] ── [B] ── [C] ── [F] ── [G]    [D] ── [E]
                     
4. Merge Feature (3-way)
main
 ↓
[A] ── [B] ── [C] ── [F] ── [G]
                \           /
                 [D] ── [E]
                         ↓
                   [M] Merge commit
                    ↑
                   HEAD
```

### Gitflow Branches

```
Production:
main    ─────────────────[v1.0]────────────[v2.0]───→
                          ↑                  ↑
                          │                  │
Integration:              │                  │
develop ──[feat1]──[feat2]┴─[release]──[fix]┴───→
           ↑       ↑         ↑           ↑
           │       │         │           │
Features:  │       │         │           │
feature1 ──┘       │         │           │
feature2 ──────────┘         │           │
release/1.0 ─────────────────┘           │
hotfix/bug ──────────────────────────────┘

Branch Types:
main       → Production releases
develop    → Integration branch
feature/*  → New features
release/*  → Release preparation
hotfix/*   → Emergency fixes
```

---

## Quick Visual Reference

### Branch Pointer Movement

```
Create Branch:
[A] ── [B] ── [C]
              ↑
            main
              ↑
         new-branch

Switch Branch:
[A] ── [B] ── [C]
              ↑
            main
         (HEAD moves)

Delete Branch:
[A] ── [B] ── [C]
              ↑
            main
     (pointer removed)
```

### Merge Types

```
Fast-Forward:
Before: main─[A]─[B]  feature─[C]─[D]
After:  main/feature──[A]─[B]─[C]─[D]

3-Way Merge:
Before: main─[A]─[C]  feature─[B]─[D]
After:  main─[A]─[C]──[M]
              └─[B]─[D]─┘
```

---

This guide covers Git branching and merging with detailed visual representations of branch operations, HEAD movement, and merge strategies.