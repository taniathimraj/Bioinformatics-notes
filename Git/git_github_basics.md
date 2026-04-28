# Git & GitHub Basics

## What is Git?

Git is a version control system that tracks changes to your files over time. Every time you make a meaningful change, you take a snapshot (called a commit). You can always go back to any previous snapshot.

**GitHub** is the cloud backup of those snapshots — it stores your full project history online.

---

## First-Time Setup

- Check if Git is already configured :
  - `git config --global user.name`
  - `git config --global user.email`
- Configure your identity (only needed once) :
  - `git config --global user.name "Your Name"`
  - `git config --global user.email "your@email.com"`

---

## Starting a New Project

- Create and initialize a project folder :
  - `mkdir project_name`
  - `cd project_name`
  - `git init`
- Create subfolders for organisation :
  - `mkdir data results figures notebooks scripts`
- Copy existing files into the project :
  - `cp -r /path/to/data ./data`
  - `cp -r /path/to/figures ./figures`
  - `cp -r /path/to/script.ipynb ./notebooks`
- Verify the transfer :
  - `ls data/`
  - `ls figures/`
  - `ls notebooks/`

---

## The .gitignore File

A `.gitignore` file tells Git which files to never track — large data files, temporary files, etc.

- Create a .gitignore : `touch .gitignore`
- Example entries for bioinformatics projects :
  - `*.fastq` `*.fastq.gz` `*.bam` `*.bai` `*.rds` `*.h5` `*.h5ad`
  - `.DS_Store` `*.log` `.Rhistory` `.RData` `.Rproj.user/`
- Always add large data files to .gitignore — GitHub has a 100MB file size limit

---

## Daily Workflow — Stage, Commit, Push

- Check the status of your files : `git status`
- Stage a specific file : `git add filename.qmd`
- Stage all changed files : `git add .`
- Commit (take the snapshot) : `git commit -m "Brief description of what you changed"`
  - Good commit messages: `"Add QC filtering step"`, `"Fix mito threshold"`, `"Add UMAP plot"`
- Push to GitHub : `git push`

---

## Connecting to GitHub

- Create a new repo on GitHub :
  - Go to https://github.com
  - Click New repository
  - Give it a name and set visibility (public/private)
  - Do NOT check any boxes (no README, no .gitignore)
  - Click Create repository
- Connect your local repo to GitHub :
  - `git remote add origin https://github.com/USERNAME/REPO_NAME.git`
  - `git branch -M main`
  - `git push -u origin main`

---

## Checking History

- See all your commits : `git log`
- See a compact version : `git log --oneline`
- See what changed in a specific commit : `git show COMMIT_ID`

---

## Making Changes to Existing Files

- Rename a file and track it with Git :
  - `git mv old_filename.ipynb new_filename.ipynb`
  - `git commit -m "Rename analysis notebook"`
  - `git push`
- Add a new file :
  - `git add new_script.qmd`
  - `git commit -m "Add new QC script"`
  - `git push`
- Delete a file and track the deletion :
  - `git rm filename.qmd`
  - `git commit -m "Remove outdated script"`
  - `git push`
- Recover an acidentaally deleted file
  - `git log --oneline`          
  - `git checkout COMMIT_ID -- ATAC-seq/ATAC_seq_basics.md`
  - `git commit -m "Restore ATAC-seq file"`
  - `git push`
---

## Undoing Changes

- Undo changes to a file before staging (go back to last commit) : `git checkout -- filename.qmd`
- Unstage a file (remove from staging but keep changes) : `git reset HEAD filename.qmd`
- Undo the last commit (keep the changes, just undo the commit) : `git reset --soft HEAD~1`
- Revert a commit (create a new commit that undoes it — safer) : `git revert COMMIT_ID`

---

## Troubleshooting

- Push is rejected :
  - `git pull origin main`
  - `git push`
- Check what remote your repo is connected to : `git remote -v`
- See all branches : `git branch`
