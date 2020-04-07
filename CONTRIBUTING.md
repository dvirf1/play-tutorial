# Contributing

## Thank you for contributing to the play-framework tutorial!

## How to contribute

We use a simple [fork and pull](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests#fork--pull) approach.  

Follow these steps to contribute:

- Open an issue to let us know what is wrong and that you are working on it
- Fork the repo by clicking the `Fork` button at the top of the [repository page](https://github.com/dvirf1/play-tutorial)
- Clone the fork of the repository:
  ```bash
  git clone git@github.com:<your_user>/play-tutorial.git && cd play-tutorial
  # or
  # git clone https://github.com/<your_user>/play-tutorial.git && cd play-tutorial
  ```
- Optional: If you have global git hooks or you want to change user for this repo:
  ```bash
  git config core.hooksPath '' # disables global hooks
  git config user.email "desired@user.com" # set another identity than the one in your gitconfig file https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup
  ```
- Create and checkout a new branch. Give it a name that describes what you are working on:
  ```bash
  git checkout -b my-new-feature
  ```
- Make the changes you need. The post are at `_posts` dir.
  To set up the project for the first time, you need ruby. See installation instructions in the [README](README.md).  
  You can use `bundle exec jekyll serve` to serve the tutorial at http://localhost:4000/play-tutorial/
- Commit and push. If your change fixes an existing issue (e.g issue #42), add it in the commit message or pull request:
  ```bash
  git add .
  git commit -m "meaningful commit message. fixes #42"
  git push -u origin my-new-feature # change `my-new-feature` to the name of your branch
  ```
- Submit a new pull request via the github page of your fork

Thank you!

### Syncing your fork

You can sync your fork with updates from the [upstream](https://github.com/dvirf1/play-tutorial) repository.  
You should first [configure a remote for your fork](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/configuring-a-remote-for-a-fork) (one-time setup):

1. Open Terminal.

2. List the current configured remote repository for your fork.

```bash
$ git remote -v
origin  git@github.com:YOUR_USERNAME/play-tutorial.git (fetch)
origin  git@github.com:YOUR_USERNAME/play-tutorial.git (push)
```

3. Specify a new remote upstream repository that will be synced with the fork.

```bash
git remote add upstream git@github.com:dvirf1/play-tutorial
# or
# git remote add upstream https://github.com/dvirf1/play-tutorial.git
```

4. Verify the new upstream repository you've specified for your fork.

```bash
$ git remote -v
origin    git@github.com:YOUR_USERNAME/play-tutorial.git (fetch)
origin    git@github.com:YOUR_USERNAME/play-tutorial.git (push)
upstream  git@github.com:dvirf1/play-tutorial (fetch)
upstream  git@github.com:dvirf1/play-tutorial (push)
```

Now you can [sync your fork](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/syncing-a-fork) to pull new changes from the upstream remote:

1. Open Terminal.

2. Change the current working directory to your local project.

3. Fetch the branches and their respective commits from the upstream repository.

```bash
git fetch upstream
```

4. Check out your fork's local master branch.

```bash
$ git checkout master
> Switched to branch 'master'
```

5. Merge the changes from upstream/master into your local master branch.
This brings your fork's master branch into sync with the upstream repository, without losing your local changes.  
If your local branch didn't have any unique commits, Git will instead perform a "fast-forward".

```bash
$ git merge upstream/master
> Updating aabb123..aabb456
> Fast-forward
>  README.md                 |    5 +++--
>  1 file changed, 3 insertions(+), 2 deletions(-)
```