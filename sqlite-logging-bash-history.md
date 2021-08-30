> Logging bash command history into a sqlite database

## Basics

All the commands you type go into a sqlite database file named ~/.bash_history-$HOST.db to used facilitate syncing of multiple hosts into one central location (by rsync, onedrive, dropbox, etc).

It works like a normal shell history - only with the power of the SQL synthax!

But wait, it's not just the commands: thanks to trap fuctions and custom bash hooks using the prompt variables, other important things go there too, such as the path, the start timestamp, the stop timestamp and the error code.

That makes this sqlite database much more versatile than a regular shell history.

## A complementing or a substitute?

The resulting database can either complement (by keeping HISTSIZE small) or fully replace (if HISTSIZE is set to 1) logging to .bash_history.

Personally, I recommend fully replacing bash own history, to force yourself into a full immersion. This way, you will learn faster, and get exposed to the multiple advantages without letting the friction due to small differences keep you into your old ways.

In any case, comments from the current session will be kept in memory, so the effect on your usual bash use should be minimal: both up and down arrows to navigate the history, type 'history' and you will see it. Actually, unless you bind Ctrl-R to the new fuzzy matcher, even the reverse-search of your command history will work ask usual - only on the commands seen until you close this prompt.

## Shortcuts

But you should not do that: instead, you should give a try to the full sqlite experience, with the shortcuts! With the shortcuts also bound:

 - Ctrl-R is remapped to use an internal bash function that shows your history in a way that will be more useful. It is not just filtered by success (so you are *NOT* offered an autocomplete that is known to return an error code), but also restricted to the same path from the sqlite history (so the suggestions become context sensitive!). Also, your reverse search is now dynamic and colored, by showing the best 20 of 10000 matches using fzy, an equivalent of fzf (so you can refine the list by adding more characters)

 - Ctrl-T is remapped to do a fuller search, meaning the last 1000 commands that were successful, regardless of which path you typed them. This can be sometimes useful, like if you create a new dictory and need to use there one of the long commands you usually type in some other directory.

This is not standard: Ctrl-T is normally use for transposing the last 2 characters typed. However, it's something almost no-one uses, while the letter T is being conveniently placed very close to the letter R: this makes Ctrl-T a prime candidate for substitution! Based on my experience, muscle memory adapts very quickly, and you will use Ctrl-T way more than before!

## Typical uses

Besides Ctrl-R and Ctrl-T, you can use sqlite for many things!

This allows to to quickly see for example what are the last 10 commands you are actually still running, into any of your the sessions still opened:

```{text}
sqlite3 ~/.bash_history-$HOST.db "
SELECT start, stop, seq, err, path, what FROM commands WHERE stop is NULL GROUP BY ssid,seq  ORDER BY ssid desc, seq desc limit 10;
"
```

Or the last 10 commands ran on other bash sessions than the current one:

```{text}
sqlite3 ~/.bash_history-$HOST.db "
SELECT start, ssid, what FROM commands WHERE ssid != $SID ORDER BY start,ssid,seq LIMIT 10;
"
```

This one returns the number of errors, per hour, for this month:

```{text}
select ymdh, 1.0*sum(ok)/(sum(ok)+sum(ko)) from (select strftime ('%H', stop) as ymdh, case when err>1 then 1 else 0 end as ko, case when err=0 then 1 else 0 end as ok from commands where err != 130 and stop BETWEEN datetime('now', 'start of month') AND datetime('now', 'localtime')) as sub group by 1 order by 1;
```

If you have a sixel capable terminal, you can use that for metrics, like for plotting your failure rate for this year of 2021:

```{text}
#!/bin/sh
# can plot '-' or '<cat'
# set timefmt \"%Y-%m-%d %H\";
#     .headers on
sqlite3 ~/.bash_history-$HOST.db \
 " select ymdh, 1.0*sum(ok)/(sum(ok)+sum(ko)) from (select strftime ('%H', stop) as ymdh, case when err>1 then 1 else 0 end as ko, case when err=0 then 1 else 0 end as ok from commands where stop > '2021-01-01' and err != 130) as sub group by 1 order by 1;" \
  | \
/c/Program\ Files/gnuplot/bin/gnuplot -e "set xdata time;
  set timefmt \"%H\";
  set datafile sep '|';
  set xtics rotate;
  plot '-' using 1:2 w l title 'shell success by hour'"
```

## Database design

This is not a wtmp replacement: so the remote IP, tty, or the session login are not logged, as this would be redundant with wtmp or its equivalent.

Instead, each bash command has a CID, and is attached to a session SID.

A foreign key constraint is added, to be able to delete all the commands from a given session by deleting the SID.

At login, the tables are created if they don't exist, and SID is exported if it's not set yet.  This is done to avoid discontinuities, otherwise, every time you open a new bash inside vim, the SID would be updated and exported!

To handle the sessions, the bash_profile contains *blocking* content, as we *NEED* $SID from the first command.

Since this part is blocking, it is kept to a minimum to avoid slowing down opening new shells on msys2 (where fork is dog slow): we don't even call 'date', as sqlite defaults can take care of populating the timestamps automatically.

If you like speed and are stuck on Windows/msys2, uname could be added in a separate non blocking (background) update to only run 1 blocking command - but this may be premature optimisation!

The general principle is only to keep the .bash_profile as small as reasonably possible, so that one extra command will not make much difference.

```{text}
## If not running interactively, do not do anything more
[[ $- != *i* ]] && return

## Otherwise, do SQLite logging, starting with this new session: get kernel info
UNAME=$(uname -a)
export UNAME

## Facilitate upload to a central repository
SQLITEFILE="$HOME/.bash_history-$HOST.db"
export SQLITEFILE

## Then only update the session ID if not already set to avoid discontinuities
[[ -z $SID ]] && export SID=$(sqlite3 "$SQLITEFILE" "

-- In case the files are deployed on a new host
CREATE TABLE IF NOT EXISTS sessions (   -- table of the sessions
sid INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
login TIMESTAMP NOT NULL DEFAULT current_timestamp,
                                        -- bash login timestamp
logout TIMESTAMP NULL,                  -- bash logout timestamp
user TEXT,                              -- username to merge different databases later
uname TEXT                              -- complete kernel version
);

CREATE TABLE IF NOT EXISTS commands (   -- table of the commands
cid INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT, -- pk autoincremented
ssid INTEGER NOT NULL,                  -- foreign key
seq INTEGER NOT NULL,                   -- cid order inside a ssid, to deduplicate pipes
start TIMESTAMP NOT NULL DEFAULT current_timestamp,
                                        -- execution begins when enter is pressed
stop TIMESTAMP NULL,                    -- ends when prompt shown again, empty if SIGINT
err INTEGER NULL,                       -- eventual returned code
what TEXT,                              -- command line as it was typed
path TEXT,                              -- context where the command line was typed
FOREIGN KEY (ssid) REFERENCES sessions(sid) ON DELETE CASCADE,
UNIQUE (ssid, seq)                      -- constraint to avoid dupes
);

-- for the new session
INSERT INTO sessions (user,uname) VALUES (
 '${USER//\'/''}', '${UNAME//\'/''}'
 );

-- for the conditional export SID
SELECT max(sid) from sessions;
")
```

## The session table

In the code above, 1) when starting bash interactively and 2) if there is no running session yet ($SID is empty), a new session is added by automatically increasing sid due to the insertion of (user,uname)- this also automatically sets the session start timestamp.

At the end, the select allows for this new SID to be exported: $SID will then be used as the master context for each command typed in this bash session.

This way, we do not have to use say the bash current process ID from $PPID, which could create conflicts by rollover or by chance.

Still, just like $PPID, $SID does allow grouping by bash session each command entered in each terminal, instead of having them mashed and merged alltogether in your bash_history.

On logout, the session closing time is added by the following from .bash_logout:

```{text}
sqlite3 "$SQLITEFILE" "UPDATE sessions SET logout = current_timestamp WHERE sid ='${SID//\'/''}';"
```
This avoids using a trap on exit, which can be used for better purposes, like in your scripts!

This also lets you detect "unusual" session closing, like due to a crash.

## The commands table

In .bashrc, a DEBUG trap is used to log each command and timestamp its starting point: the sqliteaddstart function is called whenever a new command starts.

There are several potential issues here, mostly for pipes: a|b|c will be trapped 3 times and could add 3 redundant lines.

sqliteaddstart uses "history 1" instead of $BASH_COMMAND to avoid splitting pipe chains into their individual commands, something you can see for yourself with:

```{text}
function testcmd { echo -n "# #$HISTCMD " ; echo -n "[$BASH_COMMAND], h="; history 1;}
trap testcmd DEBUG
```
Then if you run a pipe, you understand why that matters:

```{text}
echo 1|grep 1 |grep -v 2|grep -v 3
# #1 [echo 1], h=   27  echo 1|grep 1 |grep -v 2|grep -v 3
# #1 [grep 1], h=   27  echo 1|grep 1 |grep -v 2|grep -v 3
# #1 [grep -v 2], h=   27  echo 1|grep 1 |grep -v 2|grep -v 3
# #1 [grep -v 3], h=   27  echo 1|grep 1 |grep -v 2|grep -v 3
```

If that is not clear, [here is a good tutorial for the DEBUG trap](https://jichu4n.com/posts/debug-trap-and-prompt_command-in-bash/).

#### Handling pipes

There are no perfect solutions unless we ramp up complexity to an unacceptable level.

Let's start with what can't be used "out of the box" without thinking long and hard about the implications:

 - $HISTCMD (1 in the example above), because it is not robust to the function context. Also, as explained in bash manpage:
> "If HISTCMD is unset, it loses its special properties, even if it is subsequently reset"

 - the prefix from history (27 in the example above), since exporting new values of HISTSIZE changes the number: it may not be unique!!!

A potential solution would be to insert blindly into the sqlite database, either with INSERT OR IGNORE, or a UNIQUE constraint on (ssid,cid,what,start) if for some reason we care about dupes: if all that matches, it means we are trying to insert again something we did before.

However, it could be due to a pipe *OR* repeating commands. For repeated command ("cd ..", then "cd .." again 10 seconds later), the difference in time when the command is started should be enough to keep logging, even with HISTCMD unset. If not, the UNIQUE constraint could be expanded to (ssid,cid,what,path,start) - but that's starting to get long, and it may degrade performance.

Another possibility would be a TRIGGER on INSERT that uses some algorithm with the command currently entered, the previous commands, and maybe the time or the path - some arcane rules could decide that the same commands within a sliding 10 seconds window should not be inserted because they are dupes - then again, what about pipes where individual commands take a long time?

An alternative would be to insert nothing to get the cid for a blank record then populate the fields of this record. However, in case of a pipe for example, we would then have to do INSERT OR IGNORE (to keep the start timestamping of the first pipe command), or INSERT OR UPDATE (to update the start date to the default value now at the start of the final pipe command)

It seems needlessly complicated. Do we really want to have to build a SQLite database with conditional inserts, triggers, complex unique constraints, arcane deduplication heuristics and all that, just for handling pipes?

The goal here is to have a simple and robust solution, so to me, the answer is no. If you disagree, and argue it's necessary for security reasons, submit a patch, but I fear you are not understanding the scope of the problem: a dedicated user can simply remove the DEBUG trap to stop the logging.

Therefore, even if tweaking HISTSIZE seems close to sabotage, we have to trust the user knows what's being done. There might even be good reasons to do that. If the logging fails... so what? If you need to keep logs for auditing reasons, you should have 2 logs anyway: the full session logging output, as can be generated by GNU screen, and this database.

This is because the goal is to have each separate log under different custody, and check if they can each corroborate the other.

The prefix from history is thus stored as seq, with a simple constraint on (ssid, seq) being unique to at least avoid corrupting existing logs when the user tweaks HISTSIZE.

So the .bashrc contains:

```{text}
function sqliteaddstart {
  # don't slow down logout
  [[ $BASH_COMMAND =~ "^logout$" ]] && return

  # get the command from history then strip the command number
  local numandwhat="$(history 1)"
  # remove leading spaces
  numandwhat="${numandwhat#"${numandwhat%%[![:space:]]*}"}"
  # read the sequence number
  local num="${numandwhat%%' '*}"
  # to avoid having to unset the trap in PROMPT_COMMAND and to deal with pipes
  [[ $SEEN -eq $num ]] && return
  # remove the number and the leading spaces
  numandwhat="${numandwhat#*' '}"
  what="${numandwhat#"${numandwhat%%[![:space:]]*}"}"

  # avoid logging empty commands
  if [[ -z $what ]]; then return; fi

  # IGNORE to avoid failing the UNIQUE
  sqlite3 "$SQLITEFILE" "
   INSERT OR IGNORE INTO commands (ssid, seq, what, path) VALUES (
  '${SID//\'/''}', '${HISTCMD//\'/''}', '${what//\'/''}', '${PWD//\'/''}',
   );"
   
  # PROMPT_COMMAND contains several commands, only run once to optimize
  export SEEN=$num
}

# upon starting a command, log it
trap sqliteaddstart DEBUG
```

This is a good beginning, but only half the problem: this trap *CAN'T* give when the command ends, unless you make some heroic assumptions such as supposing typing commands is immediate and instant (!!)

To avoid wrong statistics when a prompt is left idle for a while, an update is made by a function that's called when the command ends, as proxied by when the next prompt is displayed: the function is thus started by PROMPT_COMMAND

It will populate both the stop time entry and the error code obtained:
```{text}
# once done, complete with the error code and stop timestamp through PROMPT_COMMAND
function sqliteaddstop {
 local ERR=$1
 [[ -z $ERR ]] && ERR=0
 # get the command from history then strip the command number
 local numandwhat="$(history 1)"
 # remove leading spaces
 numandwhat="${numandwhat#"${numandwhat%%[![:space:]]*}"}"
 # read the sequence number
 local num="${numandwhat%%' '*}"
 # don't do anything right after login
 [[ $num <1 ]] && return
 sqlite3 "$SQLITEFILE" "
   UPDATE commands
    SET err='${ERR//\'/''}', stop=current_timestamp
    WHERE seq =${num//\'/''} AND ssid =${SID//\'/''}
   ;"
 }
```

### "Fuzzy" queries

Navigating in your history is often done by calling reverse history search, with Ctrl-R on bash.

This can be fully replaced, simply by binding the key sequence to an internal function:

```{text}
# map ^R to the custom search using sqlite and fzy on current path
bind -x '"\C-r": sqlitehistorysearchpath'
# map ^T to the custom search using sqlite and fzy on everything
bind -x '"\C-t": sqlitehistorysearch'
```

With bind -x, not only an ordinary executable, but also a shell function can be called. The function can then read and write the variables READLINE_POINT (zero based position of cursor on line) and READLINE_LINE (text of current line): as explained on the bash mailing list, if the command changes the value of READLINE_LINE or READLINE_POINT, [those new values will be reflected in the editing state](https://lists.gnu.org/archive/html/help-bash/2020-03/msg00050.html)

You can notice that I also bind Ctrl-T:  Ctrl-T is remapped to do a fuller search, meaning the last 10000 commands that were successful, regardless of which path you typed them.

Ctrl-T is normally used for transposing the last 2 characters typed, something almost no-one uses, while being conveniently placed very close to Ctrl-R. This makes it a prime candidate for substitution, as muscle memory adapts very quickly!

Ctrl-T works essentially like what you are used to, while Ctrl-R is remapped to use a slightly more advanced internal bash function that use the current working directory (pwd) as part of the query: this is based on the observation that the commands you type are often directory dependant. 

```{text}
# For Ctrl-R
function sqlitehistorysearchpath {
  # First, check if we are within 20 lines off the bottom that will be used by
  # fzy to display completion entries, and cause it to scroll the display
  __notbottom 20 || export overwritecursorposition=y
  # With no argument, blind search, just based on date
  [[ -z $READLINE_LINE ]] && INITSEARCH="" || INITSEARCH="--query=$READLINE_LINE"
  # On the database, look for unique (distinct: don't duplicate) successful
  # commands (err=0) entered in the current path, with the most recent first
  selected=`sqlite3 ~/.bash_history-$HOST.db "select distinct what from
  commands where err=0 and path is \"$PWD\" order by stop desc limit 1000;" |
  fzy -l 20 $INITSEARCH`
  # With argument, change bash prompt and jump to the entry end
  [[ -n "$selected" ]] && export READLINE_LINE="$selected" && READLINE_POINT=${#READLINE_LINE}
  # And if fzy caused a scroll, do a SCP up there to overwrite the RCP at the bottom
  [[ -n "$overwritecursorposition" ]] && echo "\e[1A\e7" && unset overwritecursorposition
}

# For Ctrl-T
function sqlitehistorysearch {
  __notbottom 20 || export overwritecursorposition=y
  [[ -z $READLINE_LINE ]] && INITSEARCH="" || INITSEARCH="--query=$READLINE_LINE"
  selected=`sqlite3 ~/.bash_history-$HOST.db "select distinct what from commands where err=0 order by stop desc limit 10000;" | fzy -l 20 $INITSEARCH`
  [[ -n "$selected" ]] && export READLINE_LINE="$selected" && READLINE_POINT=${#READLINE_LINE}
  [[ -n "$overwritecursorposition" ]] && echo "\e[1A\e7" && unset overwritecursorposition
}
```

Let's have a closer look at what this last function does when Ctrl-R is pressed, as Ctrl-T is essentially the same thing without condition 2c:

0. If we are within 20 lines from the bottom of the screen, take note by exporting a variable
1. If READLINE_LINE is not empty, initially filter the fzy output with what you have just started entering before pressing Ctrl-R (so that you can use backspace to rub it out and get more matches)
2. Then using sqlite,
        - 2a. select the different (distinct) commands you previously typed (so no repeat ls)
        - 2b. that were successfull in the sense no error code was returned (err=0)
        - 2c. from within the same directory (path is...)
        - 2d. ordered by decreasing date of completion (order by stop desc) so you get to see the most recent first
        - 2e. for a maximum of 1000 results (limit 1000) to avoid too much clutter
        - 2f. using the bash history database specific for this host (.bash-history-$HOST.db)
3. Display the top 20 matches to you using fzy (an equivalent of fzf) so you can refine the list by typing more characters
4. Initially filters the fzy output with what you have started entering before (which uses the variable first defined, INITSEARCH)a
5. When something is selected, pass it to bash readline for display and put the cursor at the end (READLINE_POINT)
6. And if all this cause the screen to scroll, go a line up from the current line and save the cursor position there (for the multiline prompt)

About this final point, the reason is not immediately obvious unless you are used to issues with multiline prompts, so here is a quick explanation: when starting fzy, the bash prompt looks like `[ timestart         ]` on one line with the `# ` right below: since 20 lines may not be free downscreen, starting fzy will cause a scroll of the screen to fit its 20 lines of matches.

Once your selection is done, the prompt would be identical but on top of your screen, with the fzy selection filled in: so far, so good, but pressing enter would the cause the PS0 exported by PROMPT_COMMAND to restore of the cursor position at the initial position of the SCP, right after the timestart string on the bottom of your screen, while this line has moved to the top of your screen due to fzy.

This would cause the RCP to fill timestop to happen below the restored prompt! To avoid that, we override the SCP position by another SCP done just a line above the current `# ` so the RCP will be able to complete  `[ timestart         ]` to  `[ timestart,timestop ]` the way it should be. 

By the way, notice how we did not talk about a specific session, but we are working with the results from all session grouped together.

This is mostly useful for Ctrl-T which doesn't restrict the results to the current directory: this way, if you have multiple tabs or ssh into that host and type commands at the same time, press Ctrl-T and you will see them all!

## Design pitfalls

Adding commands was simplified as much as possible, following [saurik example which makes a creative use of debug and exit traps](https://news.ycombinator.com/item?id=10695305): this allows to leave PROMPT_COMMAND mostly alone.

The trap design used by saurik is not perfect: he recognizes pipes only work because the same entry can be overwritten, but this saves a lot of complexity, which I believe is more important than completion.

However, we can't do without PROMPT_COMMAND to get the stop timestamp. A function call seems like a small price to pay, especally given the complex but beautiful prompts already displayed!

Most other solutions I've found seem needlessly complicated or even dangerous:

 - you [should not have to draw a sequence diagram to understand how logging works](https://github.com/barabo/advanced-shell-history)

 - you [should not need special initialization commands or Go](https://github.com/andmarios/bashistdb)

 - you [should not need to emulate zsh preexec](https://pastebin.com/zJkPW79C) by [importing other scripts manually](https://github.com/rcaloras/bash-preexec), or even worse [by automatic execution of unknown code through wget](https://github.com/thenewwazoo/bash-history-sqlite/blob/master/bash-history-sqlite.sh)

 - you [should not use so many pipes as to slow down to a crawl on msys2](https://www.outcoldman.com/en/archive/2017/07/19/dbhist/)

The design I used may seem overkill in some parts, and too relaxed in other, but given the various horrors seen above, don't you want to play it safe and keep it simple?

If you find this approach too complicated, the next best is [bash command timer](https://github.com/jichu4n/bash-command-timer), but it feels dated: in 2015, bash-preexec was necessary. Not anymore!

### Concerns

I do not call these issues, but concerns, as they should not impact 99% of the users.

But if you want to fork and improve this, some food for your thoughts:

 - On remote hosts, to avoid giving intruder pointers to interesting things, you may want to set HISTFILESIZE=0 and only depend on the sqlite database for your history, 

 - Of course, this assumes an intruder will be familiar with sqlite, a risky bet at best. It may be better to either 1) copy and remove the database upon logout, or 2) encrypt additions to the database with a public key (so anyone can add but only you can read), ideally using some timestamp salting or a Merkle Tree (to avoid hackers hiding their tracks with replays)

 - If the file is not kept locally but copied and deleted, it should be timestamped with at least the creation date (.bash_history-$hostname-$date.db), and eventually some random salt, in case multiple bash are started at the very same second.

 - On the remote server, the file should go to a specific directory, watched by a daemon, with the files moved to another directory, so a local intruder couldn't overwrite the database by uploading an empty file or using similar Denial-Of-Service inspired approaches.

