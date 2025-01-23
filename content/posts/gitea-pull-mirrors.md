+++
title = "Updating Gitea Pull Mirrors in Bulk"
date = "2025-01-23"
+++

## Background

I use [mirror-to-gitea](https://github.com/jaedle/mirror-to-gitea) to backup my personal GitHub repositories onto a private [Forgejo](https://forgejo.org) instance.

mirror-to-gitea expects a GitHub access token in case you want to mirror private repositories.
I use a _fine-grained personal access token_ to only allow read access to my repositories.

This works very well for my use case.

## Problem

tl;dr: I'm lazy.

By default, GitHub PATs have an expiration date.
It is possible to disable the expiration, but depending on your risk aversion you might not want to do that.

A few days ago my token expired, which led to errors when Forgejo tried to update the mirrored repositories.
This meant that I had to update the mirroring configuration through the GUI in every single private repository on my Forgejo instance.

## Solution

Gitea/Forgejo store the auth token inside the `config` file of each Git repository.
This allows us to use a combination of CLI tools to update all of the tokens in bulk:

```bash
grep -rl 'YOUR_OLD_TOKEN' $FORGEJO_VOLUME | xargs sed -i 's/YOUR_OLD_TOKEN/YOUR_NEW_TOKEN/g'
```

## Tips

Use a leading space when copying the command into your terminal to prevent your token from leaking into your `.bash_history`.
This requires setting `$HISTCONTROL` to either `ignoredups` or `ignoreboth` (Bash).
