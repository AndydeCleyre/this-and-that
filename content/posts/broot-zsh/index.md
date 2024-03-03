+++
title = 'Using broot as an fzf-like path completer and interactive folder jumper in Zsh'
date = 2024-01-15T14:17:11-05:00
tags = ['broot', 'zsh']
showComments = true
draft = false
+++

[Broot](https://dystroy.org/broot/) is a great file manager TUI
which can be adapted for very specific workflows.
But it takes some configuration up front to get things going,
even for the basics.

I'll go over some of my own configurations for using it from Zsh:

- Minimum Suggested Setup:
  - [wrap `broot` in a Zsh function](#wrap-broot-in-a-zsh-function)
  - [launch it with a non-disruptive shortcut](#launch-the-wrapper-with-a-non-disruptive-shortcut)
- Broot Basics:
  - [basic broot configuration](#basic-broot-configuration)
  - [basic broot usage](#basic-broot-usage)
- Seamless Zsh Hookups:
  - [drill down and fuzzy filter to set your working folder](#drill-down)
  - [complete a file argument](#insert-or-complete-paths)

My dotfiles containing the snippets in this post
are on GitHub at
[dotfiles-broot](https://github.com/AndydeCleyre/dotfiles-broot/)
and [dotfiles-zsh/broot.zsh](https://github.com/AndydeCleyre/dotfiles-zsh/blob/main/broot.zsh).

If you already use broot, skip to [Drill down](#drill-down).
To follow along, go ahead and [install broot](https://dystroy.org/broot/install/) first.

# Demo videos

## Drill down to change folder

{{< video src="drill_down" >}}

## Complete a partially typed argument with a path filter

{{< video src="complete_partial_with_path_filter" >}}

## Complete with a file-content filter

{{< video src="complete_with_content_filter" >}}

# Wrap broot in a Zsh function

`broot` is compiled (from Rust).
Since it's not a shell function,
it can't simply change your shell state --
in particular, change the current working directory.
That's best done from within the shell.
[Broot's solution](https://dystroy.org/broot/install-br/) is:

- launch `broot` via a small shell function
- `broot` writes a temporary file with content like `cd deep/down/folder`
- the shell function then `eval`s the content of the tempfile

## The simple way

If you launch `broot` in Zsh,
it'll ask to add a line to your `.zshrc`,
sourcing a file containing the shell function,
which you can also print out with:

```console
$ broot --print-shell-function zsh
```

```zsh
function br {
    local cmd cmd_file code
    cmd_file=$(mktemp)
    if broot --outcmd "$cmd_file" "$@"; then
        cmd=$(<"$cmd_file")
        command rm -f "$cmd_file"
        eval "$cmd"
    else
        code=$?
        command rm -f "$cmd_file"
        return "$code"
    fi
}
```

So you can just launch `broot`,
let it add that function,
and move on to the
[next section](#launch-the-wrapper-with-a-non-disruptive-shortcut).
Quit with `ctrl-q`.

## A Zsh (or control) enthusiast's way

But I don't like tools editing my `.zshrc`,
and I think Zsh syntax has an irresistible beauty,
so I defined this similar function instead:

```zsh
# -- Run broot, eval cmdfile if successful --
br () {  # [<broot-opt>...]
  emulate -L zsh

  local cmdfile=$(mktemp)
  trap "rm ${(q-)cmdfile}" EXIT INT QUIT
  if { broot --outcmd "$cmdfile" $@ } {
    if [[ -r $cmdfile ]]  eval "$(<$cmdfile)"
  } else {
    return
  }
}
```

To stop broot from asking to install its own `br` function, run:

```console
$ broot --set-install-state installed
```

### Room for improvement

While writing this post I remembered that `eval` is overkill for the task,
given that the `cmdfile` only ever contains a `cd` command.
I'll add [a bonus section at the end of this post](#bonus-rewrite-br-to-avoid-eval),
rewriting `br` to avoid `eval`.

# Launch the wrapper with a non-disruptive shortcut

There are other ways to do this,
but my approach is:

- push whatever's currently on the line out of the way
- put `br` on the line and run it
- when `br` exits, restore whatever was initially on the line

This is accomplished with a Zsh Line Editor
([ZLE](https://linux.die.net/man/1/zshzle))
function.
ZLE functions can do some extra things,
like manipulate your shell's current input line (`BUFFER`),
and call other ZLE functions.

I name this one `.zle_broot`,
and register it with `zle -N`:

```zsh
# -- Clear line, run broot, restore line --
# Key: ctrl+b
# Depends: br
.zle_broot () {
  zle .push-input
  BUFFER="br"
  zle .accept-line
}
zle -N       .zle_broot
bindkey '^b' .zle_broot  # ctrl+b
```

# Basic broot configuration

The first time you launch it,
some [default configuration files](https://github.com/Canop/broot/tree/main/resources/default-conf)
are created for you,
in [Hjson](https://hjson.github.io/) format.
You can learn about the possibilities by reading the comments therein,
and [the config docs](https://dystroy.org/broot/conf_file).

So you've now got `config.hjson`,
which imports a list of ["verb"](https://dystroy.org/broot/conf_verbs/)
definitions from `verbs.hjson`.

These defaults are OK,
so feel free to jump to [Basic broot usage](#basic-broot-usage).
Stick around for some configuration tips.

## Flags

You can set [`default_flags`](https://dystroy.org/broot/conf_file/#default-flags)
in `config.hjson`
to control which files and properties appear when you launch `br`.
I've got:

```yaml
# -- Do show: git-ignored files, git info, file sizes --
default_flags: --git-ignored --show-git-info --sizes
```

## Verbs

Have a look at `verbs.hjson`,
where you can [define](https://dystroy.org/broot/conf_verbs/#verb-definition-attributes)
actions you might want to trigger,
from either broot's ["internal"](https://dystroy.org/broot/conf_verbs/#internals)
actions (e.g. `back`, `line_down`, `toggle_preview`, `toggle_second_tree`),
or "external" commands,
each with access to the
[verb arguments](https://dystroy.org/broot/conf_verbs/#verb-arguments)
as parameters.
A verb can alternatively be a "cmd" type,
wherein you can provide a sequence of broot actions
as if you typed them interactively.

Here are some handy examples that I use:

```yml
verbs:
[
  {
    key: alt-f
    internal: ":toggle_files"
  }
  {
    key: alt-s
    internal: ":toggle_sizes"
  }
  {
    invocation: mkcd {new_dir}
    cmd: ":mkdir {new_dir};:focus {new_dir}"
  }
  {
    invocation: rmdir
    external: rmdir {directory}
    leave_broot: false
  }
  {
    key: alt-up
    internal: ":up_tree"
  }
  {
    key: alt-left
    internal: ":back"
  }
  {
    key: alt-down
    invocation: go
    internal: ":focus"
  }
  {
    invocation: go {path}
    internal: ":focus {path}"
  }
  {
    shortcut: ~
    internal: ":focus ~"
  }
  {
    invocation: clip
    internal: ":copy_path"
    leave_broot: false
  }
  {
    invocation: dolphin
    external: dolphin {directory}
    leave_broot: false
  }
]
```

# Basic broot usage

Typing within broot puts text into a buffer at the bottom of the screen,
which is used to do two different things:

- filter files and folders
- trigger a verb, making parameters from the currently selected file available to it

If there is a filter,
it has to come first.

## Filters

As you type almost anything but space, slash, or colon,
files and folders are fuzzy filtered by pathname.

But you can change this by prefixing your filter with
`/`, `c/`, or `cr/`:

prefix  |  filter by  |  filter syntax
---  | --- | ---
None  |  path  |  fuzzy match
`f/`  |  file name  |  fuzzy match
`/`  |  file name  |  regular expression
`c/`  |  file *content*  |  literal
`cr/`  |  file *content*  |  regular expression

Suffix any regular expression with `/i` to make it case insensitive.

When the filter is empty,
enter `?` to see help for all this,
and browse the built in verbs and descriptions.

## Verbs

A space or colon marks the beginning of a verb.
I like to use a space interactively,
and a colon in saved verb definitions, for readability.

So a buffer might look like:

```
projname cd
```

or

```
oldfle mv new-file-name
```

Yes, `oldfle` (not `oldfile`), because it's a fuzzy filter, not a filename.

## Keys

Some useful default keyboard shortcuts to know:

- `alt`+`i`: toggle display of git-ignored files
- `alt`+`h`: toggle display of `.hidden` files
- `enter`: if folder is focused: navigate into it within broot; otherwise open the focused file externally
- `alt`+`enter`: exit broot and `cd` into the focused folder

# Drill down

The setup here is that you're at your prompt,
maybe with a command already typed in full or part,
and you want to quickly change folder,
especially to a descendant of the current folder,
then have your prompt restored.

We can already do this with the
[wrapper we wrote above](#launch-the-wrapper-with-a-non-disruptive-shortcut):

- hit `ctrl`+`b`
- filter and navigate
- hit `alt`+`enter` (or type out "` cd`" in the broot buffer)

But we can streamline it for this case,
using a new Zsh function which
launches `broot` with an alternate verb configuration.

This alternate launcher will feature these differences:

- only folders shown, no files
- `enter` on a folder quits broot and changes folder
- `enter` on a file quits broot and changes to file's parent
- no temporary cmdfile, since we know we want to cd; use stdout

## Drill down verb definitions

We create a new config file
with some overriding verb definitions
to associate with the `enter` key.

Since we plan on using stdout to send the path to Zsh,
these verbs use broot's `print_path` internal.

We also redefine the verbs for `alt`+`enter` and the ` cd` invocation
to do the same thing.

`select-folder.hjson`:

```yml
verbs:
[
  {
    key: enter
    internal: ":print_path"
    apply_to: directory
  }
  {
    key: enter
    cmd: ":parent;:print_path"
    apply_to: file
  }
  {
    key: alt-enter
    invocation: cd
    internal: ":print_path"
    apply_to: directory
  }
  {
    key: alt-enter
    invocation: cd
    cmd: ":parent;:print_path"
    apply_to: file
  }
]
```

Overriding verb definitions via additional broot configs works well,
but overriding `default_flags` is best done in the CLI invocation
(in our drill down launcher function).

## Drill down launcher

We create a function `.zle_cd-broot` in `.zshrc`,
the core of which will be:

```zsh
broot \
  --color yes \
  --git-ignored \
  --show-git-info \
  --only-folders \
  --conf ".../select-folder.hjson;.../conf.hjson"
```

I bind it to `alt`+`down`,
to match
[other navigation bindings I have](https://github.com/AndydeCleyre/dotfiles-zsh/blob/main/clear_and_foldernav.zsh#L38)
for Zsh (`alt`+`up`/`left`/`right`).

Some funny bits need explaining,
but let's see the whole thing:

```zsh
# -- Folder Navigation: Down --
# Key: alt+down
# Depends: ~/.config/broot/select-folder.hjson
# Optional: .zle_redraw-prompt
.zle_cd-broot () {
  echoti rmkx
  cd "$(
    <$TTY broot --color yes --git-ignored --show-git-info --only-folders \
    --conf "${XDG_CONFIG_HOME:-${HOME}/.config}/broot/select-folder.hjson;${XDG_CONFIG_HOME:-${HOME}/.config}/broot/conf.hjson"
  )"
  if (( $+functions[.zle_redraw-prompt] ))  zle .zle_redraw-prompt
  if (( $+functions[_zsh_highlight]     ))  _zsh_highlight
}
zle -N            .zle_cd-broot
bindkey '^[[1;3B' .zle_cd-broot  # alt+down
```

### `.zle_redraw-prompt`?

Your fancy Zsh prompt may rely on certain hooks being run
after a change of your current working directory,
and then need to get poked by the ZLE system to get redrawn.

I lifted and restyled
[a function to do just that](https://github.com/romkatv/zsh4humans/blob/v2/fn/-z4h-redraw-prompt)
from [Zsh for Humans](https://github.com/romkatv/zsh4humans) by Roman Perepelitsa,
of [Powerlevel10k](https://github.com/romkatv/powerlevel10k/) fame.

```zsh
# -- Refresh prompt, rerunning any hooks --
# Credit: Roman Perepelitsa
# Original: https://github.com/romkatv/zsh4humans/blob/v2/fn/-z4h-redraw-prompt
.zle_redraw-prompt () {
  for 1 ( chpwd $chpwd_functions precmd $precmd_functions ) {
    if (( $+functions[$1] ))  $1 &>/dev/null
  }
  zle .reset-prompt
  zle -R
}
zle -N .zle_redraw-prompt
```

Drop this in your `.zshrc` to ensure
the prompt reflects your newly-changed working directory.

To skip the rest of the `.zle_cd-broot` explanations,
jump to [complete a file argument](#insert-or-complete-paths).

### `echoti rmkx`?

Debian based systems usually have a bit like this in `/etc/zsh/zshrc`:

```zsh
# Make sure the terminal is in application mode, when zle is
# active. Only then are the values from $terminfo valid.
zle-line-init () { ... smkx ... }
zle-line-finish () { ... rmkx ... }
```

This changes the handling of keyboard input
whenever a ZLE function is run,
and returns it to normal afterward.

In "application" mode,
broot won't recognize your keys properly.

`echoti rmkx` does the same as `zle-line-finish` above:
it takes the shell back out of application mode.

### `<$TTY`?

As the [`zshzle` manual](https://linux.die.net/man/1/zshzle) says:

> The standard input of the function is redirected from `/dev/null` . . .

`<$TTY` redirects your TTY to stdin again,
the way it usually is.

Without doing this, you may find broot unresponsive to your keyboard input.

### `_zsh_highlight`?

This is only useful if you use
[fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting),
and forces f-s-h to rescan and recolor the input line, now that it's changed.

### `${XDG_CONFIG_HOME:-${HOME}/.config}`?

Using `:-` described in the [`zshexpn` manual](https://linux.die.net/man/1/zshexpn),
this expression will evaluate to the value of `XDG_CONFIG_HOME` if that's set and non-empty.
Otherwise, it will be `~/.config`.

# Insert or complete paths

OK, this is the big one.

Like last time,
we'll use a set of alternate verb definitions,
and a new launcher function.

But this launcher has a lot more work to do.

## File selection verb definitions

These verbs will make it easier to print out one or more file paths.

All we're going to do here is remap `tab` and `enter`.

If we have not added anything to the selection list,
which would appear as a right-hand pane,
hitting `enter` will now print the path of the focused file
and quit.

We use the internal `panel_right_no_open` so that
if there *is* a right-hand pane (housing selected files),
`enter` will *first* switch pane-focus to that one,
*then* apply the `print_path` action to each selected item.

Whereas `tab` usually triggers broot's internal completion helper,
we'll now have it add the currently focused file to the list ("selected" or "marked"),
then move focus down to the next path.
This way, you can keep tapping `tab` to select consecutive file paths.

`select.hjson`:

```yml
verbs:
[
  {
    key: tab
    cmd: ":toggle_stage;:next_match"
  }
  {
    invocation: pright
    internal: panel_right_no_open
  }
  {
    key: enter
    cmd: ":pright;:print_path"
  }
]
```

We can't use the `panel_right_no_open` internal directly in a `cmd` entry,
as by default it has no "invocation" defined.

## File selection launcher

I call it `.zle_insert-path-broot`, and bind it to `ctrl`+`/`.

### Simple file selection launcher

```zsh
.zle_insert-path-broot () {
  echoti rmkx
  local locations=(${(f)"$(
    <$TTY broot --color yes --conf "${HOME}/.config/broot/select.hjson;${HOME}/.config/broot/conf.hjson"
  )"})
  locations=(${(q-)locations})
  LBUFFER+="$locations "
}
zle -N       .zle_insert-path-broot
bindkey '^_' .zle_insert-path-broot  # ctrl+/
```

The `(f)` splits broot's output into lines,
which we store as an array.
The `(q-)` adds shell quoting to any array item which needs it,
before we flatten the array into a space-separated string we cram into
`LBUFFER`, which corresponds to all text left of the cursor on our input line.

This is good enough to be useful,
but it can be much better.

### Complicated file selection launcher

We'll improve on this by handling any
already partially typed path, using it to:

- guess a starting filter
- guess a starting folder
- guess whether we should see hidden files

At its core,
the launcher will run:

```zsh
broot \
  --color yes \
  --cmd "$broot_search" \
  $show_hidden \
  --no-sizes \
  --conf ".../select.hjson;.../conf.hjson" \
  $start_dir
```

It's big, so let's see the whole thing first,
then take it piece by piece.

One known limitation is that it expects no newlines within your filenames.

```zsh
# -- Complete current word as path using broot --
# Key: ctrl+/
# Depends: ~/.config/broot/select.hjson
.zle_insert-path-broot () {
  setopt localoptions extendedglob

  # If the current word is partially typed, grab it (partial),
  # pop it off the line, and create a search string for it (broot_search)
  local partial broot_search
  if [[ ${LBUFFER[-1]/ } ]] {

    # Store it (partial)
    partial=(${(z)LBUFFER})
    partial=${partial[-1]}

    # Remove it from the line
    LBUFFER=${LBUFFER[1,-$#partial-1]}

    # Expand home folder
    partial=${partial/#(\$HOME\/|\~\/)/~\/}

    # Store a sanitized version for broot search (broot_search)
    broot_search=${partial/#~\/}                 # strip leading '~/'
    broot_search=${broot_search//( |:|..\/|\;)}  # strip ' ', ':', '../', ';'
    broot_search=${broot_search//.\/}            # strip './'
    broot_search=${broot_search//\//\\/}         # replace '/' with '\/'

  }

  # If resulting search string starts with '.' or ends with '/.',
  # search hidden files (can be slow)
  local show_hidden=--no-hidden
  if [[ $broot_search[1] == . || $broot_search[-2,-1] == /. ]]  show_hidden=--hidden

  # If partial has leading '/', '../', or ~/,
  # set broot start folder (start_dir)
  local start_dir
  if [[ $partial == ~/* ]] {
    start_dir=~
  } else {
    local updirs=${(M)partial##(../|/)##}
    start_dir=${updirs:-.}
  }

  # Get a location (or not) from broot,
  # using select.hjson config
  echoti rmkx
  local locations=(${(f)"$(
    <$TTY broot --color yes --cmd "$broot_search" $show_hidden --no-sizes \
    --conf "${XDG_CONFIG_HOME:-${HOME}/.config}/broot/select.hjson;${XDG_CONFIG_HOME:-${HOME}/.config}/broot/conf.hjson" \
    $start_dir
  )"})

  # Add location to line or restore original partial
  if [[ $locations ]] {

    # If GNU coreutils realpath is present,
    # convert some paths from absolute to relative
    if (( $+commands[realpath] )) {
      if [[ $(realpath --help 2>&1) == *--relative-base* ]] {
        local locations_abs=($locations) loc
        locations=()
        for loc ( $locations_abs )  locations+=("$(realpath --relative-base ~ --relative-to . $loc)")
      }
    }

    # Ensure minimal and sufficient quoting
    locations=(${(q-)locations})

    LBUFFER+="$locations "
  } else {
    LBUFFER+=$partial
  }

  # If using fast-syntax-highlighting, retrigger it
  if (( $+functions[_zsh_highlight] ))  _zsh_highlight

}
zle -N       .zle_insert-path-broot
bindkey '^_' .zle_insert-path-broot  # ctrl+/
```

That's it; if you don't want or need any further explanation,
I've got nothing more for you except the
[`eval`-avoiding `br` implementation bonus project](#bonus-rewrite-br-to-avoid-eval).

### Are we mid-word?

First, we determine if we're mid-word on the input line.
If not, we just need to insert whatever path `broot` spits out.
But if so, we:

  - define variable `partial` -- the unfinished path
  - define variable `broot_search` -- a broot-friendly filter
  - remove the unfinished path from `BUFFER`, so we can insert a replacement

```zsh
  # If the current word is partially typed, grab it (partial),
  # pop it off the line, and create a search string for it (broot_search)
  local partial broot_search
  if [[ ${LBUFFER[-1]/ } ]] {
```

This amounts to: "if there is a non-space character immediately to the left of the cursor,"
or: "if we are mid-word."

`LBUFFER` contains all input text left of the cursor.
We get its last character.
If it's a space, we replace it with an empty string,
which turns this whole condition `false`-y.

#### Set `partial`

We're mid-word,
so we split the `LBUFFER` content
into an array of words,
then store just the last word in `partial`.

```zsh
    # Store it (partial)
    partial=(${(z)LBUFFER})
    partial=${partial[-1]}
```

#### Erase partial on the input line

```zsh
    # Remove it from the line
    LBUFFER=${LBUFFER[1,-$#partial-1]}
```

"`$#partial`" gives us the length of `partial`,
so we know how much to trim off the end of `LBUFFER`.

#### Normalize leading user home paths

If the partially typed word starts with "`$HOME/`" or "`~/`",
we replace that bit with the full path of your home folder.

```zsh
    # Expand home folder
    partial=${partial/#(\$HOME\/|\~\/)/~\/}
```

#### Set `broot_search`

```zsh
    # Store a sanitized version for broot search (broot_search)
    broot_search=${partial/#~\/}                 # strip leading '~/'
    broot_search=${broot_search//( |:|..\/|\;)}  # strip ' ', ':', '../', ';'
    broot_search=${broot_search//.\/}            # strip './'
    broot_search=${broot_search//\//\\/}         # replace '/' with '\/'

  }
```

To turn our `partial` path into a good broot filter string,
we:

- remove any leading home path
- remove any of these substrings `' ', ':', '../', ';', './'`
- escape/quote any remaining forward slashes

Those forward slashes will match folder boundaries,
so for example you can type:

```console
$ head Co/actr/wk/aoc23
```

Then use broot to complete your line as:

```console
$ head Code/factor-99/work/aoc-2023/day05/day05.factor
```

### Set `show_hidden`

The idea is to spare you from hidden files
unless you probably want them,
as they add both visual noise and performance penalties,
if there are many.

```zsh
  # If resulting search string starts with '.' or ends with '/.',
  # search hidden files (can be slow)
  local show_hidden=--no-hidden
  if [[ $broot_search[1] == . || $broot_search[-2,-1] == /. ]]  show_hidden=--hidden
```

### Set `start_dir`

We use `partial` to guess a good starting folder
for the `broot` search.

```zsh
  # If partial has leading '/', '../', or ~/,
  # set broot start folder (start_dir)
  local start_dir
  if [[ $partial == ~/* ]] {
    start_dir=~
  } else {
    local updirs=${(M)partial##(../|/)##}
    start_dir=${updirs:-.}
  }
```

If `partial` starts with our home, we use our home.
Otherwise, we check for a leading substring of one or more `../` or `/`,
and use that.
If not, we use the current folder.

### The core

The `(f)` is used to split the result on newlines,
and we store the lines as an array.

```zsh
  # Get a location (or not) from broot,
  # using select.hjson config
  echoti rmkx
  local locations=(${(f)"$(
    <$TTY broot --color yes --cmd "$broot_search" $show_hidden --no-sizes \
    --conf "${XDG_CONFIG_HOME:-${HOME}/.config}/broot/select.hjson;${XDG_CONFIG_HOME:-${HOME}/.config}/broot/conf.hjson" \
    $start_dir
  )"})
```

`--no-sizes` is a compromise for performance.

### Insertion

```zsh
  # Add location to line or restore original partial
  if [[ $locations ]] {
```

#### Final touches

We use `realpath` to turn some absolute paths to relative ones.
Some `realpath` implementations don't support this feature.

```zsh
    # If GNU coreutils realpath is present,
    # convert some paths from absolute to relative
    if (( $+commands[realpath] )) {
      if [[ $(realpath --help 2>&1) == *--relative-base* ]] {
        local locations_abs=($locations) loc
        locations=()
        for loc ( $locations_abs )  locations+=("$(realpath --relative-base ~ --relative-to . $loc)")
      }
    }
```

```zsh
    # Ensure minimal and sufficient quoting
    locations=(${(q-)locations})
```

The `q-` parameter expansion flag
only quotes the items that need quoting,
back on the input line.

```zsh
    LBUFFER+="$locations "
```

Adding to `LBUFFER` also moves our `CURSOR` location for us,
and any `RBUFFER` content is preserved.

```zsh
  } else {
    LBUFFER+=$partial
  }
```

If `locations` ended up empty,
we probably quit broot with `ctrl`+`q`,
indicating a canceled operation.
So we just put the almost-original `partial`
back on the line as it was.

But if it had a leading `~/` or `$HOME/`,
that's now expanded.
It's a bug, but I can live with it.

```zsh
  # If using fast-syntax-highlighting, retrigger it
  if (( $+functions[_zsh_highlight] ))  _zsh_highlight
```

As we saw earlier, this last bit is only helpful if you use
[fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting),
and forces f-s-h to rescan and recolor the input line, now that it's changed.

# Bonus: Rewrite br to avoid eval

{{< alert >}}
This method only works with `broot>1.33.1`.
{{< /alert >}}

Instead of `--outcmd`,
we'll use a new option, `--verb-output`,
and a new "internal," `write_output`.

First, we add a verb definition to our `verbs.hjson` like:

```yml
{
  key: alt-enter
  invocation: cd
  cmd: ":write_output {directory};:quit"
}
```

This is will write the focused folder path to a file (specified by `--verb-output`, coming up soon).

Now we can update our function to:

```zsh
zmodload zsh/mapfile

# -- Run broot, cd into pathfile if successful --
# Depends: zmapfile
br () {  # [<broot-opt>...]
  emulate -L zsh

  local pathfile=$(mktemp)
  trap "rm ${(q-)pathfile}" EXIT INT QUIT
  if { broot --verb-output "$pathfile" $@ } {
    if [[ -r $pathfile ]] {
      local folder=${mapfile[$pathfile]}
      if [[ $folder ]]  cd $folder
    }
  } else {
    return
  }
}
```

It hasn't changed much,
but here we use `--verb-output` to tell broot what tempfile to write to,
then `cd` into whatever path that file contains, if the file is written at all (and non-empty).

Additionally, a micro-optimization has been used to avoid a subshell;
we *could* use

```zsh
local folder=$(<$pathfile)
```

But instead we load the [zsh/mapfile module](https://linux.die.net/man/1/zshmodules),
and access the file contents as the value in a magic dictionary,
which maps file names to contents.

# That's all, folks!

That's all I have to say about this for now,
and if you've gotten this far, I hope some part of this was useful to you.

Did I make a mistake? Have I been unclear? Yell at me in the comments!
