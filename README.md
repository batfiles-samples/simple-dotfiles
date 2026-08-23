# simple-dotfiles

A small, working [batfiles](https://github.com/abatkin/batfiles) repository you
can clone and install, built to be read top to bottom. It is shaped like a
dotfiles repository someone would actually keep — zsh, git, neovim, tmux, and a
script on the `PATH` — and every feature it uses is one batfiles implements
today.

Batfiles is early. Right now `sync` makes symlinks, and that is the whole of it,
so **every action here is one of the two symlink actions**. As action types
arrive this repository grows to use them; what is in it is always what works.

## The idea

Your dotfiles stay ordinary files in an ordinary Git repository. A
`batfiles.toml` at the root declares where each one belongs in your home
directory, and `batfiles sync` makes the home directory match.

```
simple-dotfiles/                      installs as
├── batfiles.toml
├── README.md                          (nothing — never named by the manifest)
├── notes/
│   └── setting-up-a-new-machine.md    (nothing — never named by the manifest)
│
├── files/                        -->  ~/.<name>             ] one action,
│   ├── ackrc                     -->  ~/.ackrc              ] whatever is
│   ├── gitconfig                 -->  ~/.gitconfig          ] in here
│   ├── zshenv                    -->  ~/.zshenv             ]
│   └── zshrc                     -->  ~/.zshrc              ]
├── bin/                          -->  ~/.local/bin/<name>   ] one action,
│   ├── proj                      -->  ~/.local/bin/proj     ] undotted
│   └── scratch                   -->  ~/.local/bin/scratch  ]
│
├── git/gitignore                 -->  ~/.config/git/ignore      (renamed)
├── shell/aliases.zsh             -->  ~/.config/zsh/aliases.zsh (moved)
├── tmux/tmux.conf                -->  ~/.config/tmux/tmux.conf
└── editor/
    └── nvim/                     -->  ~/.config/nvim  (the directory itself)
        ├── init.lua
        └── lua/options.lua
```

Six actions, ten links. Only `batfiles.toml` has intrinsic meaning. Every other
name in the tree becomes meaningful when an action references it — which is why
`README.md` and `notes/` are installed nowhere, and why `files/ackrc` and
`editor/nvim/lua/options.lua` are installed without ever being named.

Read [`batfiles.toml`](batfiles.toml) next. It is commented action by action,
and it is the actual thing that runs.

## The two actions, and when to reach for each

This is the one decision the manifest asks you to make.

|                     | `symlink`                        | `symlink-dir`                             |
|---------------------|----------------------------------|-------------------------------------------|
| **Names**           | one source, one exact `dest`     | a `source-dir` and a `dest-dir`           |
| **Makes**           | one link                         | one link per direct child                 |
| **Use it when**     | the destination has a different name, or a place of its own | the destination is the file's own name in some directory |
| **Adding a file**   | needs a new entry                | needs nothing                             |

The rule of thumb: **if you would be typing the same name twice, a directory
action already covers it.** `files/zshrc → ~/.zshrc` is the same name with a
dot; `bin/scratch → ~/.local/bin/scratch` is the same name in another
directory. Both are `symlink-dir`'s job, and there are two entries in this
manifest for six files because of it.

What is left over is the interesting half. `git/gitignore` installs as
`~/.config/git/ignore` — a different name, in a different place — and
`shell/aliases.zsh` lands two directories from where it lives. Nothing about a
directory can express either, so each gets a `symlink` that says exactly what
it means.

### Two ways to say "directory", and they are not the same

Worth pausing on, because the names are close:

- **`symlink-dir` on `bin/`** makes *two* links, one per child. `~/.local/bin`
  stays a real directory that other things can write into.
- **`symlink` with `source = "editor/nvim"`** makes *one* link. `~/.config/nvim`
  *is* a symlink into this repository, and everything under it is reached
  through that one link.

Neither is more correct. Pick per directory: a real directory where other
programs also install things, one link where the whole tree is yours.

## Try it without touching your home directory

`--home-dir` moves the destination of every `~` in the manifest, so you can
install the whole repository into a scratch directory and look at the result.
Nothing outside that directory is written.

```console
$ git clone https://github.com/batfiles-samples/simple-dotfiles.git
$ mkdir ~/batfiles-demo
$ batfiles --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
linked /home/you/batfiles-demo/.ackrc -> /home/you/simple-dotfiles/files/ackrc
linked /home/you/batfiles-demo/.gitconfig -> /home/you/simple-dotfiles/files/gitconfig
linked /home/you/batfiles-demo/.zshenv -> /home/you/simple-dotfiles/files/zshenv
linked /home/you/batfiles-demo/.zshrc -> /home/you/simple-dotfiles/files/zshrc
linked /home/you/batfiles-demo/.local/bin/proj -> /home/you/simple-dotfiles/bin/proj
linked /home/you/batfiles-demo/.local/bin/scratch -> /home/you/simple-dotfiles/bin/scratch
linked /home/you/batfiles-demo/.config/git/ignore -> /home/you/simple-dotfiles/git/gitignore
linked /home/you/batfiles-demo/.config/zsh/aliases.zsh -> /home/you/simple-dotfiles/shell/aliases.zsh
linked /home/you/batfiles-demo/.config/nvim -> /home/you/simple-dotfiles/editor/nvim
linked /home/you/batfiles-demo/.config/tmux/tmux.conf -> /home/you/simple-dotfiles/tmux/tmux.conf
```

Ten links from six actions. The first four came from one `symlink-dir` entry
and the next two from another; the directory actions install their children in
sorted order, so the output is the same on every machine and diffs cleanly.

`~/batfiles-demo/.config`, `.config/zsh`, and `.local/bin` did not exist a
moment ago. Missing parent directories are created — and so is a missing
`dest-dir` — but nothing else is.

When you are done, `rm -rf ~/batfiles-demo` removes every trace. The links are
the only thing that was created, and deleting a link never touches what it
pointed at.

To install it for real, clone to `~/dotfiles` — where batfiles looks by
default — and run `batfiles sync` with no arguments. Read
[`batfiles.toml`](batfiles.toml) first: it will want `~/.zshrc` and
`~/.gitconfig`, and it will refuse rather than replace whatever you already
have there.

## What each part of the repository demonstrates

### Adding a dotfile without editing the manifest

This is what `symlink-dir` is for, and it is worth seeing once:

```console
$ echo "set completion-ignore-case on" > simple-dotfiles/files/inputrc
$ batfiles --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
linked /home/you/batfiles-demo/.inputrc -> /home/you/simple-dotfiles/files/inputrc
```

No entry was added. `batfiles.toml` was not opened. The `rcfiles` action names
the directory, not its contents, so a file that appears there is installed and
a file that leaves it stops being.

`files/ackrc` in this repository is the same thing already done — it is
installed, and nothing in the manifest mentions it.

### Running it twice changes nothing

```console
$ batfiles --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
$
```

Silence is the successful case. An action whose destination is already correct
says nothing, because a repository that is already installed is the ordinary
one, and forty lines of "unchanged" is how output stops being read. `-v` prints
them, along with the four roots it resolved:

```console
$ batfiles -v --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
repository: /home/you/simple-dotfiles
home:       /home/you/batfiles-demo
config:     /home/you/.config/batfiles
cache:      /home/you/.cache/batfiles
unchanged /home/you/batfiles-demo/.ackrc
unchanged /home/you/batfiles-demo/.gitconfig
unchanged /home/you/batfiles-demo/.zshenv
...
```

### A source and its destination need not agree on a name

`git/gitignore` installs as `~/.config/git/ignore`, and `shell/aliases.zsh`
installs two directories away from where it lives. A dotfiles repository reads
better without a tree of dot-directories in it; the manifest is where the two
naming schemes get reconciled. These are the entries a directory action cannot
replace, which is why they are still written out one by one.

### `dot-prefix`, and the file it will not install

`files/` holds `zshrc`, not `.zshrc`. The repository stays greppable and `ls`
shows you everything, and `dot-prefix = true` puts the dot on at install time.

The corner worth knowing: a file in there that is *already* dotted is refused
rather than installed as `..name`, which is a legal filename and never what
anyone meant. If you want a genuinely dotted name in a dot-prefixed directory,
that file wants its own `symlink` entry.

### `id` and `group`

Every action here carries an `id`, and most a `group`. Neither selects anything
yet — the commands that take an address (`apply-action`, `disable-group`, and
the rest) are not built. They are validated now, so writing them now means the
manifest is ready when those commands arrive.

For a `symlink-dir` the `id` names the whole action, never one of its children:
the ten links here are addressable as six things, not ten. That is deliberate —
a directory action is all-or-nothing, and an address that could name a file
which may or may not exist tomorrow would not be much of an address.

### A link batfiles made is repaired; anything else is refused

Point an action's `source` at a different file and re-run, and the link already
at that destination is repointed. A symlink holds no content of its own, and
what it pointed at is untouched, so there is nothing to lose:

```console
$ batfiles --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
relinked /home/you/batfiles-demo/.zshrc -> /home/you/simple-dotfiles/files/zshrc (was /home/you/simple-dotfiles/files/zshenv)
```

Anything batfiles did not create is somebody's data, and until there is a
backup policy to give it back with, it is refused by name:

```console
$ batfiles --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
error: /home/you/batfiles-demo/.gitconfig already exists and is a regular file; move it aside and run sync again
```

A directory, or a symlink pointing somewhere outside the repository, is refused
on the same terms and says which it found:

```console
error: /home/you/batfiles-demo/.zshenv already exists and is a symlink to /etc/hostname, which is outside the repository; move it aside and run sync again
```

Note that this is decided by where a link *points*, not by how it is spelled. A
link written `../simple-dotfiles/files/zshrc` is one batfiles owns; one written
inside the repository that climbs back out of it is not. And "where it points"
means what the operating system makes of it, read from the directory the link
is physically in — so if `~/bin` is itself a symlink somewhere else, a relative
link sitting in it is judged from *there*, not from `~/bin`.

The same goes for a whole directory a `symlink-dir` installs into: an existing
`~/.local/bin` is used as it is, whatever it holds, and files in it that
batfiles did not put there are left alone.

### The manifest is read strictly, and read whole

Every check below happens before the first link is written, so a manifest
batfiles will not honor stops the run rather than installing half of it.

A typo is an error, not a setting that looks accepted and does nothing:

```console
error: invalid TOML in /home/you/simple-dotfiles/batfiles.toml:
TOML parse error at line 1, column 1
  |
1 | [[actions]]
  | ^^^^^^^^^^^
unknown field `mode`, expected one of `id`, `group`, `source`, `dest`
```

Each action type's fields are its own, so borrowing one from the other is the
same kind of error — and the message doubles as the list of what the type does
take:

```console
unknown field `source`, expected one of `id`, `group`, `source-dir`, `dest-dir`, `dot-prefix`
```

So is a section belonging to a part of the format that is specified but not yet
built — `[vars]`, `[remotes]`, `[default-disabled]`. Better to be told they do
nothing than to believe they did something:

```console
error: invalid TOML in /home/you/simple-dotfiles/batfiles.toml:
TOML parse error at line 1, column 2
  |
1 | [vars]
  |  ^^^^
unknown field `vars`, expected `actions`
```

And so are the rules TOML itself cannot express — a `source` that climbs out of
the repository, or two actions claiming one `id`:

```console
error: invalid configuration in /home/you/simple-dotfiles/batfiles.toml: action 1: source `../elsewhere/zshrc` resolves outside the repository
error: invalid configuration in /home/you/simple-dotfiles/batfiles.toml: action 2 repeats the id `zshrc`, which action 1 already uses
```

A `source` naming a file the repository does not contain is caught when the
action runs, since it is the one rule that needs a filesystem — and so is a
`source-dir` that turns out not to be a directory, which has no children to
link:

```console
error: no such file in the repository: /home/you/simple-dotfiles/files/nope
error: not a directory: /home/you/simple-dotfiles/files/zshrc
```

### An option that is not live yet is refused, not ignored

```console
$ batfiles sync --dry-run
error: `--dry-run` is not implemented yet; it arrives at step 2.4
```

The whole command-line surface parses from the first release, so the shape of
the tool is visible before all of it works. Accepting `--dry-run` silently
would let you believe a dry run had happened. Same for commands:

```console
$ batfiles enable-group shell
error: `enable-group` is not implemented yet
```

Exit status `0` means the command did what was asked, `1` that it ran and failed
partway, and `2` that it did not run at all — a usage error, or something not
built yet. The distinction that matters: after a `1` the filesystem may have
been touched, after a `2` it definitely was not.

## What this repository does not show yet

Because it does not exist yet. Roughly in planned order:

| What arrives                                                        |
|---------------------------------------------------------------------|
| The `create-dir` and `copy` actions                                 |
| `--dry-run`                                                         |
| Groups you can enable and disable, `--skip`, `apply-action`         |
| Fetching files and archives, and cloning Git repositories           |
| Variables, and `when`/`unless` conditions                           |
| Git remotes, and splicing a remote's actions into your own manifest |
| `init` and `clone` for setting up a new machine                     |

Filtering which children a `symlink-dir` picks up — `include` and `exclude` —
is specified but not built, and writing either is an error rather than a filter
that silently does nothing. Until it exists, a file you do not want installed
does not go in the directory.

Also: this repository installs on Unix only. Windows compiles and every command
runs there, but both action types are refused by name rather than performed,
since both of them make symlinks.

## See also

- [batfiles](https://github.com/abatkin/batfiles) — the tool, and how to build
  it. No binaries are published yet.
- [Repository format](https://github.com/abatkin/batfiles/blob/main/docs/repoformat.md)
  — the manifest, and what an action may declare.
- [Command-line surface](https://github.com/abatkin/batfiles/blob/main/docs/cmdline.md)
  — commands, global options, exit statuses.
