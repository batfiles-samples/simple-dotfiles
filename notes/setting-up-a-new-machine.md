# Setting up a new machine

Nothing in `batfiles.toml` names this file, so `batfiles sync` will not install
it anywhere. That is deliberate: a dotfiles repository accumulates notes,
scripts, and half-finished ideas, and only the entries the manifest declares
have any meaning to batfiles. Everything else is just a file in a Git
repository.

The steps that are not yet automated:

1. Install the package manager.
2. Install zsh, tmux, neovim, git.
3. Clone this repository to `~/dotfiles`.
4. Run `batfiles sync`.
