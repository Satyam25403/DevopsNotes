# Git Basics - Initialize, Status, Add, Commit

Essential Git commands for starting and tracking your repository with visual diagrams.

## Table of Contents
- [Initialize Repository](#initialize-repository)
- [Check Status](#check-status)
- [Add Files to Staging](#add-files-to-staging)
- [Commit Changes](#commit-changes)
- [View History](#view-history)

---

## Initialize Repository

### git init

**Creates a new Git repository in the current directory.**

```bash
git init
```

**What it does:**
- Creates `.git/` directory
- Initializes configuration files
- Sets up staging area
- Creates default branch (main/master)

**Visual:**
```
Before:
my-project/
├── file1.txt
└── file2.txt

After git init:
my-project/
├── .git/              ← New hidden directory
│   ├── config
│   ├── objects/
│   └── refs/
├── file1.txt
└── file2.txt

Repository State:
┌─────────────────┐
│  Untracked      │
│                 │
│  file1.txt      │
│  file2.txt      │
└─────────────────┘
```

---

## Check Status

### git status

**Shows the state of your working directory and staging area.**

```bash
git status
```

**Visual:**
```
Working Directory         Staging Area         Repository
┌──────────────┐         ┌──────────────┐    ┌──────────────┐
│              │         │              │    │              │
│ file1.txt ✓  │   →     │              │    │              │
│ file2.txt ✗  │         │              │    │              │
│ file3.txt ?  │         │              │    │              │
└──────────────┘         └──────────────┘    └──────────────┘

Legend:
✓ = Modified (tracked)
✗ = Modified (not staged)
? = Untracked
```

**Output Example:**
```
On branch main

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   file1.txt

Changes not staged for commit:
  (use "git add <file>..." to update)
        modified:   file2.txt

Untracked files:
  (use "git add <file>..." to include)
        file3.txt
```

### git status -s

**Short format status.**

```bash
git status -s
```

**Visual:**
```
Output:
M  file1.txt    ← Modified, staged
 M file2.txt    ← Modified, not staged
?? file3.txt    ← Untracked
```

---

## Add Files to Staging

### git add (specific file)

**Adds specific file to staging area.**

```bash
git add file1.txt
```

**Visual:**
```
Before:
Working Directory         Staging Area         Repository
┌──────────────┐         ┌──────────────┐    ┌──────────────┐
│              │         │              │    │              │
│ file1.txt ✗  │         │              │    │              │
│ file2.txt ✗  │         │              │    │              │
└──────────────┘         └──────────────┘    └──────────────┘

After git add file1.txt:
Working Directory         Staging Area         Repository
┌──────────────┐         ┌──────────────┐    ┌──────────────┐
│              │         │              │    │              │
│ file1.txt ✓  │   →     │ file1.txt ✓  │    │              │
│ file2.txt ✗  │         │              │    │              │
└──────────────┘         └──────────────┘    └──────────────┘
```

### git add . (all files)

**Adds all changes to staging area.**

```bash
git add .
```

**Visual:**
```
Before:
Working Directory         Staging Area         Repository
┌──────────────┐         ┌──────────────┐    ┌──────────────┐
│ file1.txt ✗  │         │              │    │              │
│ file2.txt ✗  │         │              │    │              │
│ file3.txt ?  │         │              │    │              │
└──────────────┘         └──────────────┘    └──────────────┘

After git add .:
Working Directory         Staging Area         Repository
┌──────────────┐         ┌──────────────┐    ┌──────────────┐
│ file1.txt ✓  │   →     │ file1.txt ✓  │    │              │
│ file2.txt ✓  │   →     │ file2.txt ✓  │    │              │
│ file3.txt ✓  │   →     │ file3.txt ✓  │    │              │
└──────────────┘         └──────────────┘    └──────────────┘
```

### git add *.js (by type)

**Adds all files of specific type.**

```bash
git add *.js
```

**Visual:**
```
Working Directory         Staging Area
┌──────────────┐         ┌──────────────┐
│ app.js ✗     │   →     │ app.js ✓     │
│ util.js ✗    │   →     │ util.js ✓    │
│ test.js ✗    │   →     │ test.js ✓    │
│ README.md ✗  │         │              │  ← Not added
└──────────────┘         └──────────────┘
```

### git add src/ (directory)

**Adds entire directory.**

```bash
git add src/
```

**Visual:**
```
Working Directory              Staging Area
┌─────────────────┐           ┌─────────────────┐
│ src/            │           │ src/            │
│   ├── app.js ✗  │   →       │   ├── app.js ✓  │
│   └── lib.js ✗  │   →       │   └── lib.js ✓  │
│ test/           │           │                 │
│   └── test.js ✗ │           │                 │  ← Not added
└─────────────────┘           └─────────────────┘
```

---

## Commit Changes

### git commit -m

**Records snapshot of staged changes.**

```bash
git commit -m "Add user authentication"
```

**Visual:**
```
Before Commit:
Working Directory    Staging Area       Repository
┌──────────────┐    ┌──────────────┐   ┌──────────────┐
│ file1.txt ✓  │    │ file1.txt ✓  │   │              │
│ file2.txt ✓  │    │ file2.txt ✓  │   │              │
└──────────────┘    └──────────────┘   └──────────────┘

After Commit:
Working Directory    Staging Area       Repository
┌──────────────┐    ┌──────────────┐   ┌──────────────┐
│              │    │              │   │  [Commit]    │
│              │    │              │   │  abc1234     │
│              │    │              │   │  file1.txt   │
│              │    │              │   │  file2.txt   │
└──────────────┘    └──────────────┘   └──────────────┘

Commit Graph:
main
 ↓
[abc1234] "Add user authentication"
```

### git commit -am

**Add and commit in one step (tracked files only).**

```bash
git commit -am "Fix login bug"
```

**Visual:**
```
Tracked Modified Files → Staging → Repository

Working Directory              Repository
┌──────────────────┐          ┌──────────────────┐
│ file1.txt ✗      │  ─────→  │  [Commit]        │
│ (tracked)        │          │  def5678         │
└──────────────────┘          │  file1.txt       │
                              └──────────────────┘

⚠️ Untracked files NOT included!

Commit Graph:
main
 ↓
[abc1234] "Add user authentication"
 ↓
[def5678] "Fix login bug"
```

### git commit --amend

**Modify the last commit.**

```bash
git commit --amend -m "Updated message"
```

**Visual:**
```
Before Amend:
main
 ↓
[abc1234] "Original message"

After Amend:
main
 ↓
[xyz9999] "Updated message"  ← New commit hash!

⚠️ Original commit abc1234 is replaced
⚠️ Never amend commits that have been pushed!
```

---

## View History

### git log

**Shows commit history.**

```bash
git log
```

**Visual:**
```
Commit History (newest first):

main
 ↓
[def5678] "Fix login bug"
Author: John Doe
Date: 2024-01-15
 ↓
[abc1234] "Add user authentication"
Author: John Doe
Date: 2024-01-14
 ↓
[xyz0000] "Initial commit"
Author: John Doe
Date: 2024-01-13
```

### git log --oneline

**Compact one-line format.**

```bash
git log --oneline
```

**Visual:**
```
Output:
def5678 Fix login bug
abc1234 Add user authentication
xyz0000 Initial commit

Timeline:
main
 ↓
[def5678] ── [abc1234] ── [xyz0000]
```

### git log --graph --oneline --all

**Graph view with all branches.**

```bash
git log --graph --oneline --all
```

**Visual:**
```
* def5678 (HEAD -> main) Fix login bug
* abc1234 Add user authentication
│
├─* ghi7890 (feature) Add feature
│ * jkl3456 Work in progress
│/
* xyz0000 Initial commit

Branch Visualization:
       feature
          ↓
    [ghi7890] ── [jkl3456]
       ↙
[xyz0000] ── [abc1234] ── [def5678]
                            ↑
                           main
                           HEAD
```

### git log -5

**Last 5 commits.**

```bash
git log -5
```

**Visual:**
```
Shows last 5:
main
 ↓
[5] newest
[4] ↓
[3] ↓
[2] ↓
[1] oldest shown
...
(older commits not shown)
```

### git log -- filename.txt

**History of specific file.**

```bash
git log -- filename.txt
```

**Visual:**
```
Commits affecting filename.txt only:

[def5678] "Update filename.txt"
Modified: filename.txt
 ↓
[abc1234] "Add filename.txt"
Created: filename.txt

Other commits not shown
```

---

## Visual Summary

**Complete Git Workflow:**

```
1. Initialize Repository
   git init
   ┌─────────────┐
   │ Empty Repo  │
   └─────────────┘

2. Create/Modify Files
   Working Directory
   ┌──────────────┐
   │ file1.txt ✗  │
   └──────────────┘

3. Stage Changes
   git add file1.txt
   Working Dir → Staging Area
   ┌──────────┐   ┌──────────┐
   │ file1 ✗  │ → │ file1 ✓  │
   └──────────┘   └──────────┘

4. Commit Changes
   git commit -m "message"
   Staging → Repository
   ┌──────────┐   ┌──────────┐
   │ file1 ✓  │ → │ [Commit] │
   └──────────┘   └──────────┘

5. View History
   git log
   main
    ↓
   [abc1234] "message"
```

**Three Areas of Git:**

```
┌────────────────┐   git add   ┌────────────────┐   git commit   ┌────────────────┐
│    Working     │  ────────→  │    Staging     │  ──────────→   │   Repository   │
│   Directory    │             │      Area      │                │   (.git dir)   │
│                │             │                │                │                │
│  Modified      │             │   Staged       │                │   Committed    │
│  files         │             │   files        │                │   snapshots    │
└────────────────┘             └────────────────┘                └────────────────┘
```

---

This guide covers Git basics with visual representations of each command's effect on your repository.