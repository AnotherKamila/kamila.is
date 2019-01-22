---
title: Shell Tutorial for Absolute Beginners
add_to_head: <link rel="stylesheet" type="text/css" href="asciinema-player.css" />
---

Almost all software you use has a graphical user interface, where you make things happen in the software mostly by clicking/tapping on things. But there's another type of user interface, called the "shell" or "command line", where you make things happen by telling the computer what to do by typing specific words (they're called "commands").

The command line is a really powerful tool that allows you to do much more than any graphical interface would. But it can also be scary and unfamiliar! So this tutorial aims to make you comfortable in the shell, assuming no previous knowledge."

# Preparation: What you need to follow this tutorial

This tutorial is full of "videos" (not really videos) of what it looks like to do the things I am talking about. You can pause and replay everything, and you can copy things from the window (it's just text). Click this "video" to play it:

<asciinema-player src="./cast/asciinema-demo.cast" speed="2" theme="solarized-dark" size="medium" idle-time-limit="0.2">If you see this message instead of the "video", something went wrong. Sorry about that. Try reloading the page.</asciinema-player>

This tutorial uses [fish](https://fishshell.com/) (the "friendly interactive shell"), which you need to install. If your system has a graphical package manager, you can use that. Otherwise, see [their website](https://fishshell.com/#platform_tabs) for installation instructions. For example, on FreeBSD, you can install it by typing the command `pkg install fish`. [Ask me](https://github.com/AnotherKamila/shell-for-beginners/issues/new) if you need help.

<!-- TODO Installation instructions that don't require the command line would be much preferred :D -->

After you have installed it, you can run it inside your "normal" shell by typing `fish`.

This is what the process looks like on FreeBSD:

<asciinema-player src="cast/install-fish.cast">
If you see this, reload the page.
</asciinema-player>

If you have `fish` running, we can dive right in!

# Moving around the filesystem

## Basics

Similarly to other operating systems, files on Unix-like systems are arranged in directories (folders).

All commands are run from a directory, and your shell knows which directory you are in. Many commands will work in the current directory unless you tell them otherwise.

You can use the`ls` command to list the contents of a directory, and the `cd` (change directory) command to go to another directory:

<asciinema-player src="cast/cd-ls.cast">If you see this, reload the page.</asciinema-player>

The *shortened* path to the current directory is displayed in the command prompt: if you see e.g. `kamila@mymachine ~/M/Album2>`, you know you are in the directory `Album2`, which is inside something that begins with an M (in this case `Music`). The `~` stands for your home directory. (The first part of the prompt is your username and hostname.)

Unlike on Windows, all files are arranged in a single file tree (not in separate trees for `C:\`, `D:\`, etc.).
The top of the file tree is called `/`. (This is like `C:\` on Windows, except there is no `D:\`.) The character used to separate directories in paths is a `/` (Windows uses `\` for the same thing).

You can specify files and directories either relative to the current directory, like above, or with an absolute (i.e. full) path -- starting at the top, with a `/`.

The command `pwd` **p**rints **w**orking (i.e. current) **d**irectory.

<asciinema-player src="cast/rel-and-abs-paths.cast">If you see this, reload the page.</asciinema-player>

## Creating and modifying files

Start by going to `/tmp` (directory for temporary files), because that's where you can't do much damage :-)

Some useful commands are:

* `mkdir`: make directory
* `echo`: just prints everything you give it, which sounds useless, but...
* `>` is a special character that tells the shell to write things into files instead of on the screen
* so `echo hello world > hello.txt` will write "hello world" into a file called `hello.txt`
* `cat` prints file contents into the terminal

<asciinema-player src="cast/creating.cast">If you see this, reload the page.</asciinema-player>

### Bonus: Some more useful file-related commands

* `less` lets you scroll in big files (quit with `q`; press `h` for help):
* `tree` prints the file tree (you may need to install it first)
* `rmdir` removes an empty directory
* `cp -a` copies a directory and all its contents.
* `rm` removes a file.
* `rmdir` removes an empty directory. It will refuse to remove a directory if it isn't empty, to prevent accidentally deleting things.
* `ls -l` shows more info about the files, like when it was changed or file size; add `-h` to show human-readable file sizes

Try them!

# Awesome fish sauce

## Command Suggestions

Notice that Fish suggests things you might want to type, and the predictions get better over time. Press the right arrow to accept the suggestion. This does not immediately execute the command, you can still edit it, so do not be afraid to press right arrow a lot :-)

Also: Notice that Fish colors what you type to help you easily see what you're doing. For example, text highlighted red is an error (non-existent command, unclosed quotes, or something like that); underlined text is an existing file; and similar. This helps immediately see things like typos, and over time your subconsciousness will magically correct such things for you (speaking from experience here).

## Tab completion

Note: This is difficult to explain when you don't see which exact keys I am pressing. Go try it yourself!

You can press the Tab key to autocomplete commands or files at any time. If there is a unique completion, fish will complete it; if not, it will list the options. If you press Tab multiple times, it will cycle through possible options; press Enter to accept a completion. 

<asciinema-player src="cast/tab-completion-1.cast">If you see this, reload the page.</asciinema-player>

You can search among the possible completions (after pressing Tab at least twice) by simply starting to type when the options are displayed; type Enter to accept a completion. A combination of typing to filter and tabbing to cycle through options can get you to what you want very quickly with a bit of practice. Don't forget to accept the completion when you are happy by pressing Enter.

<asciinema-player src="cast/tab-completion-search.cast">If you see this, reload the page.</asciinema-player>

Fish is smart and will try to tab complete things even if they are not an exact match. This is useful e.g. when you make a typo or when you don't remember the exact file name. Play with it!

### Tab completion is magic

As mentioned, Tab can complete all sorts of things. Importantly, it can complete commands; and it provides short descriptions of what each command does. This is great if you can't remember exactly which command you are looking for:
<asciinema-player src="cast/tab-completion-commands.cast">If you see this, reload the page.</asciinema-player>

Some important commands are so important that Fish knows more about them; in that case, it can complete the arguments too:
<asciinema-player src="cast/tab-completion-commands-args.cast">If you see this, reload the page.</asciinema-player>

## Using previously typed commands

When doing things in the shell, you often need to run the same or similar command several times; or you do several things with the same file; or similar. Fish lets you save time in such cases by letting you search the command history by pressing the up arrow.

If you press Up without typing anything, it will go through the previous commands. If you type something and then press Up, it will go through only those commands which contain the characters you had typed. You can also press Down to go to more recent commands, of course.

<asciinema-player src="cast/history.cast">If you see this, reload the page.</asciinema-player>

If you want to go through the previous *words* you had typed, instead of whole commands, use Alt+Up instead. This also works with the searching: typing a few letters and then pressing Alt+Up will go through only matching words.

<asciinema-player src="cast/history-alt-1.cast">If you see this, reload the page.</asciinema-player>

You will often do several things with the same file: for this, type your command and then press Alt+Up once to find the previous file name again:

<asciinema-player src="cast/history-alt-2.cast">If you see this, reload the page.</asciinema-player>

<!--
# TODO things

 You can also set it as your default shell for your user (but NOT for root) with `chsh -s fish` (on less smart systems, you may need to tell it something like `/usr/bin/fish`).
 -->

# Suggestions?

This document lives [on GitHub](https://github.com/AnotherKamila/shell-for-beginners/). Comments, suggestions, improvements, or anything else is welcome! Feel free to open an issue or send a pull request.

# Credits

* [asciinema](https://asciinema.org/) is used for the "videos"

<!-- the stuff below is needed to configure and enable asciinema -->
<script>
var ds = document.getElementsByTagName('asciinema-player');
for (var i = 0; i < ds.length; ++i) {
  var d = ds[i];
  d.setAttribute('theme', 'solarized-dark');
  d.setAttribute('font-size', '1em');
  d.setAttribute('speed', 1.7);
  //d.setAttribute('idle-time-limit', 0.2);
}

// doing it this way because of the very annoying interaction with MDL on https://kamila.is
window.addEventListener('load', function() {
setTimeout(function() {
    var script_tag = document.createElement('script');
    script_tag.setAttribute('src','./asciinema-player.js');
    document.head.appendChild(script_tag);
  }, 500);
});
</script>
