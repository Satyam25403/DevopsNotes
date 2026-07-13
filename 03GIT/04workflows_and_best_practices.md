# Git Workflows and Best Practices - Visual Guide

Complete guide to Git workflows, collaboration patterns, commit conventions, and production best practices with detailed diagrams.

## Table of Contents
- [Feature Branch Workflow](#feature-branch-workflow)
- [Gitflow Workflow](#gitflow-workflow)
- [Pull Requests](#pull-requests)
- [Commit Message Conventions](#commit-message-conventions)
- [Git Configuration](#git-configuration)
- [Protected Branches](#protected-branches)

---

## Feature Branch Workflow

### Overview

**Simple workflow for feature development.**

**Visual:**
```
Branch Structure:
main (production)
  │
  ├──→ feature/user-auth
  ├──→ feature/dashboard
  └──→ feature/api-v2

Flow:
main ─────────────────────────────→
      \         \         \
       feature1  feature2  feature3
```

### Step 1: Create Feature Branch

```bash
git checkout -b feature/user-auth
```

**Visual:**
```
Before:
main
 ↓
[A]─[B]─[C]
         ↑
        HEAD

After:
main  feature/user-auth
 ↓          ↓
[A]─[B]─[C]
         ↑
        HEAD

New branch created and checked out
```

### Step 2: Work on Feature

```bash
# Make changes
git add .
git commit -m "Add user authentication"
```

**Visual:**
```
Progress:
main          feature/user-auth
 ↓                  ↓
[A]─[B]─[C]───[D]─[E]─[F]
                      ↑
                     HEAD

Multiple commits on feature branch
Main branch unchanged
```

### Step 3: Push to Remote

```bash
git push -u origin feature/user-auth
```

**Visual:**
```
Local                          Remote
┌────────────────────┐        ┌─────────────────────┐
│ main               │        │ origin/main         │
│  ↓                 │        │  ↓                  │
│ [A]─[B]─[C]        │  ════→ │ [A]─[B]─[C]         │
│                    │        │                     │
│ feature/user-auth  │        │ origin/user-auth    │
│  ↓                 │  ════→ │  ↓                  │
│ [A]─[B]─[C]─[D]─[E]│        │ [A]─[B]─[C]─[D]─[E] │
└────────────────────┘        └─────────────────────┘

Feature branch pushed to remote
Ready for Pull Request
```

### Step 4: Create Pull Request

**Visual:**
```
GitHub/GitLab UI:
┌────────────────────────────────┐
│ Pull Request                   │
├────────────────────────────────┤
│ feature/user-auth → main       │
│                                │
│ Changes:                       │
│  + Added login page            │
│  + Added authentication API    │
│  + Added tests                 │
│                                │
│ Reviewers: @alice, @bob        │
│                                │
│ [Create Pull Request]          │
└────────────────────────────────┘

Review Process:
Developer → PR → Reviewer → Approve → Merge
```

### Step 5: Merge to Main

```bash
git switch main
git pull origin main
git merge feature/user-auth
git push origin main
```

**Visual:**
```
Before Merge:
main              feature/user-auth
 ↓                      ↓
[A]─[B]─[C]        [D]─[E]─[F]

After Merge (Fast-Forward):
main/feature/user-auth
           ↓
[A]─[B]─[C]─[D]─[E]─[F]
                     ↑
                    HEAD

Or 3-Way Merge:
main                  feature/user-auth
 ↓                          ↓
[A]─[B]─[C]───────────[M]  [D]─[E]─[F]
           \          /
            └─[D]─[E]─[F]

Merge commit [M] combines changes
```

### Step 6: Delete Feature Branch

```bash
git branch -d feature/user-auth
git push origin --delete feature/user-auth
```

**Visual:**
```
Before Delete:
main        feature/user-auth
 ↓                ↓
[A]─[B]─[C]─[D]─[E]─[F]

After Delete:
main
 ↓
[A]─[B]─[C]─[D]─[E]─[F]

Branch pointer removed
Commits remain in history
```

### Complete Feature Workflow

**Visual:**
```
Timeline:

Day 1: Create Branch
main ──────────→
      \
       feature

Day 2-5: Development
main ──────────→
      \
       feature ─→ [commits]

Day 6: Pull Request
main ──────────→
      \
       feature ─→ [ready] → PR created

Day 7: Review & Merge
main ──────────→ ← merged
      \              ↑
       feature ──────┘

Day 8: Cleanup
main ──────────→
(feature branch deleted)
```

---

## Gitflow Workflow

### Branch Structure

**Visual:**
```
Permanent Branches:
main     ──────────[v1.0]────────[v2.0]───→ (production)
develop  ────[feat]─────[feat]────[feat]──→ (integration)

Temporary Branches:
feature/*   ──→ merged to develop
release/*   ──→ merged to main & develop
hotfix/*    ──→ merged to main & develop

Complete Structure:
main      ─────────────[1.0]──────────[1.1]──────→
                        ↑              ↑
                        │              │
develop   ──[f1]─[f2]──┴─[release]────┴──[f3]────→
          ↑     ↑        ↑              ↑
          │     │        │              │
feature/a ┘     │        │              │
feature/b ──────┘        │              │
release/1.0 ─────────────┘              │
hotfix/bug ─────────────────────────────┘
```

### Main Branch (Production)

```bash
# Only receives merges from:
# - release branches
# - hotfix branches
```

**Visual:**
```
main
 ↓
[A]─────[v1.0]─────[v1.1]─────[v2.0]───→
         ↑          ↑          ↑
         │          │          │
    release/1.0  hotfix   release/2.0

Only tagged releases
Always production-ready
Never commit directly
```

### Develop Branch (Integration)

```bash
# Receives merges from:
# - feature branches
# - release branches (after release)
# - hotfix branches (after hotfix)
```

**Visual:**
```
develop
   ↓
[A]─[B]─[C]─[D]─[E]─[F]───→
    ↑   ↑       ↑   ↑
    │   │       │   │
  feat1 feat2   │   │
           release  hotfix

Integration branch
Latest development
Pre-release code
```

### Feature Branches

**Create feature:**
```bash
git checkout -b feature/new-feature develop
```

**Visual:**
```
Create:
develop      feature/new-feature
   ↓              ↓
[A]─[B]─────[C]

Work:
develop                 feature
   ↓                       ↓
[A]─[B]─────[C]       [D]─[E]─[F]

Merge back:
develop
   ↓
[A]─[B]─[C]─[G]
         \  /
          \/
       [D]─[E]─[F]

Convention:
feature/user-auth
feature/payment-gateway
feature/dashboard
```

### Release Branches

**Create release:**
```bash
git checkout -b release/1.0.0 develop
```

**Visual:**
```
Flow:
develop          release/1.0
   ↓                  ↓
[A]─[B]─[C]─────[D]  [C]─[D']─[D'']
                      ↓
                   (bug fixes)
                      ↓
              ┌───────┴───────┐
              ↓               ↓
           main           develop
            ↓               ↓
           [v1.0]    [merged back]

Purpose:
- Final testing
- Bug fixes
- Version bumps
- Documentation
- No new features

Lifespan: 1-2 weeks
```

### Hotfix Branches

**Create hotfix:**
```bash
git checkout -b hotfix/critical-bug main
```

**Visual:**
```
Emergency Fix Flow:
main                hotfix/critical
 ↓                        ↓
[v1.0]───────────────[v1.0]─[fix]
                            ↓
                     ┌──────┴──────┐
                     ↓              ↓
                   main          develop
                    ↓              ↓
                  [v1.0.1]    [merged back]

Purpose:
- Production bugs
- Critical fixes
- Immediate deployment

Branches from: main
Merges to: main AND develop
Creates: New version tag
```

### Complete Gitflow Timeline

**Visual:**
```
Time →

Week 1-2: Feature Development
develop  ──[f1]──[f2]──[f3]──→
          ↑     ↑     ↑
          │     │     │
feature/a │     │     │
feature/b ──────┘     │
feature/c ─────────────┘

Week 3: Release Preparation
develop  ────────────→
          \
           release/1.0 ──[fixes]──→

Week 4: Release
main     ─────────[v1.0]──→
develop  ─────────[v1.0]──→
                   ↑
                   │
           release/1.0 (merged)

Week 5: Hotfix
main     ──[v1.0]────[v1.0.1]──→
                      ↑
                      │
              hotfix/bug (merged)
```

---

## Pull Requests

### Creating Pull Request

**Visual:**
```
Process:
┌─────────────────┐
│ 1. Push Branch  │
│    to Remote    │
└────────┬────────┘
         ↓
┌─────────────────┐
│ 2. Open PR UI   │
│    on GitHub    │
└────────┬────────┘
         ↓
┌─────────────────┐
│ 3. Fill Details │
│    - Title      │
│    - Description│
│    - Reviewers  │
└────────┬────────┘
         ↓
┌─────────────────┐
│ 4. Create PR    │
└─────────────────┘

PR Interface:
┌────────────────────────────────────┐
│ Add user authentication            │
├────────────────────────────────────┤
│ feature/user-auth → main           │
│                                    │
│ Description:                       │
│ - Implements JWT authentication    │
│ - Adds login/logout endpoints      │
│ - Includes unit tests              │
│                                    │
│ Files Changed: 15                  │
│ +342 -127                          │
│                                    │
│ Reviewers: @alice, @bob            │
│ Labels: enhancement, security      │
│                                    │
│ ✓ All checks passed                │
│ ✓ No merge conflicts               │
│                                    │
│ [Create Pull Request]              │
└────────────────────────────────────┘
```

### Reviewing Pull Request

**Fetch PR locally:**
```bash
git fetch origin pull/123/head:pr-123
git checkout pr-123
```

**Visual:**
```
Local Repository After Fetch:
┌─────────────────────────────┐
│ main                        │
│  ↓                          │
│ [A]─[B]─[C]                 │
│                             │
│ pr-123 (Pull Request #123)  │
│  ↓                          │
│ [A]─[B]─[C]─[D]─[E]         │
│             ↑               │
│            HEAD             │
└─────────────────────────────┘

Test locally:
npm test
npm run build
npm start

Review code:
git log main..pr-123
git diff main pr-123
```

### Review Process

**Visual:**
```
PR States:
┌─────────────────────────────┐
│ Draft                       │
│ - Work in progress          │
│ - Not ready for review      │
└─────────────┬───────────────┘
              ↓
┌─────────────────────────────┐
│ Open                        │
│ - Ready for review          │
│ - Awaiting feedback         │
└─────────────┬───────────────┘
              ↓
        ┌─────┴─────┐
        ↓           ↓
┌───────────┐  ┌────────────┐
│ Approved  │  │ Changes    │
│           │  │ Requested  │
└─────┬─────┘  └─────┬──────┘
      ↓              ↓
      │         ┌────────────┐
      │         │ Updated    │
      │         └─────┬──────┘
      │               ↓
      └───────┬───────┘
              ↓
┌─────────────────────────────┐
│ Merged                      │
│ - Changes in main branch    │
│ - PR closed                 │
└─────────────────────────────┘

Review Workflow:
Author    Reviewer    Author    Reviewer
  │          │          │          │
  │─ Submit ─→│          │          │
  │          │          │          │
  │          │─ Review ─→│          │
  │          │          │          │
  │          │←─ Changes ─│          │
  │          │          │          │
  │          │─ Approve ─→│          │
  │          │          │          │
  │          │          │─ Merge ──→│
```

---

## Commit Message Conventions

### Conventional Commits Format

```
<type>: <subject>

<body>

<footer>
```

**Visual:**
```
Structure:
┌─────────────────────────────────┐
│ feat: add user authentication   │ ← Type + Subject (50 chars)
├─────────────────────────────────┤
│                                 │
│ Implemented JWT-based auth      │ ← Body (72 chars per line)
│ system with refresh tokens.     │
│                                 │
│ - Added login endpoint          │
│ - Added token validation        │
│ - Added user session mgmt       │
│                                 │
├─────────────────────────────────┤
│ Closes #123                     │ ← Footer (issue refs)
│ Breaking Change: Auth required  │
└─────────────────────────────────┘
```

### Commit Types

**Visual:**
```
Type        Purpose                 Example
─────────────────────────────────────────────────
feat        New feature             feat: add payment gateway
fix         Bug fix                 fix: resolve login timeout
docs        Documentation           docs: update API readme
style       Code formatting         style: format with prettier
refactor    Code restructure        refactor: extract auth logic
test        Add tests               test: add user service tests
chore       Maintenance             chore: update dependencies
perf        Performance             perf: optimize DB queries
ci          CI/CD changes           ci: add deployment workflow

Prefix Meaning:
feat     → 🎉 New capability
fix      → 🐛 Bug squashed
docs     → 📝 Documentation
refactor → ♻️  Code cleanup
test     → ✅ Testing
```

### Good vs Bad Commits

**Visual:**
```
❌ Bad Commits:
[A] "fixes"
[B] "update"
[C] "wip"
[D] "asdf"

Why bad:
- No context
- Not descriptive
- Unhelpful in history
- Hard to revert

✅ Good Commits:
[A] "feat: add user authentication system"
[B] "fix: resolve race condition in payment processing"
[C] "refactor: extract database connection logic"
[D] "docs: add API endpoint documentation"

Why good:
- Clear purpose
- Descriptive
- Easy to understand
- Easy to revert
```

### Atomic Commits

**Visual:**
```
❌ One Large Commit:
[A] "Add feature + fix bugs + update docs"
     ├── feature code
     ├── bug fixes
     ├── documentation
     └── refactoring

Problems:
- Hard to review
- Can't revert partially
- Mixed concerns

✅ Atomic Commits:
[A] "feat: add user registration"
[B] "fix: resolve validation bug"
[C] "docs: add registration API docs"
[D] "refactor: extract validation logic"

Benefits:
- Easy to review
- Can revert individually
- Clear history
- Single responsibility
```

---

## Git Configuration

### User Configuration

```bash
git config --global user.name "John Doe"
git config --global user.email "john@example.com"
```

**Visual:**
```
Configuration Levels:
┌──────────────────────────────────┐
│ System Level (--system)          │
│ /etc/gitconfig                   │
│ Applies to all users             │
└────────────┬─────────────────────┘
             ↓
┌──────────────────────────────────┐
│ Global Level (--global)          │
│ ~/.gitconfig                     │
│ Applies to current user          │
└────────────┬─────────────────────┘
             ↓
┌──────────────────────────────────┐
│ Local Level (default)            │
│ .git/config                      │
│ Applies to current repo          │
└──────────────────────────────────┘

Priority: Local > Global > System
```

### Useful Aliases

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual 'log --graph --oneline --all'
```

**Visual:**
```
Usage After Setup:
Instead of:          Use:
git status       →   git st
git checkout     →   git co
git branch       →   git br
git commit       →   git ci

Custom Aliases:
git unstage file.txt  →  git reset HEAD -- file.txt
git last              →  git log -1 HEAD
git visual            →  git log --graph --oneline --all

Result:
Faster workflow
Less typing
Custom commands
```

### .gitignore Configuration

```bash
# .gitignore file
node_modules/
.env
*.log
dist/
build/
.DS_Store
*.swp
```

**Visual:**
```
Repository Structure:
my-project/
├── .gitignore          ← Ignore rules
├── src/
│   ├── app.js         ✓ Tracked
│   └── config.js      ✓ Tracked
├── node_modules/       ✗ Ignored
├── .env                ✗ Ignored
├── debug.log           ✗ Ignored
└── dist/               ✗ Ignored

Ignore Patterns:
node_modules/     → Directory
*.log             → All .log files
.env              → Specific file
dist/             → Build directory

Benefits:
- Keep repo clean
- Avoid sensitive data
- Reduce repo size
- Focus on source code
```

### Global .gitignore

```bash
git config --global core.excludesfile ~/.gitignore_global
```

**Visual:**
```
Scope:
Local .gitignore       Global .gitignore
(per repository)       (all repositories)
┌─────────────┐       ┌─────────────┐
│ node_modules│       │ .DS_Store   │
│ .env        │       │ *.swp       │
│ dist/       │       │ .idea/      │
│ *.log       │       │ *.bak       │
└─────────────┘       └─────────────┘
      │                     │
      └──────┬──────────────┘
             ↓
      Combined Ignore Rules

Use Cases:
Local:  Project-specific
Global: OS files, editor configs
```

---

## Protected Branches

### Branch Protection Rules

**Visual:**
```
GitHub Settings:
┌─────────────────────────────────────┐
│ Branch Protection Rules             │
├─────────────────────────────────────┤
│ Branch: main                        │
│                                     │
│ ☑ Require pull request reviews     │
│   ├─ Required reviewers: 2         │
│   └─ Dismiss stale reviews         │
│                                     │
│ ☑ Require status checks             │
│   ├─ CI/CD must pass               │
│   ├─ Tests must pass               │
│   └─ Code coverage > 80%           │
│                                     │
│ ☑ Require signed commits            │
│                                     │
│ ☑ Include administrators            │
│   (No bypass for admins)           │
│                                     │
│ ☑ Restrict who can push             │
│   Only: release managers           │
│                                     │
│ ☐ Allow force pushes                │
│ ☐ Allow deletions                   │
└─────────────────────────────────────┘

Effect:
┌─────────┐                ┌─────────┐
│ Direct  │    ✗ BLOCKED   │  main   │
│  Push   │   ─────────────→│         │
└─────────┘                └─────────┘

┌─────────┐                ┌─────────┐
│   PR    │    ✓ ALLOWED   │  main   │
│ (reviewed)─────────────→│         │
└─────────┘                └─────────┘
```

### Code Review Workflow

**Visual:**
```
Protected Branch Flow:
┌────────────────────────────────┐
│ 1. Developer creates PR        │
└───────────┬────────────────────┘
            ↓
┌────────────────────────────────┐
│ 2. Automated Checks            │
│    - Unit tests                │
│    - Integration tests         │
│    - Linting                   │
│    - Security scan             │
└───────────┬────────────────────┘
            ↓
        ✓ Pass? ─────No──→ Fix and retry
            │
           Yes
            ↓
┌────────────────────────────────┐
│ 3. Code Review                 │
│    - 2 reviewers required      │
│    - All comments resolved     │
└───────────┬────────────────────┘
            ↓
┌────────────────────────────────┐
│ 4. Merge to main               │
│    - Squash or merge commit    │
│    - Protected branch updated  │
└────────────────────────────────┘

Enforcement:
Developer ─→ PR ─→ Checks ─→ Review ─→ main
                     ↓          ↓
                   Must Pass  Must Approve
```

---

## Quick Reference

### Feature Workflow
```
1. git checkout -b feature/name
2. Make changes + commit
3. git push -u origin feature/name
4. Create PR
5. Review & merge
6. Delete branch
```

### Gitflow Branches
```
main     → Production
develop  → Integration
feature/ → New features
release/ → Release prep
hotfix/  → Emergency fixes
```

### Commit Convention
```
<type>: <subject>
feat, fix, docs, style, refactor, test, chore
```

---

This guide covers Git workflows, collaboration patterns, commit conventions, and production best practices with detailed visual representations.