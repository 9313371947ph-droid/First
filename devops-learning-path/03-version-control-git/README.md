# Module 03: Version Control with Git

## 🎯 Learning Objectives

By the end of this module, you will:
- Understand version control concepts and why Git is essential
- Master Git fundamentals and workflows
- Implement branching strategies for teams
- Resolve merge conflicts confidently
- Use Git in DevOps workflows (GitOps, CI/CD integration)

---

## 📚 Part 1: Why Version Control Matters

### The Problem Before Version Control

```
❌ "Who changed the production config?"
❌ "Where's the backup of yesterday's code?"
❌ "My changes broke everything, how do I revert?"
❌ "We can't work on the same file!"
❌ "Which version is deployed to production?"
```

### The Git Solution

```
✅ Complete history of every change
✅ Who changed what and when
✅ Easy rollback to any previous version
✅ Multiple people can work simultaneously
✅ Clear audit trail for compliance
✅ Branching for features, fixes, experiments
```

### Why Git Specifically?

- **Distributed**: Every developer has full repository history
- **Fast**: Optimized for performance
- **Branching**: Lightweight, easy-to-create branches
- **Industry Standard**: Used by 90%+ of developers
- **Ecosystem**: Massive tooling and platform support

---

## 🔧 Part 2: Git Fundamentals

### Git Configuration

```bash
# First-time setup
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global init.defaultBranch main

# Useful configurations
git config --global color.ui true
git config --global core.editor "vim"
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status

# View configuration
git config --list
```

### Basic Git Workflow

```bash
# Initialize a repository
git init

# Clone an existing repository
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git    # SSH

# Check status
git status

# Stage changes
git add file.txt                  # Add specific file
git add .                         # Add all changes
git add -p                        # Interactive staging

# Commit changes
git commit -m "Descriptive commit message"
git commit -am "Add feature and fix bug"  # Stage and commit tracked files

# View history
git log
git log --oneline
git log --graph --oneline --all
git log --since="2 weeks ago"
git log --author="John"
```

### Understanding Git States

```
Working Directory → Staging Area → Repository → Remote
     ↓                ↓              ↓           ↓
  Modified        Staged        Committed    Pushed
```

```bash
# See what changed
git diff                    # Unstaged changes
git diff --staged          # Staged changes
git diff HEAD~1            # Changes from last commit

# Unstage changes
git reset HEAD file.txt

# Discard changes (CAREFUL!)
git checkout -- file.txt   # Git < 2.23
git restore file.txt       # Git >= 2.23
```

---

## 🌿 Part 3: Branching & Merging

### Branch Operations

```bash
# List branches
git branch                  # Local branches
git branch -a               # All branches (including remote)
git branch --merged         # Branches merged into current

# Create branches
git branch feature-name
git branch -d feature-name  # Delete (safe, checks merge)
git branch -D feature-name  # Delete (force)

# Switch branches
git checkout branch-name
git switch branch-name      # Newer command

# Create and switch
git checkout -b new-branch
git switch -c new-branch    # Newer command
```

### Merging Strategies

#### Fast-Forward Merge
```bash
# When target branch has no new commits
git checkout main
git merge feature-branch
```

#### Three-Way Merge
```bash
# Creates merge commit when both branches have changes
git checkout main
git merge --no-ff feature-branch
```

#### Merge Conflicts

```bash
# When conflict occurs:
# 1. Git will mark the conflicted file
# 2. Edit the file to resolve
# 3. Stage the resolved file
# 4. Complete the merge

<<<<<<< HEAD
Our changes
=======
Their changes
>>>>>>> feature-branch

# After resolving:
git add resolved-file.txt
git commit
```

### Rebasing

```bash
# Rebase instead of merge (rewrites history)
git checkout feature-branch
git rebase main

# Interactive rebase (rewrite commits)
git rebase -i HEAD~3

# Common rebase operations:
# pick   - use commit
# reword - use commit, edit message
# edit   - use commit, stop for amending
# squash - combine with previous commit
# drop   - remove commit
```

**⚠️ Golden Rule of Rebasing**: Never rebase public/shared branches!

---

## 🏗️ Part 4: Git Workflows for DevOps

### Feature Branch Workflow

```
main ────────────────────────────────●
         \                          /
feature ──●──●──●──●────────────────●
```

```bash
# Create feature branch
git checkout -b feature/new-payment

# Work on feature
git add .
git commit -m "Add payment processing"

# Keep updated with main
git fetch origin
git rebase origin/main

# Push feature branch
git push -u origin feature/new-payment

# Create Pull Request/Merge Request
# Review → Approve → Merge to main
```

### Git Flow (Classic)

```
main (production)
  ↑
develop (integration)
  ↑
feature/*, release/*, hotfix/*
```

```bash
# Initialize gitflow
git flow init

# Start feature
git flow feature start new-feature
git flow feature finish new-feature

# Start release
git flow release start 1.0.0
git flow release finish 1.0.0

# Hotfix for production
git flow hotfix start critical-fix
git flow hotfix finish critical-fix
```

### GitHub Flow (Simpler)

```
main (always deployable)
  ↑
feature branches (short-lived)
```

**Rules**:
1. `main` is always deployable
2. Create branch for features
3. Open Pull Request early
4. Discuss and review
5. Deploy and test
6. Merge to main

### Trunk-Based Development (Modern DevOps)

```
main/trunk (continuously deployed)
  ↑
short-lived feature branches (< 1 day)
  OR
feature flags in code
```

**Benefits for DevOps**:
- ✅ Enables Continuous Deployment
- ✅ Reduces merge conflicts
- ✅ Faster feedback loops
- ✅ Smaller, safer changes

---

## 🔐 Part 5: Working with Remotes

### Remote Operations

```bash
# View remotes
git remote -v

# Add remote
git remote add origin https://github.com/user/repo.git

# Fetch from remote
git fetch origin
git fetch --all

# Pull changes (fetch + merge)
git pull origin main
git pull --rebase origin main

# Push changes
git push origin main
git push -u origin main    # Set upstream
git push --force           # Force push (careful!)
git push --force-with-lease # Safer force push
```

### Tagging Releases

```bash
# Create tags
git tag v1.0.0
git tag -a v1.0.0 -m "Release version 1.0.0"

# Push tags
git push origin v1.0.0
git push --tags

# List tags
git tag
git tag -l "v1.*"
```

---

## 🔍 Part 6: Advanced Git Techniques

### Undoing Mistakes

```bash
# Amend last commit
git commit --amend
git commit --amend --no-edit

# Undo commit, keep changes
git reset --soft HEAD~1

# Undo commit, discard changes
git reset --hard HEAD~1

# Undo pushed commit
git revert COMMIT_HASH

# Find lost commits
git reflog
```

### Cherry-Picking

```bash
# Apply specific commit from another branch
git cherry-pick COMMIT_HASH

# Cherry-pick range
git cherry-pick COMMIT1^..COMMIT2
```

### Stashing

```bash
# Save uncommitted changes
git stash
git stash save "WIP: feature X"

# List stashes
git stash list

# Apply stash
git stash pop        # Apply and remove
git stash apply      # Apply but keep

# Create branch from stash
git stash branch new-branch stash@{0}
```

### Bisect for Debugging

```bash
# Find which commit introduced a bug
git bisect start
git bisect bad           # Current version has bug
git bisect good v1.0.0   # Known good version

# Git will checkout commits, you mark them
git bisect good
git bisect bad

# When done
git bisect reset
```

### Submodules

```bash
# Add submodule
git submodule add https://github.com/user/repo.git path/to/submodule

# Initialize submodules
git submodule init
git submodule update

# Update submodules
git submodule update --remote
```

---

## 🚀 Part 7: Git in DevOps

### Git Hooks

```bash
# Location: .git/hooks/

# Pre-commit hook example
#!/bin/bash
# .git/hooks/pre-commit

# Run linter
npm run lint
if [ $? -ne 0 ]; then
    echo "Linting failed!"
    exit 1
fi

# Run tests
npm test
if [ $? -ne 0 ]; then
    echo "Tests failed!"
    exit 1
fi

# Make executable
chmod +x .git/hooks/pre-commit
```

### GitOps Principles

```
Git Repository = Single Source of Truth
       ↓
Automated Sync to Infrastructure
       ↓
Continuous Reconciliation
```

**GitOps Benefits**:
- Version-controlled infrastructure
- Audit trail for all changes
- Easy rollback
- Consistent environments
- Self-documenting systems

### CI/CD Integration

```yaml
# Example: GitHub Actions workflow
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build
        run: make build
      
      - name: Test
        run: make test
      
      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: ./deploy.sh
```

---

## 🏋️ Practical Exercises

### Exercise 1: Git Basics (1 hour)

**Task**: Create a repository and practice basic operations

1. Initialize a new Git repository
2. Create a README file
3. Make several commits with meaningful messages
4. Create a branch, make changes, merge back
5. View commit history in different formats

**Deliverable**: Repository with at least 10 commits

### Exercise 2: Conflict Resolution (1 hour)

**Task**: Practice resolving merge conflicts

1. Create a repository with a file
2. Create two branches from same point
3. Modify same lines in both branches
4. Attempt to merge and resolve conflicts
5. Document your conflict resolution process

### Exercise 3: Git Flow Implementation (2 hours)

**Task**: Implement Git Flow workflow

1. Initialize git-flow in a repository
2. Create a feature branch, complete it
3. Start a release branch, finish it
4. Create a hotfix for "production"
5. Document each step with screenshots

### Exercise 4: Interactive Rebase (1 hour)

**Task**: Clean up commit history

1. Create a branch with 5 messy commits
2. Use interactive rebase to:
   - Squash related commits
   - Reword commit messages
   - Reorder commits
   - Drop unnecessary commits
3. Document before/after history

### Exercise 5: Git Hooks Automation (2 hours)

**Task**: Create useful Git hooks

1. Pre-commit hook that runs linting
2. Commit-msg hook that validates message format
3. Pre-push hook that runs tests
4. Post-merge hook that updates dependencies
5. Share hooks with team (hooks directory or tool)

### Exercise 6: Disaster Recovery (1 hour)

**Task**: Practice recovering from mistakes

1. Create a repository with important commits
2. "Accidentally" delete a branch
3. Recover it using reflog
4. Reset to a previous commit
5. Revert a problematic commit

---

## 💡 Best Practices

### Commit Messages

```
❌ Bad: "fixed stuff"
❌ Bad: "update"
✅ Good: "Fix authentication timeout issue"
✅ Great: "auth: Fix timeout issue in JWT validation

- Increase timeout from 5s to 30s
- Add retry logic for transient failures
- Update error messages for clarity

Fixes #123"
```

**Conventional Commits Format**:
```
type(scope): subject

body

footer
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

### Branch Naming

```
✅ feature/user-authentication
✅ fix/login-bug
✅ hotfix/security-patch
✅ release/v1.2.0
❌ my-branch
❌ test
❌ fix_stuff
```

### General Guidelines

1. **Commit Often**: Small, atomic commits
2. **Never Commit Secrets**: Use environment variables
3. **Review Before Commit**: `git diff --staged`
4. **Pull Before Push**: Avoid conflicts
5. **Use Branches**: Never work directly on main
6. **Write Tests**: For critical functionality
7. **Document**: In commit messages and README

---

## 🔒 Security Considerations

### What NOT to Commit

```bash
❌ Passwords and API keys
❌ Database credentials
❌ Private SSH keys
❌ Personal access tokens
❌ Sensitive configuration
❌ Large binary files
```

### Prevent Accidental Commits

```bash
# .gitignore example
.env
*.key
*.pem
secrets/
credentials.json
node_modules/
__pycache__/
*.log
.DS_Store
```

### If You Accidentally Commit Secrets

```bash
# 1. Rotate the secret immediately!
# 2. Remove from history
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch PATH_TO_SECRET" \
  --prune-empty --tag-name-filter cat -- --all

# 3. Force push (notify team!)
git push --force --all

# 4. Better: Use BFG Repo-Cleaner
bfg --delete-files secret.txt
```

---

## 📝 Knowledge Check

### Quiz Questions

1. **What's the difference between `git merge` and `git rebase`?**
   <details>
   <summary>Click for Answer</summary>
   
   - **Merge**: Creates a merge commit, preserves history
   - **Rebase**: Rewrites history by replaying commits
   </details>

2. **How do you undo the last commit but keep changes?**
   <details>
   <summary>Click for Answer</summary>
   
   ```bash
   git reset --soft HEAD~1
   ```
   </details>

3. **What does `git push --force-with-lease` do?**
   <details>
   <summary>Click for Answer</summary>
   
   Safely force pushes only if remote hasn't changed since you last fetched
   </details>

4. **When should you NOT use rebase?**
   <details>
   <summary>Click for Answer</summary>
   
   On shared/public branches that others are working on
   </details>

5. **What is GitOps?**
   <details>
   <summary>Click for Answer</summary>
   
   Using Git as the single source of truth for infrastructure, with automated sync to systems
   </details>

---

## 📚 Additional Resources

### Books
- "Pro Git" by Scott Chacon (Free online!)
- "Git Pocket Guide" by Richard E. Silverman
- "Version Control with Git" by Jon Loeliger

### Tools
- [GitKraken](https://www.gitkraken.com/) - GUI client
- [SourceTree](https://www.sourcetreeapp.com/) - Free GUI
- [GitHub Desktop](https://desktop.github.com/) - Simple GUI
- [LazyGit](https://github.com/jesseduffield/lazygit) - Terminal UI

### Practice Platforms
- [Learn Git Branching](https://learngitbranching.js.org/) - Interactive tutorial
- [Git Game](https://www.git-game.com/) - Learn through challenges

### Documentation
- [Official Git Docs](https://git-scm.com/doc)
- [GitHub Guides](https://guides.github.com/)
- [Atlassian Git Tutorial](https://www.atlassian.com/git)

---

## 🎯 Next Steps

✅ Complete all practical exercises
✅ Set up Git with proper configuration
✅ Implement a branching strategy for your projects
✅ Create and share Git hooks with your team
✅ Proceed to [Module 04: CI/CD Pipelines](../04-ci-cd/README.md)

---

## 💬 Reflection Questions

1. How has Git improved your development workflow?
2. What branching strategy works best for your team?
3. How can you use Git to improve collaboration?
4. What Git practices would you implement in your organization?

---

*"Git is not just version control, it's time travel for your code."*
