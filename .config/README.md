# Dotfiles

Managed with the **bare Git repository** method: the repo internals live in
`~/.dotfiles`, while the work tree is `$HOME` itself. There is no `.git/` folder
in your home directory, so other tools never mistake `$HOME` for a repo.

All commands use a `dot` wrapper instead of plain `git`:

```sh
dot() {
    git --git-dir="$HOME/.dotfiles/" --work-tree="$HOME" "$@"
}
```

This function is defined in the tracked shell config, so it is available
automatically on any machine that has already been bootstrapped. On a brand-new
machine it does not exist yet — see [Setting up a new machine](#setting-up-a-new-machine).

---

## Daily use

### Adding / updating a file

Track a new file, or stage changes to one already tracked, then commit and push:

```sh
dot add ~/.config/nvim/init.lua
dot commit -m "nvim: add LSP config"
dot push
```

Files are opted **in** one at a time. Nothing in `$HOME` is tracked until you
`dot add` it, which is what keeps the rest of your home directory invisible.

Check status and history exactly as with normal git:

```sh
dot status          # only shows tracked files (see note below)
dot diff
dot log --oneline
```

> **Note:** `dot status` relies on `status.showUntrackedFiles no` being set
> (see step 4 of bootstrap). Without it, `status` lists your entire `$HOME`.

### Pulling updates on an existing machine

If the machine is already set up, syncing the latest changes is ordinary git:

```sh
dot pull
```

If you have local changes that would conflict, stash or commit them first:

```sh
dot stash
dot pull
dot stash pop
```

---

## Setting up a new machine

A plain `git pull` does **not** work here — there is no local repo to pull into
yet, and a bare repo must be cloned with `--bare`. Run these steps once per new
machine.

### 1. Clone the repo as bare

```sh
git clone --bare <your-repo-url> $HOME/.dotfiles
```

### 2. Define the `dot` alias temporarily

The real `dot` function lives inside a dotfile you have not checked out yet, so
define a throwaway alias for the rest of this bootstrap:

```sh
alias dot='git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
```

### 3. Check out into `$HOME` (with backup of collisions)

A fresh machine often already has default versions of some tracked files
(`~/.bashrc`, etc.). Git refuses to overwrite them and aborts. Back up the
collisions first, then retry:

```sh
dot checkout 2>&1 | egrep "\s+\." | awk '{print $1}' | \
  xargs -I{} sh -c 'mkdir -p "$(dirname .dotfiles-backup/{})"; mv {} .dotfiles-backup/{}'

dot checkout
```

Anything that collided is now preserved under `~/.dotfiles-backup/`.

### 4. Set the untracked-files flag

This is repo-local config and is **not** tracked, so set it on every new clone:

```sh
dot config status.showUntrackedFiles no
```

### 5. Reload the shell

```sh
exec $SHELL    # or just open a new terminal
```

The temporary alias from step 2 is now replaced by the proper `dot()` function
from your dotfiles. You are live.

---

## After bootstrapping: per-machine setup

Some things intentionally **do not** live in the repo and must be set up per
machine:

- **Machine-specific git config.** The shared `~/.config/git/config` does not
  hardcode a signing key. Create `~/.config/git/local.config` on this machine
  with its own `user.signingkey` (the shared config includes it optionally):

  ```ini
  [user]
      signingkey = <THIS_MACHINE_KEY_ID>
  ```

- **GPG key import.** The dotfiles carry the *config* that says "sign commits,"
  not the key itself. Import your secret key before signing will work:

  ```sh
  gpg --import secret-key.asc
  ```

- **Verify `status.showUntrackedFiles no`** is set (step 4). If `dot status`
  floods you with your whole home directory, this is missing.

- **OS-specific files.** Shell configs guarded by `case "$(uname -s)"` should
  no-op on the wrong platform. Confirm no BSD/GNU tool-flag errors appear on
  first shell load.

---

## Quick reference

| Action                         | Command                              |
| ------------------------------ | ------------------------------------ |
| Track / stage a file           | `dot add <path>`                     |
| Commit                         | `dot commit -m "..."`                |
| Push                           | `dot push`                           |
| Pull (existing machine)        | `dot pull`                           |
| Status (tracked only)          | `dot status`                         |
| New machine                    | clone `--bare` → checkout → set flag |
