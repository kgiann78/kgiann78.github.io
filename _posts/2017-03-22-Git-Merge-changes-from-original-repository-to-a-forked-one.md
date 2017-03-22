---
layout: post
title:  "Git: Merge changes from original repository to a forked one"
date:   2017-03-22 11:00:01 +0200
categories: github
---

# Git Merge changes from original repository to a forked one

Like many I forked a repository from GitHub in order to work on it. After a while I needed to update my version with commits and updates from the original repository. In a brief search in the internet I came into the same question in the [stackoverflow]:<http://stackoverflow.com/questions/3903817/pull-new-updates-from-original-github-repository-into-forked-github-repository>.

As the answer declares:

Once the clone is complete your repo will have a remote named “origin” that points to my fork on GitHub. To keep track of the original repo I will need to add another remote named “upstream”:

    $ cd my-forked-repository-name
    $ git remote add upstream git://github.com/original-creator/original-repo-name.git
    $ git fetch upstream

    # then: (like "git pull" which is fetch + merge)
    $ git merge upstream/master master
    
    # or, better, replay your local work on top of the fetched branch
    # like a "git pull --rebase"
    $ git rebase upstream/master

![upstream]({{site.url}}/assets/LtFGa.png)
