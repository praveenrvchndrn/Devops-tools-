# Git Troubleshooting Systematic Guide

## 🎯 Troubleshooting Workflow

```
1. Identify error type (merge, conflict, auth, etc.)
2. Check repository state
3. Review git logs
4. Verify configuration
5. Test operation in isolation
6. Fix and verify
7. Document resolution
```

---

## 📋 Essential Diagnostic Commands

### Quick Health Check
```bash
# Check repository status
git status

# Check current branch
git branch
git branch -a  # Include remote branches

# Check remote configuration
git remote -v

# Check last commits
git log --oneline -10

# Check local changes
git diff
git diff --staged

# Check configuration
git config --list
git config --global --list

# Verify Git version
git --version
```

---

## 🔥 Common Issues & Troubleshooting

### 1. Merge Conflicts

**Symptoms:**
- "CONFLICT (content): Merge conflict in file"
- "Automatic merge failed; fix conflicts and then commit the result"
- Files show conflict markers (<<<<<<<, =======, >>>>>>>)

**Diagnostic Commands:**
```bash
# Check repository state
git status
# Shows: "You have unmerged paths"
# Lists: "both modified: file.txt"

# See conflicting files
git diff --name-only --diff-filter=U

# View conflict details
git diff file.txt

# Check merge progress
cat .git/MERGE_HEAD  # Exists during merge
cat .git/MERGE_MSG   # Merge message

# See what's being merged
git log --merge --oneline

# Compare branches
git log --oneline --graph --all
```

**What Conflict Looks Like:**
```javascript
const apiUrl = "http://localhost:3000";
<<<<<<< HEAD (Current Change)
const timeout = 5000;
=======
const timeout = 10000;
>>>>>>> feature-branch (Incoming Change)
```

**Practice Scenario:**
```bash
# Create repository with conflict
mkdir git-conflict-test && cd git-conflict-test
git init

# Create initial file
echo "line 1" > file.txt
echo "line 2" >> file.txt
git add file.txt
git commit -m "Initial commit"

# Create and modify in branch
git checkout -b feature
echo "line 3 from feature" >> file.txt
git commit -am "Feature change"

# Modify same line in main
git checkout main
echo "line 3 from main" >> file.txt
git commit -am "Main change"

# Try to merge - creates conflict!
git merge feature

# Check status
git status
# Shows: "both modified: file.txt"

# View conflict
cat file.txt
```

**Diagnostic Process:**
```bash
# Step 1: Identify conflicting files
git status | grep "both modified"

# Step 2: Understand what changed
git diff feature main -- file.txt

# Step 3: See conflict markers
cat file.txt

# Step 4: Check history of file
git log --oneline --all -- file.txt

# Step 5: See who made each change
git blame file.txt
```

**Resolution Strategies:**

```bash
# Strategy 1: Manual resolution
# Edit file, remove markers, keep desired changes
vi file.txt
# Remove <<<<<<<, =======, >>>>>>> markers
# Keep the code you want

# Mark as resolved
git add file.txt

# Complete merge
git commit

# Strategy 2: Accept one side completely
# Keep current branch version (HEAD)
git checkout --ours file.txt
git add file.txt
git commit

# OR keep incoming branch version
git checkout --theirs file.txt
git add file.txt
git commit

# Strategy 3: Abort merge
git merge --abort
# Restores to state before merge

# Strategy 4: Use mergetool
git mergetool
# Opens configured merge tool (vimdiff, meld, etc.)

# After resolving:
git add file.txt
git commit
```

**Real-World Fix Example:**
```bash
# Scenario: Merge conflict during pull

git pull origin main
# CONFLICT (content): Merge conflict in src/config.js

# Check what's conflicted
git status
# both modified:   src/config.js

# View the conflict
cat src/config.js
<<<<<<< HEAD
const DB_HOST = "localhost";
=======
const DB_HOST = "production.db.company.com";
>>>>>>> origin/main

# Decision: Keep production DB, update code
vi src/config.js
# Remove markers, keep production DB

const DB_HOST = "production.db.company.com";

# Mark resolved and commit
git add src/config.js
git commit -m "Resolved merge conflict - keeping production DB host"

# Verify
git log --oneline -3
git diff HEAD~1 src/config.js
```

---

### 2. Authentication/Permission Issues

**Symptoms:**
- "Permission denied (publickey)"
- "fatal: Authentication failed"
- "remote: Repository not found"
- "fatal: unable to access: Could not resolve host"

**Diagnostic Commands:**
```bash
# Test SSH connection
ssh -T git@github.com
# Expected: "Hi username! You've successfully authenticated..."

# Check SSH key
ls -la ~/.ssh/
cat ~/.ssh/id_rsa.pub

# Test SSH with verbose output
ssh -vT git@github.com

# Check Git remote URL
git remote -v
# Shows: origin git@github.com:user/repo.git (fetch)

# Check credential helper
git config --global credential.helper

# Check stored credentials (HTTPS)
git credential-cache exit
git credential reject  # Clear cached credentials

# Check Git configuration
git config --global user.name
git config --global user.email
```

**What to Check in Error Messages:**
```bash
# SSH key issues:
Permission denied (publickey)
→ SSH key not added to GitHub/GitLab

# HTTPS authentication:
Authentication failed for 'https://github.com/user/repo.git'
→ Wrong username/password or token expired

# Repository access:
remote: Repository not found
→ Repository doesn't exist or no access

# Network issues:
Could not resolve host: github.com
→ DNS/network problem
```

**Practice Scenario:**
```bash
# Test SSH authentication
ssh -T git@github.com
# If fails: Need to add SSH key

# Generate new SSH key
ssh-keygen -t ed25519 -C "your_email@example.com"
# Press Enter to accept default location

# Start SSH agent
eval "$(ssh-agent -s)"

# Add key to agent
ssh-add ~/.ssh/id_ed25519

# Copy public key
cat ~/.ssh/id_ed25519.pub
# Add this to GitHub: Settings → SSH Keys

# Test again
ssh -T git@github.com
# Should succeed
```

**Real-World Fixes:**

```bash
# Scenario 1: SSH key permission denied

# Error when pushing
git push origin main
# Permission denied (publickey)

# Check SSH key exists
ls -la ~/.ssh/id_rsa.pub
# File not found

# Generate SSH key
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Add to SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa

# Copy public key and add to GitHub
cat ~/.ssh/id_rsa.pub
# Copy output, paste in GitHub → Settings → SSH Keys

# Test
ssh -T git@github.com
git push origin main  # Should work

# Scenario 2: HTTPS authentication failure

# Error
git push origin main
# fatal: Authentication failed

# Check remote URL
git remote -v
# Shows: https://github.com/user/repo.git

# Option 1: Use Personal Access Token
# Generate token on GitHub: Settings → Developer settings → Personal access tokens
# Use token instead of password when prompted

# Option 2: Switch to SSH
git remote set-url origin git@github.com:user/repo.git
git push origin main

# Scenario 3: Corporate proxy blocking Git

# Error
git clone https://github.com/user/repo.git
# fatal: unable to access: Could not resolve host

# Configure Git proxy
git config --global http.proxy http://proxy.company.com:8080
git config --global https.proxy http://proxy.company.com:8080

# Test
git clone https://github.com/user/repo.git

# Scenario 4: Wrong repository permissions

# Error
git push origin main
# remote: Repository not found

# Check repository URL
git remote -v
# Verify: correct organization/user name

# Fix URL
git remote set-url origin git@github.com:correct-user/repo.git

# Or check access permissions on GitHub
# Settings → Manage access → Verify you have write access
```

---

### 3. Detached HEAD State

**Symptoms:**
- "You are in 'detached HEAD' state"
- "HEAD detached at commit-hash"
- Commits disappear after checkout

**Diagnostic Commands:**
```bash
# Check if in detached HEAD
git status
# Shows: "HEAD detached at <commit>"

# See current HEAD
cat .git/HEAD
# Shows: commit hash instead of "ref: refs/heads/branch-name"

# See all commits (including detached)
git reflog

# Check which branch you were on
git log --all --oneline --graph
```

**Practice Scenario:**
```bash
# Create repository
mkdir detached-test && cd detached-test
git init

# Create commits
echo "v1" > file.txt
git add file.txt
git commit -m "v1"

echo "v2" > file.txt
git commit -am "v2"

echo "v3" > file.txt
git commit -am "v3"

# Checkout old commit (creates detached HEAD)
git log --oneline
# Copy first commit hash
git checkout <commit-hash>

# Check status
git status
# Shows: HEAD detached at <hash>

# Make a commit while detached
echo "detached change" > new.txt
git add new.txt
git commit -m "Detached commit"

# Checkout main - commit "disappears"
git checkout main
git log --oneline  # Detached commit not visible

# Find it with reflog
git reflog
# Shows: <hash> HEAD@{1}: commit: Detached commit
```

**Recovery Strategies:**

```bash
# Strategy 1: Create branch from detached HEAD
# While still detached:
git checkout -b recovery-branch
# Now the commit is saved on a branch

# Strategy 2: Already left detached HEAD
# Find lost commit
git reflog
# Look for: HEAD@{1}: commit: Detached commit
# Note the hash: abc1234

# Create branch pointing to that commit
git branch recovery-branch abc1234

# Or cherry-pick the commit
git cherry-pick abc1234

# Strategy 3: Return to normal
# If you don't care about detached commits
git checkout main
# Or whatever branch you were on

# Strategy 4: Merge detached changes
# If you created branch from detached HEAD
git checkout main
git merge recovery-branch
```

**Real-World Example:**
```bash
# Scenario: Accidentally checked out commit, made changes

# Working on feature
git log --oneline
# a1b2c3d Feature X
# d4e5f6g Feature Y
# g7h8i9j Initial

# Accidentally checkout old commit
git checkout g7h8i9j
# HEAD is now at g7h8i9j Initial

# Make important changes
vi important-fix.txt
git add important-fix.txt
git commit -m "Critical hotfix"

# Realize detached HEAD
git status
# HEAD detached at g7h8i9j

# Save work immediately
git branch hotfix-branch

# Return to feature
git checkout main

# Apply hotfix
git merge hotfix-branch

# Clean up
git branch -d hotfix-branch
```

---

### 4. Corrupted Repository

**Symptoms:**
- "fatal: bad object"
- "error: object file is empty"
- "fatal: loose object is corrupt"
- Operations fail unexpectedly

**Diagnostic Commands:**
```bash
# Check repository integrity
git fsck --full

# Check for corruption
git fsck --lost-found

# Check object database
git count-objects -v

# Verify specific object
git cat-file -p <object-hash>

# Check packed objects
git verify-pack -v .git/objects/pack/*.idx
```

**Recovery Steps:**

```bash
# Step 1: Identify corruption
git fsck --full
# Shows: error: object file .git/objects/ab/cd1234 is empty

# Step 2: Try to recover from remote
git fetch origin
git reset --hard origin/main

# Step 3: If fetch fails, clone fresh
cd ..
git clone <repository-url> repository-recovered
cd repository-recovered

# Step 4: Copy local changes if any
# From old repo:
git diff > /tmp/local-changes.patch
# In new repo:
git apply /tmp/local-changes.patch

# Step 5: Cleanup old corrupted repo
cd ..
rm -rf repository-old
```

**Prevention:**
```bash
# Regular integrity checks
git fsck --full

# Garbage collection
git gc --aggressive --prune=now

# Verify after operations
git verify-pack -v .git/objects/pack/*.idx
```

---

### 5. Large File Issues

**Symptoms:**
- "remote: error: File X is 100.00 MB; this exceeds GitHub's file size limit"
- "fatal: The remote end hung up unexpectedly"
- Push/pull very slow

**Diagnostic Commands:**
```bash
# Find large files in repository
git rev-list --objects --all \
| git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
| sed -n 's/^blob //p' \
| sort --numeric-sort --key=2 \
| tail -20

# Or use git-sizer
git-sizer --verbose

# Check repository size
du -sh .git

# Find large files in current commit
find . -type f -size +10M

# Check what would be pushed
git diff --stat origin/main..HEAD
```

**Real-World Fix:**

```bash
# Scenario: Accidentally committed large file

# Error on push
git push origin main
# remote: error: File data.sql is 200.00 MB

# Find the large file in history
git rev-list --objects --all | grep data.sql

# Option 1: Remove from last commit (if not pushed yet)
git rm --cached data.sql
echo "data.sql" >> .gitignore
git commit --amend -m "Remove large file"

# Option 2: Remove from history (use BFG Repo-Cleaner)
# Download BFG: https://rtyley.github.io/bfg-repo-cleaner/
java -jar bfg.jar --delete-files data.sql
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Option 3: Use Git LFS for large files
git lfs install
git lfs track "*.sql"
git add .gitattributes
git add data.sql
git commit -m "Track SQL files with LFS"

# Push
git push origin main
```

---

### 6. Branch Management Issues

**Symptoms:**
- Cannot switch branches
- Branch not tracking remote
- Lost commits after branch deletion

**Diagnostic Commands:**
```bash
# List all branches
git branch -a
git branch -vv  # Show tracking info

# Check current branch
git branch --show-current

# See remote branches
git branch -r

# Check tracking configuration
git config --get branch.main.remote
git config --get branch.main.merge

# Find branches containing commit
git branch --contains <commit-hash>

# Find dangling branches
git reflog
git fsck --lost-found
```

**Common Issues & Fixes:**

```bash
# Issue 1: Can't switch branches (uncommitted changes)

git checkout feature-branch
# error: Your local changes would be overwritten

# Fix Option 1: Commit changes
git add .
git commit -m "WIP: Save current work"
git checkout feature-branch

# Fix Option 2: Stash changes
git stash
git checkout feature-branch
# Later: git stash pop

# Fix Option 3: Force checkout (LOSES CHANGES)
git checkout -f feature-branch

# Issue 2: Branch not tracking remote

git branch -vv
# Shows: main [no upstream set]

# Fix: Set upstream
git branch --set-upstream-to=origin/main main
# Or during push:
git push -u origin main

# Issue 3: Deleted branch with unpushed commits

# Find lost commits
git reflog
# Shows: abc1234 HEAD@{5}: commit: Important feature

# Recover branch
git branch recovered-branch abc1234

# Or cherry-pick specific commits
git cherry-pick abc1234

# Issue 4: Too many local branches

# List merged branches
git branch --merged main

# Delete merged branches (except main)
git branch --merged main | grep -v "main" | xargs git branch -d

# Force delete unmerged branch (CAREFUL!)
git branch -D old-feature
```

---

### 7. Rebase Conflicts

**Symptoms:**
- "CONFLICT (content): Merge conflict during rebase"
- "Cannot rebase: You have unstaged changes"
- Rebase stuck/interrupted

**Diagnostic Commands:**
```bash
# Check rebase status
git status
# Shows: "rebase in progress"

# See which commit is being applied
cat .git/rebase-merge/msgnum
cat .git/rebase-merge/end

# Check rebase head
cat .git/rebase-merge/head-name

# See original head
cat .git/rebase-merge/orig-head
```

**Practice Scenario:**
```bash
# Create repository with rebase conflict
mkdir rebase-test && cd rebase-test
git init

# Create base commits
echo "line1" > file.txt
git add file.txt && git commit -m "Initial"

echo "line2" >> file.txt
git commit -am "Add line2"

# Create feature branch
git checkout -b feature
echo "line3-feature" >> file.txt
git commit -am "Feature change"

# Add commit to main
git checkout main
echo "line3-main" >> file.txt
git commit -am "Main change"

# Try to rebase (creates conflict)
git checkout feature
git rebase main
# CONFLICT!
```

**Resolution Steps:**

```bash
# During rebase conflict

# Step 1: Check status
git status
# Shows conflicting files

# Step 2: Resolve conflicts
vi file.txt
# Remove <<<<<<<, =======, >>>>>>> markers

# Step 3: Mark as resolved
git add file.txt

# Step 4: Continue rebase
git rebase --continue

# If more conflicts, repeat steps 2-4

# Option: Abort rebase
git rebase --abort
# Returns to state before rebase

# Option: Skip problematic commit
git rebase --skip
# Use carefully - loses that commit's changes
```

**Real-World Example:**
```bash
# Scenario: Rebase feature branch onto updated main

# Update main
git checkout main
git pull origin main

# Rebase feature
git checkout feature-x
git rebase main

# Conflict in src/api.js
# CONFLICT (content): Merge conflict in src/api.js

# Resolve conflict
vi src/api.js
# Fix conflicts, remove markers

# Continue
git add src/api.js
git rebase --continue

# Another conflict in tests/api.test.js
vi tests/api.test.js
git add tests/api.test.js
git rebase --continue

# Rebase complete
git log --oneline --graph

# Force push (rewrites history)
git push --force-with-lease origin feature-x
```

---

## 🔍 Advanced Git Debugging

### Git Bisect (Find Bad Commit)
```bash
# Start bisect
git bisect start

# Mark current as bad
git bisect bad

# Mark old working commit as good
git bisect good <commit-hash>

# Git checks out middle commit - test it
# If bad: git bisect bad
# If good: git bisect good

# Git automatically finds the breaking commit

# End bisect
git bisect reset
```

### Git Reflog (Recovery Tool)
```bash
# See all ref updates
git reflog

# Recover deleted branch
git branch recovered-branch HEAD@{2}

# Recover from bad reset
git reset --hard HEAD@{1}

# See reflog for specific branch
git reflog show feature-branch
```

### Blame & History Analysis
```bash
# See who changed each line
git blame file.txt

# See when line was changed
git blame -L 10,20 file.txt

# Follow file renames
git log --follow file.txt

# See file at specific commit
git show commit-hash:path/to/file.txt
```

---

## 📊 Systematic Troubleshooting Checklist

When Git issues occur:

```bash
# 1. Check repository state
git status

# 2. Check current branch
git branch

# 3. Check remote configuration
git remote -v

# 4. Check recent commits
git log --oneline -5

# 5. Check for local changes
git diff

# 6. Verify remote connectivity
git fetch --dry-run

# 7. Check configuration
git config --list | grep -i remote

# 8. Check reflog for history
git reflog -10

# 9. Verify integrity
git fsck --full

# 10. Test in clean clone
git clone <repo-url> test-clone
```

---

## 💡 Key Takeaways

1. **Always check git status** - Shows current state
2. **Use git reflog** - Recovers "lost" commits
3. **Understand HEAD** - Prevents detached HEAD issues
4. **Stash before switching** - Prevents lost changes
5. **Pull before push** - Reduces conflicts
6. **Small commits** - Easier to resolve conflicts
7. **Test before force push** - Can't undo history rewrite
8. **Use branches** - Isolates experimental work
9. **Regular backups** - Clone to another location
10. **Read error messages** - Git tells you exactly what's wrong

---

## 📚 Quick Reference Card

| Issue Type | First Command | Solution Pattern |
|------------|---------------|------------------|
| Merge Conflict | `git status` | Edit files, `git add`, `git commit` |
| Auth Failed | `ssh -T git@github.com` | Check/add SSH keys |
| Detached HEAD | `git branch` | `git checkout -b branch-name` |
| Lost Commits | `git reflog` | `git branch recovery <hash>` |
| Large File | `git rev-list --objects --all` | BFG Repo-Cleaner or Git LFS |
| Can't Switch | `git status` | `git stash` or commit |
| Rebase Conflict | `git status` | Resolve, `git add`, `git rebase --continue` |

---

**Remember**: Git never truly loses data. Almost everything can be recovered with reflog. When in doubt, create a backup clone before risky operations!
