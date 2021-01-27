---
layout:  post
title:   "Git Tricks"
summary: "Trying to free up some disk space in my brain by saving git tricks here."
date:    2021-01-26 14:10:00 -0800
categories: all
---

### Change a commit's author email

1. Take note of the commit number(s) for which you want to change the author's email.

2. Rebase interactively until one commit before the oldest one you want to modify.

```
git rebase -i <commit>
```

3. In the text editor, find the commit numbers you want to amend, and change the `pick` prefix for `edit`. Save the file and exit the editor.

4. Execute these two commands as many times as commits you want to amend:

```
git rebase --continue
git commit --amend --no-edit --author="New name <new@email.com>"
```

You'll know when to stop when you get a message like this:

```
Successfully rebased and updated refs/heads/RRS.
```

### Hide your email and use the GitHub one

GitHub allows you to specify a GitHub domain email for the author, to avoid showing your own personal or corporate email.

The address can be located in [https://github.com/settings/emails](https://github.com/settings/emails) and it has the format `<RandomNumber>+<YourAlias>@users.noreply.github.com`.

Copy that email address (optionally, you can omit the `<RandomNumber>`), then execute:

```
git config --global user.email "Your Name <YourAlias>@users.noreply.github.com"
```

In the same settings page, you can prevent showing your personal or corporate email if you check the option:

> Keep my email addresses private

You can also prevent accidentally pushing commits that contain your private email address if you check the option:

> Block command line pushes that expose my email