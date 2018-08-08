---
layout: post
title: Learn some bash scripting by adding current git branch name to bash prompt
---

Maybe you saw something like this:
{% highlight bash %}
 04:13 PM:~/www/project (dev) $ 
{% endhighlight %}
The terminal is telling what is the current git branch name of repository in current directory. 
And maybe you are wondering how it works. Let's implement this functionality!

PS1 environment variable
========================
The text that is displayed at the beginning of your command line input is called "primary prompt string"
and it is controlled by `PS1` environment variable.
Just type `echo $PS1` in your terminal and it will output something like `\h:\W \u\$`. You may say WTF?
Take it easy! Manual explains all this stuff (`man bash` PROMPTING section).

| Character | Meaning                                 |
|-----------|-----------------------------------------|
| \d        | Date in "Weekday Month Date" format     |
| \h        | The hostname up to the first point      |
| \H        | Host name                               |
| \n        | New line                                |
| \s        | Name of the shell                       |
| \t        | Current time in 24-hour HH:MM:SS format |
| \u        | Username of the current user            |
| \w        | Current working directory               |

...


Append text to PS1 environment variable
===========================
So we should change the `PS1` environment variable to also display current git branch name.
Let's create a function that will output current git branch name and append the function output to `PS1` variable.
For that we should know that:
- `(command)` construct is used to execute the command inside the brackets.
- `$(command)`construct is used to execute the command inside the brackets AND return the output of that command.
- `$VAR` is used to refer to `VAR` variable value inside the string.


Open `~/.bashrc` file (this file is loaded when you start new [non-login shell](https://www.unixmen.com/non-login-shell-login-shell/)) and add next code to it:

{% highlight bash %}
function current_git_branch() {
  echo "(dev)"
  return
}

PS1="$PS1 \$(current_git_branch) "
{% endhighlight %}

Now we have a stub `current_git_branch` function and result of this function is appended to
`PS1` environment variable.

Get the real current git branch name
============================
For that we should know that:
 - `|` is a pipe operator and it's used to direct the output of command placed before pipe operator to
 input of the command placed after pipe operator.
 The small example of `|` operator:
     - `ls -l` command outputs filenames in current directory, each filename on new line
     - `grep "^d"` filters each line of input by regular expression, in our case `"^d"` means that line should start with a 'd' symbol
  which means that file should be a directory.
  And by doing  
  `ls -l | grep ^d`  we will filter the output of `ls` command and 
  get the list of subdirectories in current directory.
 - `colrm from to` is a command for removing characters from input 
 (e.g `colrm  2 4` will output `Ho world` for `Hello world` input)
 

 

Let's continue. We all know `git branch` command. But the output is not acceptable for us.

{% highlight bash %}
 04:13 PM:~/www/project $ git branch
  CVW-3
  DEFAULT-CONTENT
  UVW-3
* dev
{% endhighlight %}

We should transform this output to just `(dev)`.

The first step: 
{% highlight bash %}
 04:13 PM:~/www/project $ git branch | grep '^*'
* dev
{% endhighlight %}

Also we should remove the star and the space after star (2 characters).

{% highlight bash %}
 04:13 PM:~/www/project $ git branch | grep '^*' | colrm 1 2
dev
{% endhighlight %}

And now we can assign the output of these piped commands to a variable

{% highlight bash %}
 04:13 PM:~/www/project $ CURRENT_GIT_BRANCH=$(git branch | grep '^*' | colrm 1 2)
* dev
{% endhighlight %}

Nitpick!
What if we execute this command from directory that is not a part of git repository?

{% highlight bash %}
 04:13 PM:~/www $ CURRENT_GIT_BRANCH=$(git branch | grep '^*' | colrm 1 2)
fatal: Not a git repository (or any of the parent directories): .git
{% endhighlight %}

We do not need this error in output. There is an easy way to fix that. We should redirect the
error output to another place. Error output redirection can be done by appending `2>file_for_errors_output.txt`
to the command. And there is a special file named `/dev/null`. This file discards all data written to it, 
you can think about it like about a trash can.

{% highlight bash %}
 04:13 PM:~/www/project $ CURRENT_GIT_BRANCH=$(git branch 2>/dev/null | grep '^*' | colrm 1 2)
{% endhighlight %}

Now we can change our `.bash_profile` file.

{% highlight bash %}
function current_git_branch() {
  CURRENT_GIT_BRANCH=$(git branch 2>/dev/null | grep '^*' | colrm 1 2)
  echo $CURRENT_GIT_BRANCH
  return
}

PS1="$PS1 \$(current_git_branch) "
{% endhighlight %}

IF statement 
============
But we also want to display round braces around current git branch name and only when our branch output is not empty.
We need some conditional statement for this functionality. Here is the syntax:

{% highlight bash %}
if [ condition ]
then
  commands
fi
{% endhighlight %}

`[ condition ]` is a shorthand for `test` command (crazy, yeah?) [Documentation](https://linux.die.net/man/1/test)

There are a lot of keys for testing values. We need the `-n` key which checks that "the length of STRING is nonzero".

{% highlight bash %}
if [ -n "$CURRENT_GIT_BRANCH" ] 
    then echo "($CURRENT_GIT_BRANCH)"; 
fi
{% endhighlight %}

Colors!
========================

We can also make the name of git branch yellow or green or whatever color we want (almost any).
To output text in color use the following syntax:
`echo -e "\e[x;ym YOUR_TEXT \e[m"` where

 - `-e` is a flag for echo command that turns on escape sequences output (colors are escape sequences)
 - `\e[` is used to begin the colors modifications
 - `x;y` is a color code to use
 - `\e[m` is used to reset the color to default one.


| Color	   | Code  |
|----------|-------|
| Black	   | 0;30  |
| Blue	   | 0;34  |
| Green	   | 0;32  |
| Cyan	   | 0;36  |
| Red	   | 0;31  |
| Purple   | 0;35  |
| Brown	   | 0;33  |
| Blue	   | 0;34  |
| Green	   | 0;32  |
| Cyan	   | 0;36  |
| Red	   | 0;31  |
| Purple   | 0;35  |
| Brown	   | 0;33  |
| Yellow   | 1;33  |
| Cyan	   | 1;36  |
| White	   | 1;37  |

[...](https://gist.github.com/chrisopedia/8754917)

So we can rewrite the function like this: 

{% highlight bash %}
function current_git_branch() {
  YELLOW_COLOR='\033[1;33m'
  NO_COLOR='\033[0m'
  CURRENT_GIT_BRANCH=$(git branch 2>/dev/null | grep '^*' | colrm 1 2)
  if [ -n "$CURRENT_GIT_BRANCH" ] 
    then echo -e "${YELLOW_COLOR}($CURRENT_GIT_BRANCH)${NO_COLOR}"
  fi
  return ;
}
{% endhighlight %}

Put functionality in separate file.
===============================
We can put all our custom code in separate file and then include it inside `~/.bashrc` file
with help of `source file` command (this command executes the content of passed file).

Here is the final code:

`~/.bashrc` :

{% highlight bash %}
 ...
 source '~/.bash_git_branch'
{% endhighlight %}

`~/.bash_git_branch`:

{% highlight bash %}
function current_git_branch() {
  YELLOW_COLOR='\033[1;33m'
  NO_COLOR='\033[0m'
  CURRENT_GIT_BRANCH=$(git branch 2>/dev/null | grep '^*' | colrm 1 2)
  if [ -n "$CURRENT_GIT_BRANCH" ] 
    then echo -e "${YELLOW_COLOR}($CURRENT_GIT_BRANCH)${NO_COLOR}"
  fi
  return
}

PS1="$PS1 \$(current_git_branch) "
export PS1
{% endhighlight %}

P.S
====
It's great that we can customize prompt string by ourselves but there are already existing solutions with much more
functionality. I use [this one](https://github.com/arialdomartini/oh-my-git) and very happy with it.
