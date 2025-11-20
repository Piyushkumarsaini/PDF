# 📘 Git Notes (Beginner Friendly)

Git is a tool that helps you track changes in your code. Think of it like a **time machine** for developers — you can go back, fix mistakes, and collaborate with others easily.

---

## ✅ What is Git?

Git is a **version control system**. It:

* Saves different versions of your project
* Lets you undo mistakes
* Helps you work with a team
* Keeps your code safe

---

## ⭐ Install Git

Download from: [https://git-scm.com/](https://git-scm.com/)

Check if installed:

```bash
git --version
```

---

## ⭐ Create a New Git Project

```bash
git init
```

**Explanation:**
`git init` creates a hidden folder (`.git`) that starts tracking your project.

---

## ⭐ Add Files to Git

```bash
git add filename
```

Add everything:

```bash
git add .
```

**Explanation:**
`git add` tells Git which files you want to save in the next version.

---

## ⭐ Save Changes (Commit)

```bash
git commit -m "Your message"
```

**Explanation:**
A commit is like taking a **snapshot** of your project.

---

## ⭐ Check Status

```bash
git status
```

Shows:

* New files
* Modified files
* Tracked/untracked

---

## ⭐ See Commit History

```bash
git log
```

Press `q` to exit.

---

## ⭐ Connect Local Code to GitHub

```bash
git remote add origin https://github.com/username/repo.git
```

**Explanation:**
`origin` = nickname for your GitHub repo.

---

## ⭐ Push Code to GitHub

```bash
git push -u origin main
```

**Explanation:**
Uploads your local commits to GitHub.

---

## ⭐ Clone a Project

```bash
git clone https://github.com/user/repo.git
```

**Explanation:**
Downloads a GitHub project to your computer.

---

## ⭐ Create a New Branch

```bash
git branch dev
```

Switch to it:

```bash
git checkout dev
```

Shortcut:

```bash
git checkout -b dev
```

**Explanation:**
Branches let you add features without touching the main code.

---

## ⭐ Merge Branches

First switch to main branch:

```bash
git checkout main
```

Merge:

```bash
git merge dev
```

---

## ⭐ Delete a Branch

```bash
git branch -d dev
```

---

## ⭐ Pull Latest Code

```bash
git pull
```

**Explanation:**
Downloads changes from GitHub into your local project.

---

## ⭐ Important Git Commands (Quick Summary)

| Command                   | Use                   |
| ------------------------- | --------------------- |
| `git init`                | Start Git in a folder |
| `git add .`               | Add all files         |
| `git commit -m "msg"`     | Save changes          |
| `git status`              | Check current state   |
| `git log`                 | See history           |
| `git branch`              | Check branches        |
| `git checkout branchname` | Switch branches       |
| `git merge branchname`    | Merge branch          |
| `git push`                | Upload to GitHub      |
| `git pull`                | Download from GitHub  |

---

## Want Advanced Git Topics?

I can add:

* Git stash
* Git rebase
* Solve merge conflicts
* Forking workflow
* PR workflow
* GitHub Actions

Just tell me! 🚀

---

# 🚀 Advanced Git Topics

## ⭐ Git Stash

Temporarily save your uncommitted changes.

```bash
git stash
```

Apply them back:

```bash
git stash apply
```

List stashes:

```bash
git stash list
```

Delete stash:

```bash
git stash drop
```

---

## ⭐ Git Rebase

Rebase cleans your commit history by placing your commits on top of another branch.

Rebase onto main:

```bash
git checkout dev
git rebase main
```

Continue after fixing conflicts:

```bash
git rebase --continue
```

Abort:

```bash
git rebase --abort
```

---

## ⭐ Solving Merge Conflicts

When Git can't automatically merge:

1. Open the file
2. Look for:

```
<<<<<<<<< HEAD
Your code
===========
Other branch code
>>>>>>>>>> branchname
```

3. Edit manually
4. Save
5. Commit:

```bash
git add .
git commit -m "fixed conflict"
```

---

## ⭐ Forking Workflow (GitHub)

Used in open-source projects.

1. Fork a repo
2. Clone your fork
3. Add original repo as upstream:

```bash
git remote add upstream https://github.com/original/repo.git
```

4. Pull updates:

```bash
git pull upstream main
```

5. Push to your fork & open a Pull Request

---

## ⭐ Pull Request Workflow

1. Create a new branch
2. Make changes
3. Push the branch
4. Create a Pull Request on GitHub
5. Review + merge

---

## ⭐ Git Reset (Dangerous but Powerful)

Soft reset (keep files):

```bash
git reset --soft HEAD~1
```

Hard reset (deletes changes):

```bash
git reset --hard HEAD~1
```

---

## ⭐ Git Revert (Safe Undo)

Undo a commit without changing history:

```bash
git revert <commit-id>
```

---

## ⭐ Git Tag (Versioning)

Create tag:

```bash
git tag v1.0
```

Push tags:

```bash
git push --tags
```

---

## ⭐ Git Cherry-Pick

Move a specific commit to another branch:

```bash
git cherry-pick <commit-id>
```

---

If you want, I can also add:

* Real-world workflows (GitFlow)
* CI/CD with GitHub Actions
* Interactive rebase tutorials
* Visual diagrams for each concept 🔥
