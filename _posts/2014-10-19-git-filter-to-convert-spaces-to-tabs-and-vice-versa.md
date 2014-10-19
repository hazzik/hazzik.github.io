---
layout: post
title: "Git Filter to Convert Spaces to Tabs and Vice Versa"
date: 2014-10-19T11:32:29+13:00
tags:
  - Git
---
Usually people have different preferences on tabs vs spaces. Here is a git filter to convert tabs to spaces, and spaces to tabs. Git allows you to write filters - a small commands which will be run after checkout (smudge) or before add (clean). As usual all settings are stored in git config files.

To make a work I use two GNU tools [expand](http://man7.org/linux/man-pages/man1/expand.1.html) (converts tabs to spaces) and [unexpand](http://man7.org/linux/man-pages/man1/unexpand.1.html) (converts spaces to tabs). If you are using Windows, then you'll need these tools to be reachable from your PATH environment varible. You can get this tools as part of [GnuWin32](http://gnuwin32.sourceforge.net/) or any other port of GNU tools to Windows.

DISCLAMER: Be really careful before commiting, as this filters can produce a huge changesets.

## Tabify - spaces to tabs on add
1. Open your ~/.gitconfig (%SystemDrive%\\Users\\<username>\\.gitconfig in case of Windows)
2. Add following lines

	[filter "tabify"]
	clean = unexpand --tabs=4 --first-only

It says that we want to convert each 4 space characters at the beginning of a line to one tab.

3. To make the filter active add following lines to your .gitconfig (or to .git\info\attributes)

	*.cs filter=tabify

It says that for files with ".cs" extension we want to use this filter.

## Spacify - spaces to tabs on add
1. Open your ~/.gitconfig (%SystemDrive%\Users\<username>\.gitconfig in case of Windows)
2. Add following lines

	[filter "spacify"]
	clean = expand --tabs=4 --initial

It says that we want to convert each tab character at the beginning of line to 4 space characters.

3. To make the filter active add following lines to your .gitconfig (or to .git\info\attributes)

	*.cs filter=spacify

It says that for files with ".cs" extension we want to use this filter.

## Spaces2Tabs - spaces to tabs on add, and tabs to spaces on checkout.
This filter is just a combination of previous two filters. This filter is usefull if your repository requires you to use tabs for indents, but you prefer spaces. The code

	[filter "spaces2tabs"]
		clean = unexpand --tabs=4 --first-only
		smudge = expand --tabs=4 --initial

## Tabs2Spaces - tabs to spaces on add, and spaces to tabs on checkout.
This filter is just a combination of first two filters. This filter is usefull if your repository requires you to use spaces for indents, but you prefer tabs. The code

	[filter "tabs2spaces"]
		clean = expand --tabs=4 --initial
		smudge = unexpand --tabs=4 --first-only

Happy filtering!






