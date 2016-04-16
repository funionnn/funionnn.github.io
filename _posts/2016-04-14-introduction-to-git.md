---
layout: post
title:  "Introduction to Git"
date:   2016-04-16
comments: on
---
{% include header-image image='/assets/git-intro.jpg' %}

# Introduction to Git

This post is an adaptation of an introduction to Git that I put together for [Flawless Hacks](flawlesshacks.com). Check out the [companion repository](github.com/funionnn/flawless)!

## Git vs GitHub

Before we dive in too deep, let's figure out what we're talking about. Git and GitHub get used interchangeably, but these are different things. 

Git is a version control system. We'll be talking about features later like branching and merging - these are features of Git. Version control allows you to safely work on a repository alongside other people without overwriting eachothers' work. Additionally, Git is great at keeping track of history. This allows you to roll back to previous versions of your app in case anything goes wrong. It also lets you gain insight into the history of the code by reading old commit messages.

GitHub is a company that hosts Git repositories, and provides a web interface for many aspects of managing them. Other companies offer similar services, like Atlassian's BitBucket. 

## Getting set up

### Desktop app

There's a lot of reasons why you should use Git via the command line, but for beginners, it can be difficult to use. The desktop application is also the fastest way to get set up and start hacking, and is available for both OSX and Windows, so that's what we'll do today.

First, sign up for a [GitHub account](https://github.com).

Second, grab the [GitHub desktop client](https://desktop.github.com). 

Follow through the setup directions in the desktop app, and you'll be ready in no time.

### Command line - OS X

To get set up on the command line with OS X, you'll first need to install the OS X Command Line Tools. If you already have Xcode installed, you can skip this step. Otherwise, head over to [https://developer.apple.com/downloads](https://developer.apple.com/downloads) to download and install Command Line Tools.

OS X comes with Git installed, but you should check your version to ensure that it's up to date. Enter `git --version` into your terminal and compare it to the latest version [here](https://git-scm.com/download/mac). If you're behind versions, download and install the latest one.

In the terminal, let Git know who you are:
```
$ git config --global user.name "YOUR NAME"
$ git config --global user.email "YOUR EMAIL ADDRESS"
```

You can store your password so that you don't need to enter it every time you interact with a remote repository. Get the directions [here](https://help.github.com/articles/caching-your-github-password-in-git/).

## Steps for using source control

* Create, fork, or clone a repository
* Make a feature branch
* Make and save your changes on your computer
* Commit your changes
* Push your branch to the remote
* Open a pull request
* Code review
* Merge and delete your branch

### Create, fork, or clone a repository

There's a few different ways that you can get a repository to work with. 

If you're starting fresh, create a new repository. GitHub lets you create public repositories (visible to the world) for free. 

If you'd like to create a private repository without paying a monthly fee, check out [Bitbucket](https://bitbucket.org/).

If you'd like to make your own copy of an existing repository, you can *fork* it. Forking a repository typically means that you won't be merging your changes back into the original repository. I set up a demo Rails repository with Devise and Heroku deployment ready to go - [feel free to fork it  here](github.com/funionnn/flawless) to get a head-start on your project!

Once you have a repository to work with, whether it's a new one you created, one that you forked, or an existing one that you'd like to collaborate on with teammates, you'll need to *clone* the repository to your computer. If you have the desktop app installed, there's a clone button in the GitHub web interface that will do this. 

To clone a repository via the command line: 
```
git clone git@github.com:funionnn/flawless.git 
```
This will clone the repository into a directory with the same name as the repository within the directory that you're currently in. To clone it to a folder with a specific name: 

```
git@github.com:funionnn/flawless.git flawless-app
```

### Make a feature branch

Before you make any changes, you need to create a *feature branch*. Branching is a big part of the Git workflow - it's the first step you'll take every time you start working on something new in your codebase. 

First, make sure that you're branching from the latest version of Master. You want to make sure that your computer knows locally about all the latest changes. In the command line:

```
git pull origin master
```

A feature branch is how we can safely work in isolation, and then later on merge our changes into the master branch without worrying about overwriting other people's work. It will keep track of the state that the master branch was in when we branched off. This will also let Git identify what happened in the meantime when we go to merge our changes, so that it can tell us whether it's safe to merge or not. 

Once you've created a branch on your computer, you can start working!

### Make and save your changes on your computer, committing them as you go

A commit is the act of telling Git to record your changes. You'll give your commit a commit message, describing what your changes do. You can make as few or as many commits as you like on your branch. It's good practice to break your commits down into small bundles of related changes. That will allow people reading through the Git history later on to understand what your changes were intended to do, and get better context into the codebase. To do this through the command line:

```
git add .
git commit -m "Your commit message"
```

### Push your branch to the remote

Making a commit on your computer doesn't send those changes to the internet - you need to push your branch to get those changes onto GitHub. To do this, you'll push your branch to a remote branch. You can do this by clicking "Publish" in the desktop app, or through the command line: 

```
git push -u origin your-branch-name
```

Or if you've already pushed the branch before, and you want to add on more changes:

```
git push origin your-branch-name
```

### Open a pull request

Once your branch has been pushed up to GitHub, you can open a pull request. This is a request to merge the changes you've made into the master branch. The pull request is where you and your collaborators can discuss the changes and do code review. 

### Merge and delete your branch

If GitHub gives you the green checkmark go-ahead, you're safe to merge your changes! If you get a merge conflict, [check out this guide](https://help.github.com/articles/resolving-a-merge-conflict-from-the-command-line/) on how to resolve it.

### Done!

That's it! Rinse and repeat. 

Git has a lot of power and complexity. This guide is meant to get you up and running as quickly as possible, but there's a lot more to learn. Check out the freely available [Git Pro book by Scott Chacon](https://git-scm.com/book) to learn about any of these topics.