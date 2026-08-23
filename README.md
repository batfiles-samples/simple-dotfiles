# simple-dotfiles

A small, working [batfiles](https://github.com/abatkin/batfiles) repository you
can clone and install, built to be read top to bottom. It is shaped like a
dotfiles repository someone would actually keep — zsh, git, neovim, tmux, and a
script on the `PATH` — and every feature it uses is one batfiles implements
today.

Batfiles is early. Right now `sync` installs symlinks, and that is the whole of
it, so **every action here is a `symlink` action**. As action types arrive this
repository grows to use them; what is in it is always what works.

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
├── shell/
│   ├── zshrc                     -->  ~/.zshrc
│   ├── zshenv                    -->  ~/.zshenv
│   └── aliases.zsh               -->  ~/.config/zsh/aliases.zsh
├── git/
│   ├── gitconfig                 -->  ~/.gitconfig
│   └── gitignore                 -->  ~/.config/git/ignore
├── editor/
│   └── nvim/                     -->  ~/.config/nvim        (the directory)
│       ├── init.lua
│       └── lua/options.lua
├── tmux/
│   └── tmux.conf                 -->  ~/.config/tmux/tmux.conf
└── bin/
    └── scratch                   -->  ~/.local/bin/scratch
```

Only `batfiles.toml` has intrinsic meaning. Every other name in the tree becomes
meaningful when an action references it — which is why `README.md` and
`notes/` are installed nowhere, and why `editor/nvim/lua/options.lua` is
installed without ever being named.

Read [`batfiles.toml`](batfiles.toml) next. It is commented action by action,
and it is the actual thing that runs.

## Try it without touching your home directory

`--home-dir` moves the destination of every `~` in the manifest, so you can
install the whole repository into a scratch directory and look at the result.
Nothing outside that directory is written.

```console
$ git clone https://github.com/batfiles-samples/simple-dotfiles.git
$ mkdir ~/batfiles-demo
$ batfiles --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
linked /home/you/batfiles-demo/.zshrc -> /home/you/simple-dotfiles/shell/zshrc
linked /home/you/batfiles-demo/.zshenv -> /home/you/simple-dotfiles/shell/zshenv
linked /home/you/batfiles-demo/.config/zsh/aliases.zsh -> /home/you/simple-dotfiles/shell/aliases.zsh
linked /home/you/batfiles-demo/.gitconfig -> /home/you/simple-dotfiles/git/gitconfig
linked /home/you/batfiles-demo/.config/git/ignore -> /home/you/simple-dotfiles/git/gitignore
linked /home/you/batfiles-demo/.config/nvim -> /home/you/simple-dotfiles/editor/nvim
linked /home/you/batfiles-demo/.config/tmux/tmux.conf -> /home/you/simple-dotfiles/tmux/tmux.conf
linked /home/you/batfiles-demo/.local/bin/scratch -> /home/you/simple-dotfiles/bin/scratch
```

`~/batfiles-demo/.config`, `.config/zsh`, and `.local/bin` did not exist a
moment ago. Missing parent directories are created; nothing else is.

When you are done, `rm -rf ~/batfiles-demo` removes every trace. The links are
the only thing that was created, and deleting a link never touches what it
pointed at.

To install it for real, clone to `~/dotfiles` — where batfiles looks by
default — and run `batfiles sync` with no arguments. Read
[`batfiles.toml`](batfiles.toml) first: it will want `~/.zshrc` and
`~/.gitconfig`, and it will refuse rather than replace whatever you already
have there.

## What each part of the repository demonstrates

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
unchanged /home/you/batfiles-demo/.zshrc
unchanged /home/you/batfiles-demo/.zshenv
unchanged /home/you/batfiles-demo/.config/zsh/aliases.zsh
...
```

### A source and its destination need not agree on a name

`git/gitignore` installs as `~/.config/git/ignore`, and `shell/aliases.zsh`
installs two directories away from where it lives. A dotfiles repository reads
better without a tree of dot-directories in it; the manifest is where the two
naming schemes get reconciled.

### Linking a directory

`editor/nvim` is a single action linking a whole directory, so
`~/.config/nvim` is one symlink and everything under it is reachable through
that one link. Add `editor/nvim/lua/keymaps.lua` tomorrow and there is nothing
to add to `batfiles.toml`.

Linking each file individually is the other option, and the two differ in what
happens to files neovim writes into that directory itself. Whichever you want,
say it in the manifest.

### `id` and `group`

Most actions here carry an `id` and a `group`. Neither selects anything yet —
the commands that take an address (`apply-action`, `disable-group`, and the
rest) are not built. They are validated now, and writing them now means the
manifest is ready when those commands arrive. One action, `shell/zshenv`, has
no `id` on purpose: it is optional, and an action without one still runs.

### A link batfiles made is repaired; anything else is refused

Point an action's `source` at a different file and re-run, and the link already
at that destination is repointed. A symlink holds no content of its own, and
what it pointed at is untouched, so there is nothing to lose:

```console
$ batfiles --home-dir ~/batfiles-demo --batfiles-dir ./simple-dotfiles sync
relinked /home/you/batfiles-demo/.zshrc -> /home/you/simple-dotfiles/shell/zshrc (was /home/you/simple-dotfiles/shell/zshenv)
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
link written `../simple-dotfiles/shell/zshrc` is one batfiles owns; one written
inside the repository that climbs back out of it is not.

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
action runs, since it is the one rule that needs a filesystem:

```console
error: no such file in the repository: /home/you/simple-dotfiles/shell/nope
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

Also: this repository installs on Unix only. Windows compiles and every command
runs there, but a `symlink` action is refused by name rather than performed, and
`symlink` is the only action there is.

## See also

- [batfiles](https://github.com/abatkin/batfiles) — the tool, and how to build
  it. No binaries are published yet.
- [Repository format](https://github.com/abatkin/batfiles/blob/main/docs/repoformat.md)
  — the manifest, and what an action may declare.
- [Command-line surface](https://github.com/abatkin/batfiles/blob/main/docs/cmdline.md)
  — commands, global options, exit statuses.
