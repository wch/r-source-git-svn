R-source metadata for Git-SVN mirroring
=======================================

This repository uses GitHub Actions to do mirroring of R source code from the SVN repository at https://svn.r-project.org/R/, to the git repository at https://github.com/wch/r-source. (NOTE: it is currently mirroring to https://github.com/wch/rs for testing.)

It contains:

* A GitHub Actions workflow in `.github/workflows/`
* git-svn metadata in `/svn/`
* git-svn remote information in `refs/remotes/Rsvn`


## How it works

A git repository that is set up to mirror a SVN repository using git-svn contains a directory `/.git/svn`, which has metadata about the mappings between SVN and git commits.

The `/.git/svn` directory does _not_ get pushed to remote git repositories, but normally that's not a problem because the repository persists on disk. However, with GitHub Actions, there isn't persistent storage, so we need to save the `/.git/svn/` metadata some other way. We do it here by storing it in `/svn/`, and then symlinking it to `r-source/.git/svn/` every time the workflow runs.

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

# Push newly mirrored commits to r-source
git push

# Commit changes to SVN metadata and push to r-source-git-svn
cd ..
git add svn refs
git commit -m"Update from SVN"
git push
```
