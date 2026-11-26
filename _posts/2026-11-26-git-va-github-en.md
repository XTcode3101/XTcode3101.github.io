---
title: "Git and GitHub"
description: "Core Git and GitHub guide: the three working areas, daily commit workflow, branching, remote syncing, merge conflicts, and safe rollbacks."
date: 2026-11-26 17:53:00 +0700
categories: [DevOps, Git & GitHub]
tags: [git, github, version-control, branching, devops]
pin: false
toc: true
---

## 1. Overview

**Git** is a Distributed Version Control System (DVCS) created by Linus Torvalds in 2005. It tracks source code changes over time, allowing developers to inspect history, roll back to earlier versions when bugs occur, and merge code cleanly when multiple people collaborate on the same project.

**GitHub** is a cloud-based source code hosting platform (Remote Repository) built around Git. It enables individuals and teams to back up, synchronize, and collaborate on projects over the Internet. *(Note: Microsoft acquired GitHub in 2018, whereas Git itself is an independent tool created much earlier.)*

---

## 2. Main Components

To understand how Git works, it helps to see how the three local working areas interact with branches and remote repositories:

- **Working Directory**: The actual project folder on your machine where you create, edit, or delete files. Changes made here are immediately visible on disk, but Git has not recorded them in history yet.
- **Staging Area**: An intermediate preparation zone between the Working Directory and the Local Repository. It lets you selectively group the exact changes you want to include in the next snapshot.
- **Local Repository**: Stored inside the hidden `.git` folder within your project. It holds the complete history of committed versions on your machine, allowing you to inspect past states or restore earlier snapshots at any time.
- **Branch**: An independent line of development split off from the main codebase (typically `main` or `master`). Branches let you build new features or collaborate without disrupting stable code on the main branch.
- **Remote Repository (GitHub)**: A copy of the repository hosted on GitHub's servers. Team members push local commits to the remote repository and pull updates made by others back to their machines.

---

## 3. Usage

### Initial Setup

Before creating your first commit, configure your author identity so Git can attach your name and email to every commit in the history:

```bash
git config --global user.name "<Your Name>"
git config --global user.email "<Your Email>"
```

Next, choose how to initialize your project depending on whether it starts locally or already exists on GitHub:

- **Start a new local repository**: Run `git init` inside your project folder to initialize a Git repository (creating the `.git` directory). When you are ready to connect this local repo to an existing empty repository on GitHub, link them with `git remote add origin <url>` and verify the connection with `git remote -v`.
- **Download an existing repository from GitHub**: Run `git clone <url>` once to download the entire project and its commit history to your machine; this automatically sets up the `origin` remote connection.

To prevent Git from tracking unwanted or sensitive files (such as `node_modules/`, build outputs, `.env` files, passwords, or API keys), create a `.gitignore` file at the project root and list those file or folder patterns inside it.

### Daily Versioning Workflow

While editing files in your Working Directory, check the current state before saving a snapshot to see which files are modified, which are staged, and which branch you are on:

```bash
git status
```

After adding or modifying code and confirming your changes, record a new version in your Local Repository in two steps:

1. **Stage your changes**: Run `git add .` to move all modified, added, and deleted files into the Staging Area.
2. **Create a commit**: Run `git commit -m "<change description>"` to save the staged snapshot into the Local Repository. Always include `-m` with a clear message to document why the change was made and prevent Git from opening a terminal text editor.

To view a compact list of past commits and look up a commit hash when inspecting history or rolling back, run:

```bash
git log --oneline
```

Once you have a commit hash from `git log --oneline` and want to go back to an earlier version, choose the approach that fits your goal:

- **Temporarily inspect or test an older version**: Run `git switch --detach <commit-hash>` (or `git checkout <commit-hash>`) to view your working directory at that exact point in time. When you are done, run `git switch main` (or your branch name) to return to the latest commit.
- **Undo a faulty commit while preserving shared history**: Run `git revert <commit-hash>` to create a new commit that reverses the faulty commit's changes (safe even after pushing to GitHub — see *Handling Conflicts and Safe Rollbacks* below).
- **Reset your local branch back to an earlier commit**: Run `git reset --hard <commit-hash>` to move the current branch back to that commit and discard later changes. *(Use only on local commits that have not been pushed to GitHub, as this rewrites commit history.)*

### Working with Branches

When developing a new feature or working in a team, isolate your work on a dedicated branch instead of committing directly to `main`:

1. **Create and switch to a new branch**: Run `git switch -c <branch-name>` (or `git checkout -b <branch-name>`). Note that `git branch <branch-name>` only creates the branch without switching to it, which can easily lead to committing on the wrong branch.
2. **Develop and commit**: Write and test your code on the new branch, then run `git add .` and `git commit -m "..."` as usual.
3. **Return to the main branch and merge**: Once the feature is stable, switch back to the main branch with `git switch main`, and then run `git merge <branch-name>` to integrate your branch changes into `main`.

### Syncing with GitHub (Remote)

When collaborating with others, keep your local repository synchronized with GitHub:

- **Preview remote updates without merging**: Run `git fetch` if you only want to download new metadata and commits from GitHub to inspect them without modifying your current local branch.
- **Download and merge updates**: Run `git pull` before starting new work; this fetches the latest commits from GitHub and immediately merges them into your current branch.
- **Upload local commits**: After completing your local workflow (`branch` → `add` → `commit` → `merge`), run `git push` to upload your commits to GitHub.

### Handling Conflicts and Safe Rollbacks

**Resolving Merge Conflicts**:
If two people modify the same lines of code and you run `git merge` or `git pull`, Git pauses and flags a conflict. Open the affected file, locate the conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`), keep the correct code, remove all marker lines, and finish the merge by running `git add .` followed by `git commit`.

**Undoing Mistakes Safely**:
- **Discard uncommitted changes**: Run `git restore <file-name>` to revert unsaved edits in the Working Directory back to the last commit. If the file was already staged with `git add`, run `git restore --staged <file-name>` first to unstage it.
- **Undo a commit already pushed to GitHub**: Find the faulty commit hash with `git log --oneline` and run `git revert <commit-hash>`. This creates a new commit that reverses the faulty commit's changes without rewriting shared history, making it safe for team branches.

### Quick Command Reference

| Category | Command | Purpose |
| :--- | :--- | :--- |
| **Setup & Init** | `git config --global ...` | Set `user.name` and `user.email` before the first commit |
| | `git init` / `git clone <url>` | Initialize a new local repo or clone an existing repo from GitHub |
| | `git remote add origin <url>` | Link a local `git init` repo to GitHub (`git remote -v` to verify) |
| **Versioning** | `git status` / `git log --oneline` | Check current working tree status or view compact commit history |
| | `git add .` | Stage changes into the Staging Area |
| | `git commit -m "<message>"` | Record staged changes as a new commit in the Local Repository |
| **Branching** | `git switch -c <branch-name>` | Create a new branch and switch to it immediately |
| | `git switch main` | Switch back to the main branch |
| | `git merge <branch-name>` | Merge changes from a feature branch into the current branch |
| **Remote Sync** | `git fetch` / `git pull` | Download changes to inspect (`fetch`) or download and merge (`pull`) |
| | `git push` | Upload local commits to GitHub |
| **Undo & Revert** | `git restore <file-name>` | Discard uncommitted edits (add `--staged` to unstage files) |
| | `git revert <commit-hash>` | Safely undo a pushed commit by creating a new inverse commit |
