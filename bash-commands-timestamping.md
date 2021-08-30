> Timestamping bash commands using the prompt and ANSI codes

## Concept

The initial idea was something looking like:

```{text}
an eventual error code [timestamp of when input began, then of when enter was pressed] (a cute prompt) the command typed
```

During experiments, this felt a bit too cluttered, so the final result is a multiline prompt which allows to also feature the username, the hostname and the current directory, all that without wasting any screen space for the commands:

```{text}
# !eventual error code [timestamp of when input began, and when enter was pressed ] (user@host):/directory
# command typed that can be very long but is more important than all this eye-candy
```

The two # instead of the usual $ is because I like to copy paste from my consoles, and I don't want to take the risk of having anything executed by accident!

To compensate for the risk of confusion coming from a multiline prompt, the part where commands are typed is separated using visual attributes.

## Graphical design

### Multiline

The prompt is multiline to give you as much room as possible to type your commands: the only wasted-space luxury is a pound sign (aka hash sign, aka octhrope) on the newline.

### Black-and-white

The use of colors is voluntarily kept to a minimum, since extremely colored prompts are often too distracting to me: I prefer to keep syntax coloring for when I'm typing actual bash code, like making sure all my parenthesis are matched, my quotes are closed, and my loops end in done. This is [easily done thanks to ble.sh](https://github.com/akinomyoga/ble.sh).

The only color "luxury" is that the error code (if any) is in red, to quickly catch your attention, along with an exclam point to make it easy to parse your saved sessions if using minicom or some other serial console for embedded systems.

But unless you use ble.sh or the likes, everything you type yourself will be in the default color, and likewise for the prompt.

However, it will be made to look different thanks to ANSI attributes: this will let you use various color themes on your terminal such as Solarized, without requiring any changes in the ANSI sequences code of your bashrc or any other complex logic that PS1-promp-pimping often requires.

### Typographic attributes

Thanks to ANSI attributes, font changes can be used instead of colors: thin, italic and bold can mimick different greytones.

Visually, the part above the command typed is both in a thin font and italic, to avoid focusing on that instead of the command being type.

The use of bold is mostly intended as a fallback: if your terminal supports neither thin nor italic, at least the user@host:/directory part will be bold so you know where you are!

### Terminal support

Ideally, your terminal should support thin, italic, bold and ANSI colors: you may have to export TERM to the right value, or edit your terminfo if your terminal support all, yet you can't see any of it.

Another nice thing to have is unicode output. To give an example of why, a small unicode trick is added to distinguish outputs that do not contain a newline, as can be done with echo -n something.

```{text}
# echo -n $TERM
xterm-256color⏎
# echo $TERM
xterm-256color
```

Can you notice how there is a small difference? It's a very faint "carriage return" arrow at the end of the line, to tell us we may have forgotten something, but without cluttering the display.

A fancier version could use a stroke combining character over a bolder arrow like ↵, but combining characters on symbols doesn't always work, and this faint arrow is far less obstructive.

If you can't see anything special and the unicode output is garbled, consider using a modern terminal such as mintty or mlterm. This is because besides being pleasant to see, advanced console font typographics can help you visually WITHOUT distracting you from the console commands, unlike ANSI colors.

If your terminal is only missing unicode, simply use a space instead: keeping the function that detects missing newlines will at least keep your prompt nice and clutter free!

## Internal magic to update the prompt after execution

If you have seen the demo, you may have noticed something almost magical: the timestamp prompt that sits ABOVE the command you type is updated with the time the typing stops and the enter key is pressed AFTER the command is run! How is that possible without keeping copies of all the text and writing it again?

The magic on how this works is quite simple, but requires some explanations.

There are basically 2 parts: PS1 and PS0.

Everybody knows PS1 is for customizing the bash prompt. PS0 is somehow of a more obscure/secret code, not even shown in the man page: this is because PS0 variable is relatively recent, having been introduced to Bash in 2016.

Simply put, PS0 is expanded after the enter key is pressed but before the command is run, so it often used in fancy prompts to avoid the accumulation of colors.

Let's look at PS1 and PS0 line-by-line.

### PS1 : detect error codes, display timestamp, save cursor position, output eye candy

#### Step 1: detect errors

```{text}
PS1='`RETURN=$?; if [ $RETURN != 0 ]; then \
```

The PS1 start with a backtick to detect if the last command gave an error

```{text}
printf "\e[3m\e[2m\e[31m#!%03d\e[39m\e[49m" $RETURN ; else \
```

First, we toggle italic with \e[3m and faint with \e[2m : the changes are cumulative.

The \e means escape, [ starts the sequence, and m finishes it; if you want to learn more about these \e codes, check the [wikipedia page for ANSI escape codes](https://en.wikipedia.org/wiki/ANSI_escape_code)

In the case of an error, the color is switched to red and the error code shown: \e[31m switches the foreground (3) color to red.

The error code is then printed in decimal, padded with zeroes to fit in 3 characters with %03d, after which the foreground color is returned to normal with \e[39m.

The background color is also returned to default, with \e[49m: this part is not needed and therefore totally superfluous.

It is included for your convenience only, should you want to customize colors, for example if you prefer say black text on a red background.

If there was no error:

```{text}
printf "\e[3m\e[2m\e[39m#    \e[49m" ; fi ; \
```

The font attributes are similarly changed to italic and faint with \e[3m\e[2m, and \e[39m switches the foreground color to normal: this is more of a superfluous precaution, but I don't like bad surprises when I tweak colors.

Instead of an error code, now 4 spaces are printed as a placeholder: this allows the timestamps to align nicely:

As before, the background color is returned to normal with \e[49m - another precaution.

#### Step 2: output the timestamp

```{text}
printf "\e[39m\e[12m[\D{%Y-%m-%d_%H:%M:%S}\e7\e[20C]\e[1m (\u@\h:\w)\e[22m\n\[\e[10m\e[3m\e[2m\]#\[\e[0m\] "`'
```

We start by changing the foreground color to normal with \e[39m, as another precaution, before switching the font before the timestamp with \e[12m : on mintty, this [selects a nice thin](https://github.com/mintty/mintty/wiki/Tips#text-attributes-and-rendering) font.

Then, the timestamp of the beginning of input is displayed inside brackets, using the ISO format YmdHMS with pretty separator: \D followed by curly braces let bash know this is a date-time specification.

\e7 saves the position of the cursor, since once the command is executed we will want to complete the "begin" timestamp with an "end" timestamp.

\e[20C advances by 20 characters to create a placeholder for the end timestamp, after which ] closes the timestamp braket.

To know where we are, \e[1m switches the font to bold before outputting the username \u, the hostname with \h, and the curent path with \w all inside parenthesis to make it pretty.

\e[22m disables both bold and faint, then a newline is printed: this makes the prompt a multiline prompt, which allows you to have more room to type.

After the newline, \\[ signifies to bash the output is not printable until it sees a \\]: this is to avoid creating unnecessary line wraps

Ffor both pound signs to have matching a shape and color, we go back to the primary font with \e[10m, toggle italic with 3 and faint with 2, then say with \\] the output is now printable

We can now print the leading pound sign with the correct font attributes, after which we have another non printable sequence turning all attributes off with \e[0m, then finally a space

### PS0

Here, PS0 is necessary to complete the timestamp with the time at which you press on enter.

However, PS0 is very limited and can't be used as such as we need 2 different version depending on where on the screen the cursor is. Instead, PROMPT_COMMAND is used to set the PS0 to the right thing.

### PROMPT_COMMAND

PROMPT_COMMAND is evaluated before printing PS1.

It is the perfect mix with PS0: here, PROMPT_COMMAND is used to detect if the bottom of the screen is reached, in which case a slightly different PS0 will be used to adjust for that

Inside PROMPT_COMMAND, a notbottom function is used to detect the bottom of the screen: the precise position is read by querying the terminal.

This can be done using escape codes:

```{text}
# printf '\e[6n'
^[[3;1R^[[3;8R⏎
```

The coordinates are between the semicolon and R: they are read into an array, and one is added to the X coordinate.

This is only used for debugging, so the code could be shorter, but this is a neglictible overhead, and I find it much better to have a reusable function also giving me the $CURLN variable I can use wherever.

The code below is pretty self explainatory; the only candy is a sanity check in case the position read goes beyond the known screen geometry.

```{text}
# Detect the bottom of the screen without stty, read with -s to avoid displaying
function __notbottom() {
  local pos
  ## Detect the cursor position
  IFS='[;' read -p $'\e[6n' -d R -a pos -rs || echo "failed with error: $? ; ${pos[*]}"
  if [ -z $1 ] ; then
  ## add one to the 0-initiated x-pos (+ no offset), to give an error code at the bottom
    CURLN=$((${pos[1]} +1 ))
  else
  ## same but add the parameter as well, for the comparison that follows
    CURLN=$((${pos[1]} +1 +$1 ))
  fi
  if [ "$CURLN" -ge "$LINES" ] ; then return -1 ; fi
}
```

We can now start analyzing the PROMPT_COMMAND:

```{text}
PROMPT_COMMAND='__notbottom && export PS0="\e8\r\e[0m\e[39m\e[2m\e[3m\e[5C\e[0m\e[20C\e[12m\e[2m\e[3m\e[12m\D{,%Y-%m-%d_%H:%M:%S}\e[0m\e[2E" \
```

If the bottom of the screen has not been reached, \e8 restores the cursor position ; an alternative to \e8 would be \033[u but this RCP is not supported by mosh - so we stick with what will work on most setups.

The carriage return \r returns to the beginning of the line, where we apply the same sequence as before to get the font to the same state, as we can't know which font is being used - well, we might find a way to query the terminal but that would be too complicated, it's simpler to return to a default state!

First, \e[0m returns the font to normal, and \e[39m likewise switches the foreground to normal, after which we toggle faint and italic.

Then, we advance forward by 5 characters with \e\5C to leave either the error code or the placeholder as is - we then toggle off all font attributes with \e[0m, after which we advance by 20 characters with \e[20C to leave the beginning timestamp as such.

We can finally output the ending timestap: \e[12 switches to the alternative font (since we returned to a "known good" default state), \e[2m\e[3m toggle faint and italic, and we output the iso date after which we can return the font attribute to normal with \e[0m and move the cursor 2 lines below

```{text}
                       || export PS0="\e8\e[2A\r\e[0m\e[39m\e[2m\e[3m\e[5C\e[0m\e[20C\e[12m\e[2m\e[3m\D{,%Y-%m-%d_%H:%M:%S}\e[0m\e[2E" ; \
```

If we have reached the screen bottom, we do the same thing except right after restoring the cursor position with \e8, we move the cursor up twice with \e[2A

Some more advanced state logic could be used, but repeating everything and keeping all the ANSI on one line adds very little overhead while making the code far easier to understand.

The next part is tricker:

```{text}
printf "⏎%$((COLUMNS-1))s\\r\\033[K"'
```

The COLUMN decreased by 1 is [a simple trick found on a blog](https://www.vidarholen.net/contents/blog/?p=878) and discussed [more in detail there](https://news.ycombinator.com/item?id=23520240) : the output is padded with COLUMNS-1 spaces, followed by a carriage return \r to move to the first column, and \e[K to erase from the cursor to the end of the line.

### Issues

#### Full-screen

Returning from full-screen apps like vim can show "^[[?1049l⏎" at the bottom.

This is sequence to "Exit alternate screen and restore cursor", so this is not due to anything we do, but to an interaction between $TERM, terminfo and the ~/.vimrc for which a quick fix is:

```{text}
" Either do not clear the screen
set t_ti=
set t_te=

" Or do not damage scrollback
"set t_ti=^[[?1049h
"set t_te=^[[?1049l
```

#### Completions

Tab completions and interruptions (such as with SIGINT) are not handled, but I believe interruptions do not need a redundant timestamp: they show up as an error 130

It should be possible to trap completion attempts to clear the saved line with something like:

```{text}
PS0="`if [ $COMP_LINE ] ; printf "\e8\e[2K"; fi`"
```
However, I find the current functionality sufficient - so submit a patch if you want the feature included.

