# Devdash

`devdash` is a small tmux launcher for a development workspace. It creates one tmux session with a fixed six-pane layout so the same tools are always in the same place when you start working. The current version opens Git on the left, files below that, Codex in the top-middle pane, Claude in the bottom-middle pane, a shell on the top-right, and a dev server on the bottom-right.

The script is designed to be fast and predictable rather than interactive. You run it once, it creates the session, and if that session already exists it attaches to the existing one instead of rebuilding it. That behavior matters, because if you change the script and test it against an already-running tmux session, you will still see the old panes until the session is killed and recreated.

## What It Needs

The only hard requirement is `tmux`. If `tmux` is missing, the script exits immediately with `tmux is not installed.` Everything else is optional, but the behavior changes depending on what is available on your machine.

For the best experience, install `codex`, `claude`, `lazygit`, and `yazi`. The script will use them automatically if they exist. If they do not exist, it falls back to simpler commands. That means the script still works on a more minimal setup, but some panes will open into plain shell output instead of the tool you expected.

The script also assumes it is being run in a Unix-like shell environment with `bash`, `sed`, and standard terminal behavior. Since it uses `#!/usr/bin/env bash`, you should run it in an environment where `bash` is available on the `PATH`.

## What Each Pane Does

The left column is the Git area. The top-left pane tries to run `lazygit`. If `lazygit` is not installed, it falls back to `git status`. The bottom-left pane is the file browser. It prefers `yazi`, then falls back to `nvim .`, and then finally to `ls -la` if neither of those tools is available.

The middle column is the AI column. The top-middle pane now opens `codex` if the `codex` command exists. If it does not, that pane shows `Codex not found.` The bottom-middle pane opens `claude` if available, otherwise it shows `Claude Code not found.`

The right column is for general development work. The top-right pane opens a login shell with `bash -l`. The bottom-right pane is the dev server pane. If the target project contains a `package.json`, the script tries to pick the correct dev command based on the lockfile. It uses `bun run dev` for Bun projects, `pnpm dev` for pnpm projects, `yarn dev` for Yarn projects, and `npm run dev` otherwise. If there is no `package.json`, it opens with a message saying the dev pane is ready.

When one of these commands exits, the pane does not disappear. The wrapper command prints `[exited] shell ready` and drops you into an interactive login shell so the pane remains usable.

## How The Session Works

By default, the tmux session name is `dev` and the window name is `main`. Those defaults come from the `SESSION` and `WINDOW` environment variables.

When you run `devdash`, the first thing it does is check whether a tmux session with the chosen session name already exists. If it does, the script does not create a new layout. It simply attaches to the existing session and exits. That is why layout changes seem like they did nothing if you forget to kill the old session first.

If you want to test a new script version, either kill the old session before starting again or use a different session name. The two most useful commands are:

```bash
tmux kill-session -t dev
./devdash
```

and:

```bash
SESSION=dev2 ./devdash
```

The first recreates the normal session from scratch. The second keeps your existing `dev` session intact and starts a second session named `dev2`.

## Running It From This Folder

If you are standing in the same directory as the script, the safest way to run the exact file you are looking at is:

```bash
./devdash
```

That leading `./` matters. It tells the shell to run the script in the current directory. If you leave it off and type only `devdash`, your shell will search the `PATH` and may run some other installed copy instead. That is exactly the problem you ran into earlier: the local script had been edited, but the global command still pointed at an older copy in `/usr/local/bin/devdash`.

## Installing It So `devdash` Works Everywhere

If you want to run `devdash` globally from any directory, install the script into a directory that is already on your `PATH`. On most Linux systems and WSL setups, `/usr/local/bin` is the standard place for a personal script like this.

From the directory that contains the updated `devdash`, run:

```bash
sudo install -m 755 "./devdash" /usr/local/bin/devdash
```

This copies the file into `/usr/local/bin`, sets it executable, and replaces the older global copy if one already exists.

After that, verify what the shell will use:

```bash
type -a devdash
```

If the global install worked, you should see `/usr/local/bin/devdash` in the result. If there are multiple entries, read them carefully. The shell will use the first matching command it finds on the `PATH`.

You can also inspect the installed file directly:

```bash
sed -n '1,120p' /usr/local/bin/devdash
```

If you want to confirm that the top-middle pane is really the Codex pane, look for `CODEX_RUN` and the line that feeds `"$CODEX_RUN"` into the top-middle split.

## Making Sure It Is Executable

If you copied the file with `install -m 755`, you do not need to run `chmod` again. The install command already gave it executable permissions.

If you are using the local file directly and want to make it executable by hand, run:

```bash
chmod +x ./devdash
```

You can verify permissions with:

```bash
ls -l ./devdash
```

The important part is that the mode string contains `x`, such as `-rwxr-xr-x`.

## Running It Against A Project

The script accepts an optional project path as its first argument. If you pass a folder, `devdash` uses that folder as the tmux working directory and as the place where it looks for `package.json` and lockfiles.

Example:

```bash
devdash /path/to/project
```

If you do not pass an argument, it uses the current working directory.

That means the normal pattern is simple: `cd` into your project and run `devdash`, or give it the project directory directly if you are launching it from somewhere else.

## Changing The Default Behavior

The script is configurable through environment variables. You do not need to edit the file if you only want to change the commands it runs for a single launch.

`SESSION` changes the tmux session name. `WINDOW` changes the tmux window name. `PROJECT` is handled through the first positional argument rather than an environment variable in normal use. `GIT_RUN`, `FILES_RUN`, `CODEX_RUN`, `CLAUDE_2_RUN`, `SIDE_RUN`, and `DEV_RUN` let you override the command used in each pane.

For example, if you want the dev pane to run some other command, you can do:

```bash
DEV_RUN="npm run start" devdash
```

If you want the shell pane to open `zsh` instead of a Bash login shell, you can do:

```bash
SIDE_RUN="zsh -l" devdash
```

If you want a second isolated workspace without touching your normal `dev` session, you can do:

```bash
SESSION=experiment devdash
```

This is often a better way to test changes than destroying your main session.

## Layout Notes

The layout is not hardcoded to exact terminal dimensions. The script reads the terminal size with `tput cols` and `tput lines`, then calculates pane sizes as percentages. The left column is roughly twenty-four percent of the width, the right column is roughly twenty-five percent, and the middle column gets the remaining space. The top panes also get percentage-based heights, with minimum sizes enforced so the layout does not collapse on small terminals.

The script disables the tmux status bar, enables mouse support, adds pane titles along the top border, uses gray borders for inactive panes, and a bright green border for the selected pane. If your tmux version supports heavy border lines, it enables those too.

## Common Problems

The most common problem is thinking the script edit did not work when the real issue is that tmux is reattaching to an older session. If you changed the file and still see the old layout, kill the session and start again. The command is:

```bash
tmux kill-session -t dev
```

Then launch the script again.

The second most common problem is running the wrong copy of the script. If `./devdash` behaves one way but `devdash` behaves another way, your global copy is outdated. Check that with:

```bash
type -a devdash
```

If needed, reinstall the updated file into `/usr/local/bin`.

Another common issue is a missing dependency. If the top-middle pane says `Codex not found.`, the `codex` command is not available on your `PATH` inside that shell environment. If the lower middle pane says `Claude Code not found.`, the same thing is true for `claude`. If the Git pane does not open `lazygit`, or the files pane does not open `yazi`, those commands are simply not installed or not visible to the shell.

If the bottom-right pane does not start your app, check whether you launched the script in the correct project folder and whether that folder contains `package.json`. Also check whether the lockfile matches the package manager you expect. The script does not inspect package scripts in detail; it assumes the usual `dev` script exists.

## Updating The Script Later

If you edit the local `devdash` file and want those edits to become the global command, reinstall it with the same `install` command you used originally:

```bash
sudo install -m 755 "./devdash" /usr/local/bin/devdash
```

Then kill the old tmux session and start a new one. Updating the file alone is not enough if you are still running an older global copy or reattaching to an old tmux session.

## Quick Start

If you only want the shortest correct workflow, it is this:

```bash
chmod +x ./devdash
sudo install -m 755 "./devdash" /usr/local/bin/devdash
tmux kill-session -t dev 2>/dev/null
devdash
```

Run that from the directory containing the script. After that, `devdash` should start the global updated version, and the top-middle pane should open Codex.
