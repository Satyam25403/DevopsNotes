# Git Advanced Operations - Visual Guide

Complete guide to advanced Git operations including tags, cherry-pick, reflog, bisect, and submodules with detailed diagrams.

## Table of Contents
- [Tags](#tags)
- [Cherry Pick](#cherry-pick)
- [Reflog](#reflog)
- [Bisect](#bisect)
- [Stash](#stash)
- [Reset Operations](#reset-operations)

---

## Tags

### git tag (list tags)

**List all tags in repository.**

```bash
git tag
```

**Visual:**
```
Repository with tags:
        v1.0.0      v1.1.0      v2.0.0
           ↓           ↓           ↓
[A]───[B]───[C]───[D]───[E]───[F]───[G]
                                    ↑
                                   main
                                   HEAD

Output:
v1.0.0
v1.1.0
v2.0.0
```

### git tag v1.0.0 (lightweight tag)

**Create simple tag pointer.**

```bash
git tag v1.0.0
```

**Visual:**
```
Before:
main
 ↓
[A]───[B]───[C]
             ↑
            HEAD

After git tag v1.0.0:
main   v1.0.0
 ↓       ↓
[A]───[B]───[C]
             ↑
            HEAD

Tag Properties:
- Just a pointer to commit
- No metadata stored
- Lightweight (minimal)
```

### git tag -a v1.0.0 -m (annotated tag)

**Create tag with metadata.**

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
```

**Visual:**
```
After git tag -a v1.0.0:

┌────────────────────────┐
│  Tag Object: v1.0.0    │
│  Message: Release 1.0  │
│  Tagger: John Doe      │
│  Date: 2024-01-15      │
│  Points to: [C]        │
└───────────┬────────────┘
            ↓
main     v1.0.0
 ↓          ↓
[A]───[B]───[C]
             ↑
            HEAD

Tag Properties:
- Full Git object
- Contains metadata
- Recommended for releases
```

### git tag v1.0.0 <commit-hash>

**Tag specific commit.**

```bash
git tag v1.0.0 abc1234
```

**Visual:**
```
Current State:
main
 ↓
[A]───[B]───[C]───[D]───[E]
                         ↑
                        HEAD

Tag older commit:
git tag v1.0.0 <hash-of-B>

Result:
         v1.0.0   main
            ↓       ↓
[A]───[B]───[C]───[D]───[E]
                         ↑
                        HEAD

Use Case:
- Forgot to tag release
- Retroactive versioning
```

### git push origin v1.0.0

**Push tag to remote.**

```bash
git push origin v1.0.0
```

**Visual:**
```
Before Push:
Local                    Remote
v1.0.0                  (no tag)
   ↓
[A]─[B]─[C]             [A]─[B]─[C]

After git push origin v1.0.0:
Local                    Remote
v1.0.0                  v1.0.0
   ↓                       ↓
[A]─[B]─[C]             [A]─[B]─[C]

⚠️ Tags not pushed automatically!
Must push explicitly
```

### git push origin --tags

**Push all tags.**

```bash
git push origin --tags
```

**Visual:**
```
Local                    Remote
v1.0   v1.1   v2.0      (no tags)
  ↓     ↓     ↓
[A]─[B]─[C]─[D]─[E]     [A]─[B]─[C]─[D]─[E]

After git push --tags:
Local                    Remote
v1.0   v1.1   v2.0      v1.0   v1.1   v2.0
  ↓     ↓     ↓           ↓     ↓     ↓
[A]─[B]─[C]─[D]─[E]     [A]─[B]─[C]─[D]─[E]

All tags pushed at once
```

### git tag -d v1.0.0 (delete local)

**Delete local tag.**

```bash
git tag -d v1.0.0
```

**Visual:**
```
Before:
v1.0.0
   ↓
[A]─[B]─[C]─[D]
             ↑
            main

After git tag -d v1.0.0:
[A]─[B]─[C]─[D]
             ↑
            main

Tag removed locally
Commit remains unchanged
```

### git push origin --delete v1.0.0

**Delete remote tag.**

```bash
git push origin --delete v1.0.0
```

**Visual:**
```
Local                    Remote
                        v1.0.0 (delete this)
                           ↓
[A]─[B]─[C]             [A]─[B]─[C]

After push delete:
Local                    Remote
[A]─[B]─[C]             [A]─[B]─[C]

Remote tag deleted
```

---

## Cherry Pick

### git cherry-pick <commit-hash>

**Apply specific commit to current branch.**

```bash
git cherry-pick abc1234
```

**Visual:**
```
Initial State:
main          feature
 ↓               ↓
[A]─[B]─[C]    [A]─[B]─[X]─[Y]
         ↑
        HEAD

Want [X] in main:
git switch main
git cherry-pick <hash-of-X>

After Cherry-Pick:
main              feature
 ↓                   ↓
[A]─[B]─[C]─[X']    [A]─[B]─[X]─[Y]
             ↑
            HEAD

Result:
- [X'] is copy of [X]
- New commit hash
- Original [X] unchanged
- Changes from [X] applied to main
```

### Multiple cherry-picks

```bash
git cherry-pick abc1234 def5678
```

**Visual:**
```
feature branch:
[A]─[B]─[X]─[Y]─[Z]

main branch (before):
[A]─[B]─[C]
         ↑
        HEAD

Cherry-pick [X] and [Z]:
git cherry-pick <X-hash> <Z-hash>

main branch (after):
[A]─[B]─[C]─[X']─[Z']
                   ↑
                  HEAD

Skipped [Y] - only picked [X] and [Z]
```

### git cherry-pick -n (no commit)

**Apply changes without committing.**

```bash
git cherry-pick -n abc1234
```

**Visual:**
```
Process:
feature           main
  ↓                ↓
[X]              [C]

git cherry-pick -n <X>

Working Directory    Staging Area    Repository
┌──────────────┐    ┌──────────────┐    ┌──────┐
│ Changes from │ →  │ Changes from │    │ [C]  │
│ [X] applied  │    │ [X] staged   │    │  ↑   │
└──────────────┘    └──────────────┘    │ HEAD │
                                        └──────┘

No commit created yet
Can modify before committing
```

---

## Reflog

### git reflog (view reference log)

**Shows history of HEAD movements.**

```bash
git reflog
```

**Visual:**
```
HEAD History:
abc1234 (HEAD@{0}) commit: Add feature
def5678 (HEAD@{1}) checkout: moving to main
ghi9012 (HEAD@{2}) commit: Fix bug
jkl3456 (HEAD@{3}) reset: moving to HEAD~1
mno7890 (HEAD@{4}) commit: Initial

Timeline:
HEAD@{4}    HEAD@{3}    HEAD@{2}    HEAD@{1}    HEAD@{0}
   ↓           ↓           ↓           ↓           ↓
[Initial] → [reset] → [Fix bug] → [checkout] → [Add feature]
                                                    ↑
                                                   NOW
```

### Recover lost commit

**Using reflog to find lost work.**

```bash
git reflog
git checkout abc1234
git branch recovery-branch
```

**Visual:**
```
Scenario: Accidentally deleted commits

Before Reset:
main
 ↓
[A]─[B]─[C]─[D]─[E]
                 ↑
                HEAD

After git reset --hard HEAD~2:
main
 ↓
[A]─[B]─[C]
         ↑
        HEAD

[D]─[E] appear lost!

Recovery with reflog:
git reflog
> abc1234 HEAD@{1} commit: Add [E]

git checkout abc1234
[A]─[B]─[C]─[D]─[E]
                 ↑
              HEAD (detached)

git branch recovery-branch
main          recovery-branch
 ↓                 ↓
[A]─[B]─[C]    [D]─[E]
         ↑          ↑
        main       HEAD

Commits recovered!
```

---

## Bisect

### git bisect (binary search for bugs)

**Find commit that introduced a bug.**

```bash
git bisect start
git bisect bad           # Current commit is bad
git bisect good abc1234  # This old commit was good
```

**Visual:**
```
Commit History (50 commits):
[A]─[B]─...─[X]─...─[Y]─...─[Z]
 ↑                           ↑
good                        bad
(works)                  (broken)

Binary Search Process:

Step 1: Test middle [M1]
[A]────────[M1]────────[Z]
 ↑           ↑          ↑
good        test?      bad

Test → broken
git bisect bad

Step 2: Test new middle [M2]
[A]─────[M2]─────[M1]
 ↑        ↑        ↑
good    test?     bad

Test → works
git bisect good

Step 3: Test [M3]
     [M2]──[M3]──[M1]
       ↑     ↑     ↑
      good  test? bad

Step 4: Found!
[M3] is the first bad commit!

Instead of testing 50 commits:
Binary search = log₂(50) ≈ 6 tests
```

### Complete bisect workflow

```bash
git bisect start
git bisect bad                    # Current is broken
git bisect good v1.0.0           # v1.0.0 worked

# Git checks out middle commit
# Test the code
git bisect good  # or bad

# Repeat until found
# When done:
git bisect reset
```

**Visual:**
```
Process Flow:
┌─────────────────┐
│  bisect start   │
└────────┬────────┘
         ↓
┌─────────────────┐
│  Mark bad/good  │
└────────┬────────┘
         ↓
┌─────────────────┐
│  Git checks out │
│  middle commit  │
└────────┬────────┘
         ↓
┌─────────────────┐
│  Test manually  │
└────────┬────────┘
         ↓
    ┌────┴────┐
    ↓         ↓
  Works?   Broken?
    ↓         ↓
git bisect  git bisect
  good        bad
    ↓         ↓
    └────┬────┘
         ↓
   ┌─────────┐
   │  Found  │
   │  or     │
   │Continue │
   └─────────┘
```

---

## Stash

### git stash (save work)

**Temporarily save uncommitted changes.**

```bash
git stash
```

**Visual:**
```
Before Stash:
Working Directory    Staging Area    Repository
┌──────────────┐    ┌──────────────┐    ┌──────┐
│ file1.txt ✗  │    │ file2.txt ✓  │    │ [C]  │
│ file3.txt ✗  │    │              │    │  ↑   │
└──────────────┘    └──────────────┘    │ HEAD │
                                        └──────┘

After git stash:
Working Directory    Staging Area    Repository
┌──────────────┐    ┌──────────────┐    ┌──────┐
│ (clean)      │    │ (clean)      │    │ [C]  │
│              │    │              │    │  ↑   │
└──────────────┘    └──────────────┘    │ HEAD │
                                        └──────┘

Stash Storage:
┌──────────────────────────┐
│ stash@{0}                │
│ - file1.txt changes      │
│ - file2.txt changes      │
│ - file3.txt changes      │
└──────────────────────────┘

Changes saved, working directory clean
```

### git stash save "message"

**Stash with description.**

```bash
git stash save "WIP: feature implementation"
```

**Visual:**
```
Stash List:
stash@{0}: WIP: feature implementation
stash@{1}: On main: fixing bug
stash@{2}: On develop: experiment

Each stash has:
- Unique identifier
- Branch name
- Custom message
- Timestamp
```

### git stash list

**View all stashes.**

```bash
git stash list
```

**Visual:**
```
Stash Stack (most recent first):
┌────────────────────────────┐
│ stash@{0}: WIP: feature    │ ← Most recent
├────────────────────────────┤
│ stash@{1}: Bug fix         │
├────────────────────────────┤
│ stash@{2}: Experiment      │ ← Oldest
└────────────────────────────┘

Timeline:
stash@{2} → stash@{1} → stash@{0}
(oldest)               (newest)
```

### git stash apply

**Apply stash without removing.**

```bash
git stash apply
```

**Visual:**
```
Before Apply:
Stash:                Working Directory:
┌──────────────┐     ┌──────────────┐
│ stash@{0}    │     │ (clean)      │
│ - changes    │     │              │
└──────────────┘     └──────────────┘

After git stash apply:
Stash:                Working Directory:
┌──────────────┐     ┌──────────────┐
│ stash@{0}    │ ──→ │ changes ✗    │
│ - changes    │     │ restored     │
└──────────────┘     └──────────────┘
    (kept)

Stash still in storage
Can apply multiple times
```

### git stash pop

**Apply and remove stash.**

```bash
git stash pop
```

**Visual:**
```
Before Pop:
Stash Stack:          Working Directory:
┌──────────────┐     ┌──────────────┐
│ stash@{0}    │     │ (clean)      │
│ stash@{1}    │     │              │
└──────────────┘     └──────────────┘

After git stash pop:
Stash Stack:          Working Directory:
┌──────────────┐     ┌──────────────┐
│ stash@{1}    │     │ changes ✗    │
│ (0 removed)  │     │ from @{0}    │
└──────────────┘     └──────────────┘

stash@{0} deleted after applying
stash@{1} becomes new @{0}
```

### git stash apply stash@{2}

**Apply specific stash.**

```bash
git stash apply stash@{2}
```

**Visual:**
```
Stash List:
┌────────────────┐
│ stash@{0}      │
│ stash@{1}      │
│ stash@{2} ←──  │ Apply this one
│ stash@{3}      │
└────────────────┘
        ↓
Working Directory
┌────────────────┐
│ Changes from   │
│ stash@{2}      │
└────────────────┘

Can apply any stash, not just latest
```

### git stash drop / clear

**Remove stashes.**

```bash
git stash drop stash@{0}  # Remove specific
git stash clear           # Remove all
```

**Visual:**
```
Before:
┌────────────────┐
│ stash@{0}      │
│ stash@{1}      │
│ stash@{2}      │
└────────────────┘

After git stash drop stash@{1}:
┌────────────────┐
│ stash@{0}      │
│ stash@{1}      │ (was @{2})
└────────────────┘

After git stash clear:
┌────────────────┐
│ (empty)        │
└────────────────┘

⚠️ Cannot undo stash clear!
```

---

## Reset Operations

### git reset HEAD~1 (soft)

**Undo commit, keep changes staged.**

```bash
git reset HEAD~1
# Or explicitly:
git reset --soft HEAD~1
```

**Visual:**
```
Before Reset:
main
 ↓
[A]─[B]─[C]
         ↑
        HEAD

After git reset HEAD~1:
main
 ↓
[A]─[B]
     ↑
    HEAD

Working Directory    Staging Area    Repository
┌──────────────┐    ┌──────────────┐    ┌──────┐
│ (unchanged)  │    │ Changes from │    │ [B]  │
│              │    │ [C] staged   │    │  ↑   │
└──────────────┘    └──────────────┘    │ HEAD │
                                        └──────┘

Commit [C] undone
Changes still staged
Ready to recommit
```

### git reset --hard HEAD~1

**Undo commit, discard all changes.**

```bash
git reset --hard HEAD~1
```

**Visual:**
```
Before:
main
 ↓
[A]─[B]─[C]
         ↑
        HEAD

After git reset --hard HEAD~1:
main
 ↓
[A]─[B]    [C]
     ↑      ↑
    HEAD  (orphaned)

Working Directory    Staging Area    Repository
┌──────────────┐    ┌──────────────┐    ┌──────┐
│ (clean)      │    │ (clean)      │    │ [B]  │
│              │    │              │    │  ↑   │
└──────────────┘    └──────────────┘    │ HEAD │
                                        └──────┘

⚠️ DANGER:
- Commit [C] removed
- All changes lost
- Cannot undo (unless using reflog)
```

### Reset modes comparison

**Visual:**
```
Initial State:
[A]─[B]─[C]
         ↑
        HEAD

git reset --soft HEAD~1:
[A]─[B]
     ↑
    HEAD
Working: unchanged
Staging: has [C] changes
Repo: [B]

git reset (--mixed) HEAD~1:
[A]─[B]
     ↑
    HEAD
Working: has [C] changes
Staging: clean
Repo: [B]

git reset --hard HEAD~1:
[A]─[B]
     ↑
    HEAD
Working: clean
Staging: clean
Repo: [B]

Summary:
--soft:  Keep in staging
--mixed: Keep in working (default)
--hard:  Discard everything
```

---

## Quick Visual Reference

### Commit Recovery

```
Lost Commit → reflog → Find hash → Create branch
    [X]         ↓         ↓            ↓
             History   abc1234    recovery-branch
```

### Stash Workflow

```
Working ─┬─→ stash save ─→ Stash Storage
         │                      ↓
         └←── stash apply/pop ──┘
```

### Cherry-Pick Flow

```
Source Branch:  [A]─[B]─[X]─[Y]
                       ↓
                  cherry-pick
                       ↓
Target Branch:  [A]─[B]─[C]─[X']
```

---

This guide covers advanced Git operations with detailed visual representations of tags, cherry-pick, reflog, bisect, stash, and reset operations.