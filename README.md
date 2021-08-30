> bash commandline timestamping using a sqlite database for personal analytics, activity logging and auditing

## License

Copyright (c) by CS DVRX, 2020,2021 - a data consultant who prefers to remain pseudonymous :)

This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 3 of the License, or
(at your option) any later version.

My email address is these 5 letters followed by @outlook.com

## Example

![demo of the interactive features](https://raw.githubusercontent.com/csdvrx/bash-timestamping-sqlite/master/bash-timestamping.gif)

## What is it?

I have replaced my bash history by a sqlite based backend, fed by pure bash functions, and read using a mix of fzy (for the interactive UI) or shell-scripts (for the analytics) sometimes using gnuplot (for the graphs).

It also comes with eye-candy changes to your prompt that will show you the timestamp of the commands you type, along with the eventual error code they return.

## Why use it?

There are 2 typical use-cases:

 - If you work in an industry where keeping logs of what was done, and by who, matters a lot,

 - If you are into personal analytics, and want to focus your improvements on the pain points.

Fortunately, I'm into both :) Also, I believe [you need benchmarks](https://danluu.com/why-benchmark/) to understand, validate and improve, even if I may not be [as obsessive about that as some people](https://www.technologyreview.com/2013/05/08/178528/stephen-wolfram-on-personal-analytics/).

Even if you don't work in a sensitive field, when was the last time you scrolled in your shell history and couldn't remember when you had started this command, or how long did that long batch process take? The prompt changes solve these issues, by putting forward such informations.

My approach goes further by saving it all into a database: that gives you the power of personal analytics. It is quite unique, especially if you spend most of your time using command-line tools: you will get far more actionable data!

## What to do with more data?

Let's start with a simple example: I rarely schedule important work in the beginning of the day, or in the early afternoon. This is because, in my experience, I am the most creative at these times: I will try various new approaches, some of which may work and yield large gains, while most will fail.

I could easily confirm part of my hunch with my commit times, but I was lacking data on the failures. Then I started working on this project: while initially I was planning to solve a much more mundane issue, I quickly realized what collecting all this data could do, and started doing some small changes to extend the scope of the project.

Now, after accumulating data for 10 months, I can confirm the second part of my idea with actual data because I can clearly notice more shells errors using using a request like:

``
 select m, case when (h+6)%24 >=0 then (h+6)%24 else (h+6)%24-24 end as localtime,
         case when ok is null or ko is null then "N/A" else printf("%.2f", 1.0*sum(ok)/(sum(ok)+sum
(ko))) end as success
         from (
          select strftime ('%m', stop) as m,
                 strftime ('%H', stop) as h,
                 case when err>1 then 1 else 0 end as ko,
                 case when err=0 then 1 else 0 end as ok
          from commands where err != 130
         ) as sub1 group by 1 order by 1,2;
``

After a copy-paste of this code into sqlite3 ~/.bash_history-$HOST.db I get a result like:
```
0|0.88
1|0.87
2|0.82
3|0.86
4|0.85
5|0.88
6|0.87
7|0.92
8|0.73
9|0.90
10|0.85
11|0.88
12|0.76
13|0.92
14|0.36
15|0.77
16|0.85
17|0.77
18|0.88
19|0.81
20|0.83
21|0.92
22|0.81
23|0.82
```

This is hard to read, so I can do the same request per hour and per month, while also pivoting the lines to fit everything in the screen:
``
select m,
       sum(case when localtime is 0 then success else '' end) as "00",
       sum(case when localtime is 1 then success else '' end) as "1a",
       sum(case when localtime is 2 then success else '' end) as "2a",
       sum(case when localtime is 3 then success else '' end) as "3a",
       sum(case when localtime is 4 then success else '' end) as "4a",
       sum(case when localtime is 5 then success else '' end) as "5a",
       sum(case when localtime is 6 then success else '' end) as "6a",
       sum(case when localtime is 7 then success else '' end) as "7a",
       sum(case when localtime is 8 then success else '' end) as "8a",
       sum(case when localtime is 9 then success else '' end) as "9a",
       sum(case when localtime is 10 then success else '' end) as "10a",
       sum(case when localtime is 11 then success else '' end) as "11a",
       sum(case when localtime is 12 then success else '' end) as "12",
       sum(case when localtime is 13 then success else '' end) as "1p",
       sum(case when localtime is 14 then success else '' end) as "2p",
       sum(case when localtime is 15 then success else '' end) as "3p",
       sum(case when localtime is 16 then success else '' end) as "4p",
       sum(case when localtime is 17 then success else '' end) as "5p",
       sum(case when localtime is 18 then success else '' end) as "6p",
       sum(case when localtime is 19 then success else '' end) as "7p",
       sum(case when localtime is 20 then success else '' end) as "8p",
       sum(case when localtime is 21 then success else '' end) as "9p",
       sum(case when localtime is 22 then success else '' end) as "10p",
       sum(case when localtime is 23 then success else '' end) as "11p"
       from (
         select m, case when (h+6)%24 >=0 then (h+6)%24 else (h+6)%24-24 end as localtime,
         case when ok is null or ko is null then "N/A" else printf("%.2f", 1.0*sum(ok)/(sum(ok)+sum
(ko))) end as success
         from (
          select strftime ('%m', stop) as m,
                 strftime ('%H', stop) as h,
                 case when err>1 then 1 else 0 end as ko,
                 case when err=0 then 1 else 0 end as ok
          from commands where err != 130
         ) as sub1 group by 1,2 order by 1,2
 ) as sub2 group by 1;
 ``

The same dips in success rate are generally present around 8-9am and 2pm.  For some months, there is not a single entry for both 3pm and 4pm: this is not a mistake in the simple modulo arithmetics formula I use to convert from UTC to EST - it's just that I usually don't work at these times!

Of course, a proper confirmation would require statistical tests, but this is beyond the scope of this quick introduction.

Another simple example: I like to optimize the commands I type the most. What are the commands you type the most? You may be able to get that information from your .bash_history, but can you do this by month?

Here's how I can get the top 5 commands of the last 10 months:
``
-- cheap pivot using aggregating functions
select ym,
       sum(case when top=1 then cmd end) as first,
       sum(case when top=2 then cmd end) as second,
       sum(case when top=3 then cmd end) as third,
       sum(case when top=4 then cmd end) as fourth,
       sum(case when top=5 then cmd end) as fifth
       from (
  select ym, cmd,
       -- popularity rank of the cmd per month
       rank() over (partition by ym order by nbr desc, cmd) as top from (
    select ym,
           -- TODO: remove the optional .exe
           -- split path: keep the last element like c in /a/b/c
           replace(firstword, rtrim(firstword, replace(firstword, '/', '')),'') as cmd,
           count(*) as nbr from (
                select strftime ('%Y-%m', stop) as ym,
                   -- split words: keep the first thing before a space
                   substr(trim(what),1,instr(trim(what)||' ',' ')-1) as firstword
                   from commands where err=0
        -- TOOD: remove at least local variables before, and maybe backticks too
        -- like: CFLAGS="-g" make #comment here
        -- test with:
        --and what not like '%=%'
                ) as sub1
    group by 1, 2 order by 1,3 desc) as sub2
  ) as sub3 where top<6 group by ym;
``

![top 5 bash commands per month](https://raw.githubusercontent.com/csdvrx/bash-timestamping-sqlite/master/top5-per-month.png)

By my high use of oathtool in June, you can infer I started a different position which required configuring (and accessing) many 2FA services, with far less network related activity that right before as ssh, scp, ping, ifconfig were more important in February-May. However, ping has made a comeback in August due to my experiments with multiple ISPs to better work from home!

This is the kind of interesting insights you can get from personal analytics.  I have also included some of the analytics scripts I use, like with a gnuplot output.

To do more, you will have to write your own SQL queries and scripts.

I have started using JetBrains Datagrip to write my SQL code, which looks nice:
![sql code inside datagrip](https://raw.githubusercontent.com/csdvrx/bash-timestamping-sqlite/master/sql-code-inside-datagrip.png)

You can get the exact same results from your shell. If you prefer to do anything and everything in the shell, I recommend litecli to write queries in a frienly CLI environment.

## Installation

Add the content of bashrc.txt to your bashrc, the content of bash_profile.txt to your .bash_profile, and bash_logout to your .bash_logout.

If you don't have a .bash_profile but use a .bash_login instead, considering renaming the file to .bash_profile. [The order does matter](https://www.thegeekstuff.com/2008/10/execution-sequence-for-bash_profile-bashrc-bash_login-profile-and-bash_logout/) but it will make your setup more portable: out-of-the-box, a new msys2+mintty install ignored .bash_login.

If you don't like my bashrc defaults, add at least to your bashrc:

### For visual timestamping only:

```{text}
shopt -s checkwinsize

function __notbottom() {
  local pos
  IFS='[;' read -p $'\e[6n' -d R -a pos -rs || echo "failed with error: $? ; ${pos[*]}"
  CURLN=$((${pos[1]} +1 ))
  if [ "$CURLN" -ge "$LINES" ] ; then return -1 ; fi
}

PS1='`RETURN=$?; if [ $RETURN != 0 ]; then \
printf "\e[3m\e[2m\e[31m#!%03d\e[39m\e[49m" $RETURN ; else \
printf "\e[3m\e[2m\e[39m#    \e[49m" ; fi ; \
printf "\e[39m\e[12m[\D{%Y-%m-%d_%H:%M:%S}\e7\e[20C]\e[1m (\u@\h:\w)\e[22m\n\[\e[10m\e[3m\e[2m\]#\[\e[0m\] "`'

PROMPT_COMMAND='__notbottom && export PS0="\e8\r\e[0m\e[39m\e[2m\e[3m\e[5C\e[0m\e[20C\e[12m\e[2m\e[3m\D{,%Y-%m-%d_%H:%M:%S}\e[0m\e[2E" \
                       || export PS0="\e8\e[2A\r\e[0m\e[39m\e[2m\e[3m\e[5C\e[0m\e[20C\e[12m\e[2m\e[3m\D{,%Y-%m-%d_%H:%M:%S}\e[0m\e[2E" ; \
 printf "⏎%$((COLUMNS-1))s\\r\\033[K"'
````

### For SQLite timestamping only:


```{text}
# If you really don't want a fancy shell, just use:
#PS1='`RETURN=$?; sqliteaddstop "$RETURN" 2>/dev/null ; printf "# "`'

# Otherwise:
PS1='`RETURN=$?; sqliteaddstop "$RETURN" 2>/dev/null ; if [ $RETURN != 0 ]; then \
printf "\e[3m\e[2m\e[31m#!%03d\e[39m\e[49m" $RETURN ; else \
printf "\e[3m\e[2m\e[39m#    \e[49m" ; fi ; \
printf "\e[39m\e[12m[\D{%Y-%m-%d_%H:%M:%S}\e7\e[20C]\e[1m (\u@\h:\w)\e[22m\n\[\e[10m\e[3m\e[2m\]#\[\e[0m\] "`'

## SQLite search
# map ^R to the custom search using sqlite and fzy on current path
bind -x '"\C-r": sqlitehistorysearchpath'
# map ^T to the custom search using sqlite and fzy on everything
bind -x '"\C-t": sqlitehistorysearch'

# SQLite searching and logging
function sqlitehistorysearch { # Ctrl-T
  # First, check if we are within 20 lines off the bottom that will be used by
  # fzy to display completion entries, and cause it to scroll the display
  __notbottom 20 || export overwritecursorposition=y
  # With no argument, blind search, just based on date
  [[ -z $READLINE_LINE ]] && INITSEARCH="" || INITSEARCH="--query=$READLINE_LINE"
  selected=`sqlite3 ~/.bash_history-$HOST.db "select distinct what from commands where err=0 order by stop desc limit 10000;" | fzy -l 20 $INITSEARCH`
  # With argument, change bash prompt and jump to the entry end
  [[ -n "$selected" ]] && export READLINE_LINE="$selected" && READLINE_POINT=${#READLINE_LINE}
  # And if fzy caused a scroll, do a SCP up there to overwrite the RCP at the bottom
  [[ -n "$overwritecursorposition" ]] && echo "\e[1A\e7" && unset overwritecursorposition     }

function sqlitehistorysearchpath { # Ctrl-R
  # First, check if we are within 20 lines off the bottom that will be used by
  # fzy to display completion entries, and cause it to scroll the display
  __notbottom 20 || export overwritecursorposition=y
  # With no argument, blind search, just based on path and date
  [[ -z $READLINE_LINE ]] && INITSEARCH="" || INITSEARCH="--query=$READLINE_LINE"
  selected=`sqlite3 ~/.bash_history-$HOST.db "select distinct what from commands where err=0 and path is \"$PWD\" order by stop desc limit 10000;" | fzy -l 20 $INITSEARCH`
  # With argument, change bash prompt and jump to the entry end
  [[ -n "$selected" ]] && export READLINE_LINE="$selected" && READLINE_POINT=${#READLINE_LINE}
  # And if fzy caused a scroll, do a SCP up there to overwrite the RCP at the bottom
  [[ -n "$overwritecursorposition" ]] && echo "\e[1A\e7" && unset overwritecursorposition     
}

# SQLite logging
function sqliteaddstart {
  [[ $BASH_COMMAND =~ "^logout$" ]] && return
  local numandwhat="$(history 1)"
  numandwhat="${numandwhat#"${numandwhat%%[![:space:]]*}"}"
  local num="${numandwhat%%' '*}"
  [[ $SEEN -eq $num ]] && return
  numandwhat="${numandwhat#*' '}"
  what="${numandwhat#"${numandwhat%%[![:space:]]*}"}"
  [[ -z $what ]] && return
  sqlite3 "$SQLITEFILE" "
   INSERT OR IGNORE INTO commands (ssid, seq, what, path) VALUES (
    '${SID//\'/''}', '${num//\'/''}', '${what//\'/''}', '${PWD//\'/''}'
    );"
  export SEEN=$num
}

trap sqliteaddstart DEBUG

function sqliteaddstop {
 local ERR=$1
 [[ -z $ERR ]] && ERR=0
 local numandwhat="$(history 1)"
 numandwhat="${numandwhat#"${numandwhat%%[![:space:]]*}"}"
 local num="${numandwhat%%' '*}"
 [[ $num <1 ]] && return
 sqlite3 "$SQLITEFILE" "
   UPDATE commands 
    SET err='${ERR//\'/''}', stop=current_timestamp
    WHERE seq =${num//\'/''} AND ssid =${SID//\'/''}
   ;"
 }
```

inputrc.txt is included only as a suggestion for nice key mapping defaults, such as navigating on the prompt using control-arrows like on Windows.

If you don't like my inutrc defaults either, I suggest using at least the following in your .inputrc:

```{text}
# Send & Receive 8 bit chars
set meta-flag on
set convert-meta off
set input-meta on
set output-meta on

# protect against copy-paste
set enable-bracketed-paste on

# Do not wrap command line
set horizontal-scroll-mode on
```

## Design overview and history

This project started after realising that very often we forget when a very long file processing job was started.

Timestamps can be use to estimate even roughly how long it took, but are never shown by default. However, having even a rougth idea of how long some commands took in the past can be helpful, like during deployments or when writing documentation. And when doing system administration on a remote server with a shared tmux session, it is also extremely helpful to know what was done, and when.

While searching about that, I found a post about a [simple clock on the bash prompt to known when the command was started](https://redandblack.io/blog/2020/bash-prompt-with-updating-time/) and another [about measuring commands execution time](https://jichu4n.com/posts/debug-trap-and-prompt_command-in-bash/)

This proved an timestamped bash prompt was possible, but I really didn't like how any of that was done, so I reimplemented everything from scratch in a simpler way, that also does not depend on "shopt -s histverify" and other options which I really dislike.

Now everything works fine, and session logs have become much more useful! If you are not familiar with session logs, start GNU screen, then type Ctrl-A H to start logging to a file, and again when you want to stop. It is good practice to record your sessions when working with sensible systems.

If you prefer an automatic recording of all your sessions, simply uncomment the script line in the proposed bashrc.txt.

If you want an even fancier prompt that goes way beyond what oh-my-zsh and the likes can do, [check out ble.sh: it is the ideal companion of this project](https://github.com/akinomyoga/ble.sh).

If you want to not just copy-paste things but understand how it all works under the hood, the [logging of bash command history to sqlite](sqlite-logging-bash-history.md) and the use of ANSI attributes for [visually timestamping bash commands](bash-commands-timestamping.md) are explained separately in great details. You should read them.

### Major design limitation: not tamper-proof

A database is not the ideal tool if you want tamper-proof record-keeping: there's nothing preventing the records from being edited.

I would not call that an issue, but more of a concern, or a design limitation that should not impact 99% of the users.

But if you want to fork and improve this, some food for your thoughts!

 On remote hosts, to avoid giving intruder pointers to interesting things, you may want to set HISTFILESIZE=0 and only depend on the sqlite database for your history, ideally using some kind of password protection or decryption similar to how local password are protected before logging

 If you don't protect the sqlite database, you rely on obscurity: you assumes in order of probability that the attack will be fully automated, or that an intruder will be a script-kiddie with a standard toolset, or if you have an actual human on the other side, that this person will not be familiar with sqlite, a risky bet at best. It may be better to either 1) copy and remove the database upon logout, or 2) encrypt additions to the database with a public key (so anyone can add but only you can read), ideally using some timestamp salting or a Merkle Tree (to avoid hackers hiding their tracks with replays)

Why do you need at least a cryptographic solution, and ideally a Merckle tree? If you use just a public key solution, tampered records will become evident. They could still be erased, even if that may stand out even more, as the commands table contain 'seq' that should be sequential, but at least the tampering can't be hidden.

Still, if you haven't taken additional protections, a malicious agent could simply reorder the sequential IDs after removing the "bad" entries to hide such malveolent commands: anything is possible with write access to the database!

Even worse: if there is no replay protection (like by salting with the timestamp and the sequential ID) the problematic entries could be overwritten! With no sequence dependance, nothing prevent the innucuous "ls" signed entry from overwriting the "problematic" entries showing data has been exfiltrated or that an attack has occured.

Another concern is what to do with the files if the sqlite db files are not kept locally but copied and deleted: they should be timestamped with at least the creation date (.bash_history-$hostname-$date.db), and eventually some random salt, in case multiple bash are started at the very same second.

On the remote server, the file should go to a specific directory, watched by a daemon, with the files moved to another directory, so a local intruder couldn't overwrite the database by uploading an empty file or using similar Denial-Of-Service inspired approaches. You should also be careful to make one directory per user, with quotas, for similar reasons.

These are the most obvious limitations I see in my approach, and there may be others, so think carefully before trusting this solution too much.

Or just get in touch: I can do all the hard thinking for you and design the best solution for your specific needs!

After all, I'm a consultant :)
