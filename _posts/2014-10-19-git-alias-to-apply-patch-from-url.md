---
layout: post
title: "Git aliases to apply patches from url"
date: 2014-10-19T02:18:53+13:00
tags:
  - Git
  - Git Alias
---
Git has two commands to apply patches: [_git am_](https://www.kernel.org/pub/software/scm/git/docs/git-am.html) and [_git apply_](https://www.kernel.org/pub/software/scm/git/docs/git-apply.html), but none of them are able to apply patches from an url. So here are my aliases to do so.

###git apply-url - Apply a patch to files and/or to the index from url

	$git config --global alias.apply-url "!f() { curl -s $1 2>nul | git apply ${@:2}; }; f"

The alias itself is pretty simple. First it declares callable function `f` and then calls it. The function itself calls `curl` in silent mode with first argument passed to alias as url. Then it pipes `curl`s output to `git apply` with remaing arguments. This alias applies the patch but does not create a commit. 

Usage

	$git apply-url http://example.org/sample.patch args

The first argument MUST be an url for a patch, other arguments will be passed to `git apply`. 

###git apply-url - Apply a series of patches from an url (git am-url):

	$git config --global alias.am-url "!f() { curl -s $1 2>nul | git am ${@:2}; }; f"

This alias works exactly the same as previous.

Usage

	$git am-url http://example.org/sample.patch args

The first argument MUST be an url for a patch, other arguments will be passed to `git am`.
You can apply a pull request from github by performing following:

	$git am-url https://github.com/user/repo/pull/777.patch

Happy aliasing!