---
title: One Git, Many Hats = How I Manage Multiple Git Identities
description: How I configure my git identities for work and personal projects
author: James Ingold
published: true
---

Manually managing Git identities for multiple projects can be tedious. I would know, over the past couple of years, I’ve been using bits and pieces of this guide, but I recently automated my workflow entirely. My goal is to configure Git identities automatically based on a repository's location while keeping project repositories organized in separate directories.

For instance, if I have ~/Repos/work and ~/Repos/personal, each with its own Git identity, I want commits within those directories to automatically use the appropriate Git identity. This can be achieved using Git's gitconfig and SSH configuration features.

Here’s a step-by-step guide to managing multiple Git identities effectively.

#### Global Git Configuration

Start by creating a base Git configuration file as your default. This will be saved as `$HOME/.gitconfig`.

```
[user]
  name = Your Name
  email = your.email@example.com
  signingKey = ~/.ssh/mykey.pub
```

This establishes your default identity for all repositories.

#### Project-Specific Git Config

To override the global identity for specific projects, create a project-specific Git configuration file at the root of your project folder. For example, if all your work projects are under `$HOME/Repos/work`, create a .gitconfig file at `$HOME/Repos/work/.gitconfig`.

```
[user]
    name = Work Name
    email = work@example.com
    signingKey = ~/.ssh/work.pub
```

#### Automate Git Identity Selection

To automate identity selection based on directory paths, use Git’s conditional includes. Edit your global Git configuration file `($HOME/.gitconfig)` and add the following:

```
[includeIf "gitdir:~/Repos/work/"]
    path = ~/Repos/work/.gitconfig
```

With this setup, any repository within the `$HOME/Repos/work/` directory will automatically
use your work git identity.

### SSH Configuration

There are a couple different ways to handle SSH configuration. One way is to use
the `sshComand` in gitconfig to tell Git use a specific ssh command for all Git
operations in the repository. We can tell Git to always use our work ssh private
key when interacting with repositories in the ~/Repos/work directory by adding
an sshCommand to our work .gitconfig file.

```
[core]
    sshCommand = "ssh -i ~/.ssh/work"
```

The option I prefer is to manage all SSH configuration in my `$HOME/.ssh/config` file. If
both work and personal projects use github, we will have a problem that the
right SSH key is not being used for work projects. This is because both SSH
entries use the same hostname. Here's an example of how an SSH config file would
be set up:

```
Host github
 AddKeysToAgent yes
 Hostname github.com
 User git
 IdentityFile ~/.ssh/github

Host github-work
 AddKeysToAgent yes
 Hostname github.com
 User git
 IdentityFile ~/.ssh/work
```

To fix this, we will use the url mapping feature in our gitconfig. This rewrites specific repository URLs automatically. When we are in a work
repository, git will automatically rewrite the URL to use github-work and use the correct SSH config.

#### Update Project Specific with URL Overwrite

```
[url "git@github-work:"]
    insteadOf = git@github.com:
```

#### Full Project-Specific Root .gitconfig

```
[user]
    name = Work Name
    email = work@example.com
    signingKey = ~/.ssh/work.pub

[url "git@github-work:"]
    insteadOf = git@github.com:
```

With this setup, you can manage Git identities for different
projects. Git will default to your global configuration, but will automatically
switch to your project specific configuration when you are in a repository that
matches the conditional include.
