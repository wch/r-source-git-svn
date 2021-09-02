R-source metadata for Git-SVN mirroring
=======================================

This repository uses GitHub Actions to do mirroring of R source code from the SVN repository at https://svn.r-project.org/R/, to the git repository at https://github.com/wch/r-source. (NOTE: it is currently mirroring to https://github.com/wch/rs for testing.)

It contains:

* A GitHub Actions workflow in `.github/workflows/`
* git-svn metadata in `/svn/`
* git-svn remote information in `refs/remotes/Rsvn`


## How it works

The process of mirroring the SVN repository involves two git repositories:

* r-source-git-svn (this one), which contains git-svn metadata
* r-source, which is a git repository which mirrors the SVN repository

A git repository which is set up to mirror a SVN repository using `git-svn` contains a directory `/.git/svn`, which has metadata about the mappings between SVN and git commits.

The `/.git/svn` directory does _not_ get pushed to remote git repositories. Normally that's not a problem because the repository persists on disk. However, with GitHub Actions, there is not persistent storage, so we need to save the `/.git/svn/` metadata some other way. We do it here by storing it in `/svn/`, and then symlinking it to `r-source/.git/svn/` every time the workflow runs.

The strategy used by GitHub Actions workflow is:

* Clone this r-source-git-svn repository.
* Clone the r-source repository, into `r-source-git-svn/r-source/`.
* Symlink `r-source-git-svn/r-source/.git/svn` to point to `r-source-git-svn/svn/`.
* Symlink the `r-source-git-svn/r-source/.git/refs/remotes/Rsvn` dir to `r-source-git-svn/refs/remotes/Rsvn/`.
* Do the `git svn` steps for mirroring SVN commits to the local `r-source-git-svn/r-source/` repo. This will also update contents in the `.git/svn/` directory.
* Commit the changes to `r-source-git-svn/svn/` (the files here have changed because `r-source-git-svn/r-source/.git/svn/` is a symlink to it).
* Push the changes to the r-source repository.
* Push the changes to the r-source-git-svn repository.


### Mirroring manually

The same process can be done manually, with the commands below.

```bash
git clone https://github.com/wch/r-source-git-svn.git
cd r-source-git-svn

git clone https://github.com/wch/r-source.git
cd r-source/.git
ln -s ../../svn

cat <<EOF >> config
[svn-remote "svn"]
    url = https://svn.r-project.org/R
    fetch = trunk:refs/remotes/Rsvn/trunk
    branches = branches/*:refs/remotes/Rsvn/*
    tags = tags/*:refs/remotes/Rsvn/tags/*
EOF

cd refs/remotes
ln -s ../../../../refs/remotes/Rsvn

cd ../../..

git checkout trunk
git svn fetch
git svn rebase

# Push all branches from Rsvn to origin (r-source)
git push origin refs/remotes/Rsvn/*:refs/heads/*

# Commit changes to SVN metadata and push to r-source-git-svn
cd ..
git add svn refs
git commit -m"Update from SVN"
git push
```

## Problems and how to fix (some of) them

#### What happens if the two repositories get out of sync?

The two repositories could get out of sync if a push to one of them succeeds, but a push to the other fails.

**If the r-source repository gets ahead of the r-source-git-svn repository:** that is not a problem (I think), because the next time `git svn fetch` and `git svn rebase` are run, they correctly match up the commits.

**If the reverse happens, with r-source-git-svn getting ahead of r-source:** this can be a problem, because the refs in `refs/remotes/Rsvn` will point to git commits which do not exist. Here's a sample of the output you might see if that happens:

```
$ git svn fetch
error: refs/remotes/Rsvn/trunk does not point to a valid object!

$ git svn rebase
error: refs/remotes/Rsvn/trunk does not point to a valid object!
fatal: invalid upstream 'refs/remotes/Rsvn/trunk'
rebase refs/remotes/Rsvn/trunk: command returned error: 128
```

If this happens, the way to fix it is to go back in time, and check out commits in each repository at a point when they were in sync. Then you may need to do a `git checkout -B trunk` (or whichever branch) to point it to the selected (old) commit. After doing that, the `git svn fetch` and `rebase` should work.


#### What's the deal with the files in `r-source/.git/refs/remotes/Rsvn` disappearing?

A `git gc` in the r-source repo will remove the remote branches in `.git/refs/remotes/Rsvn`. (This directory is a symlink to the `r-source-git-svn/refs/remotes/Rsvn`.) The GitHub Actions workflow checks that these files are present; if not, it errors out before trying to commit and push changes.
